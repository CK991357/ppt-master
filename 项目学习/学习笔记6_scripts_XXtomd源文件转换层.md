## 脚本实现层（scripts/）第一批：源文件转换脚本详细解析

你已经完成了 `references/` 目录的学习，现在进入 `scripts/` 目录。第一批是**源文件转换**脚本，负责将各种输入格式（PDF、DOCX、网页等）统一转换为 Markdown，供后续策略师和执行师使用。

以下详细解析三个 Python 脚本：`doc_to_md.py`、`pdf_to_md.py`、`web_to_md.py`。对于 `web_to_md.cjs`（Node.js 版本），我会在最后提供一段提示词，你可以在 IDE 中让 AI 解析。

---

# 一、doc_to_md.py —— 文档转 Markdown（基于 Pandoc）

### 1. 核心定位

**作用**：利用 Pandoc 将多种文档格式（DOCX、ODT、EPUB、HTML、LaTeX、RST、Jupyter Notebook、Typst 等）转换为 Markdown，并提取嵌入的媒体文件。

**依赖**：Pandoc 必须安装（`brew install pandoc` / `sudo apt install pandoc` / 官网下载）。

### 2. 支持格式与常量

```python
SUPPORTED_FORMATS = {
    # Office
    ".docx": ("docx", "Microsoft Word"),
    ".doc": ("doc", "Microsoft Word 97-2003"),
    ".odt": ("odt", "OpenDocument Text"),
    ".rtf": ("rtf", "Rich Text Format"),
    # eBooks
    ".epub": ("epub", "EPUB"),
    # Web
    ".html": ("html", "HTML"),
    ".htm": ("html", "HTML"),
    # Academic/technical
    ".tex": ("latex", "LaTeX"),
    ".latex": ("latex", "LaTeX"),
    ".rst": ("rst", "reStructuredText"),
    ".org": ("org", "Emacs Org-mode"),
    ".ipynb": ("ipynb", "Jupyter Notebook"),
    ".typ": ("typst", "Typst"),
}

MEDIA_FORMATS = {".docx", ".odt", ".epub", ".html", ".htm"}
```

- `SUPPORTED_FORMATS`：后缀 → (Pandoc 输入格式名, 描述)
- `MEDIA_FORMATS`：这些格式可能包含嵌入的图片/媒体，需要用 `--extract-media` 提取。

### 3. 核心函数 `convert_to_markdown`

#### 3.1 参数与前置检查
- 输入文件存在性、后缀支持性。
- 确定输出路径（默认与输入同目录，后缀改为 `.md`）。
- 创建媒体提取目录：`{stem}_files`（相对路径）。

#### 3.2 构建 Pandoc 命令
```python
cmd = [
    "pandoc",
    "-f", input_format,
    "-t", "gfm",                # GitHub Flavored Markdown
    str(input_file.resolve()),
    "-o", str(out_file.resolve()),
    "--wrap", "none",           # 不自动换行
    "--strip-comments",         # 删除 HTML 注释
]
if suffix in MEDIA_FORMATS:
    cmd.extend(["--extract-media", rel_media_dir])
```
- `-t gfm`：输出 GFM 风格 Markdown（支持表格、任务列表等）。
- `--extract-media`：提取嵌入媒体到指定目录，Pandoc 会在 Markdown 中生成相对路径引用。

#### 3.3 后处理：修复媒体路径
Pandoc 有时会在提取目录内再建一个 `media/` 子目录，脚本会将其扁平化：
```python
nested_media = media_dir / "media"
if nested_media.exists():
    for f in nested_media.iterdir():
        shutil.move(str(f), str(media_dir / f.name))
    nested_media.rmdir()
    markdown_content = markdown_content.replace(f"{rel_media_dir}/media/", f"{rel_media_dir}/")
```

同时将绝对路径转换为相对路径，避免跨平台问题。

