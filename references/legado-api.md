# Legado Web Service API Reference

> Legado Web 服务的完整 API 文档。供 legado-reading 技能调用时参考。

## Overview

| 服务 | 默认端口 | 说明 |
|------|----------|------|
| HTTP | 1122 | RESTful API + 静态页面 |
| WebSocket | 1123 | 实时搜索和调试 |

## 响应格式

```json
{"isSuccess": true, "errorMsg": "", "data": <any>}
```

## 数据模型

### Book
- `name`, `author`, `bookUrl`（唯一标识）, `tocUrl`, `origin`, `originName`
- `type`: 0=文本, 1=音频, 2=图片, 3=文件
- `group`: 分组索引
- `kind`: 分类标签（逗号分隔）, `wordCount`, `intro`
- `totalChapterNum`, `durChapterIndex`, `durChapterTitle`, `durChapterPos`, `durChapterTime`
- `latestChapterTitle`, `latestChapterTime`, `lastCheckTime`, `lastCheckCount`
- `coverUrl`, `customCoverUrl`, `addTime`, `order`, `originOrder`, `syncTime`
- `canUpdate`, `readConfig`, `variable`

### BookChapter
- `url`, `title`, `bookUrl`, `index`, `baseUrl`
- `isVolume`, `isVip`, `isPay`
- `resourceUrl`, `tag`, `start`, `end`

### BookProgress
- `name`, `author`, `durChapterIndex`, `durChapterPos`, `durChapterTime`, `durChapterTitle`

### BookThought
- `id`: Long（0=新建，有值=更新）
- `bookName`, `bookAuthor`: String
- `chapterIndex`: Int（章节序号，0-based）
- `chapterPos`: Int（想法在章节中的位置）
- `chapterName`: String
- `selectedText`: String（选中的原文）
- `textHash`: String（选中文本的哈希值）
- `thought`: String（想法内容）
- `createTime`, `updateTime`: Long（Unix 毫秒）
- `underlineStyle`: Int（下划线样式，默认 0）
- `underlineWeight`: Float（下划线粗细，默认 2.5）
- `underlineColor`: String（下划线颜色，默认空）

### ReadRecordShow
- `bookName`, `readTime`（毫秒）, `lastRead`

### DetailedReadRecord
- `id`, `bookName`, `startTime`, `endTime`, `readIteration`

### BookSource
- `bookSourceUrl`, `bookSourceName`, `bookSourceGroup`, `bookSourceType`
- `enabled`, `enabledExplore`, `customOrder`, `weight`
- `searchUrl`, `exploreUrl`, `loginUrl`
- `ruleSearch`, `ruleBookInfo`, `ruleToc`, `ruleContent`

### RssSource
- `sourceUrl`, `sourceName`, `sourceIcon`, `sourceGroup`
- `enabled`, `singleUrl`, `articleStyle`, `customOrder`
- `ruleArticles`, `ruleNextPage`, `ruleTitle`, `ruleContent`

### ReplaceRule
- `id`: Long（修改时必填，新建时留空或设为 0）
- `name`: String（规则名称）
- `group`: String?（分组名）
- `pattern`: String（匹配模式）
- `replacement`: String（替换内容；高亮规则用 HTML 格式）
- `isRegex`: Boolean（是否正则，默认 false）
- `isHighlight`: Boolean（是否高亮规则，默认 false；需配合 `isRegex=true`）
- `scope`: String?（作用书源 URL，空=全部）
- `scopeTitle`: Boolean（是否作用于标题，默认 false）
- `scopeContent`: Boolean（是否作用于正文，默认 true）
- `excludeScope`: String?（排除范围，逗号分隔书源 URL）
- `isEnabled`: Boolean（是否启用）
- `order`: Int（执行顺序，越小越先）

**高亮规则 replacement 格式**（`isHighlight=true` 时）：
- 捕获组：`$0`整个匹配、`$1`第1括号、`$N`第N括号
- HTML 标签：`<b>`加粗、`<i>`斜体、`<u>`下划线、`<font color="#颜色">`颜色、`<big>`/`<small>`字号
- 示例：`<b><font color="#D32F2F">$1</font></b>`

### ThemeConfig.Config
- `themeName`: String（必填，同名则覆盖）
- `isNightTheme`: Boolean（true=夜间，false=日间）
- `primaryColor`: String（主色，16进制如 `"#607D8B"`）
- `accentColor`: String（强调色）
- `backgroundColor`: String（背景色，夜间模式必须为深色）
- `bottomBackground`: String（底部栏背景色）
- `cardBackground`: String?（卡片背景色，默认 `"#F3EDF7"`）
- `cardBackgroundAlpha`: Int（卡片透明度 0-100，默认 100）
- `transparentNavBar`: Boolean（透明导航栏）
- `backgroundImgPath`: String?（背景图路径；null=纯色；http=网络图；本地路径）
- `backgroundImgBlur`: Int（模糊强度 0-25）

