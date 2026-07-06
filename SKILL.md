---
name: reading-skill
description: Legado 阅读助手 — 通过 Legado Web API 管理用户的书架、书源、阅读记录与笔记想法。当用户询问阅读相关问题时使用。支持扩展接入其他阅读平台（如微信读书等，持续缓慢更新中）。
version: 2.5.0
tags: [reading, legado, reeden, books, notes, statistics]
---

# Legado 阅读助手

通过 Legado 内置 Web 服务的 HTTP + WebSocket API 访问用户的阅读数据。

## 连接

Legado Web 服务绑定手机的局域网 IP（非 127.0.0.1），默认端口 `1122`（HTTP）/ `1123`（WebSocket）。

**`<LEGADO_HOST>`** = `手机IP:端口`，如 `192.168.3.133:1122`。

获取方式：引导用户在 Legado → 我的 → Web 服务中开启，通知栏会显示地址。

| 场景 | LEGADO_HOST |
|------|-------------|
| 同一局域网 | 通知栏显示的局域网 IP |
| 同一设备（如 Termux） | 同上（不监听 localhost） |
| 跨网络 | Tailscale 组网后的 `100.x.x.x` IP |

连接超时排查：确认 Web 服务已开启、网络互通、防火墙放行 1122/1123。

---

## API

| 项 | 值 |
|----|-----|
| 响应格式 | `{"isSuccess": true, "errorMsg": "", "data": <any>}` |
| 认证 | 无 |
| 时间戳 | Unix **毫秒** |
| 中文参数 | `bookUrl`/`bookName`/`bookAuthor` 等需 URL 编码 |

完整数据模型见 `references/legado-api.md`。

### 数据结构注意事项

Agent 解析 API 返回的 JSON 时需注意：

- **`dur` 前缀** = 当前阅读位置（如 `durChapterIndex` = 当前章节索引，`durChapterPos` = 章节内字符偏移）
- **`toc`** = Table of Contents（目录）
- **时间戳**均为 Unix **毫秒**，展示时需转换
- **`type` 是位掩码**（非枚举）：text=8, audio=32, local=256 等，一个书可同时有多个类型
- **`origin`**：本地书为 `"loc_book"`，网络书为书源 URL
- **`originName`** 是书源名称/本地文件名，**`name`** 才是书名
- **`wordCount`** 是 `String`（如 "120万字"），非数字
- **`variable` / `readConfig`** 可能是 JSON 字符串（双重编码），需要二次解析
- **`custom*` 字段优先**：`customCoverUrl` > `coverUrl`，`customIntro` > `intro`，`customTag` > `kind`
- **书籍关联方式不一致**：BookChapter 用 `bookUrl`，ReadRecord/BookThought 用 `bookName`+`bookAuthor`
- **`readIteration`**：0=未读完, 1=读完一轮, 2=二刷, 3=二刷完, 依此类推

### 书架

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getBookshelf` | 返回 `Book[]` |
| POST | `/saveBook` | Body: Book JSON |
| POST | `/deleteBook` | Body: `{"bookUrl":"..."}` |
| POST | `/saveBookProgress` | Body: BookProgress JSON |

Book 关键字段：`name`, `author`, `bookUrl`（唯一标识）, `kind`（逗号分隔标签，可能为空）, `totalChapterNum`, `durChapterIndex`, `durChapterPos`, `durChapterTime`, `durChapterTitle`, `latestChapterTitle`, `originName`, `coverUrl`。

阅读状态：`durChapterIndex == 0 && durChapterPos == 0` → 未开始；`durChapterIndex >= totalChapterNum - 1` → 已读完；`totalChapterNum == 0` → 本地书。

### 章节与正文

| 方法 | 端点 | 参数 |
|------|------|------|
| GET | `/getChapterList` | `url` (bookUrl) |
| GET | `/refreshToc` | `url` (bookUrl) |
| GET | `/getBookContent` | `url` + `index` |

### 笔记想法

| 方法 | 端点 | 参数 |
|------|------|------|
| GET | `/getBookThoughts` | `bookName` + `bookAuthor`（必填） |
| GET | `/getThoughtsByChapter` | `bookName` + `bookAuthor` + `index` |
| POST | `/saveBookThought` | Body: BookThought（`id=0` 新建，有值更新） |
| POST | `/deleteBookThought` | Body: BookThought |

### 阅读统计

| 方法 | 端点 | 参数 | 返回 |
|------|------|------|------|
| GET | `/getReadRecords` | `searchKey`(可选) | `ReadRecordShow[]`（`bookName`, `readTime` 毫秒, `lastRead`） |
| GET | `/getReadTime` | `bookName`(可选) | 毫秒数 |
| GET | `/getDetailedReadRecords` | `bookName`(可选,模糊), `startTime`(可选,毫秒), `endTime`(可选,毫秒) | `DetailedReadRecord[]` |

大响应接口（`/getReadRecords`, `/getBookThoughts`）建议先保存到文件再处理。

### 搜索（WebSocket）

端口 `1123`（HTTP+1），协议 `ws`。

```
Send: {"key": "关键词"}
```

每个书源返回一条消息（数组），需收集所有消息按 `bookUrl` 去重合并。耗时 10-30 秒。

返回字段：`name`, `author`, `bookUrl`, `origin`, `originName`, `type`, `coverUrl`, `intro`, `latestChapterTitle`, `tocUrl`。

### 书源 / 订阅源 / 替换规则

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getBookSources` | 所有书源 |
| GET/POST | `/getBookSource` `/saveBookSource` `/saveBookSources` `/deleteBookSources` | 书源 CRUD |
| GET | `/getRssSources` | 所有订阅源 |
| GET/POST | `/getRssSource` `/saveRssSource` `/saveRssSources` `/deleteRssSources` | 订阅源 CRUD |
| GET | `/getReplaceRules` | 所有替换/高亮规则 |
| POST | `/saveReplaceRule` `/deleteReplaceRule` `/testReplaceRule` | 替换规则操作 |

