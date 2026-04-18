# Whisky 游戏库升级 PRD

## 1. 背景

Whisky 当前已经有基础的 `Library` 首页，但它的核心对象仍然是 `Bottle`，而不是 `Game`。这导致当用户的游戏数量变多时，首页选择、分组归类、素材管理、搜索过滤都会明显掉队，整体更像“容器管理器 + 程序快捷入口”，而不是“游戏库”。

这份 PRD 的目标不是简单模仿 Lutris 的视觉样式，而是补齐 Lutris 在“海量游戏管理”上的关键产品能力，让 Whisky 能够：

- 在首页直接按游戏选取与启动，而不是先找 Bottle 再找程序。
- 支持游戏分组/分类，适合管理几十到几百个游戏。
- 为每个游戏独立管理三种素材：`cover`、`banner`、`icon`。
- 在大量游戏场景下仍然具备清晰、快速、稳定的查找和整理体验。

## 2. 现状基线

### 2.1 当前 Whisky 已有能力

基于当前仓库代码，Whisky 已经有这些基础能力：

- 有单独的 `LibraryHomeView` 首页和 `LibraryEntry` 卡片模型。
- 首页支持 `icon`、`banner`、`cover` 三种展示形态。
- 可以从程序图标中提取 icon 作为展示素材。
- 支持 pin 程序，并在首页展示最多 4 个 quick launch。
- 支持按 Bottle 名称搜索。

### 2.2 当前 Whisky 的结构性限制

当前实现的核心限制如下：

| 维度 | 当前实现 | 问题 |
| --- | --- | --- |
| 核心实体 | 以 `Bottle` 为首页主对象，`Program` 为子对象 | 用户想管理的是游戏，不是 Wine 容器 |
| 分组能力 | 无游戏分组/分类模型 | 无法对大量游戏做长期整理 |
| 素材能力 | `cover/banner/icon` 仅是运行时渲染模板 | 没有独立、可持久化、可替换的游戏素材资源 |
| 搜索能力 | 首页只按 Bottle 名搜索；Programs 只在单 Bottle 内搜程序名 | 不能跨库快速定位游戏 |
| 首页组织 | 首页展示 Bottle 卡片和 featured Bottle | 不能以“游戏库”视角浏览、筛选、排序 |
| 批量管理 | 基本没有 | 多游戏场景下操作成本高 |

### 2.3 代码证据摘要

- `Whisky/Views/Library/LibraryHomeView.swift`
  - 首页侧边栏和主内容均以 `LibraryEntry(bottle: ...)` 构建，说明首页主对象是 Bottle。
- `Whisky/Views/Library/LibraryTypes.swift`
  - `LibraryEntry` 直接持有 `Bottle`。
  - `quickLaunchPrograms` 仅取 `bottle.pinnedPrograms` 的前 4 个。
  - `artwork(for:)` 返回的是根据调色板即时生成的展示数据，而不是持久化媒体资源。
- `Whisky/Views/Library/LibraryArtworkView.swift`
  - `icon/banner/cover` 都是 UI 绘制逻辑，未见素材路径、上传、替换、下载、重置逻辑。
- `WhiskyKit/Sources/WhiskyKit/Whisky/Bottle.swift`
  - 只有 `pins` / `pinnedPrograms`，没有游戏级分类、标签、媒体资源或元数据模型。
- `WhiskyKit/Sources/WhiskyKit/Whisky/Program.swift`
  - `Program` 只持有 `url`、`settings`、`pinned`，没有游戏库层面的展示信息、分组或素材定义。
- `Whisky/Views/Programs/ProgramsView.swift`
  - 搜索范围仅限当前 Bottle 下的程序名，无法满足大库场景。

## 3. Lutris 对标结论

本次对标基于你提供的 `../lutris` 本地源码，而不是只参考公开介绍。

### 3.1 Lutris 的关键产品特征

