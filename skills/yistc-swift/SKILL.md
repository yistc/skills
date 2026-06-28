---
name: yistc-swift
description: 以符合用户偏好的方式编写 Swift、SwiftUI、iOS 和 macOS 代码。涵盖最新的 Apple 技术栈选择、代码风格、架构决策、macOS Settings/window management 和常见陷阱规避。
---

# Swift & SwiftUI Development Guidelines

## Goal
确保 AI 生成的 Swift/SwiftUI 代码：
1. 优先使用 Apple 最新官方技术栈（Observable macro、Swift Testing、SwiftData 等）
2. 遵循现代 Swift 并发模型（Swift Concurrency / async-await）
3. 避免已被标记为 deprecated 的旧模式
4. 符合用户个人的架构偏好和代码风格

## Technology Stack Preferences

### 状态管理：使用 Observable Macro
- **首选**：`@Observable` macro（Swift 5.9+ / iOS 17+ / macOS 14+）
- **废弃**：`ObservableObject` + `@StateObject` / `@ObservedObject` / `@Published`
- **注意**：在 struct 中使用 `@State`，在 view hierarchy 间传递用 `@Environment` 和 `@Bindable`

### 数据持久化：使用 SwiftData
- **首选**：`SwiftData`（iOS 17+ / macOS 14+）
- **废弃**：Core Data
- **原因**：SwiftData 是 Core Data 的继任者，与 Swift 语言特性（如 macros、Codable）集成更紧密，声明式语法更简洁

### Sub-skill: SwiftData Migrations
- **触发场景**：涉及 `VersionedSchema`、`SchemaMigrationPlan`、SwiftData store 迁移、legacy store 兼容、migration tests。
- **执行指南**：先读取 `references/swiftdata-migrations.md`。保持 app schema/stages 属于各项目；公共化只做流程、测试和护栏，不猜 SwiftData 内部行为。

### Sub-skill: SwiftData + CloudKit/iCloud Sync
- **触发场景**：涉及 SwiftData 自动 iCloud sync、CloudKit-backed `ModelContainer`、`cloudKitDatabase`、CloudKit schema、跨 iOS/macOS 同步、production schema、同步延迟/失败、或准备把本地 SwiftData store 接入 iCloud。
- **执行指南**：先读取 `references/swiftdata-cloudkit-sync.md`。把 CloudKit production schema 视为不可逆设计面；先判断业务是否真的需要 CloudKit，尤其是已有服务端 source of truth 的 app。

### Sub-skill: iOS SwiftUI WidgetKit
- **触发场景**：涉及 iOS widgets、WidgetKit、SwiftUI widget extension、`TimelineProvider`、`AppIntentConfiguration`/`StaticConfiguration`、widget gallery preview、Home Screen widget skeleton/redacted placeholder、App Group 共享数据、widget 刷新频率、或 widget 布局溢出。
- **执行指南**：先读取 `references/ios-widgetkit-swiftui.md`。默认把 widget 当作受限、短生命周期、只展示预先准备数据的 extension；不要默认让 widget 自己同步完整业务数据。

### Sub-skill: iOS SwiftUI Entry List Performance
- **触发场景**：涉及 iOS SwiftUI `List`/entry list/feed reader 滚动卡顿、context menu 呼出卡顿、row favicon/image 解码、每行 swipe/context menu 热路径、或用户提供性能调研报告要求修复。
- **执行指南**：先读取 `references/ios-swiftui-entry-list-performance.md`。不要默认替换 `List` 或分页；先独立核查 row body、菜单构建、图片解码和 projection 热路径。

### macOS 大列表与内存
- 当任务涉及 macOS 上千/上万行列表、SwiftUI `List` / `Table` 卡顿、CPU 100%、内存超标、SwiftData `@Query` 大数据集、RSS/log/mail/feed reader 类 UI 时，先读取 `references/macos-large-list-memory.md`。
- 该引用文件是 agent 执行指南，不是用户文档；按其中流程做代码审查、实现和验证。