#### 3.4 将 `<img>` HTML 标签转换为 Markdown 图片语法
```python
def _img_to_md(match):
    src = match.group("src")
    alt = match.group("alt") or Path(src).stem
    return f"![{alt}]({src})"

markdown_content = re.sub(r'<img\s[^>]*?src="(?P<src>[^"]+)"...', _img_to_md, ...)
```
- 处理两种顺序（src 在前或 alt 在前）。

#### 3.5 输出统计
- 显示 Markdown 文件大小，以及提取的媒体文件数量。

### 4. 设计亮点

- **通用性**：支持 10+ 种文档格式，覆盖学术、办公、网页等场景。
- **媒体处理**：自动提取图片并修正路径，保证 Markdown 可移植。
- **GFM 输出**：表格、代码块等被良好保留。
- **错误处理**：Pandoc 未安装时给出友好提示。

---

# 二、pdf_to_md.py —— PDF 转 Markdown（基于 PyMuPDF）

### 1. 核心定位

**作用**：直接解析 PDF 文件，提取文本、表格、图片，并智能识别标题、列表、粗体/斜体等 Markdown 结构。**不依赖外部转换器**，直接操作 PDF 内部对象。

**依赖**：`PyMuPDF`（`fitz`），`pip install PyMuPDF`。

### 2. 核心设计思想

PDF 没有明确的语义结构（如 `<h1>`），只能通过**字体大小、粗体、位置**等启发式规则推断标题、正文、列表。该脚本实现了一套完整的启发式分析引擎。

### 3. 字体分析 `analyze_font_sizes`

```python
def analyze_font_sizes(doc):
    size_counter = Counter()
    for page in doc:
        for span in ...:
            size = round(span["size"], 1)
            size_counter[size] += len(text)
    sorted_sizes = sorted(size_counter.items(), key=lambda x: x[1], reverse=True)
    body_size = sorted_sizes[0][0]  # 出现次数最多的字号视为正文
    larger_sizes = [s for s in all_sizes if s > body_size + 1]
    # 最大的几个字号分别映射为 H1, H2, H3
```

- **原理**：正文通常使用频率最高的字号，标题字号更大但出现次数少。
- **返回值**：`{"body": 12, "h1": 24, "h2": 18, "h3": 14}`（单位是点）

### 4. 标题级别判断 `get_heading_level`

多重启发式：
- 字号阈值（基于分析结果）
- 文本长度（>80 字符不是标题）
- 结尾标点（不以句号、问号等结尾，但允许编号如“1. 概述”）
- 是否粗体（`flags & 16`）

严格模式（`strict=True`）会施加更多限制，避免误判。

### 5. 列表检测 `detect_list_item`

- 无序列表：`•`、`-`、`*`、`▪` 等符号开头
- 有序列表：`1.`、`1、`、`1)` 等数字开头

返回 `(is_list, list_type, content)`，其中 `list_type` 为 `'ul'` 或 `'ol'`。

### 6. 页眉页脚去除 `detect_headers_footers`

**原理**：统计所有页面的顶部和底部区域（各 15% 高度）的文本块，如果某个文本块出现在超过 60% 的扫描页面中，则视为重复噪音（页码、章节名等），在生成 Markdown 时过滤。

```python
def detect_headers_footers(doc, threshold_ratio=0.6):
    headers = []
    footers = []
    for i in pages_to_scan:
        top_rect = fitz.Rect(0, 0, rect.width, h * 0.15)
        bottom_rect = fitz.Rect(0, h * 0.85, rect.width, h)
        # 收集文本...
    counter = Counter(collection)
    for text, count in counter.items():
        if count / total_scanned > threshold_ratio:
            noise_texts.add(text)
```

### 7. 图片过滤 `should_keep_image`

PDF 中可能包含大量小图标、装饰性图形，需要过滤。过滤条件：