Lutris 在多游戏管理上强的不是某一个页面，而是一整套“游戏库优先”的产品结构：

| 维度 | Lutris 能力 | 结论 |
| --- | --- | --- |
| 核心实体 | 游戏是数据库中的一等实体 | 不是容器附属物 |
| 分类系统 | 独立 categories 表 + games_categories 关联表 | 支持真正的多对多分组 |
| 虚拟分类 | 支持 `uncategorized`、`favorite`、`hidden` 等逻辑 | 方便清理和管理大库 |
| 搜索过滤 | 支持 `category`、`categorized`、`favorite`、`hidden`、`source`、`runner`、`platform`、`playtime`、`lastplayed` 等 | 适合海量游戏 |
| 素材系统 | `banner`、`coverart`、`icon` 是三套独立媒体资源 | 不只是 UI 样式 |
| 素材操作 | 支持自定义上传、下载、重置、转码 | 素材是可管理资产 |
| 视图模式 | 支持 `grid` / `list`，且可切换展示素材类型 | 大库浏览成本低 |
| 库同步 | 可同步游戏、游玩时间、分类 | 是高阶能力，不一定要第一期就做 |

### 3.2 Lutris 代码证据摘要

- `../lutris/lutris/database/categories.py`
  - 有 `categories` 和 `games_categories` 的完整读写逻辑。
  - `add_game_to_category()`、`remove_category_from_game()` 表明分类是正式数据关系。
  - `get_uncategorized_game_ids()` 明确支持“未分类”视图。
- `../lutris/lutris/search.py`
  - `GameSearch` 支持 `installed`、`hidden`、`favorite`、`categorized`、`category`、`source`、`runner`、`platform`、`playtime`、`lastplayed`、`directory` 等查询维度。
- `../lutris/lutris/services/base.py`
  - `LutrisBanner`、`LutrisCoverart`、`LutrisIcon` 是三套独立媒体定义。
  - 各自有不同的 `dest_path`、`size`、`file_patterns`。
- `../lutris/lutris/gui/config/game_info_box.py`
  - 同时暴露 `Cover`、`Banner`、`Icon` 三种自定义图片入口。
  - 支持本地选择、下载、重置、保存和转码。
- `../lutris/lutris/gui/widgets/sidebar.py`
  - 有单独的 `CategorySidebarRow`，说明分类不仅存在于数据层，也是一等导航入口。
- `../lutris/lutris/gui/lutriswindow.py`
  - 支持 `grid` / `list` 视图切换。
  - `load_icon_type()` 默认让 grid 偏向 `coverart`，list 偏向 `banner`。
- `../lutris/lutris/util/library_sync.py`
  - 可同步 playtime / lastplayed / categories，这属于后续可参考的增强方向。

## 4. 距离 Lutris 还差哪些功能

### 4.1 P0: 必须补齐

这些是本次需求范围内，Whisky 距离 Lutris 的最核心差距。

#### A. 游戏作为一等实体

当前缺口：

- 没有独立的 `Game` / `LibraryGame` 数据模型。
- 首页、搜索、精选、快速启动都依赖 Bottle 推导。

必须补齐：

- 独立游戏实体，至少包含：
  - `id`
  - `displayName`
  - `sortName`
  - `sourceBottle`
  - `launchTarget`
  - `createdAt`
  - `lastPlayedAt`
  - `playtime`
  - `isFavorite`
  - `isHidden`
  - `groupIDs`
  - `artwork`
- 游戏实体必须能独立于 Bottle 首页展示和检索。

#### B. 游戏首页而不是 Bottle 首页

当前缺口：

- 首页是 Bottle 列表，不是游戏列表。
- featured/quick launch 也是从 Bottle 推导。

必须补齐：

- 首页默认浏览对象改为“游戏”。
- 提供游戏库首页，支持：
  - 最近游玩
  - 收藏/固定
  - 分组入口
  - 未分类入口
  - 最近添加
