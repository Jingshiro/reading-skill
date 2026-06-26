# Reeden 备份数据解析参考

Reeden 全量备份的数据格式文档，供 legado-reading 技能通过「本地数据解析」方式接入时参考。

## 概述

备份格式 `reeden_full_backup`，版本 `1`。结构：

| 路径 | 说明 | 必需 |
|------|------|:----:|
| `manifest.json` | 备份元信息 | ✓ |
| `metadata.zip` | 核心数据（书架、笔记、阅读记录、标签等） | ✓ |
| `books/` | 书籍内容 | 按需 |
| `fonts/`、`cover_gallery/`、`reader_bg_images/` 等 | UI 资源 | ✗ |

---

## 获取备份数据

### 方式一：用户手动导出（推荐）

**适用场景**：用户手机未 Root，或 Agent 无法直接访问设备文件系统。

**Agent 引导话术**：

```
请帮我导出 Reeden 的阅读数据，步骤如下：

1. 打开 Reeden 应用
2. 点击底部最右边的「我的」标签
3. 在页面中找到「同步」选项，点击进入
4. 点击「备份」
5. 点击「导出备份文件」
6. 选择保存位置，建议保存到「Reeden」文件夹（路径：/storage/emulated/0/Reeden/）
7. 等待导出完成后，告诉我文件的完整路径

如果你只需要查看阅读统计和笔记，可以只导出 metadata（通常只有几 MB），不用导出全部内容，节省时间。
```

**按需导出建议**：

| 内容 | 是否需要 | 说明 |
|------|:--------:|------|
| metadata | ✅ 必需 | 书架、笔记、阅读记录、标签等核心数据 |
| books | ⚠️ 按需 | 仅当需要分析正文内容或章节结构时 |
| cover_gallery | ⚠️ 按需 | 仅当需要展示封面时 |
| fonts / background_images / custom_splash 等 | ❌ 不需要 | 纯 UI 资源，与阅读数据无关 |

**⚠️ Agent 注意事项**：
- Reeden 界面可能因版本不同略有差异，如果用户找不到选项，请让用户尝试在「我的」页面搜索「同步」或「备份」
- 导出的文件是 `.zip` 格式，文件名通常为`reeden_full_backup`
- 用户完成后，请立即读取文件并验证是否包含 `manifest.json`

---

### 方式二：同设备直接访问

**前提条件**：Agent 与 Reeden 在同一设备上（如 Termux 环境）。

**Agent 引导话术**：

```
请帮我查看 Reeden 的数据存储路径，步骤如下：

1. 打开 Reeden 应用
2. 点击底部最右边的「我的」标签
3. 找到「存储管理」选项，点击进入
4. 查看「数据目录」显示的路径（通常是 /data/user/0/app.reeden/files）
5. 告诉我这个路径
```

**⚠️ Android 权限限制**：

| 方案 | 能否访问 `/data/data/` 私有目录 |
|------|:-----:|
| 普通应用 / 文件管理器 | ❌ 无法访问 |
| Shizuku / ADB shell | ❌ 无法访问（仅限系统 API） |
| Root (Magisk/KernelSU) | ✅ 可以访问 |

**Agent 需告知用户**：

- 如果设备 **未 root**：
  ```
  抱歉，由于 Android 系统的安全限制，我无法直接读取 Reeden 的内部数据。
  请改用以下方式之一：
  1. 手动导出备份文件（推荐）
  2. 通过云端同步拉取（需要你已配置 WebDAV 或 S3）
  ```

- 如果设备 **已 root**：
  ```
  检测到你的设备已 Root，我将直接读取 Reeden 的数据目录。
  注意：如果 Reeden 正在运行，部分文件可能被锁定，建议先关闭 Reeden。
  ```

---

### 方式三：云端同步拉取

**前提条件**：用户已在 Reeden 中配置 WebDAV 或 S3 同步。

**Agent 引导话术**：