## HTTP API

### Bookshelf
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getBookshelf` | 书架所有书籍 |
| POST | `/saveBook` | 保存/添加书籍 |
| POST | `/deleteBook` | 删除书籍 |
| POST | `/saveBookProgress` | 保存阅读进度 |

### Chapters & Content
| 方法 | 端点 | 参数 |
|------|------|------|
| GET | `/getChapterList` | `url` (bookUrl, required) |
| GET | `/refreshToc` | `url` (bookUrl, required) |
| GET | `/getBookContent` | `url` + `index` (required) |
| GET | `/cover` | `path` (required) |
| GET | `/image` | `url` + `path` + `width`(640) |

### Book Sources
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getBookSources` | 所有书源 |
| GET | `/getBookSource` | 指定书源（`url`） |
| POST | `/saveBookSource` | 保存单个 |
| POST | `/saveBookSources` | 批量保存 |
| POST | `/deleteBookSources` | 批量删除 |

### RSS Sources
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getRssSources` | 所有订阅源 |
| GET | `/getRssSource` | 指定订阅源 |
| POST | `/saveRssSource` | 保存单个 |
| POST | `/saveRssSources` | 批量保存 |
| POST | `/deleteRssSources` | 批量删除 |

### Replace Rules & Highlight Rules
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getReplaceRules` | 所有替换/高亮规则 |
| POST | `/saveReplaceRule` | 保存规则（完整 ReplaceRule JSON） |
| POST | `/deleteReplaceRule` | 删除规则（需 `id`） |
| POST | `/testReplaceRule` | 测试规则（Body: `{rule, text}`） |

### Theme Config（主题配色）
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getThemeConfigs` | 所有已保存的主题配色列表 |
| POST | `/saveThemeConfig` | 新建或覆盖主题（Body: ThemeConfig.Config JSON） |
| POST | `/deleteThemeConfig` | 删除主题（Body: `{"themeName":"..."}`） |
| POST | `/applyThemeConfig` | 应用主题并触发 UI 刷新（Body: `{"themeName":"..."}`） |

### Book Thoughts（读书想法）
| 方法 | 端点 | 参数 |
|------|------|------|
| GET | `/getBookThoughts` | `bookName` + `bookAuthor` |
| GET | `/getThoughtsByChapter` | `bookName` + `bookAuthor` + `index` |
| POST | `/saveBookThought` | Body: BookThought（id=0 新建） |
| POST | `/deleteBookThought` | Body: BookThought |

### Read Records（阅读记录）
| 方法 | 端点 | 参数 | 返回 |
|------|------|------|------|
| GET | `/getReadRecords` | `searchKey`(optional) | `ReadRecordShow[]` |
| GET | `/getReadTime` | `bookName`(optional) | `number`（毫秒） |
| GET | `/getDetailedReadRecords` | `bookName`(optional) | `DetailedReadRecord[]` |

### Reading Config
| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/getReadConfig` | Web 阅读配置 |
| POST | `/saveReadConfig` | 保存配置 |

### Local Book Upload
| 方法 | 端点 | 说明 |
|------|------|------|
| POST | `/addLocalBook` | multipart: `fileName` + `fileData` |

## WebSocket API

端口: HTTP + 1（默认 1123）

### `/searchBook`
```
Send: {"key": "关键词"}
Receive: [{name, author, bookUrl, origin, originName, type, coverUrl, intro, latestChapterTitle, tocUrl}]
```
每个书源返回一条消息（数组），需收集所有消息合并去重。搜索完成后连接自动关闭。耗时取决于书源数量，可能需要 10-30 秒。

### `/bookSourceDebug`
```
Send: {"tag": "书源URL", "key": "搜索关键词"}
Receive: 调试日志文本
```

### `/rssSourceDebug`
```
Send: {"tag": "订阅源URL"}
Receive: 调试日志文本
```

## 常用工作流

1. **获取书架并阅读**: `/getBookshelf` → `/getChapterList` → `/getBookContent`
2. **搜索并添加**: WebSocket `/searchBook` → `/saveBook`
3. **管理书源**: `/getBookSources` → `/saveBookSource` / `/deleteBookSources`
4. **同步进度**: `/saveBookProgress` → `/getBookshelf`

## CORS

所有 API 支持跨域，OPTIONS 返回 `Access-Control-Allow-Origin: *`

## Static Pages

| Path | 说明 |
|------|------|
| `/` | 导航首页 |
| `/vue/index.html` | Vue SPA 书架界面 |
| `/vue/index.html#/chapter` | 章节阅读 |
| `/vue/index.html#/bookSource` | 书源编辑 |
| `/uploadBook/index.html` | WiFi 传书 |