- Bottle 退居为游戏详情中的归属信息，而不是主页主导航。

#### C. 游戏分组/分类系统

当前缺口：

- 完全没有游戏分组模型和交互。

必须补齐：

- 支持用户创建、重命名、删除分组。
- 一个游戏可属于多个分组。
- 提供系统分组：
  - 全部游戏
  - 收藏
  - 未分类
  - 已隐藏
- 支持单个游戏和批量游戏分配/移除分组。

#### D. 三种独立素材资源

当前缺口：

- `cover/banner/icon` 只是 UI 渲染形态，不是可管理资源。

必须补齐：

- 每个游戏支持三套独立素材：
  - `cover`: 竖版封面，适合 grid / shelf
  - `banner`: 横幅，适合 list / featured / hero
  - `icon`: 方形图标，适合侧栏、列表、细节页
- 素材支持：
  - 本地选择
  - 替换
  - 删除/重置
  - 默认回退
- 素材需要持久化保存，不能只靠运行时生成。

#### E. 面向海量游戏的搜索、过滤、排序

当前缺口：

- 只能按 Bottle 名搜，或在单个 Bottle 内按程序名搜。

必须补齐：

- 全库搜索游戏名。
- 过滤条件至少包括：
  - 分组
  - Bottle
  - 收藏
  - 隐藏
  - 未分类
  - 最近玩过
  - 缺少封面/横幅/icon
- 排序至少包括：
  - 名称 A-Z
  - 最近游玩
  - 最近添加
  - 游玩时长

### 4.2 P1: 强烈建议补齐

#### A. 多视图浏览

- `grid` / `list` 双视图切换。
- grid 默认突出 `cover`，list 默认突出 `banner`，细节/紧凑视图使用 `icon`。

#### B. 批量管理

- 批量加入分组。
- 批量移出分组。
- 批量收藏 / 取消收藏。
- 批量隐藏 / 取消隐藏。

#### C. 元数据编辑

- 自定义游戏显示名。
- 自定义排序名。
- 自定义副标题或备注。
- 查看与跳转所属 Bottle、启动目标。

#### D. 游戏首页模块化

- Continue Playing
- Favorites
- Recently Added
- Missing Artwork
- Uncategorized Cleanup

### 4.3 P2: Lutris 领先但可后置

- 自动拉取封面/横幅/icon 元数据。
- 与外部平台或在线库同步游戏、分类、游玩时间。
- 智能集合 / Saved Search。
- 跨来源去重与合并。
- 依据来源平台、Runner、系统兼容性自动建组。

## 5. 产品目标

### 5.1 目标

1. 让用户在首页以“游戏”为中心完成浏览、筛选、启动。
2. 让拥有 50-500 个游戏的用户仍能快速找到目标游戏。
3. 让每个游戏具备稳定可管理的 `cover/banner/icon` 资源。
4. 让用户能够长期维护自己的游戏分组体系。
5. 不破坏现有 Bottle 和 Program 的运行能力。

### 5.2 非目标

1. 第一阶段不要求做成与 Lutris 等价的在线平台同步。
2. 第一阶段不要求完整复刻 Lutris 的全部搜索语法。
3. 第一阶段不重写 Bottle 配置页和运行器能力。
4. 第一阶段不强依赖在线元数据抓取服务。

## 6. 目标用户

### 用户类型 A: 轻度用户

- 有 5-20 个游戏。
- 希望首页更像游戏库，点击即玩。

### 用户类型 B: 重度用户

- 有 50-500 个游戏。
- 需要分组、搜索、收藏、隐藏、排序。

### 用户类型 C: 整理型用户

- 对封面、横幅、图标有明确审美诉求。
- 愿意手动维护素材与分类。

## 7. 关键用户故事

