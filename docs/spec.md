# Whisky 游戏库升级技术规格

## 1. 文档目标

本文档将 [prd.md](/Users/xdwanj/Project/Swift/Whisky/docs/prd.md) 转化为可实现的规格说明，回答以下问题：

- 需要新增哪些核心模型。
- 这些模型如何持久化。
- 首页和详情页如何重构。
- 搜索、分组、素材管理如何落地。
- 如何兼容现有 `Bottle` / `Program` / `pins` 数据。

本文档默认范围为 PRD 的 `P0 + 关键 P1`，即：

- 游戏是一等实体
- 游戏首页
- 游戏分组
- `cover / banner / icon` 三类素材
- 全库搜索、过滤、排序
- 基础批量操作
- 兼容迁移

## 2. 设计原则

1. 不破坏现有 Bottle 运行链路。
2. 新游戏库层独立存在，但与 Bottle/Program 保持引用关系。
3. 优先本地模型和本地素材，不引入第一期在线依赖。
4. 先建立稳定的数据层，再重构 UI。
5. 默认支持 50-500 游戏规模。

## 3. 范围

### 3.1 In Scope

- 新增游戏库领域模型
- 新增游戏分组模型
- 新增游戏素材存储与读取逻辑
- Library 首页从 Bottle 视角转为 Game 视角
- 新增游戏详情页和素材管理入口
- 新增全库搜索、过滤、排序
- 新增基础批量操作
- 新增首次迁移和后续同步机制

### 3.2 Out of Scope

- 在线元数据抓取
- 在线库同步
- Saved Search
- 智能推荐或自动分类
- 重写 Bottle 配置页
- 彻底替换现有 Program 设置存储

## 4. 术语

| 术语 | 含义 |
| --- | --- |
| `Bottle` | 现有 Wine 容器实体 |
| `Program` | Bottle 内可执行目标 |
| `LibraryGame` | 新增的游戏库实体，供首页和搜索直接使用 |
| `GameGroup` | 用户自定义或系统提供的分组 |
| `Artwork` | 游戏素材，包含 `cover` / `banner` / `icon` |
| `LibraryIndex` | 游戏库索引数据，用于持久化和启动时加载 |

## 5. 总体架构

### 5.1 分层

建议新增 4 层结构：

1. `WhiskyKit` 领域模型层
2. `WhiskyKit` 仓储与持久化层
3. `Whisky` ViewModel 状态层
4. `Whisky` SwiftUI 视图层

### 5.2 推荐模块划分

建议新增或扩展如下目录：

- `WhiskyKit/Sources/WhiskyKit/Whisky/LibraryGame.swift`
- `WhiskyKit/Sources/WhiskyKit/Whisky/GameGroup.swift`
- `WhiskyKit/Sources/WhiskyKit/Whisky/GameArtwork.swift`
- `WhiskyKit/Sources/WhiskyKit/Whisky/LibraryIndex.swift`
- `WhiskyKit/Sources/WhiskyKit/Utils/LibraryStore.swift`
- `WhiskyKit/Sources/WhiskyKit/Utils/LibraryMigrator.swift`
- `Whisky/View Models/LibraryVM.swift`
- `Whisky/Views/Library/` 下重构首页、详情、分组、素材管理视图

### 5.3 依赖关系

- `Bottle` 和 `Program` 继续负责运行与容器配置
- `LibraryGame` 只引用 `Bottle` / `Program` 的稳定标识
- 首页 UI 不再直接以 `Bottle` 为主模型
- `LibraryVM` 通过 `LibraryStore` 读写持久化

## 6. 数据模型

## 6.1 LibraryGame

建议定义为可编码模型：

```swift
public struct LibraryGame: Codable, Identifiable, Hashable {
    public var id: String
    public var displayName: String
    public var sortName: String
    public var bottleURL: URL
    public var programURL: URL
    public var source: LibraryGameSource
    public var addedAt: Date
    public var lastPlayedAt: Date?
    public var playtimeSeconds: TimeInterval
    public var isFavorite: Bool
    public var isHidden: Bool
    public var groupIDs: [String]
    public var artwork: GameArtworkSet
}
```

