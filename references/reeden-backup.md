# Reeden 备份数据解析参考

> Reeden 全量备份的数据格式文档。供 legado-reading 技能通过「本地数据解析」方式接入时参考。

## 概述

Reeden 备份格式为 `reeden_full_backup`，当前版本 `1`。备份是一个包含 `manifest.json`、`metadata.zip`（SQLite 导出的 JSON 表）和多个资源目录的文件夹（或压缩包）。

| 顶层路径 | 说明 | 阅读数据 |
|----------|------|:--------:|
| `manifest.json` | 备份元信息：格式、版本、创建时间、数据类型及文件计数 | ✓ |
| `metadata.zip` | 数据库导出，内含全部核心 JSON 表（一个 JSON 文件对应一张表） | ✓ 必需 |
| `books/` | 按书籍哈希 ID 组织的子目录，每本书包含书文件、目录、配置、统计等 | ✓ 按需 |
| `fonts/` | 全局字体文件 | ✗ |
| `cover_gallery/` | 封面图 | 可选 |
| `thumbs/` | 封面缩略图 | 可选 |
| `reader_bg_images/` | 阅读背景图 | ✗ |
| `custom_splash/` | 启动动画（Lottie JSON） | ✗ |
| `bottom_navbar_pack/` | 底部导航栏图标 | ✗ |

---

## 获取备份数据

Agent 可通过以下三种方式获取用户的 Reeden 备份数据：

### 方式一：用户手动导出备份文件

1. 打开 Reeden → **我的** → **同步** → **备份**
2. 选择**分享备份文件**，勾选**全部文件**
3. 将生成的备份文件发送给 Agent

Agent 收到后解压即可使用。

### 方式二：同设备直接访问数据目录

当 Agent 与 Reeden 处于同一设备时（如 Termux、本地脚本环境），可直接读取 Reeden 的数据目录：

1. 打开 Reeden → **我的** → **存储管理**，查看「数据目录」路径
2. 用户将该路径告知 Agent
3. Agent 直接访问该路径下的 `metadata.zip`、`books/` 等文件

此方式无需导出备份，Agent 可获取最新的实时数据。注意 Reeden 运行时部分文件可能被锁定，建议在 Reeden 未运行时读取。

同样遵循「按需拉取」原则：仅查看统计数据时只需读取 `metadata.zip`，需要正文时再读 `books/` 目录。

### 方式三：通过云端同步拉取

Reeden 支持配置 WebDAV 或 S3 协议进行云端同步。用户提供以下凭据后，Agent 可从云端拉取最新备份：

**S3 协议（如 MinIO、阿里云 OSS、AWS S3）：**

| 参数 | 说明 |
|------|------|
| Endpoint | 服务端点 URL |
| Access Key | 访问密钥 ID |
| Secret Key | 访问密钥 |
| Bucket | 存储桶名称 |
| Region | 区域（可选） |
| Folder | 备份文件夹路径（可选） |

**WebDAV 协议（如坚果云）：**

| 参数 | 说明 |
|------|------|
| Server | WebDAV 服务器地址 |
| Username | 用户名 |
| Password | 密码或应用专用密码 |
| Folder | 备份文件夹路径（可选） |

Agent 使用对应协议客户端（如 `aws s3 cp`、`rclone`、`curl`）下载备份文件后解压处理。

#### 按需拉取

云端备份通常包含大量与阅读数据无关的资源文件，Agent 应按需拉取以节省时间和流量：

| 优先级 | 路径 | 说明 |
|--------|------|------|
| 必需 | `metadata.zip` | 包含全部核心数据（书架、笔记、阅读记录、标签、设置等） |
| 按需 | `books/<HASH>/` | 需要正文内容或章节结构时拉取 |
| 可选 | `cover_gallery/`、`thumbs/` | 需要展示封面时拉取 |
| 跳过 | `fonts/`、`reader_bg_images/`、`bottom_navbar_pack/`、`custom_splash/` | 纯 UI 资源，与阅读数据无关 |

> 如果用户仅需查看书架、笔记、阅读统计等信息，只需拉取 `metadata.zip` 一个文件即可。

---

## Manifest

`manifest.json` 描述备份的元信息：

```json
{
  "format": "reeden_full_backup",
  "version": 1,
  "createdAt": "<ISO 8601 时间戳>",
  "dataTypes": ["metadata", "themeLayoutConfig", "books", "fonts",
                "dictionaries", "backgroundImages", "customSplashAssets",
                "coverThumbs", "coverGalleryImages", "bottomNavbarAssets"],
  "fileCounts": {
    "metadata": 1, "books": 0, "fonts": 0, "dictionaries": 0,
    "backgroundImages": 0, "customSplashAssets": 0, "coverThumbs": 0,
    "coverGalleryImages": 0, "bottomNavbarAssets": 0
  }
}
```

---

