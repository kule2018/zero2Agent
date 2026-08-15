---
name: scrape-nowcoder
description: 基于 CDP 原生 WebSocket 抓取牛客网面经文章。当用户说"抓牛客"、"爬牛客面经"、"nowcoder 抓取"、"抓取面经列表"时触发。通过 Chrome 调试端口直接连接已登录的浏览器会话，支持首页、话题、搜索分页和详情全文抓取，输出 Markdown。
---

# scrape-nowcoder：牛客面经 CDP 抓取

基于 Chrome DevTools Protocol 原生 WebSocket，零依赖。直接连接已运行的 Chrome 调试端口，复用浏览器登录态。

## 工作方式

1. 连接 Chrome 调试端口（默认 9222）
2. 如果端口不可达，自动启动独立 Chrome 实例（`~/.chrome-nowcoder`）
3. 支持首页推荐流、面经话题流和搜索结果三种分页方式
4. 在已登录的浏览器中操作，抓取完成后 Chrome 保持运行

## 前置条件

- Node.js >= 22（需要原生 WebSocket 和 fetch）
- Google Chrome（Windows 或 macOS）以调试端口运行
- 已在该 Chrome 中登录牛客网

### 首次使用

启动带调试端口的 Chrome（会自动创建 `~/.chrome-nowcoder` profile）：

```bash
node .claude/skills/scrape-nowcoder/scrape.mjs --login
```

在弹出的 Chrome 中登录牛客，之后 cookie 永久保存。后续直接抓取即可。

## 用法

```bash
node .claude/skills/scrape-nowcoder/scrape.mjs [选项]
```

### 选项

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--login` | — | 首次使用：启动 Chrome 并打开登录页 |
| `--home` | — | 抓取首页推荐流，通过连续下拉加载后续内容 |
| `--topic <url\|id>` | `818_1` | 抓取话题流；支持完整牛客 URL 或 `type` 值 |
| `--pages <n>` | 1 | 最大页数；首页模式下为连续滚动批次 |
| `--since <date>` | (空) | 标准 `type=<tab>_<category>` 话题接口仅保留该日及之后内容；连续两页全部更早时停止 |
| `--keyword <kw>` | (空) | 按关键词筛选标题（如“AI”、“大模型”） |
| `--search <query>` | (空) | 搜索模式，在搜索页按关键词抓取面经 |
| `--out <dir>` | `.claude/skills/scrape-nowcoder/nowcoder-output` | 输出目录 |
| `--port <port>` | 9222 | Chrome 调试端口 |
| `--delay <ms>` | 2000 | 请求间隔，避免触发反爬 |

### 常用示例

```bash
# 默认抓牛客面经话题 type=818_1 的第 1 页
node .claude/skills/scrape-nowcoder/scrape.mjs

# 从面经话题抓 10 页，连续两页全部早于指定日期时自动停止
node .claude/skills/scrape-nowcoder/scrape.mjs --topic "https://www.nowcoder.com/?type=818_1" --pages 10 --since "2026-08-12"

# 首页推荐流连续滚动 3 个批次，只保留 AI 相关内容
node .claude/skills/scrape-nowcoder/scrape.mjs --home --pages 3 --keyword "AI"

# 搜索模式：搜"字节跳动 后端 面经"，抓 5 页
node .claude/skills/scrape-nowcoder/scrape.mjs --search "字节跳动 后端 面经" --pages 5

# 搜索模式：搜 Redis 八股
node .claude/skills/scrape-nowcoder/scrape.mjs --search "Redis 面经 八股" --pages 3