### 字段约束

- `id`
  - 必须稳定，不能直接用展示名。
  - 建议使用 `bottleURL + programURL` 的规范化哈希。
- `displayName`
  - 默认来自 `Program.name` 去掉 `.exe`。
- `sortName`
  - 默认等于 `displayName`。
- `bottleURL`
  - 必须保留，用于回跳 Bottle 详情。
- `programURL`
  - 必须保留，用于直接启动。
- `source`
  - 第一阶段可枚举为 `detected`、`manual`、`migratedPin`。
- `groupIDs`
  - 仅存自定义组 ID；系统组通过计算得出。
- `artwork`
  - 只存相对路径或素材 ID，不直接存巨大二进制数据。

## 6.2 LibraryGameSource

```swift
public enum LibraryGameSource: String, Codable {
    case detected
    case manual
    case migratedPin
}
```

## 6.3 GameArtworkSet

```swift
public struct GameArtworkSet: Codable, Hashable {
    public var cover: GameArtwork?
    public var banner: GameArtwork?
    public var icon: GameArtwork?
}
```

## 6.4 GameArtwork

```swift
public struct GameArtwork: Codable, Hashable {
    public var kind: GameArtworkKind
    public var relativePath: String
    public var source: GameArtworkSource
    public var updatedAt: Date
}
```

补充枚举：

```swift
public enum GameArtworkKind: String, Codable {
    case cover
    case banner
    case icon
}

public enum GameArtworkSource: String, Codable {
    case imported
    case generated
    case migrated
}
```

### 约束

- 第一阶段允许 `source = generated`，用于从程序图标回退生成 icon。
- `cover` / `banner` 缺失时可用默认占位，不强制生成伪图。
- `relativePath` 必须相对 `Library Assets` 根目录，便于未来迁移。

## 6.5 GameGroup

```swift
public struct GameGroup: Codable, Identifiable, Hashable {
    public var id: String
    public var name: String
    public var createdAt: Date
    public var updatedAt: Date
    public var sortOrder: Int
    public var isSystem: Bool
}
```

### 分组约束

- 用户自定义组持久化。
- 系统组不落盘或只以配置方式落盘，不写入用户可编辑列表。
- 一个 `LibraryGame` 可属于多个 `GameGroup`。

## 6.6 系统分组

第一阶段定义 4 个系统分组：

- `all`
- `favorites`
- `uncategorized`
- `hidden`

这些分组不进入 `groupIDs`，由计算逻辑返回：

- `favorites`: `isFavorite == true`
- `hidden`: `isHidden == true`
- `uncategorized`: `groupIDs.isEmpty && !isHidden`
- `all`: 全量非删除游戏

## 6.7 LibraryIndex

建议将整个游戏库持久化为单一索引文件：

```swift
public struct LibraryIndex: Codable {
    public var version: Int
    public var migratedAt: Date?
    public var games: [LibraryGame]
    public var groups: [GameGroup]
}
```

### 版本策略

- 第一版 `version = 1`
- 后续 schema 变更通过 version 升级处理

## 7. 持久化设计

## 7.1 存储位置

建议新增目录：

- Library 索引
  - `~/Library/Application Support/Whisky/Library/index.json`
- Library 素材
  - `~/Library/Application Support/Whisky/Library/Assets/`

按游戏 ID 分目录：

- `Assets/<gameID>/cover.jpg`
- `Assets/<gameID>/banner.jpg`
- `Assets/<gameID>/icon.png`

### 原因

- 避免污染 Bottle 目录
- 方便独立备份与迁移
- 方便未来引入素材缓存和远程同步

## 7.2 文件格式

- 索引文件使用 JSON
- 素材使用实际图片文件
- 图片扩展名以最终转码结果为准

## 7.3 写入策略

- 索引采用整文件原子写入
- 素材采用先写临时文件再替换
- 所有路径写入前必须确保父目录存在

## 8. 迁移与同步

## 8.1 首次迁移触发条件

启动时：

1. 如果 `Library/index.json` 不存在，执行全量迁移
2. 如果存在，则执行增量同步