### Sub-skill: macOS SwiftUI Settings Window Management
- **触发场景**：涉及 SwiftUI `Settings`、`openSettings`、menu bar app/LSUIElement 设置窗口打不开到最前、Settings 窗口恢复到旧位置、首次打开居中、AppKit frame autosave、`center()` 后闪一下、Preferences/Settings window focus race、Settings tab 切换时窗口大小/位置漂移、或按屏幕动态设置 Settings 内容尺寸。
- **执行指南**：先读取 `references/macos-settings-window-management.md`。区分“禁用/清理位置恢复”“把已创建窗口拉到最前”和“内容驱动的 tab 尺寸变化”；不要用窗口显示后的 `center()` 修复恢复位置；macOS 15+ Scene API 必须按部署目标做 availability 检查。

### Sub-skill: macOS AX Cross-Display Window Moving
- **触发场景**：涉及 `AXUIElement` 移动/缩放其他 app 窗口、`kAXPositionAttribute`、`kAXSizeAttribute`、跨显示器移动窗口、`NSScreen.visibleFrame`/`CGDisplayBounds` 坐标映射、Dock/menu bar safe area、窗口移动后保留旧屏 Dock 空白、或窗口移动时多次变形/卡顿。
- **执行指南**：先读取 `references/macos-ax-cross-display-window-moving.md`。不要控制 Dock；不要多轮 sleep/retry；用 AX 读回实际 frame 验证。

### Sub-skill: macOS AX Launch Window Auto-Move
- **触发场景**：涉及 app 完全退出后重新启动、`NSWorkspace.didLaunchApplicationNotification`、首个窗口没有触发 auto move/maximize、`AXObserverAddNotification` 注册卡顿、同时启动多个 app、或想用短时间轮询兜底。
- **执行指南**：先读取 `references/macos-ax-launch-window-autofill.md`。优先使用“后台注册 AX notification + 注册完成后补查一次窗口”，不要把 AX notification 注册同步塞进 launch monitor，也不要先上短时间轮询。

### Sub-skill: Sparkle 2.9+ Swift Concurrency Integration
- **触发场景**：涉及 Sparkle updater、`SPUUpdater`、`SPUUpdaterDelegate`、`SPUStandardUpdaterController`、`canCheckForUpdates`、SwiftUI `Check for Updates` menu/button、或 Sparkle 相关 Swift strict-concurrency / Swift 6 warning。
- **关键判断**：Sparkle 2.9.0+ 已给 `SPUUpdater`、`SPUUpdaterDelegate` 等添加 Swift concurrency annotations；旧版里 `@MainActor` delegate conform nonisolated `SPUUpdaterDelegate` 的 warning，应优先通过升级 Sparkle 解决，不要保留旧的 `nonisolated` delegate workaround。
- **常见陷阱**：Sparkle 官方 SwiftUI 示例里的 `updater.publisher(for: \.canCheckForUpdates)` 会对 `@MainActor` `SPUUpdater.canCheckForUpdates` 形成 Swift key path，在 strict concurrency 下可能出现 “cannot form key path to main actor-isolated property” warning。`#keyPath(SPUUpdater.canCheckForUpdates)` 也会静态引用该 actor-isolated property，不能可靠规避。
- **推荐处理**：`canCheckForUpdates` 本身仍是正确的 Sparkle menu validation API。AppKit menu 优先让 `SPUStandardUpdaterController` 接管 target/action 和 `validateMenuItem`；SwiftUI menu/button 需要 disabled state 时，使用 Sparkle 文档承诺的 KVO-compliant property name 做 Obj-C KVO string fallback，并把状态发布回 `@MainActor`。
- **验证要点**：构建时确认 `SparkleUpdater.swift` 不再有 Sparkle concurrency warning；检查更新按钮在 updater busy/idle 状态下 disabled state 正常；若未来 Sparkle 或 Swift compiler 提供 Swift-native observation 修复，再回收 string KVO fallback。

