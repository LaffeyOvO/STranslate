# 图片翻译精简窗口改造设计

> 日期：2026-06-25
> 分支：feature/imagetranslate
> 关联文件：`src/STranslate/Views/ImageTranslateCompactWindow.xaml(.cs)`、`src/STranslate/Core/ImageTranslateCompactWindowPlacement.cs`、`src/Tests/STranslate.Tests/ImageTranslateCompactWindowPlacementTests.cs`

## 背景与现状

当前精简窗口（`ImageTranslateCompactWindow`）整体是一块带 `ApplicationPageBackgroundThemeBrush` 背景的窗口：图片区（`ImageZoom`）占 `Row 0`，底部 `Row 1` 是 64px 高的按钮条 `Border`。窗口尺寸 = 选区物理尺寸 + 工具栏高度，定位用 `ImageTranslateCompactWindowPlacement.CreateForImageBounds` 直接贴回选区坐标。

存在问题：
1. 窗口有一块灰底背景包裹图片和按钮，看起来像"一个窗口"，而不是"截图 + 悬浮按钮"。
2. 选区太靠下时，下方 64px 工具栏会顶出屏幕，按钮被裁或不可见。
3. 选区太窄时，按钮条（约 300+px）比选区宽，按钮条会在窗口内居中，与选区宽度不匹配，视觉割裂。

## 目标

参考微信截图工具：**截图内容是窗口显示的核心，按钮作为窗口的额外悬浮内容，不需要窗口背景**。具体三条诉求：

1. 窗口透明无边框，屏幕上只看到截图内容 + 悬浮按钮条（按钮条自带半透明胶囊背景）。
2. 图片太靠下显示不下时，按钮条翻到图片上方。
3. 选区太窄时，窗口横向延展以完整显示按钮条，图片仍对齐选区。

## 两条铁律（贯穿所有场景）

1. **贴图位置不变**：图片始终钉在用户截图选区的物理屏幕位置。任何避让都通过移动/翻向按钮条完成，绝不移动图片。
2. **按钮始终可见可点**：无论空间多紧，按钮条必须完整可见且可点击。宁可盖住图像也不裁切按钮。

## 已确认的五项决策

1. **窗口透明无边框（完全透明）**：去掉窗口 `Background`，窗口本身完全透明；屏幕上只看到截图内容 + 悬浮按钮条。**不动 `ImageZoom` 已有的图片渲染样式**（当前 `Viewbox Stretch=Uniform` + `Image Stretch=Fill` 效果已确认满意）。按钮条保留自身的半透明胶囊背景（`#CC1F1F1F` + `CornerRadius`）。

2. **选区太窄 → 贴左缘延展**：按钮条比选区宽时，图片钉选区左上角不动，窗口向右延展，按钮条左缘对齐图片左缘。

3. **右边顶到屏幕边缘 → 向左延展按钮条**：贴左缘延展会顶出屏幕右边时，按钮条改为右缘对齐图片右缘、向左延展（图片仍不动）。即按钮条右缘 = 图片右缘，按钮条向左延伸，窗口左边也随之向左扩。这是决策2「左缘对齐向右延展」的镜像。

4. **下方放不下才翻上方**：默认按钮条在图片下方；仅当「图片底到屏幕底」剩余空间容不下按钮条（含间距）时翻到图片上方。空间够就一直放下方。

5. **上下都放不下 → 压在图片上**：极端情况图片几乎占满屏幕高度、上下都容不下按钮条时，按钮条浮在图片底部之上盖住一小块图像，保住两条铁律。

## 布局算法

按以下优先级顺序判定（输入：选区物理坐标 `imageBounds`、DPI、按钮条尺寸、屏幕工作区）：

### 步骤 1：图片位置（永远不动）
- 图片左上角 = `imageBounds.Left, imageBounds.Top`
- 图片显示尺寸 = `imageBounds.Width × imageBounds.Height`（1:1，无缩放，与当前一致）

### 步骤 2：横向布局
设 `toolbarWidth` = 按钮条所需宽度（含 padding/margin），`imageWidth` = `imageBounds.Width`，`gap` = 图片与按钮条的横向间距（当前 margin 8）。