## 8.2 全量迁移流程

1. 扫描所有 `Bottle`
2. 对每个 Bottle 扫描 `programs`
3. 过滤已不存在的 `programURL`
4. 为每个 Program 生成 `LibraryGame`
5. 将原有 `pins` 映射为：
   - `isFavorite = true`
   - `source = migratedPin`
6. 写入 `LibraryIndex`

## 8.3 增量同步流程

每次 `bottleVM.loadBottles()` 或手动刷新后执行：

1. 重新扫描现有 Bottle / Program
2. 用 `programURL` 或 `gameID` 做匹配
3. 新发现的 Program 自动加入库
4. 已删除的 Program 标记为失效或直接移除
5. 保留用户编辑过的：
   - `displayName`
   - `sortName`
   - `groupIDs`
   - `isFavorite`
   - `isHidden`
   - `artwork`

### 删除策略

第一阶段建议：

- Program 文件不存在时，将游戏保留在索引中但标记为不可启动，后续可在 UI 中清理

可选字段：

```swift
public var isLaunchTargetAvailable: Bool
```

## 8.4 冲突规则

- 机器扫描结果只覆盖来源于运行时探测的字段
- 用户编辑字段优先级高于扫描字段
- `pins` 迁移只在首次迁移时执行，不持续双向同步

## 9. 素材系统

## 9.1 第一阶段支持的能力

- 读取当前素材
- 选择本地图片
- 替换图片
- 删除图片
- 回退默认展示

## 9.2 素材导入规则

### icon

- 优先接受 `png`
- 若用户选择其他格式，转码为 `png`
- 建议标准输出尺寸 `256x256`

### cover

- 接受 `jpg/png`
- 建议标准输出比例 `3:4`
- 第一阶段只做缩放，不强制智能裁剪

### banner

- 接受 `jpg/png`
- 建议标准输出比例 `16:9`
- 第一阶段只做缩放，不强制智能裁剪

## 9.3 默认回退规则

- `icon`
  - 优先使用自定义素材
  - 否则使用程序提取 icon
  - 否则使用系统默认游戏图标
- `cover`
  - 优先使用自定义素材
  - 否则使用默认占位图
- `banner`
  - 优先使用自定义素材
  - 否则使用默认占位图

## 9.4 素材管理接口

建议在 `LibraryStore` 或单独 `ArtworkStore` 提供：

```swift
func importArtwork(for gameID: String, kind: GameArtworkKind, from sourceURL: URL) throws
func removeArtwork(for gameID: String, kind: GameArtworkKind) throws
func resolvedArtworkURL(for game: LibraryGame, kind: GameArtworkKind) -> URL?
```

## 10. 搜索、过滤、排序

## 10.1 搜索模型

第一阶段搜索目标：

- `displayName`
- `sortName`
- Bottle 名称

建议定义状态：

```swift
struct LibraryQuery {
    var searchText: String
    var selectedGroupID: String?
    var selectedBottleURL: URL?
    var favoritesOnly: Bool
    var hiddenOnly: Bool
    var uncategorizedOnly: Bool
    var missingArtworkKinds: Set<GameArtworkKind>
    var sort: LibrarySort
    var viewMode: LibraryViewMode
}
```

## 10.2 排序

```swift
enum LibrarySort: String, Codable {
    case nameAscending
    case recentlyPlayed
    case recentlyAdded
    case playtimeDescending
}
```

### 排序细则

- `nameAscending`
  - 优先 `sortName`
- `recentlyPlayed`
  - `lastPlayedAt` 降序，空值排后
- `recentlyAdded`
  - `addedAt` 降序
- `playtimeDescending`
  - `playtimeSeconds` 降序

## 10.3 视图模式

```swift
enum LibraryViewMode: String, Codable {
    case grid
    case list
}
```

规则：

- `grid` 主展示 `cover`
- `list` 主展示 `banner`
- 行首和详情头图始终可用 `icon`

## 11. UI 规格

## 11.1 首页布局

建议保留 `NavigationSplitView`，但重构内容结构：