| 条件 | 阈值 | 说明 |
|------|------|------|
| 最小像素尺寸 | 宽高均 ≥ 100px | 太小忽略 |
| 最小像素面积 | ≥ 30000 像素 | 如 200×150 |
| 最小文件大小 | ≥ 2KB | 避免空图 |
| 相对页面渲染尺寸 | ≥ 5% 页面宽度或高度 | 太小忽略 |
| 宽高比 | ≤ 12 | 过细长条可能是分隔线 |
| 信息密度（BPP） | 低信息时面积 < 500k 则过滤 | 纯色块/渐变图 |

此外还通过 MD5 哈希去重（避免重复背景图）。

### 8. 表格提取

```python
tabs = page.find_tables()
for tab in tabs:
    page_elements.append({
        "type": 2,
        "content": tab.to_markdown()
    })
```
- PyMuPDF 的 `find_tables()` 可检测表格区域，`to_markdown()` 直接输出 Markdown 表格。

### 9. 文本合并逻辑 `should_merge_lines`

```python
def should_merge_lines(current, next_line):
    if current.get("is_heading") or next_line.get("is_heading"): return False
    if current.get("is_list") or next_line.get("is_list"): return False
    if is_sentence_end(current.get("content", "")): return False
    return True
```
- 标题、列表项、以句号结尾的句子不合并，其他情况合并为同一段落。

### 10. 代码块检测 `is_monospace_font`

通过字体名是否包含 `courier`, `consolas`, `monaco` 等关键词判断是否为等宽字体（代码块），输出时包裹 ` ``` `。

### 11. 输出 Markdown 结构

- 第一行为 `# {文件名（去除数字前缀）}` 作为文档标题。
- 每页添加注释 `<!-- Page {num} -->` 帮助 LLM 理解分页。
- 表格、图片、列表、代码块均正确格式化。

### 12. 设计亮点

- **无外部依赖**（仅 PyMuPDF）即可完成深度解析。
- **启发式规则丰富**：字体统计、页眉页脚去噪、列表检测、代码字体识别。
- **图片智能过滤**：避免将装饰性小图或重复背景图导入 Markdown。
- **表格原生支持**：利用 PyMuPDF 的表格检测能力。
- **页面边界标记**：插入 `<!-- Page N -->`，方便后续按页分割。

---

# 三、web_to_md.py —— 网页转 Markdown（Python 版）

### 1. 核心定位

**作用**：从任意 URL 抓取网页，提取正文内容（去除导航、广告、侧边栏），下载图片并转换为本地引用，输出结构化的 Markdown。

**依赖**：`requests`, `beautifulsoup4`, `Pillow`（可选，用于 WebP 转 PNG）。

**局限性**：某些网站（如微信公众号）会基于 TLS 指纹（JA3）屏蔽 Python `requests`，此时需要使用 Node.js 版本 `web_to_md.cjs`。

### 2. 配置与选择器

```python
CONFIG = {
    "output_dir": "./projects",
    "timeout": 30,
    "user_agent": "Mozilla/5.0 ... Chrome/120.0.0.0",
    "content_selectors": [
        {"class_": re.compile(r"tys-main-zt-show", re.I)},
        {"class_": "TRS_Editor"},
        {"class_": "article-content"},
        {"id": "Zoom"},
        {"id": "content"},
        {"name": "article"},   # 标签名
        {"name": "main"},
    ]
}
```
- `content_selectors` 是针对中文政务/新闻网站的常见正文容器类名/ID。

### 3. 正文提取 `find_main_content`

**策略**：
1. 先删除 `script`, `style`, `nav`, `header`, `footer`, `aside` 等噪音标签。
2. 遍历 `content_selectors` 中定义的类/ID，找到匹配元素并计算得分（文本长度 + 中文字符数×2）。
3. 若得分不足，则扫描所有 `<div>`，统计内部 `<p>` 数量和文本长度，选择得分最高的作为正文容器。

### 4. 元数据提取 `extract_metadata`