1. 作为一个拥有很多 Bottle 的用户，我希望首页按游戏显示，而不是按 Bottle 显示，这样我能更快找到想玩的游戏。
2. 作为一个拥有很多游戏的用户，我希望为游戏建立分组，比如“RPG”“FPS”“已通关”“多人联机”，方便长期整理。
3. 作为一个重视视觉展示的用户，我希望每个游戏分别设置 `cover`、`banner`、`icon`，而不是只能接受系统自动生成样式。
4. 作为一个库很大的用户，我希望能快速筛出“未分类”“缺少封面”“最近玩过”“收藏”等游戏。
5. 作为一个仍然需要 Bottle 能力的用户，我希望从游戏详情里能看到归属 Bottle，并能继续进入 Bottle 级设置。

## 8. 功能需求

### 8.1 数据模型

需要新增游戏库层的数据模型，建议命名为 `LibraryGame` 或 `GameRecord`。

建议字段：

| 字段 | 说明 |
| --- | --- |
| `id` | 稳定唯一 ID |
| `displayName` | 展示名称 |
| `sortName` | 排序名称 |
| `bottleID` / `bottleURL` | 所属 Bottle |
| `programURL` | 启动目标 |
| `addedAt` | 加入库时间 |
| `lastPlayedAt` | 最近游玩时间 |
| `playtime` | 游玩时长 |
| `isFavorite` | 是否收藏 |
| `isHidden` | 是否隐藏 |
| `groupIDs` | 分组 ID 列表 |
| `coverPath` | cover 路径 |
| `bannerPath` | banner 路径 |
| `iconPath` | icon 路径 |
| `source` | 来源类型，例如 imported/manual/detected |

要求：

- 该模型独立于 `BottleSettings.pins`。
- 允许未来扩展自动同步、元数据抓取、去重合并。

### 8.2 分组模型

需要新增 `GameGroup` 模型。

建议字段：

- `id`
- `name`
- `sortOrder`
- `createdAt`
- `isSystemGroup`

要求：

- 用户自定义分组与系统分组并存。
- 游戏与分组为多对多关系。
- 系统分组至少包括：
  - `All Games`
  - `Favorites`
  - `Uncategorized`
  - `Hidden`

### 8.3 首页信息架构

建议新的首页结构：

- Sidebar
  - Home
  - All Games
  - Favorites
  - Uncategorized
  - Groups
  - Bottles
- Main Content
  - Hero / Featured
  - Recently Played
  - Group Shelves
  - All Games Grid/List
- Detail / Inspector
  - 游戏详情
  - 所属 Bottle
  - 素材管理
  - 分组管理

原则：

- 游戏是一级导航对象。
- Bottle 仍然存在，但降低为运维/兼容层对象。

### 8.4 首页交互

必须支持：

- 首页按游戏卡片浏览。
- 从首页直接启动游戏。
- 从首页进入游戏详情页。
- 从首页快速加入/移出收藏。
- 从首页快速分组。
- 支持 grid/list 切换。

推荐支持：

- 支持“继续游戏”“最近添加”“最近玩过”等模块。
- 支持主页记住上次视图模式和排序方式。

### 8.5 图片资源系统

必须支持每个游戏三种素材：

| 类型 | 用途 | 建议比例 |
| --- | --- | --- |
| `cover` | 首页格子、书架、详情页 | 3:4 |
| `banner` | 列表、hero、宽卡片 | 16:9 或近似 |
| `icon` | 列表前导、侧栏、紧凑模式 | 1:1 |

必须支持的操作：

- 选择本地图片
- 替换已有图片
- 删除并回退默认图
- 显示当前素材路径/来源

建议支持：

- 自动裁剪或转码到标准尺寸
- 素材缺失时显示占位图
- `icon` 缺失时优先回退程序提取 icon

存储要求：

- 素材必须持久化到稳定路径。
- 素材不应再只由运行时渐变和图标拼装。
- 后续要能支持远程下载和缓存。

### 8.6 搜索、过滤、排序

必须支持：