### Sidebar

- Home
- All Games
- Favorites
- Uncategorized
- Hidden
- Groups
- Bottles

### Content

- Home 模式
  - Continue Playing
  - Favorites
  - Recently Added
  - All Games Preview
- List/Grid 模式
  - 工具栏
  - 搜索栏
  - 过滤条
  - 游戏集合视图

### Detail

- 游戏详情
- 素材管理
- 分组编辑
- Bottle 跳转

## 11.2 首页工具栏

必须包含：

- 刷新
- 新建 Bottle
- 视图切换
- 排序菜单

建议包含：

- 快速过滤菜单
- 批量模式开关

## 11.3 游戏卡片

### Grid 卡片

展示：

- cover
- displayName
- Bottle 名称
- favorite 状态
- 可启动状态

交互：

- 单击选中
- 双击或主按钮启动
- 右键菜单

### List 行

展示：

- banner 或 icon
- displayName
- Bottle 名称
- group 标签
- 最近游玩/游玩时长

## 11.4 游戏详情页

详情页必须包含：

- `icon`
- `displayName`
- 所属 Bottle
- 启动目标路径
- 分组列表
- favorite / hidden 开关
- `cover` / `banner` / `icon` 管理入口

操作：

- 启动游戏
- 打开 Bottle
- 编辑名称
- 编辑分组
- 替换素材

## 11.5 分组管理

需要两个层级：

1. Sidebar 中浏览分组
2. 详情页或批量面板中编辑游戏所属分组

第一阶段支持：

- 创建分组
- 重命名分组
- 删除分组
- 添加游戏到分组
- 从分组移除游戏

## 11.6 批量操作

第一阶段可采用简单多选模型：

- `Set<LibraryGame.ID>` 作为选中集
- 批量菜单作用于选中集

支持动作：

- Add to Group
- Remove from Group
- Favorite
- Unfavorite
- Hide
- Unhide

## 12. ViewModel 规格

## 12.1 LibraryVM

建议新增主视图模型：

```swift
@MainActor
final class LibraryVM: ObservableObject {
    @Published var index: LibraryIndex
    @Published var query: LibraryQuery
    @Published var selectedGameID: String?
    @Published var selectedSidebarItem: LibrarySidebarItem
    @Published var selectedGameIDs: Set<String>
    @Published var isRefreshing: Bool
}
```

## 12.2 核心职责

- 启动时加载索引
- 首次迁移 / 增量同步
- 生成过滤后的游戏列表
- 管理分组和素材编辑
- 将游戏操作映射回 `Program.run()`

## 12.3 关键方法

建议方法：

```swift
func bootstrap(with bottles: [Bottle])
func refreshLibrary(with bottles: [Bottle])
func launchSelectedGame()
func toggleFavorite(_ gameID: String)
func toggleHidden(_ gameID: String)
func createGroup(name: String)
func renameGroup(_ groupID: String, name: String)
func deleteGroup(_ groupID: String)
func assign(_ gameIDs: Set<String>, to groupID: String)
func remove(_ gameIDs: Set<String>, from groupID: String)
func importArtwork(_ kind: GameArtworkKind, for gameID: String, from url: URL)
```

## 13. Repository / Store 规格

## 13.1 LibraryStore

建议职责：

- 加载/保存 `LibraryIndex`
- 提供增量同步方法
- 提供素材导入和删除方法

建议接口：

```swift
protocol LibraryStoreProtocol {
    func load() throws -> LibraryIndex
    func save(_ index: LibraryIndex) throws
    func bootstrap(from bottles: [Bottle]) throws -> LibraryIndex
    func reconcile(current: LibraryIndex, bottles: [Bottle]) throws -> LibraryIndex
    func importArtwork(for gameID: String, kind: GameArtworkKind, from sourceURL: URL) throws -> GameArtwork
    func removeArtwork(for gameID: String, kind: GameArtworkKind) throws
}
```

## 13.2 LibraryMigrator

建议单独抽离首次迁移逻辑，避免 `LibraryStore` 过胖。

职责：