## metadata.zip 内的 JSON 表

解压 `metadata.zip` 后获得一组 JSON 文件，每个文件对应一张数据库表（数量因用户数据而异）。以下为与阅读数据相关的核心表。

### book — 书籍库

字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | 书籍哈希 ID（MD5） |
| `title` | String | 书名 |
| `sync_title` | String | 同步全名（可能含出版信息） |
| `author` | String | 作者 |
| `url` | String | 书文件在备份中的路径 |
| `description` | String | 简介 |
| `cover_url` | String | 封面图哈希引用 |
| `cover_thumb` | String | 缩略图哈希引用 |
| `type` | String | 格式：`epub`、`txt`、`pdf` |
| `charset` | String | 编码（txt 用，如 `utf-8`） |
| `section_num` | Int | 章节数 |
| `word_count` | Int | 总字数 |
| `size` | Int | 文件大小（字节） |
| `category_id` | String | 分类 UUID，关联 `category.json` |
| `read_progress` | Int | 阅读进度（0–10000，除以 100 得百分比） |
| `read_round_no` | Int | 阅读轮次（重读计数） |
| `last_read_time` | String | 最后阅读时间（ISO 8601） |
| `last_read_section_index` | Int | 最后阅读章节索引 |
| `last_read_para_index` | Int | 最后阅读段落索引 |
| `last_read_element_index` | Int | 最后阅读元素索引 |
| `read_start_time` | String | 开始阅读时间 |
| `read_finish_time` | String | 读完时间 |
| `rating` | Int | 评分（0–50，10=1星，50=5星） |
| `reviews` | String | 评语 JSON 数组 |
| `is_local` | Int | 是否本地书（1=是） |
| `is_deleted` | Int | 是否已删除（软删除） |
| `pin` | Int | 是否置顶 |
| `sort_index` | Int | 排序序号 |
| `create_at` | String | 创建时间 |
| `update_at` | String | 更新时间 |

**阅读状态判断**：`read_progress == 0` → 未开始；`read_progress >= 10000` 或 `read_finish_time` 非空 → 已读完；否则 → 在读。

### book_notes — 划线与笔记

字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | UUID |
| `book_id` | String | 关联书籍 ID |
| `section_index` | Int | 章节索引 |
| `section_name` | String | 章节标题 |
| `paragraph_index` | Int | 起始段落索引 |
| `start_index` | Int | 段内起始字符偏移 |
| `end_paragraph_index` | Int | 结束段落索引 |
| `end_index` | Int | 段内结束字符偏移 |
| `description` | String | 划线文本（原文摘录） |
| `comment` | String | 用户批注/想法 |
| `type` | String | 类型（`underline`） |
| `style_type` | String | 样式：`solid`、`dashed`、`halfBackground`、`wavy` |
| `style_color_type` | String | 颜色方案（`accent`） |
| `create_at` | String | 创建时间 |
| `update_at` | String | 更新时间 |

### read_record — 每日阅读记录

字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | UUID |
| `book_id` | String | 关联书籍 ID |
| `read_seconds` | Int | 阅读时长（秒） |
| `word_count` | Int | 阅读字数 |
| `date` | String | 日期（`YYYY-MM-DD`） |
| `create_at` | String | 创建时间 |
| `update_at` | String | 更新时间 |

### read_record_hourly — 小时级阅读记录

结构同 `read_record`，按小时粒度记录。

### book_tag — 标签

字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | UUID |
| `name` | String | 标签名 |
| `icon` | String | 图标（emoji） |
| `color` | String | 颜色（十六进制） |
| `sort_index` | Int | 排序序号 |
| `private` | Int | 是否私有 |

系统内置标签：`all`、`unread`、`reading`、`to_be_read`、`read_finished`、`abandoned` 等。其余为用户自定义标签（可含 emoji 图标和颜色）。

### book_tag_relation — 书籍-标签关联

字段：`id`、`book_id`、`tag_id`。

### category — 书籍分类

层级分类，支持 `parent_id` 嵌套。字段：`id`、`name`、`parent_id`、`sort_index`。

示例结构（用户自定义，因人而异）：

```
分类A
├── 子分类A1
├── 子分类A2
└── 子分类A3
分类B
└── 子分类B1
```

### book_character — 角色

字段：`id`、`book_id`、`name`、`aliases`、`comment`、`sort_order`、`role_type`（如 `main`）。

### book_character_relation — 角色关系

字段：`id`、`source_id`、`target_id`、`relation_type`。

### settings — 应用设置

包含用户画像和阅读目标等关键信息：

| 路径 | 说明 |
|------|------|
| `profile.name` | 用户名 |
| `profile.bio` | 个性签名 |
| `reading.goal.words` | 每日字数目标 |
| `reading.goal.minutes` | 每日时长目标（分钟） |
| `reading.goal.books` | 年度书籍目标 |
| `sync_config` | 云端同步配置（WebDAV/S3 凭据可能在此） |