- 标题：`<title>` 去除网站后缀（如“- 知乎”）。
- 发布时间：从 `<meta>` 的 `article:published_time`、`og:published_time` 等获取，若没有则在正文中正则匹配“发布时间：2024-01-01”模式，或从 URL 中提取日期。
- 作者/来源：从 `<meta name="author">` 或正文中“来源：”提取。
- 描述：`<meta name="description">` 等。

### 5. 图片下载与转换 `download_and_rewrite_images`

- 遍历正文容器内的 `<img>` 标签。
- 解析相对路径为绝对 URL，下载图片。
- 若图片为 WebP 格式且 Pillow 可用，则转换为 PNG（因为某些 Markdown 渲染器不支持 WebP）。
- 重命名图片（基于 URL 路径或序号），避免重名。
- 修改 `<img>` 的 `src` 为相对路径（如 `article_files/image_01.png`）。

### 6. HTML 转 Markdown `simple_html_to_markdown_traversal`

**递归遍历** BeautifulSoup 树，根据标签类型输出 Markdown：

| 标签 | Markdown 输出 |
|------|---------------|
| `h1`-`h6` | `# 标题` |
| `p` | 段落前后加空行 |
| `strong`/`b` | `**粗体**` |
| `em`/`i` | `*斜体*` |
| `a` | `[链接文本](url)` |
| `img` | `![alt](src)`（已处理） |
| `pre` | ` ``` ` 代码块 |
| `code` | `` `代码` `` |
| `li` | `- 列表项` |
| `blockquote` | `> 引用` |
| `hr` | `---` |
| `br` | 两个空格 + 换行 |

对于表格，仅简单提取为文本，未实现完整 Markdown 表格（因为复杂表格较少见）。

### 7. 输出 Markdown 结构

```markdown
<!--
  Source: https://...
  Crawled: 2025-01-01T...
  Published: 2024-12-01
  Author: 某某
-->

# 文章标题

> 文章描述

