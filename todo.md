# MarkMark TODO

## 🔴 Markdown 文件无 Raw 模式入口（编辑能力整体失效）
- [ ] Markdown 文件当前**没有任何切到 Raw/编辑模式的入口**，导致整套编辑能力（`SyntaxHighlightedEditor` ~534 行、`RawMarkdownView`、per-file undo swizzling）对 .md 文件永不触发，标注（replace/highlight/delete）提交后也无法回 Raw 手改 CriticMarkup。
  - 根因 1：切换按钮被删 —— `Sources/MarkMark/Views/DetailView.swift:272` 注释「MarkMark 无编辑功能，移除渲染/编辑切换」，titleBar 里没有渲染/编辑切换按钮。
  - 根因 2：通知是死代码 —— `.switchToRendered` / `.switchToRaw`（`App/MarkdownReaderApp.swift:233-234`）有人监听（`ContentView.swift:990-994`），但**全代码库无人 post**，菜单里也无对应项。
  - 根因 3：视图层不按 displayMode 分支 —— `DetailView.documentContentView`（约 807-815 行）只看 `isPlainTextDocument`：非 md → `RawMarkdownView`（只读）；md → 永远 `WebViewMarkdownView`，根本不查 `displayMode`。
  - 连带：`SettingsModel.defaultDisplayMode` 对 md 也是死的（视图不读它），本地化键 `titleBarDisplayMode`（"显示模式"）定义了却无人引用。
  - 决策点：要么恢复 Raw 模式入口（titleBar 按钮 + 菜单 + 让 `documentContentView` 按 `displayMode` 分支），要么明确砍掉编辑相关死代码。这一条直接影响下面「替换/高亮/删除无法编辑」的缺口——目前连 Raw 兜底都没有。

## 未保存提醒缺失
- [ ] 没有保存直接打开新文件、或退出 app 时，没有常规的「是否保存」弹窗提醒。
  - 线索：`handleUnsavedChangesBeforeAction` 目前只接在 `FileOpenModifier` 的 openFile / openDirectory / newFromClipboard 上（`ContentView.swift`）；窗口关闭走 `WindowCloseGuard`（`ContentView.swift`），但 **app 退出（`applicationShouldTerminate`）路径可能没接上**，需要确认关闭/退出两处都拦截脏文档。
  - 相关：`DocumentViewModel.hasAnyDirtyFile`（含缓存里的脏文件）。

## 替换/高亮/删除缺少编辑/撤销入口
- [ ] 渲染视图中只有 comment（`{>> … <<}`）可点气泡编辑/删除；replace（`{==原==>新==}`）、highlight（`{==…==}`）、delete（`{--…--}`）提交后无法再点开修改或撤销，且目前 Markdown 无 Raw 模式入口（见上节），连手动改 CriticMarkup 的兜底都没有。
  - 线索：`Sources/MarkMark/Resources/js/markdown-reader.js` 第 968 行的点击监听只匹配 `.critic-comment`，没有针对其他三种渲染元素的处理器。可考虑给这三种也加 popover 入口（编辑/删除/还原原文）；或先恢复 Raw 模式入口作为兜底。

## 统一「Enter = 提交」快捷键
- [ ] Enter 提交：在**替换（replace）**场景生效，在**评论（comment）**场景不生效，需统一。
  - 线索：`Sources/MarkMark/Resources/js/markdown-reader.js` 里评论弹窗的 keydown 提交条件与替换输入框不一致——评论当前要求 `⌘/Ctrl + Enter` 才提交，而替换允许纯 Enter 提交。把评论也改为纯 Enter 提交（多行时仍可用 Shift+Enter 换行）。

## Dark 主题选区不明显
- [ ] Dark theme 下选中的文本高亮不够明显。
  - 线索：渲染视图的选区色来自 `::selection` CSS / `ThemeColors` 派生色（`Services/ThemeColors.swift` + 渲染 CSS 变量），深色下需要换一个对比度更高的 selection 色。

## 设置页面侧栏在窄窗口下不可见
- [ ] 窗口宽度缩到很窄时，设置页左侧的 tab 菜单（通用 / 外观）被挤出可视区域、不可见。
  - 根因：`Sources/MarkMark/Views/ContentView.swift:280` 设置侧栏用 `.frame(width: appViewModel.sidebarWidth)`（默认 240，固定），右侧 `SettingsContentView` 第 935 行 `minWidth: 480`，两者合计 720；但窗口整体 `minWidth: 650`（第 90 行）小于这个合计，窄窗口下侧栏被挤掉。
  - 可选修法：① 设置模式下把窗口 minWidth 提到 ≥720；② 给设置侧栏单独的小固定宽度（如 180）或允许它跟随窗口收窄；③ 窄窗口下把 tab 改成顶部横向 TabView / Segmented Picker。

## 从 Finder 拖入文件不生效
- [ ] 从 Finder 拖入 `.md` / `.txt` 文件到窗口没有反应。
  - 线索：拖拽由 `AppDelegate.installFileDropHandler` 安装的 `FileDropOverlayView` 处理（`App/AppDelegate.swift`），覆盖层挂在 `themeFrame` 上、`hitTest` 返回 nil。需要排查拖拽类型注册 / overlay 层级 / 命中是否被 WKWebView 等子视图抢走。
  - 对照：双击文件打开（`application(_:open:)` + WindowRouter）是正常的，说明问题只在拖拽这条路径。

## 窗口大小不记忆
- [ ] 关闭窗口时的尺寸不被保存，下次打开永远是 `.defaultSize` 的 900×600（再加 SwiftUI/NSWindow 的 cascade 偏移），用户每次都得手动拖回习惯的大小。
  - 根因 1：初始尺寸写死 —— `Sources/MarkMark/App/MarkdownReaderApp.swift:104` `.defaultSize(width: 900, height: 600)`，纯静态 scene modifier，不读任何持久化值。
  - 根因 2：macOS 自带 frame restoration 被主动关掉 —— `Sources/MarkMark/Views/ContentView.swift:553` 的 `configureContentWindow` 设了 `window.isRestorable = false`，注释说明是为了避免上次会话窗口被 SwiftUI scene 恢复叠加成多窗口。所以系统那条「记忆窗口」的路径是断的。
  - 根因 3：关闭/退出时没人落盘 frame —— `WindowCloseGuardView.windowShouldClose`（`ContentView.swift:1570`）只管脏文档提醒，不存 frame；`applicationShouldTerminate` 路径同样没接（见上面「未保存提醒」那条）。
  - 实现思路：在 `SettingsModel`（`@Observable` + 手动 UserDefaults，`SettingsModel.swift:417` init 恢复处）加 `windowWidth` / `windowHeight`（可选再加 `windowOrigin`）；① `configureContentWindow` 里 `window.setFrame(...)` 还原；② 关闭时在 `windowShouldClose`、以及 `applicationShouldTerminate`（多窗口场景）里写回；③ 可选监听 `NSWindow.didResizeNotification`/`windowDidEndLiveResize` 防止异常尺寸。注意与 `isRestorable = false` 共存：自管 frame 不依赖系统 restoration，不冲突。