```
请问你是否在 Reeden 中配置了云端同步（WebDAV 或 S3）？

如果已配置，请提供以下信息：

**S3 协议**（如 Cloudflare R2、MinIO、AWS S3 等）：
- Endpoint（服务端点）
- Access Key（访问密钥 ID）
- Secret Key（访问密钥）
- Bucket（存储桶名称）
- Region（区域，可选，默认 auto）
- Folder（备份文件夹，可选）

**WebDAV 协议**（如坚果云、NextCloud 等）：
- Server（服务器地址）
- Username（用户名）
- Password（密码或应用专用密码）
- Folder（备份文件夹，可选）
```

**⚠️ 重要：metadata 文件格式**

云端同步目录中的 `metadata` 文件**没有 `.zip` 扩展名**，但实际上是 **ZIP 格式**（文件头为 `PK`）。

```
云端目录结构示例：
├── metadata          ← 实际是 ZIP 格式（无扩展名）
├── covers
├── indexes
├── books/
├── cover_gallery/
└── ...
```

Agent 下载后需按 ZIP 格式解压处理，不要被文件名误导！

**按需拉取建议**：

| 优先级 | 路径 | 说明 |
|--------|------|------|
| 必需 | `metadata` | 核心数据（书架、笔记、阅读记录、标签等） |
| 按需 | `books/<HASH>/` | 需要正文内容或章节结构时拉取 |
| 可选 | `cover_gallery/`、`thumbs/` | 需要展示封面时拉取 |
| 跳过 | `fonts/`、`reader_bg_images/` 等 | 纯 UI 资源，无需拉取 |

---

## 解析流程

### 本地 ZIP 文件解析

本地导出的备份文件是标准 ZIP 格式，Agent 可使用 Python 的 `zipfile` 模块解析：

```python
import zipfile
import json
import io

def parse_reeden_backup(zip_path):
    """解析 Reeden 本地备份 ZIP 文件"""
    zip_file = zipfile.ZipFile(zip_path)
    
    # 读取 manifest.json
    with zip_file.open('manifest.json') as f:
        manifest = json.loads(f.read())
    print(f"备份格式: {manifest['format']}, 版本: {manifest['version']}")
    print(f"创建时间: {manifest['createdAt']}")
    print(f"书籍数量: {manifest['fileCounts'].get('books', 0)}")
    
    # 解压 metadata.zip
    with zip_file.open('metadata.zip') as f:
        metadata_data = f.read()
    metadata_zip = zipfile.ZipFile(io.BytesIO(metadata_data))
    
    # 解析核心数据表
    result = {'manifest': manifest}
    
    # 读取书籍列表
    if 'book.json' in metadata_zip.namelist():
        with metadata_zip.open('book.json') as f:
            result['books'] = json.loads(f.read())
        print(f"书架: {len(result['books'])} 本")
    
    # 读取笔记
    if 'book_notes.json' in metadata_zip.namelist():
        with metadata_zip.open('book_notes.json') as f:
            result['notes'] = json.loads(f.read())
        print(f"笔记: {len(result['notes'])} 条")
    
    # 读取阅读记录
    if 'read_record.json' in metadata_zip.namelist():
        with metadata_zip.open('read_record.json') as f:
            result['read_records'] = json.loads(f.read())
        print(f"阅读记录: {len(result['read_records'])} 天")
    
    # 读取标签
    if 'book_tag.json' in metadata_zip.namelist():
        with metadata_zip.open('book_tag.json') as f:
            result['tags'] = json.loads(f.read())
        print(f"标签: {len(result['tags'])} 个")
    
    return result

# 使用示例
data = parse_reeden_backup('/storage/emulated/0/Reeden/reeden_full_backup.zip')
```

### S3/WebDAV 云端数据解析

云端同步目录中的 `metadata` 文件没有 `.zip` 扩展名，但实际上是 ZIP 格式：