正文内容（Markdown 格式）
```

### 8. 设计亮点

- **智能正文提取**：通过多级选择器 + 文本密度评分，适应不同 CMS 模板。
- **图片本地化**：下载图片并修正引用路径，保证 Markdown 可离线使用。
- **WebP 转 PNG**：提高兼容性。
- **元数据丰富**：便于后续策略师理解文章上下文。
- **批量处理**：支持从文件读取多个 URL 逐个转换。

---

# 四、web_to_md.cjs 代码结构与核心逻辑解析

### 0. 总体概述
这是一个纯 Node.js 原生模块实现的网页抓取与 Markdown 转换工具，**不依赖任何第三方库**（如 axios、cheerio、puppeteer）。通过正则表达式和字符串处理完成 HTML 解析、正文提取、图片下载和 Markdown 转换。主要针对中文政府网站、微信公众号文章等做了模式适配。

---

### 1. 绕过微信公众号等网站 TLS 指纹屏蔽的机制（与 Python 版本的区别）

**结论：该脚本并未真正实现 TLS 指纹绕过，仅通过设置常规 HTTP 头模仿浏览器。**

- **实际行为**：使用 Node.js 原生 `http`/`https` 模块发起请求，TLS 握手时暴露的指纹是 Node.js 的 `TLS` 特征（如加密套件顺序、扩展字段等），与 Chrome 等浏览器有明显差异。微信公众号等反爬系统很容易识别并拦截。
- **为什么题目认为“能绕过”**：可能是误解。与 Python `requests` 库相比，Node.js 原生 `https` 模块的默认 TLS 指纹也非浏览器指纹，**两者均无法绕过现代反爬**。真正的绕过需要 `puppeteer-extra` + `stealth-plugin` 或定制 `tls` 库（如 `curl-impersonate`）。本脚本仅修改了 `User-Agent` 和 `Accept` 头，**不具备绕过 TLS 指纹的能力**。
- **与 Python 版本的区别**：Python 版本若使用 `requests`，其 TLS 指纹为 `python-requests/urllib3`，同样易被识别。本脚本与 Python 普通版本处于同一防御级别。

**设计缺陷**：未实现任何指纹伪造，对微信公众号等强反爬站点实际成功率极低。

---

### 2. 使用的 Node.js 库

**仅使用 Node.js 内置模块**：
- `fs.promises` – 异步文件操作
- `path` – 路径处理
- `https` / `http` – HTTP/HTTPS 请求
- 无任何第三方库（无 axios、cheerio、puppeteer）

**关键影响**：
- 无法执行 JavaScript（不支持 SPA）
- HTML 解析完全基于正则，脆弱且易出错
- 无 CSS 选择器能力，正文提取靠硬编码模式匹配

---

### 3. 正文提取实现（与 Python 版本选择器配置的对比）

**提取流程**：`extractMainContent(html)` 函数，步骤如下：

1. **清理干扰元素**：正则删除 `<script>`、`<style>`、`<nav>`、`<header>`、`<footer>`、`<aside>`、HTML 注释。
2. **模式匹配容器**：定义 `contentPatterns` 数组，包含针对中文网站（政府、公众号）的常见 class/id 正则：
   - 微信公众号：`rich_media_content`、`js_content`
   - 政府网站：`TRS_Editor`、`TRS_UEDITOR`、`ucontent`、`article-content`、`zwgk_content` 等
   - 通用：`<article>`、`<main>`、`id="content"`、`class="content"`
3. **评分选择最佳内容**：
   - 对每个匹配到的内容块，计算**文本长度** + **中文字符数×2** 作为得分。
   - 选择得分最高且文本长度 > 200 的块。
4. **降级策略**：若无有效匹配，则寻找 `<body>` 中 `<p>` 标签最密集的区域；最后回退到整个 `<body>`。

**与 Python 版本的区别**：
- Python 常用 `readability-lxml` 或 `trafilatura`，基于 DOM 树和机器学习特征，准确率更高。
- 本脚本纯正则 + 字符串操作，**无 DOM 解析**，对 HTML 结构变化极其敏感。
- Python 版本通常支持用户提供 CSS 选择器配置（如 `'article', '.post-content'`），本脚本为硬编码模式，扩展需修改源码。

**设计决策**：为轻量和无依赖，牺牲鲁棒性，针对性优化已知站点模式。

---

### 4. 图片下载、重命名、路径替换实现

**核心函数**：`processImages(html, pageUrl, imageDir, relPrefix)`

**流程**：
1. **提取图片 URL**：通过正则 `/<img[^>]+src=["']([^"']+)["'][^>]*>/gi` 收集所有 `src`。
2. **去重**：使用 `Set` 避免重复下载。
3. **构建文件名**：`buildImageFilename(urlStr, index)`
   - 解析 URL，取路径最后一段作为 basename
   - 调用 `sanitizeFilename` 过滤非法字符（仅保留中文、字母数字下划线）
   - 若 basename 无有效名称，使用 `image_{index}`
   - 扩展名检测：若无法识别则默认 `.jpg`
4. **处理命名冲突**：循环检测文件是否存在，若存在则追加 `_1`、`_2` 等计数器。
5. **下载图片**：`downloadImage(url, savePath)` – 使用 `http`/`https` 模块流式写入文件，设置 User-Agent 和超时。
6. **路径替换**：
   - 生成相对路径：`relPrefix`（即 `{md文件名}_files`） + 文件名
   - 将 Windows 反斜杠转为正斜杠 `/`（Markdown 兼容）
   - 在原始 HTML 中用正则替换 `src="原URL"` 为 `src="相对路径"`（精确匹配，避免误替换其他属性）。

**注意事项**：
- 不支持 `data:` 内联图片和 base64（跳过）
- 不支持相对路径解析错误处理（依赖 `new URL(src, pageUrl)`）
- 无并发控制，串行下载大图片可能慢

---

### 5. 是否支持 JavaScript 渲染页面（如 SPA）

**不支持**。原因：
- 仅使用原生 `http`/`https` 获取初始 HTML，不执行任何 JS。
- 无 Puppeteer、Playwright 或 JSDOM 等浏览器环境。
- 对于 React/Vue 等客户端渲染的 SPA，抓取到的内容为空或只有骨架屏。

**设计决策**：保持轻量和无依赖，牺牲动态内容支持。如需 SPA 支持，需引入 Puppeteer 并修改 `fetchUrl` 逻辑。

---

### 6. 输出 Markdown 格式与 Python 版本的异同

**本脚本的 Markdown 转换**：`htmlToMarkdown(html)` 函数，通过**正则逐一替换**实现：

| HTML 元素 | 转换结果 | 备注 |
|----------|---------|------|
| `<h1>`-`<h6>` | `#` 到 `######` + 内容 | 前后加换行 |
| `<p>`, `<div>` | 内容 + 换行 | 保留内部格式 |
| `<strong>`,`<b>` | `**粗体**` | |
| `<em>`,`<i>` | `*斜体*` | |
| `<a>` | `[文本](URL)` | 过滤 javascript: 链接 |
| `<img>` | `![alt](src)` | 已替换为本地路径 |
| `<ul>`/`<ol>` + `<li>` | `- 项目` | 简单列表，无嵌套支持 |
| `<pre>`+`<code>` | ` ``` ` 代码块 | 无语言标注 |
| `<blockquote>` | `> 引用` | 每行加 `>` |
| `<table>` | 管道表格式 | 生成 `\| --- \|` 分隔符 |

**与 Python 版本的典型差异**：
- Python 常用 `markdownify` 或 `html2text` 库，支持更多选项（如代码语言、链接引用风格）。
- 本脚本：
  - **无嵌套列表处理**（内部 `<ul>` 会打乱）
  - **无 HTML 标签属性清理**（如 `class`、`style` 被简单删除）
  - **无表格对齐支持**
  - **无脚注、定义列表等扩展**
- 输出质量：适合纯文本阅读，但复杂排版（如多列、图文混排）可能丢失结构。

**共同点**：都支持基础 Markdown 语法，输出人类可读的文档。

---

### 7. 主要命令行参数和使用示例

| 参数 | 说明 |
|-----|------|
| `<url>` | 单个或多个 URL（空格分隔） |
| `-f <file>` | 从文本文件读取 URL（每行一个） |
| `-o <file>` | 指定输出文件名（仅单 URL 时有效） |
| `-d <dir>` | 指定输出目录（默认 `./projects`） |
| `-h, --help` | 显示帮助 |

**示例**：
```bash
# 单个 URL，自动生成文件名
node web_to_md.cjs https://example.com/article