**ReplaceRule 字段说明（含高亮扩展）**：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `id` | Long | 系统生成 | 唯一 ID（修改时必填，新建时留空或设为 0） |
| `name` | String | 必填 | 规则名称 |
| `group` | String? | null | 分组名 |
| `pattern` | String | 必填 | 匹配模式（普通字符串或正则） |
| `replacement` | String | 必填 | 替换内容（净化规则留空=""表示删除；高亮规则填 HTML 格式） |
| `isRegex` | Boolean | false | 是否使用正则表达式 |
| `isHighlight` | Boolean | false | 是否为高亮规则（用 HTML 渲染替换内容，需 `isRegex=true`） |
| `scope` | String? | "" | 作用书源 URL，空=全部书籍 |
| `scopeTitle` | Boolean | false | 是否作用于章节标题 |
| `scopeContent` | Boolean | true | 是否作用于正文 |
| `excludeScope` | String? | null | 排除范围（书源 URL，逗号分隔） |
| `isEnabled` | Boolean | true | 是否启用 |
| `order` | Int | 0 | 执行顺序，数字越小越先执行 |

**高亮规则 replacement 格式（`isHighlight=true` 时）**：

- 捕获组引用：`$0`（整个匹配）、`$1`（第1个括号）、`$N`（第N个括号）
- 支持 HTML 标签：`<b>`加粗、`<i>`斜体、`<u>`下划线、`<font color="#D32F2F">`颜色、`<big>`/`<small>`字号
- 示例：`pattern="\"([^\"]+)\""` + `replacement="<b><font color=\"#D32F2F\">$1</font></b>"` + `isRegex=true` + `isHighlight=true`
  → 将书中所有双引号内文字加粗并染红，引号本身被丢弃

### 主题配色管理

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getThemeConfigs` | 获取所有已保存的主题配色列表 |
| POST | `/saveThemeConfig` | 新建或覆盖主题（同名则覆盖）。Body: `ThemeConfig.Config` JSON |
| POST | `/deleteThemeConfig` | 删除指定主题。Body: `{"themeName":"主题名"}` |
| POST | `/applyThemeConfig` | 应用指定主题，触发 App UI 刷新。Body: `{"themeName":"主题名"}` |

**ThemeConfig.Config 字段**：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `themeName` | String | 必填 | 主题名称（同名则覆盖） |
| `isNightTheme` | Boolean | false | `true`=夜间，`false`=日间 |
| `primaryColor` | String | 必填 | 主色，16进制如 `"#607D8B"` |
| `accentColor` | String | 必填 | 强调色，16进制如 `"#BF360C"` |
| `backgroundColor` | String | 必填 | 背景色（夜间模式必须为深色） |
| `bottomBackground` | String | 必填 | 底部栏背景色 |
| `cardBackground` | String? | `"#F3EDF7"` | 卡片背景色 |
| `cardBackgroundAlpha` | Int | 100 | 卡片透明度 0-100 |
| `transparentNavBar` | Boolean | 必填 | 是否透明导航栏 |
| `backgroundImgPath` | String? | null | 背景图路径（null=纯色；http=网络图；本地路径） |
| `backgroundImgBlur` | Int | 必填 | 背景图模糊强度 0-25（0=不模糊） |

> ⚠️ 修改主题后需调用 `/applyThemeConfig` 才会生效；`/applyThemeConfig` 会触发 App 内所有界面刷新。

### 其他

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/cover?path=` | 封面图 |
| GET | `/image?url=&path=&width=640` | 正文图 |
| GET/POST | `/getReadConfig` `/saveReadConfig` | Web 阅读配置 |
| POST | `/addLocalBook` | multipart: `fileName` + `fileData` |

