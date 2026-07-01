---
name: legado-reading
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
| GET | `/getReplaceRules` | 所有替换规则 |
| POST | `/saveReplaceRule` `/deleteReplaceRule` `/testReplaceRule` | 替换规则操作 |

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