- 全库按游戏名搜索
- 按分组过滤
- 按 Bottle 过滤
- 按收藏过滤
- 按隐藏过滤
- 按未分类过滤
- 按缺失素材过滤

建议支持：

- 按最近游玩过滤
- 按游玩时长过滤
- 按来源过滤

排序至少包括：

- 名称
- 最近添加
- 最近游玩
- 游玩时长

### 8.7 批量操作

至少支持：

- 批量加入分组
- 批量移出分组
- 批量收藏
- 批量隐藏

原因：

- 当游戏数量很多时，单个逐条整理成本过高。

### 8.8 兼容与迁移

必须保证：

- 现有 Bottle 不受破坏。
- 现有 Program 仍可运行。
- 现有 pin 数据可迁移为 `favorite` 或首页快捷入口。

迁移建议：

1. 首次进入新库时扫描现有 Bottle 和 Program。
2. 为每个可识别启动目标生成 `LibraryGame`。
3. 将 `pins` 迁移为 `favorite` 或 `quick access`。
4. 如果存在重复启动目标，先保留重复项，后续再做合并策略。

## 9. 体验与性能要求

### 9.1 体验要求

- 用户在 1-2 次交互内能从首页找到最近常玩的游戏。
- 用户在 3 次交互内能完成“给游戏加分组”。
- 用户在 3 次交互内能完成“替换 cover/banner/icon”。

### 9.2 性能要求

- 100 个游戏时首页滚动、搜索、切换分组应无明显卡顿。
- 300 个游戏时搜索结果应保持即时反馈。
- 500 个游戏时仍应可以稳定浏览和筛选，不出现明显掉帧或长时间阻塞。

## 10. 验收标准

### 10.1 功能验收

1. 首页默认显示游戏而不是 Bottle。
2. 用户可以创建、重命名、删除分组。
3. 用户可以将一个游戏加入多个分组。
4. 系统可以显示“未分类”游戏集合。
5. 每个游戏都可以独立设置 `cover`、`banner`、`icon`。
6. 搜索可以跨整个游戏库生效，而不是只限当前 Bottle。
7. 可以按分组、收藏、隐藏、未分类进行过滤。
8. 可以从游戏详情跳转到所属 Bottle。
9. 旧 Bottle 和旧 Program 仍然能启动。

### 10.2 质量验收

1. 缺失素材时有稳定默认占位展示。
2. 素材替换后界面即时刷新。
3. 大库场景下搜索和切换过滤不会造成明显卡顿。
4. 新库模型不会破坏原有 Bottle 设置文件。

## 11. 分阶段建议

### Phase 1: 先做可用的游戏库

- 新增 `LibraryGame` / `GameGroup`
- 首页改为游戏视角
- 分组 CRUD
- `cover/banner/icon` 本地素材支持
- 全库搜索、分组过滤、收藏/隐藏/未分类

目标：

- 先解决“游戏太多不好管”的核心问题。

### Phase 2: 做强多游戏整理能力

- grid/list 切换
- 批量操作
- 最近游玩 / 最近添加
- 缺图筛选
- 更完整排序

目标：

- 让 100-500 游戏规模也能顺畅使用。

### Phase 3: 再追 Lutris 的高级能力

- 自动拉取元数据和素材
- 在线库同步
- Saved Search / 智能集合
- 重复项识别与合并

目标：

- 向 Lutris 的“成熟游戏库平台”靠近，但不阻塞第一阶段落地。

## 12. 最终结论

Whisky 距离 Lutris 的核心差距，不是“首页不够好看”，而是：

1. 还没有把“游戏”当成一等实体。
2. 还没有正式的分组/分类系统。
3. 还没有真正的三格式素材资源系统。
4. 还没有面向海量游戏的搜索、过滤、排序和批量整理能力。

因此，本次需求的本质不是改一页首页，而是补齐一层“游戏库系统”。

只有把这层补起来，Whisky 才能真正具备类似 Lutris 的多游戏管理体验。
