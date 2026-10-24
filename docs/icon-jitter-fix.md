# 总览进出图标抖动：根因与修复

分支 `feature/restore-theme-in-overview`。症状：进入/退出 GNOME 总览时，dock
应用图标（含左侧/右侧 dock 与面板模式 extend-height）发生一次性位置跳动：
剧烈版为每个 tile 4px（12 个 tile 共 48px+），轻微版为 item 边距 4px。

## 根因：St 伪类的 idle 延迟

mutter 的 St 引擎在 `src/st/st-widget.c` 中为容器的**可见**首/末子节点分配
`:first-child` / `:last-child` 伪类，逻辑在 `st_widget_update_child_styles()`，
通过 `g_idle_add_once` 延迟执行。子节点加入（`clutter_actor_add_child`）时只
置 dirty 标志并排队 idle。

关键时序：**帧分配是 priority 10 的 timeout，idle 是 priority 20** —— 同一批
main loop 中帧回调永远先于 idle 执行。因此：

> 批量加入子节点后的**第一帧分配**必然缺首/末伪类，CSS 中依赖
> `:first-child`/`:last-child` 的尺寸规则首帧不生效，下一帧才自纠正。

dash-to-dock 的触发链（fork 的 restore-theme 特性）：每次**退出**总览
`_onOverviewHiding` → `updateCustomTheme(true)` → themeManager 发 `'updated'`
→ `dash.resetAppIcons()` → 重建全部 12 个 item 容器。重建后的第一帧即总览
过渡帧 → 用户看到跳动。登录时首次显示图标同样命中。

## 修复策略：消除尺寸对伪类的依赖

原则：**会被重建的 actor（item 容器、tile）的尺寸不能依赖
`:first-child`/`:last-child`**；把间距转移到**从不重建的稳定 actor**
（dock 容器、scrollview、show-apps item）上补偿，保证稳态像素级不变、
首帧=稳态。

`_stylesheet.scss`（`dock-container-style` mixin 内）：

1. **tile 轴 padding**：tile 永远是其 item 容器的唯一子节点，旧 mixin
   `set-internal-children-property($side, padding, $spacing)` 生成的首/末
   归零规则在稳态永远匹配 → 直接显式声明轴 padding 为 0
   （`padding-left/right: 0` 或 `padding-top/bottom: 0`），不再走伪类。
   边缘 padding 保留：`padding-#{$side}: $padding + if($is_extended,
   $dock_side_margin, $side_margin)`。
2. **item margin**：去掉首/末归零，固定 `margin: ($dock_icons_distance/2) 0`
   （竖排）/ `0 ($dock_icons_distance/2)`（横排）。
3. **容器 padding 补偿**：`#dashtodockDashContainer` 轴 padding 由
   `$padding` 减为 `$padding - $dock_spacing`（非 extended），吸收首/末
   item 各 2px。extended 模式容器 padding 为 0，改由 scrollview edge
   padding 吸收：`$edge_items_padding - $dock_spacing`（scrollview 从不
   重建，伪类稳定）。
4. **show-apps item margin**：show-apps item 从不重建，其伪类稳定 →
   归零其**靠近 dash box 一侧**的 margin（`#dashtodockDashContainer >
   .dash-item-container` / `#dashtodockBoxContainer > .dash-item-container`
   的 `:first-child`/`:last-child`），吸收最后一个 item 的 2px。
5. **extended show-apps tile**：保留 edge padding（Fitts：图标可点到 dock
   边缘），规则只匹配 `.show-apps` 子元素。

### 图标宽度锚定（配合项，`appIcons.js`）

`DockAbstractAppIcon` / `ShowAppsButton` 的 `vfunc_get_preferred_width`
返回 `iconWidth + 24`（left/right）/ `iconWidth + 4`（bottom/top），即
图标自身首选宽 + 该方向所有状态下的最大 CSS padding 总和。vfunc 不调
父类 → tile 公共宽度与 CSS padding 解耦，图标中心在各状态（normal/
extended/custom-theme）不移动。高度不锁定（由 CSS 决定）。

## 关键约束（踩坑记录）

- **croco 不支持 `:not()`**：croco 的 FUNCTION 伪类只解析
  `:function( IDENT )` 单标识符；`:not(:first-child)` 解析失败，可能导致
  整张样式表被丢弃。绝对不要用。
- **JS 同帧通知无效**：手动 `notify('first-child')` 只是重排同一个 idle
  回调（`update_child_styles_id != 0` 时直接跳过），改变不了
  "帧先于 idle" 的架构时序。一帧延迟对任何重建都不可避免 → 只能从 CSS
  消除伪类依赖。
- `needsFirstLastChildWorkaround`（dash.js，mutter MR !2047）在 GNOME
  Shell ≥ 49 为 FALSE，与本问题无关。

## 复现/验证方法（可复用）

临时插桩（验证后移除）：在 `docking.js` 的 `_onOverviewShowing/Hiding/
Hidden` 里于 0/100/250/400ms 采样 `dash._box` 及每个子节点的
`get_allocation_box()` + 应用名，`print('D2D-ALLOC ...')`；
`journalctl --user | grep D2D-ALLOC` 读取。判定标准：各采样帧 box 尺寸
一致（无 52px/681 之类的异常值）。

环境：`make install-local` 编译 scss 并安装到
`~/.local/share/gnome-shell/extensions/dash-to-dock@haiquan27.github.com`；
扩展只在 **logout/login** 后重载。

## 像素预算（LEFT，icon 48，12 tile + 分隔符）

| 段落 | 稳态 |
|---|---|
| tile | 48 宽（padding 14+10，轴 padding 0） |
| item margin | 2/2（无归零） |
| 容器轴 padding | 8（=10−2） |
| box 总宽 | 12×48 + 44 + 9 = 629 |
| show-apps item margin | 靠近 box 侧 0，外侧 2 |

异常帧（修复前）：tile 52（轴 padding 2）→ box 681；item margin 未归零
→ 再 +4。修复后首帧=稳态，无异常帧。