### Sub-skill: SwiftUI 包 AppKit NSTableView 的键盘选择与自绘高亮
- **触发场景**：macOS SwiftUI app 为了性能用 `NSViewRepresentable` 包 `NSTableView`，列表支持自绘 selected background，同时用户反馈鼠标点击正常，但上下方向键会取消 selection、跳到第一行、或 detail pane 已更新但中间栏高亮消失。
- **常见原因**：SwiftUI `Binding`、`NSTableView` 原生 selection、cell reuse/reconfigure 的时序不同步。键盘事件里先写 SwiftUI binding，再依赖下一轮 `updateNSView` 或直接从 binding 读 selected id，容易在 transient deselect、reload、visible cell reconfigure 期间读到旧值或 nil，导致自绘高亮丢失。
- **解决方案**：让 AppKit coordinator 持有本地 selected id 作为 selection 的即时真相源；在 `updateNSView` 边界从 SwiftUI 同步外部值；在键盘导航和 table selection delegate 中先写 coordinator 本地值，再写 binding；cell `configure` 和可见行刷新都从 coordinator 的 current selected id 判断高亮。不要把 `NSTableView` 换回 SwiftUI `List` 来规避问题。
- **验证要点**：鼠标点击、连续向下、连续向上、从中间行向上/向下、滚动后再导航，都要同时满足 detail pane 更新、AppKit selected row 保持、cell 自绘高亮保持。

### 测试框架：必须使用 Swift Testing
- **首选**：`Swift Testing`（Swift 6.0+ / Xcode 16+）
- **废弃**：XCTest）
- **原因**：Swift Testing 提供结构化测试、参数化测试、`#expect` / `#require` 等现代化断言，与 Swift 语言特性深度集成
- **写法示例**：
  ```swift
  import Testing

  @Suite("User Service Tests")
  struct UserServiceTests {
      @Test("should create user successfully")
      func createUser() async throws {
          let service = UserService()
          let user = try await service.create(name: "Test")
          #expect(user.name == "Test")
      }
  }
  ```

### 并发模型：使用 Swift Concurrency
- **首选**：`async` / `await`、`Task`、`Actor`、`@MainActor`
- **废弃**：GCD（DispatchQueue）、回调地狱、基于 closure 的异步 API
- **原因**：结构化并发更安全、可读性更好，编译器可检查数据竞争
- **注意**：UI 更新必须标记为 `@MainActor`，后台任务使用 `Task.detached` 或自定义 Actor

### 网络层：优先使用 URLSession + async/await
- **首选**：`URLSession` 的 `async` 方法（`data(from:)`、`upload(for:from:)` 等）
- **可接受**：`URLSession` + `Combine`（仅限需要响应式流的场景）
- **废弃**：基于 completion handler 的旧 API、Alamofire（除非项目已有依赖）
- **JSON 解析**：优先使用 `Codable` + `JSONDecoder`，避免手动解析字典

## Code Style Guidelines

### 命名规范
- 类型名：UpperCamelCase（`UserService`, `MainViewModel`）
- 函数/变量/属性：lowerCamelCase（`fetchUsers`, `isLoading`）
- 常量：lowerCamelCase（`let maxRetryCount = 3`）
- 布尔属性：使用 `is` / `has` / `should` 前缀（`isLoading`, `hasMoreData`）
- 避免无意义缩写（用 `userCount` 而非 `usrCnt`）

### SwiftUI 视图结构
- 视图 body 保持简洁，复杂逻辑抽取为 `@ViewBuilder` 方法或子视图
- 优先使用 `VStack` / `HStack` / `ZStack` 的 spacing 参数，而非 `Spacer()` + padding 组合
- 共享状态通过 `@Environment` 或依赖注入传递，避免深层 prop drilling
- 动画使用 `.animation` modifier 或 `withAnimation`，优先使用显式动画

### 错误处理
- 使用 `throws` + `do-catch` 或 `Result` 类型，避免 `Optional` 承载错误信息
- 用户可见的错误要转换为本地化字符串
- 网络错误区分：网络不可用、超时、服务端错误、数据解析错误