# 多个 URL，每个生成独立文件
node web_to_md.cjs https://a.com/1 https://b.com/2

# 从文件读取 URL 列表
node web_to_md.cjs -f urls.txt

# 指定输出文件名和目录
node web_to_md.cjs https://example.com -o mydoc.md -d ./output
```

---

## 关键函数逐段解析与设计决策

### 1. `fetchUrl(url)`
- **职责**：发起 HTTP/HTTPS 请求，处理 3xx 重定向，尝试检测编码。
- **设计决策**：
  - 手动实现重定向（最多一次，非递归无限）。
  - 编码检测仅检查 `Content-Type` 头中的 `charset` 和 `<meta charset>`，但**未真正解码**（仍用 `utf-8` 读取）。对于 GBK 页面会乱码。
  - 无 gzip 解压支持（现代网站多压缩，可能导致乱码）。
- **局限性**：不支持压缩、不完善的编码、无 cookie/会话管理。

### 2. `parseHTML(html)` & `extractMainContent(html)`
- **职责**：提取标题、meta 信息、正文 HTML。
- **设计决策**：
  - 标题清理：移除常见的“政府|门户|网站”等后缀。
  - 正文提取：**正则匹配 + 文本评分**，而非 DOM 树分析。优点是无依赖、轻量；缺点是无法处理动态 class、嵌套混乱的页面。
  - 针对中文网站大量硬编码模式（如 `tys-main-zt-show`、`TRS_Editor`），体现了对特定 CMS 的适配。
- **评分策略**：中文字符加权，因为中文网站正文往往中文字符多。这会导致英文文章得分偏低。

### 3. `htmlToMarkdown(html)`
- **职责**：将清洗后的 HTML 转为 Markdown。
- **设计决策**：
  - 采用**顺序正则替换**，简单直观但易产生副作用（如替换后引入新的标签）。
  - 链接处理中，使用函数回调去除内部嵌套标签，确保 `text` 纯文本化。
  - 表格转换假设第一行为表头，生成管道分隔线，但不处理 `colspan`/`rowspan`。
  - 最后做空行合并和行首尾空格清理。
- **缺陷**：正则无法正确处理嵌套结构（如 `<ul>` 内的 `<li>` 再包含 `<ul>`），列表会损坏。

### 4. `processImages(html, pageUrl, imageDir, relPrefix)`
- **职责**：下载图片、重命名、更新 HTML 中的 src。
- **设计决策**：
  - 去重下载，节省时间。
  - 文件名生成策略：优先使用原 URL 中的文件名，并清理非法字符。冲突时加数字后缀。
  - 替换逻辑：用正则匹配原 src 字符串并替换，而非重新构建整个标签，避免破坏其他属性。
- **潜在问题**：
  - 串行下载，大量图片时性能差。
  - 无重试机制，单次失败即跳过。
  - 依赖文件系统 `access` 检测存在性，存在 TOCTOU 竞态（但影响不大）。

### 5. `extractMetadata(parsed, url)`
- **职责**：从 meta 标签和内容正则中提取发布日期、作者、描述等。
- **设计决策**：
  - 优先取 Open Graph 或 article 相关 meta。
  - 若 meta 无日期，则从正文中正则匹配中文日期格式（`发布时日间：\d{4}年...`）和 URL 中提取。
  - 作者信息同时从 meta 和正文“来源：”中提取。
- **局限性**：正则匹配日期模式有限，无法处理复杂格式（如“2024年3月5日”中的无前导零月份）。

### 6. `processUrl(url, outputPath)`
- **职责**：整合所有步骤，生成最终 Markdown 文件。
- **设计决策**：
  - 先获取 HTML → 提取内容 → 确定输出路径 → 下载图片 → 转换 Markdown → 写入文件。
  - 输出文件包含 YAML 风格的注释块（`<!-- ... -->`），记录源 URL、抓取时间、发布日期、作者等。
  - 图片目录命名规则：`{md文件名}_files`（类似浏览器保存网页的惯例）。
- **健壮性**：图片处理失败仅警告，不中断主流程。

### 7. `main()` 命令行解析
- **设计决策**：手动解析参数，支持 `-f`、`-o`、`-d`、`-h`。未使用 `commander` 等库以保持零依赖。
- **逻辑**：
  - 若参数为 `-h` 则显示帮助并退出。
  - 若为 `-f`，读取文件中的 URL。
  - 收集所有 `http://` 或 `https://` 开头的参数作为 URL。
  - 循环处理每个 URL，对单 URL 时可指定 `-o`。