- **若 `toolbarWidth ≤ imageWidth`**：按钮条横向居中于选区（`toolbarLeft = imageBounds.Left + (imageWidth - toolbarWidth)/2`）。窗口宽度 = `imageWidth`。
- **若 `toolbarWidth > imageWidth`（选区太窄）**：按钮条贴左缘，`toolbarLeft = imageBounds.Left + gap`。窗口需延展：
  - 窗口右边界 = `imageBounds.Left + imageWidth`（图片右缘）与 `toolbarLeft + toolbarWidth + gap`（按钮条右缘）取较大值，即向右延展。
  - **若向右延展后超出屏幕工作区右边界**：按钮条改为向左延展。`toolbarRight = imageBounds.Left + imageWidth - gap`，`toolbarLeft = toolbarRight - toolbarWidth`。窗口左边界 = `toolbarLeft - gap`（向左扩），右边界 = 图片右缘。图片仍不动。

### 步骤 3：纵向布局
设 `toolbarHeight` = 按钮条高度（当前 64），`imageBottom = imageBounds.Top + imageBounds.Height`，`gapV` = 图片与按钮条纵向间距（当前 margin 6）。

计算下方可用空间 `spaceBelow = workArea.Bottom - imageBottom`。

- **若 `spaceBelow ≥ toolbarHeight + gapV`（下方放得下）**：按钮条在图片下方。`toolbarTop = imageBottom + gapV`。窗口高度 = `imageBounds.Height + gapV + toolbarHeight + margin`，窗口顶 = `imageBounds.Top`。
- **若 `spaceBelow < toolbarHeight + gapV`（下方放不下）**：尝试翻上方。计算上方可用空间 `spaceAbove = imageBounds.Top - workArea.Top`。
  - **若 `spaceAbove ≥ toolbarHeight + gapV`（上方放得下）**：按钮条在图片上方。`toolbarTop = imageBounds.Top - gapV - toolbarHeight`。窗口高度 = `toolbarHeight + gapV + imageBounds.Height + margin`，窗口顶 = `toolbarTop`。
  - **若 `spaceAbove < toolbarHeight + gapV`（上下都放不下）**：按钮条压在图片上。`toolbarTop = imageBottom - toolbarHeight`（按钮条底缘对齐图片底缘，浮在图片之上）。窗口高度 = `imageBounds.Height + margin`，窗口顶 = `imageBounds.Top`，窗口底 = `imageBottom`。按钮条 `Panel.ZIndex` 高于图片，半透明背景保证图片部分可见。

### 步骤 4：窗口尺寸与定位
- 窗口 `Left/Top` = 步骤 2/3 计算的窗口左上角（可能因向左延展或翻上方而小于 `imageBounds.Left/Top`）。
- 窗口 `Width/Height` = 步骤 2/3 计算的窗口宽高。
- 图片在窗口内的偏移 = `imageBounds.Left - window.Left`（横向）、`imageBounds.Top - window.Top`（纵向），用 `Margin` 或 `Canvas` 定位 `ImageZoom` 使其钉在选区绝对位置。

## 架构改动

### 1. `ImageTranslateCompactWindowPlacement.cs`（核心算法）

重写/新增布局方法，返回一个完整的布局结果结构，包含：
- `WindowBounds`（物理像素 `Rectangle`）：窗口左上角 + 宽高，用于 `SetWindowPos`。
- `ImageOffset`（DIP）：图片在窗口内的左上角偏移，用于 XAML 内 `ImageZoom` 定位。
- `ToolbarBounds`（DIP）：按钮条在窗口内的位置 + 尺寸，用于 XAML 内按钮条定位。
- `ToolbarSide`（枚举 `Below`/`Above`/`Overlay`）：按钮条位置模式，驱动 XAML 布局。

替换当前 `CreateForImageBounds`（只算窗口矩形）为 `CreateLayout`（算完整布局）。`CreateCenteredOnWorkArea`（兜底）保留不变。`ToDipBounds` 保留。

### 2. `ImageTranslateCompactWindow.xaml`（XAML 布局）