### WebSocket 调试

| 端点 | 发送 | 接收 |
|------|------|------|
| `/bookSourceDebug` | `{"tag":"书源URL","key":"关键词"}` | 调试日志 |
| `/rssSourceDebug` | `{"tag":"订阅源URL"}` | 调试日志 |

---

## 阅读画像与推荐

### 画像分析

数据源：`/getBookshelf` + `/getReadRecords`。

| 维度 | 提取方式 |
|------|----------|
| 偏好标签 | 统计 `Book.kind`（逗号分隔，可能为空）的词频 |
| 偏好作者 | 按 `readTime` 加权统计 `Book.author` |
| 阅读状态 | 按 `durChapterIndex`/`totalChapterNum` 分为在读/已读完/未开始 |
| 阅读时长 Top | `ReadRecordShow` 按 `readTime` 降序 |

### 相似书推荐

策略：
1. 用画像 Top 标签 + Top 作者名作为关键词，调 WebSocket `/searchBook`
2. 用户指定某本书时，用该书的标签/作者搜索
3. 按 `bookUrl` 排除书架已有书籍
4. 用户可指定排除某类标签或加入书架（`/saveBook`）

---

## 跨平台数据整合

本 Skill 以 Legado 为核心，但 Agent 应具备整合多个阅读平台数据的能力。

### 接入方式

不同平台的接入方式不同，Agent 应根据可用数据源选择合适的方式：

| 方式 | 说明 | 示例 |
|------|------|------|
| 远程 API | 平台提供可调用的云端接口 | 微信读书 |
| 本地 Web 服务 | 平台在本地暴露 HTTP 接口 | Legado |
| 本地数据解析 | 解析用户导出的备份/数据库文件 | Reeden、Anxreader、Legado 备份等 |

对于本地数据解析方式，用户提供备份文件后，Agent 需自行分析数据格式（通常是 SQLite、JSON 或 XML）并提取书架、笔记、阅读记录等信息。

### 已接入平台

| 平台 | 技能 | 安装方式 | 接入方式 |
|------|------|----------|----------|
| 微信读书 | `WeChatReading` | `npx skills add Tencent/WeChatReading -g` | 远程 API，需 `WEREAD_API_KEY` |
| Legado | 本 Skill | — | 本地 Web 服务 |
| Reeden | 本 Skill | — | 本地数据解析（全量备份） |

> **关于微信读书 Skill 的放置建议**：安装后建议将 `WeChatReading` skill 放入 `references/` 目录而非与本 Skill 同级。部分 Agent 会按目录顺序加载 Skill，若同级放置可能优先读取 weread-skill 而忽略本 Skill 的主逻辑，导致行为异常。

### 前置确认

在进行跨平台数据整合之前，Agent **必须先确认**：
1. 用户是否已配置并连接了其他阅读平台（如微信读书 API Key、其他本地阅读器等）
2. 各平台的连接状态是否正常

若用户仅接入了单一平台，则直接使用该平台的数据即可，无需进行跨平台整合。

### 整合原则

不同平台的数据结构差异较大（字段名、时间戳单位、ID 体系），Agent 整合时遵循：

1. **书名+作者去重**：跨平台识别同一本书的唯一依据，不依赖平台内部 ID
2. **字段映射**：各平台字段名不同（如 Legado 的 `durChapterIndex` vs 其他平台的 `progress`），Agent 需理解语义并映射
3. **单位归一**：时间戳和时长单位各平台不同（Legado 用毫秒，其他可能用秒），展示前统一转换
4. **来源标注**：合并展示时标注数据来自哪个平台

### 整合场景

- **统一书架**："我所有的书" → 并行获取各平台数据，合并去重，标注来源
- **跨平台阅读画像**："我的阅读偏好" → 合并所有平台的阅读记录和标签，统一分析
- **笔记汇总**："我在三体这本书的笔记" → 从各平台获取同一本书的笔记，合并展示
- **时长统计**："我一共读了多久" → 汇总各平台的阅读时长（注意单位转换）

### 接入新平台

新平台的接入文档放在 `references/` 目录下，按以下方式之一接入：
- **API 方式**：记录端点、参数、返回格式
- **数据解析方式**：记录备份文件格式、关键表/字段结构、解析方法

---

## 资源

- Legado: https://github.com/Jingshiro/legado
- API 文档: `references/legado-api.md`
- Reeden 备份数据解析: `references/reeden-backup.md`