- **缺陷**：选项位置敏感（`-o` 必须紧跟在 URL 之后？实际解析逻辑是遍历参数，`-o` 后跟的值会被赋给 `outputFile`，但若多 URL 则 `outputFile` 只会影响第一个 URL，其余忽略）。

---

## 适用场景与局限性总结

### 适用场景
1. **静态技术文档、新闻文章、政府公告** – HTML 结构简单，内容集中在特定 class 容器内。
2. **离线阅读** – 将网页保存为单一 Markdown + 本地图片，便于笔记或归档。
3. **无外部依赖环境** – 仅需 Node.js 运行时，无需安装任何 npm 包。
4. **中文站点优先** – 大量模式针对中文政府、公众号做了优化。

### 局限性
| 方面 | 具体限制 |
|-----|----------|
| **反爬能力** | 无 TLS 指纹伪装，容易被公众号、Cloudflare 等屏蔽。 |
| **JavaScript 渲染** | 完全不支持 SPA，只能抓取服务端渲染页面。 |
| **解析可靠性** | 正则解析 HTML 对格式变化极其脆弱，嵌套结构处理差。 |
| **编码支持** | 未真正解码 GBK/GB2312，中文老站点可能乱码。 |
| **性能** | 串行下载图片，大页面慢；无缓存机制。 |
| **Markdown 质量** | 复杂表格、嵌套列表、图文混排会丢失格式。 |
| **错误恢复** | 单图片失败即跳过，无重试；网络超时无指数退避。 |
| **安全性** | 未验证 HTTPS 证书（`rejectUnauthorized` 默认 true，但未配置）。 |