# 指定端口
node .claude/skills/scrape-nowcoder/scrape.mjs --port 9333
```

### 三种模式对比

| | 首页模式（`--home`） | 话题模式（默认 / `--topic`） | 搜索模式（`--search`） |
|---|---|---|---|
| 数据源 | 牛客首页推荐流 | 牛客话题流；默认 `type=818_1` 面经 | 牛客搜索页 `/search/all` |
| 筛选方式 | `--keyword` 按标题和摘要过滤 | 全量面经，可叠加 `--keyword`；标准 `type` 话题可用 `--since` | 搜索词直接匹配全站面经 |
| 翻页方式 | 连续下拉并等待懒加载 | `home/tab/content?pageNo=N`；非标准话题 URL 回退连续下拉 | 客户端点击分页按钮并确认活动页码和结果 URL 已切换 |
| 适用场景 | 发现个性化推荐内容 | 按时间遍历面经，适合增量抓取 | 精准搜索公司、岗位或主题 |

三种模式都会按文章 URL 跨页去重；页码不存在、切换失败或某一页没有新增结果时会提前停止，避免重复抓取旧页。
非标准话题 URL 会回退为连续滚动，此时无法可靠获得发布时间，传入的 `--since` 会明确标记为未应用。

## 输出结构

```
.claude/skills/scrape-nowcoder/nowcoder-output/
├── index.md                # 目录索引
├── all-in-one.md           # 全部文章合并
├── 2026-05-20-标题.md      # 单篇文章（按发布日期命名）
├── 2026-05-18-标题.md
└── ...
```

## 长期 Agent 面经档案

完整抓取结果默认仍写入 Git 忽略目录。只有用户明确要求长期保留时，才从原始结果中筛选高质量 Agent 一手面经，归档到：

```text
.claude/skills/scrape-nowcoder/nowcoder-agent-excellent-full/
```

归档时遵循以下规则：

1. 只收录真实的一手面试记录，优先保留完整面试流程、连续追问和项目深挖链路。
2. 排除重复账号稿、逐题复用稿、教程、推广、付费墙、重构题库和纯笔试。
3. 一旦收录，保留整轮面试链路：自我介绍这一流程项、Agent 与传统技术题、项目追问、原作者记录的回答和面试官反馈、算法题及反问；不能只摘 Agent 题。
4. 保持原帖顺序与表述，不补答案，不把不确定术语擅自改成另一个技术名词。
5. 每篇保留发布日期和牛客来源 URL；移除用户名、学校、作者所在地、具体个人结果、互动区和相关推荐。
6. 新增单篇文件后同步更新档案目录的 `index.md`，记录日期、面试标题、本地文件和原始链接。

单篇文件整理完成后，用维护脚本校验来源 URL 唯一性并重建索引与合集：

```bash
node .claude/skills/scrape-nowcoder/rebuild-archive.mjs
```

## 执行流程

当用户触发此 skill 时：

### 1. 确认参数

询问用户：
- 从首页、话题还是搜索抓取？（默认面经话题 `type=818_1`）
- 最大抓几页？（默认 1）
- 是否需要日期下限或关键词筛选？

### 2. 执行抓取

```bash
node .claude/skills/scrape-nowcoder/scrape.mjs --topic "818_1" --pages <n> --since "<YYYY-MM-DD>"
```

脚本自动连接 Chrome 调试端口，无需用户额外操作。

### 3. 检查输出

读取 `index.md` 汇报抓取结果。

### 4. 后续操作（可选）

抓取完成后询问用户是否需要：
- 使用 `classify-interview-questions` 将面试题分发到已有维度文章
- 筛选特定文章深入处理

## 注意事项

- Chrome 必须以 `--remote-debugging-port` 启动
- 首次需用 `--login` 在独立 Chrome 中登录牛客
- 登录后 cookie 永久保存在 `~/.chrome-nowcoder`，后续无需重复登录
- Chrome 实例保持运行，不会被脚本关闭
- 默认 2 秒间隔，不建议降低以免触发反爬
- `type=818_1` 是牛客“面经”话题流，默认全量收集，不需要先按标题关键词过滤
- 抓取原文属于临时输入，应写入 Git 忽略目录，不提交浏览器配置、Cookie 或原始抓取结果