- 移除 `Window.Background`（改为 `Transparent` 或删除），确保 `AllowsTransparency` 配合 `WindowStyle=None` 实现真透明（当前已 `WindowStyle=None`，需确认透明生效）。
- ⚠️ **透明渲染风险**：完全透明窗口需 `AllowsTransparency=True`，会切换到软件渲染路径，可能影响 `ImageZoom` 渲染质量/性能（用户已确认满意当前渲染效果）。实现时优先尝试 `AllowsTransparency=True`；若渲染质量明显下降，备选方案是用 `WindowChrome`/`HwndSource` 透明（保留硬件加速）或保留 1px 不可见边框。此风险点需在实现阶段实测验证，若影响渲染则回退讨论。
- 将根 `Grid` 改为绝对定位容器（`Canvas` 或 `Grid` + 计算 `Margin`），用布局结果中的 `ImageOffset` / `ToolbarBounds` 定位 `ImageZoom` 和按钮条 `Border`。
- 按钮条 `Border` 的 `Panel.ZIndex` 保持高于 `ImageZoom`，确保叠加时按钮在上。
- `ImageZoom` 的 `IsPanAndZoomEnabled="False"` 等现有属性不变，渲染样式不动。
- 底部 `RowDefinition Height="64"` 固定行高的写法改为由布局结果动态定位（不再用固定行高 Grid）。

### 3. `ImageTranslateCompactWindow.xaml.cs`（定位调用）

- `PlaceOnPhysicalBounds` 调用新 `CreateLayout`，拿到完整布局结果。
- `PlaceOnPhysicalWindowBounds` 仍负责两步定位（DIP 写回 + `SetWindowPos`），但 `WindowBounds` 来自新算法。
- 新增：把布局结果中的 `ImageOffset` / `ToolbarBounds` / `ToolbarSide` 传给 XAML（通过代码后置设置 `ImageZoom.Margin` 和按钮条 `Border` 的 `Margin`/`Visibility`，或通过绑定属性）。
- `ToolbarReservedHeight` 常量保留，作为按钮条高度输入。

### 4. 测试 `ImageTranslateCompactWindowPlacementTests.cs`

更新现有测试（`CreateForImageBounds` 签名变化），新增覆盖各场景的 `CreateLayout` 测试：
- 选区宽 ≥ 按钮条宽 → 居中，窗口=选区+下方工具栏。
- 选区宽 < 按钮条宽，右边放得下 → 贴左缘向右延展。
- 选区宽 < 按钮条宽，右边放不下 → 向左延展，窗口左扩。
- 选区靠下，下方放不下、上方放得下 → 翻上方，窗口顶上移。
- 选区占满高度，上下都放不下 → 叠加，窗口=选区尺寸，按钮条 ZIndex 在上。
- 多 DPI 下物理像素↔DIP 换算正确（沿用现有 `ToDipBounds` 测试）。

## 不改动的部分

- `ImageZoom` 控件的渲染样式（`Viewbox`/`Image`/`Stretch` 等）。
- 精简窗口的输入绑定、右键菜单、关闭行为（`OnDeactivated`/`OnClosing`/Esc 等）。
- `ImageTranslateWindowViewModel` 及执行流程。
- `Screenshot` 截图回传坐标逻辑（`CaptureWithRegionAsync` 已稳定）。
- 兜底定位 `CreateCenteredOnWorkArea` / `PlaceNearCursorScreen`。

## 验证清单（人工，多场景）

1. 主屏 100% 缩放，框选中部区域 → 图片贴回选区，按钮条在下方居中，无窗口背景。
2. 框选窄竖区域（宽 < 按钮条）→ 图片钉选区左缘，窗口向右延展，按钮条左对齐图片。
3. 框选靠近屏幕右边缘的窄区域 → 按钮条向左延展，窗口左扩，图片不动。
4. 框选贴近屏幕底部的区域 → 按钮条翻到图片上方，窗口顶上移。
5. 框选几乎满屏高度的区域 → 按钮条压在图片底部之上，按钮可见可点。
6. 多显示器 + 非 100% 缩放 → 贴图零偏移（沿用 ScreenGrab 坐标回传）。
7. 浅色截图在浅色桌面 → 无窗口背景但不影响操作（按钮条半透明胶囊仍清晰）。