### 改进建议
- 若需真正绕过 TLS 指纹，可改用 `puppeteer-extra` + `stealth` 或 `curl-impersonate`。
- 若需支持 SPA，替换 `fetchUrl` 为 Puppeteer 或 Playwright。
- 若需提高解析准确率，引入 `jsdom` + `readability` 库。
- 添加 `--proxy`、`--cookie` 等参数增强灵活性。

---

该脚本是一个**轻量级、特定场景可用**的工具，适合快速转换已知结构简单的静态网页，但不应作为通用爬虫或生产环境解决方案。

---

# 五、源文件转换层总结（设计特点与架构）

| 脚本 | 输入格式 | 核心技术 | 智能特性 |
|------|----------|----------|----------|
| `doc_to_md.py` | DOCX, ODT, EPUB, HTML, LaTeX, RST, Typst, Jupyter | Pandoc 转换 | 媒体提取、路径修正、HTML 标签转 Markdown |
| `pdf_to_md.py` | PDF | PyMuPDF 解析 | 字体分析推断标题、列表检测、表格提取、图片过滤、页眉页脚去噪 |
| `web_to_md.py` | 任意 URL | requests + BeautifulSoup | 正文智能提取、图片下载转换、元数据提取 |
| `web_to_md.cjs` | 高安全性网站（微信等） | Node.js + 可能使用 curl_cffi/puppeteer | 绕过 TLS 指纹、处理动态内容 |

### 设计模式与工程智慧

1. **统一输出格式**：所有脚本输出标准 Markdown，使后续策略师和执行师无需关心输入来源。
2. **渐进式智能**：
   - PDF 解析最复杂（无语义结构），通过**启发式规则**（字号、位置、字体）推断结构。
   - 网页解析次之，通过**选择器 + 文本密度**提取正文。
   - 办公文档最简单，依赖 Pandoc 这种成熟转换器。
3. **媒体处理自动化**：自动提取并嵌入图片引用，保证 Markdown 自包含。
4. **错误容忍**：单个文件失败不影响批量处理；图片下载失败仅警告，继续处理。
5. **分页标记**：PDF 转换插入 `<!-- Page N -->`，便于后续按页分割（`total_md_split.py` 会用到）。
6. **去噪处理**：PDF 去除页眉页脚，网页去除导航/广告，提高内容纯度。

### 与后续流水线的衔接

- 输出的 Markdown 文件被 `project_manager.py` 导入到项目的 `sources/` 目录。
- 策略师读取该 Markdown 进行分析，生成设计规范。
- 执行师基于设计规范和原始 Markdown 生成 SVG 页面。
- 后处理脚本中的 `total_md_split.py` 会利用 Markdown 中的 `<!-- Page N -->` 注释或 `#` 标题来分割演讲备注。

---

如果你准备好了，可以继续下一批脚本（项目管理类：`project_manager.py`, `project_utils.py`, `config.py` 等），或者选择其他你感兴趣的脚本。