### highlight_rule — 高亮规则

正则匹配 + CSS 样式的自动高亮规则。字段：`id`、`name`、`pattern`、`replacement`、`style`、`enabled`。

---

## 书籍目录结构

每本书位于 `books/<HASH>/`，包含：

| 文件 | 说明 |
|------|------|
| `<HASH>`（无扩展名） | 书文件本体（epub/txt/pdf 二进制） |
| `toc.json` | 目录 |
| `sections.json` | 章节元数据 |
| `book_config.json` | 单书阅读配置与进度历史 |
| `statistics.hive` | 阅读统计（Hive 二进制格式，Flutter 专用） |
| `fonts/` | 嵌入字体 + `font_mapping.json` |
| `images/` | 嵌入图片 |
| `ai_role_memory.json` | AI 角色记忆（可选） |
| `role_config.json` | 角色配置（可选） |
| `role_annotations_<N>.json` | 章节角色标注（可选） |
| `role_result_<N>.json` | AI 分析结果（可选） |

### toc.json — 目录

```json
{
  "bookId": "HASH",
  "title": "章节标题",
  "index": 0,
  "sectionIndex": 0,
  "url": "Text/chapter.xhtml",
  "isValid": true,
  "parentIndex": null,
  "anchor": null,
  "wordCount": null
}
```

### sections.json — 章节元数据

**EPUB 格式**：

```json
{
  "uri": "Text/chapter.xhtml",
  "startOffset": null,
  "endOffset": null,
  "title": null,
  "wordCount": 1713
}
```

**TXT 格式**（基于字节偏移）：

```json
{
  "uri": null,
  "startOffset": 0,
  "endOffset": 13549,
  "title": "章节标题",
  "wordCount": 4312,
  "isRolling": false
}
```

### book_config.json — 单书配置

关键字段：

| 字段 | 说明 |
|------|------|
| `bookId` | 书籍 ID |
| `readProgressHistories` | 进度快照数组：`{timestamp, sectionIndex, paragraphIndex, elementIndex}` |
| `navigationHistories` | 导航事件数组：`{timestamp, sectionIndex, paragraphIndex, elementIndex, sourceType}` |
| `translateEnabled` | 翻译开关 |
| `selectedEngine` | 翻译引擎 |
| `sourceLanguage` / `targetLanguage` | 翻译语言对 |

---

## 字段映射（与 Legado 对照）

跨平台整合时，Reeden 字段需映射到 Legado 的通用模型：

| Legado 字段 | Reeden 来源 | 转换说明 |
|-------------|-------------|----------|
| `name` | `book.title` | — |
| `author` | `book.author` | — |
| `bookUrl` | `book.id` | Reeden 用哈希 ID 作为唯一标识 |
| `kind` | `book_tag` + `book_tag_relation` | 需查表拼接，逗号分隔 |
| `totalChapterNum` | `book.section_num` | — |
| `durChapterIndex` | `book.last_read_section_index` | — |
| `durChapterPos` | `book.last_read_para_index` | Reeden 精度到段落+元素 |
| `durChapterTime` | — | Reeden 不存储单章时长，需从 `read_record` 聚合 |
| `coverUrl` | `cover_gallery/<cover_url>` | 哈希引用拼接路径 |
| `wordCount` | `book.word_count` | — |
| `intro` | `book.description` | — |

### 笔记映射

| Legado BookThought | Reeden book_notes | 说明 |
|--------------------|-------------------|------|
| `bookName` | 通过 `book_id` 查 `book.title` | — |
| `bookAuthor` | 通过 `book_id` 查 `book.author` | — |
| `chapterIndex` | `section_index` | — |
| `selectedText` | `description` | 划线原文 |
| `thought` | `comment` | 用户批注 |
| `chapterPos` | `paragraph_index` | 近似映射 |
| `createTime` | `create_at` | ISO 8601 → Unix 毫秒 |

### 阅读时长映射

| Legado ReadRecordShow | Reeden read_record | 说明 |
|-----------------------|--------------------|------|
| `bookName` | 通过 `book_id` 查 `book.title` | — |
| `readTime` | `read_seconds * 1000` | 秒 → 毫秒 |
| `lastRead` | `date` | 日期字符串 → 时间戳 |

---

## 解析流程

1. 读取 `manifest.json` 确认格式版本
2. 解压 `metadata.zip` 到临时目录
3. 读取 `book.json` 获取书籍列表
4. 按需读取其他表（`book_notes.json`、`read_record.json`、`book_tag.json` 等）
5. 对需要正文的场景，从 `books/<HASH>/` 读取书文件（epub 可解析 XML，txt 直接读取）
6. 按「字段映射」表转换为通用阅读数据模型
7. 标注数据来源为 `reeden`