```python
import boto3
import zipfile
import json
import io
from botocore.config import Config

def parse_reeden_s3(endpoint, access_key, secret_key, bucket, folder='Reeden'):
    """从 S3 兼容存储解析 Reeden 数据"""
    s3 = boto3.client(
        's3',
        endpoint_url=endpoint,
        aws_access_key_id=access_key,
        aws_secret_access_key=secret_key,
        config=Config(signature_version='s3v4')
    )
    
    # 下载 metadata（无扩展名，实际是 ZIP）
    key = f'{folder}/metadata'
    response = s3.get_object(Bucket=bucket, Key=key)
    data = response['Body'].read()
    
    # 解压 ZIP
    metadata_zip = zipfile.ZipFile(io.BytesIO(data))
    
    # 解析 book.json
    with metadata_zip.open('book.json') as f:
        books = json.loads(f.read())
    
    print(f"从 S3 获取到 {len(books)} 本书")
    return books

# 使用示例（Cloudflare R2）
books = parse_reeden_s3(
    endpoint='https://xxx.r2.cloudflarestorage.com/',
    access_key='YOUR_ACCESS_KEY',
    secret_key='YOUR_SECRET_KEY',
    bucket='reeden'
)
```

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

## 核心数据表

### book — 书籍库

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | 书籍哈希 ID（MD5） |
| `title` | String | 书名 |
| `author` | String | 作者 |
| `url` | String | 书文件路径 |
| `description` | String | 简介 |
| `type` | String | 格式：`epub`、`txt`、`pdf` |
| `section_num` | Int | 章节数 |
| `word_count` | Int | 总字数 |
| `read_progress` | Int | 阅读进度（0–10000，除以 100 得百分比） |
| `last_read_time` | String | 最后阅读时间（ISO 8601） |
| `read_finish_time` | String | 读完时间 |
| `rating` | Int | 评分（0–50，10=1星，50=5星） |
| `category_id` | String | 分类 UUID |
| `create_at` | String | 创建时间 |
| `update_at` | String | 更新时间 |

**阅读状态判断**：
- `read_progress == 0` → 未开始
- `read_progress >= 10000` 或 `read_finish_time` 非空 → 已读完
- 其他 → 在读

### book_notes — 划线与笔记

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | UUID |
| `book_id` | String | 关联书籍 ID |
| `section_index` | Int | 章节索引 |
| `section_name` | String | 章节标题 |
| `paragraph_index` | Int | 段落索引 |
| `start_index` / `end_index` | Int | 字符偏移 |
| `description` | String | 划线文本（原文摘录） |
| `comment` | String | 用户批注/想法 |
| `type` | String | 类型（`underline`） |
| `create_at` | String | 创建时间 |

### read_record — 每日阅读记录

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | UUID |
| `book_id` | String | 关联书籍 ID |
| `read_seconds` | Int | 阅读时长（秒） |
| `word_count` | Int | 阅读字数 |
| `date` | String | 日期（`YYYY-MM-DD`） |

### book_tag — 标签

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | UUID |
| `name` | String | 标签名 |
| `icon` | String | 图标（emoji） |
| `color` | String | 颜色（十六进制） |

系统内置标签：`all`、`unread`、`reading`、`to_be_read`、`read_finished`、`abandoned` 等。其余为用户自定义标签。

### book_tag_relation — 书籍-标签关联

字段：`id`、`book_id`、`tag_id`

---

## 书籍目录结构

每本书位于 `books/<HASH>/`，包含：

| 文件 | 说明 |
|------|------|
| `<HASH>`（无扩展名） | 书文件本体（epub/txt/pdf 二进制） |
| `toc.json` | 目录 |
| `sections.json` | 章节元数据 |
| `book_config.json` | 单书阅读配置与进度历史 |
| `fonts/` | 嵌入字体 |
| `images/` | 嵌入图片 |

---

## 解析流程总结

1. 读取 `manifest.json` 确认格式版本
2. 解压 `metadata.zip`（本地导出）或下载并解压（云端拉取）
3. 读取 `book.json` 获取书籍列表
4. 按需读取其他表（`book_notes.json`、`read_record.json`、`book_tag.json` 等）
5. 对需要正文的场景，从 `books/<HASH>/` 读取书文件
6. 按「字段映射」表转换为通用阅读数据模型
7. 标注数据来源为 `reeden`