- 从 `Bottle` + `Program` 生成初始 `LibraryIndex`
- 处理 `pins -> favorite` 迁移
- 处理默认名字清洗

## 14. 启动与运行行为

## 14.1 启动游戏

规则：

- `LibraryGame` 本身不负责运行
- 启动时通过 `bottleURL + programURL` 找回 `Program`
- 若 `Program` 丢失，则按钮禁用并提示目标不存在

建议流程：

1. `LibraryVM` 根据 `selectedGameID` 定位 `LibraryGame`
2. 从当前 `BottleVM` 中找到对应 `Bottle`
3. 从 `bottle.programs` 中找到 `programURL`
4. 调用 `program.run()`
5. 成功触发后更新 `lastPlayedAt` 和 `playtime` 统计入口

## 14.2 游玩时间

第一阶段建议：

- 先记录 `lastPlayedAt`
- `playtimeSeconds` 预留字段，不强制完整实现真实计时

如果已有可复用的运行事件钩子，再做计时累计。

## 15. 兼容性要求

1. `BottleSettings` 不做破坏式修改
2. `ProgramSettings` 不做破坏式修改
3. 原 `pins` 继续可用，但首页不再依赖 pins 生成核心视图
4. 原 `Library` 渐变占位样式可作为默认素材回退

## 16. 失败处理

### 16.1 索引损坏

- JSON 解码失败时：
  - 备份旧文件到 `index.json.bak`
  - 重新执行全量迁移

### 16.2 素材导入失败

- 不修改原有素材引用
- 给出用户可见错误

### 16.3 Program 丢失

- 游戏保留在库中
- 标记不可启动
- 允许用户手动删除或等待下一次同步恢复

## 17. 测试建议

## 17.1 单元测试

建议补以下测试：

- `LibraryMigrator` 从 Bottle/Program 生成稳定 `LibraryGame`
- `pins` 正确迁移为 `favorite`
- 分组的增删改查
- 搜索/过滤/排序组合结果
- 素材导入后索引更新正确
- 缺失 Program 的 reconcile 逻辑

## 17.2 UI 验证

手动验证至少覆盖：

- 首次迁移
- 首页显示游戏而不是 Bottle
- Grid/List 切换
- 创建/删除/重命名分组
- 将游戏加入多个分组
- 替换 cover/banner/icon
- 搜索“未分类”“收藏”“缺图”
- 从详情页跳转 Bottle

## 18. 分阶段实现建议

## Phase 1

- 建立 `LibraryIndex`
- 建立 `LibraryGame` / `GameGroup` / `GameArtwork`
- 首次迁移和增量同步
- 首页切到游戏视角
- 基础搜索/过滤/排序
- 分组 CRUD
- 素材导入/删除

交付标准：

- 已能替代当前 Bottle 首页作为默认游戏库入口

## Phase 2

- 批量操作
- 完整详情页
- 缺图筛选
- 最近游玩与最近添加模块
- 更好的不可启动状态处理

## Phase 3

- 自动素材抓取
- 在线同步
- 智能集合
- 去重合并

## 19. 开放问题

这些问题不阻塞第一版 spec，但实现前需要确认：

1. `LibraryIndex` 是否放在 `WhiskyKit` 管理的统一 App Support 路径下。
2. `playtimeSeconds` 第一版是否只预留，不做精确统计。
3. 删除失效游戏时，是自动移除还是保留为灰态条目。
4. `cover` / `banner` 是否需要第一版就做裁剪 UI。
5. 旧 `ContentView` 与新 `LibraryHomeView` 是否统一，还是直接让 Library 成为默认主页。

## 20. 最终决策摘要

第一阶段的实现核心是：

- 以 `LibraryGame` 替换 `Bottle` 成为首页主对象
- 用 `LibraryIndex` 建立本地持久化游戏库
- 用 `GameGroup` 建立多对多分组体系
- 用 `GameArtwork` 建立真正的三素材资源体系
- 通过迁移层复用现有 `Bottle` / `Program` 运行能力

这样可以在不推翻现有运行架构的前提下，把 Whisky 从“容器视角管理器”升级为“游戏库视角管理器”。
