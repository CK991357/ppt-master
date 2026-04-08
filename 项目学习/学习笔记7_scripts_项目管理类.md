非常抱歉，上一轮我确实没有输出项目管理类的解析内容。现在补上，格式清晰可复制。

---

# 项目管理类脚本详细解析

## 文件清单

| 文件 | 核心职责 |
|------|----------|
| `config.py` | 统一配置管理：画布格式、配色方案、行业颜色、字体、SVG 约束等 |
| `error_helper.py` | 错误信息助手：提供友好错误提示和修复建议 |
| `project_manager.py` | 项目生命周期管理：初始化、导入源文件、验证、信息查询 |
| `project_utils.py` | 项目工具函数：解析项目名、获取信息、验证结构、SVG viewBox 校验 |

这四个文件共同构成了项目管理的基础设施，负责**创建项目目录、处理源文件转换、验证项目完整性**。

---

## 一、config.py —— 统一配置管理

### 1. 核心设计思想

将所有可配置的常量集中管理，避免硬编码。任何脚本都可以通过 `from config import Config, CANVAS_FORMATS` 引用。

### 2. 配置分类

#### 2.1 路径配置

```python
PROJECT_ROOT = Path(__file__).parent.parent          # skills/ppt-master/
SCRIPTS_DIR = PROJECT_ROOT / 'scripts'
REFERENCES_DIR = PROJECT_ROOT / 'references'
TEMPLATES_DIR = PROJECT_ROOT / 'templates'
REPO_ROOT = PROJECT_ROOT.parent.parent               # 仓库根目录
EXAMPLES_DIR = REPO_ROOT / 'examples'
PROJECTS_DIR = REPO_ROOT / 'projects'
```

#### 2.2 画布格式配置 `CANVAS_FORMATS`

包含 8 种格式：`ppt169`, `ppt43`, `wechat`, `xiaohongshu`, `moments`, `story`, `banner`, `a4`。

每个格式包含字段：
- `name`: 显示名称
- `dimensions`: 尺寸（如 `1280×720`）
- `viewbox`: SVG viewBox 字符串
- `width`, `height`: 数值
- `aspect_ratio`: 宽高比
- `use_case`: 使用场景描述

#### 2.3 设计配色方案 `DESIGN_COLORS`

预定义了 5 种风格：`consulting`（咨询）、`general`（通用）、`tech`（科技）、`academic`（学术）、`government`（政务）。每个风格包含：
- `primary`, `secondary`, `accent`, `success`, `warning`
- `text_dark`, `text_light`, `text_muted`
- `background`, `background_alt`

#### 2.4 行业颜色模板 `INDUSTRY_COLORS`

覆盖 14 个行业（金融、医疗、科技、教育、零售、制造、能源、房地产、法律、媒体、物流、农业、旅游、汽车）。每个行业提供 `primary`, `secondary`, `accent` 三种颜色。

#### 2.5 字体配置

```python
FONTS = {
    'system_ui': "-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif",
    'sans_serif': "'Helvetica Neue', Arial, 'PingFang SC', 'Microsoft YaHei', sans-serif",
    'monospace': "'SF Mono', Monaco, Consolas, 'Liberation Mono', monospace"
}
FONT_SIZES = {...}   # 9 级字号
```

#### 2.6 SVG 技术约束 `SVG_CONSTRAINTS`

包含三个子项：
- `forbidden_elements`: 禁止的 SVG 元素列表（`clipPath`, `mask`, `style`, `foreignObject`, `animate*`, `script` 等）
- `forbidden_attributes`: 禁止的属性（`class`, `id`, 事件属性, `marker-end`）
- `forbidden_patterns`: 正则表达式禁止模式（`rgba(`, `@font-face`, `@import`, `<g[^>]*opacity` 等）
- `recommended_fonts`: 推荐字体栈

### 3. Config 类方法

| 方法 | 功能 |
|------|------|
| `get_canvas_format(key)` | 获取指定画布格式配置 |
| `get_all_canvas_formats()` | 获取所有画布格式 |
| `get_color_scheme(style)` | 获取配色方案 |
| `get_industry_colors(industry)` | 获取行业颜色 |
| `get_layout_margins(format_key)` | 获取布局边距 |
| `get_font(font_type)` | 获取字体声明 |
| `get_font_size(size_name)` | 获取字号数值 |
| `validate_svg_element(element_name)` | 检查 SVG 元素是否允许 |
| `export_config(output_file)` | 导出全部配置为 JSON |

### 4. CLI 入口

```bash
python3 config.py list-formats      # 列出所有画布格式
python3 config.py list-colors       # 列出所有配色方案
python3 config.py list-industries   # 列出所有行业颜色
python3 config.py export            # 导出配置到 JSON
python3 config.py format <key>      # 查看特定格式详情
```

### 5. 设计亮点

- **单点维护**：所有配置集中，修改一处全局生效。
- **向后兼容**：其他脚本通过 `try/except ImportError` 提供后备配置。
- **可扩展**：新增画布格式只需在 `CANVAS_FORMATS` 中添加条目。
- **行业颜色库**：支持策略师根据内容行业快速推荐配色。

---

## 二、error_helper.py —— 错误信息助手

### 1. 核心设计思想

统一管理所有错误类型及其修复建议，输出用户友好的错误信息，而不是原始的 Python 异常堆栈。

### 2. 错误类型字典 `ERROR_SOLUTIONS`

每个错误条目包含：
- `message`: 错误描述
- `solutions`: 修复建议列表（字符串数组）
- `severity`: `error` 或 `warning`

支持的错误类型（部分示例）：

| 错误类型 | 说明 |
|----------|------|
| `missing_readme` | 缺少 README.md |
| `missing_spec` | 缺少设计规范文件 |
| `missing_svg_output` | 缺少 svg_output 目录 |
| `empty_svg_output` | svg_output 目录为空 |
| `invalid_svg_naming` | SVG 文件命名不规范 |
| `viewbox_mismatch` | viewBox 与画布格式不匹配 |
| `foreignobject_detected` | 检测到禁止的 `<foreignObject>` |
| `clippath_detected` | 检测到禁止的 `<clipPath>` |
| `rgba_detected` | 检测到禁止的 `rgba()` 颜色 |
| `group_opacity_detected` | 检测到禁止的 `<g opacity>` |
| `symbol_use_detected` | 检测到禁止的 `<symbol>`+`<use>` |

> 共定义了约 30 种错误类型，覆盖 SVG 兼容性、项目结构、命名规范等。

### 3. 核心方法

```python
@classmethod
def get_solution(cls, error_type, context=None) -> Dict:
    # 获取错误信息，支持上下文变量替换（如 <project_path>）

@classmethod
def format_error_message(cls, error_type, context=None) -> str:
    # 返回格式化的错误信息字符串

@classmethod
def print_error(cls, error_type, context=None):
    # 直接打印错误信息
```

### 4. 上下文变量替换

当 `context` 包含 `project_path`, `file_name`, `expected`, `actual` 时，会自动替换到错误消息中。例如：

```python
ErrorHelper.format_error_message('missing_readme', {'project_path': 'my_project'})
# 输出中所有 <project_path> 会被替换为 'my_project'
```

### 5. CLI 入口

```bash
python3 error_helper.py missing_svg_output
python3 error_helper.py viewbox_mismatch expected="0 0 1280 720" actual="0 0 1024 768"
```

### 6. 设计亮点

- **用户友好**：错误信息包含具体修复步骤，而非晦涩的技术术语。
- **可扩展**：新增错误类型只需在 `ERROR_SOLUTIONS` 中添加条目。
- **上下文感知**：支持动态替换占位符，提供定制化信息。
- **结构化输出**：便于其他脚本调用并嵌入到日志中。

---

## 三、project_manager.py —— 项目生命周期管理

### 1. 核心职责

- 创建标准化项目目录结构
- 导入源文件（本地文件或 URL）并自动转换为 Markdown
- 验证项目完整性
- 查询项目信息

### 2. 项目目录结构

```
project/
├── README.md           # 自动生成
├── svg_output/         # 原始 SVG（执行师输出）
├── svg_final/          # 后处理后的 SVG
├── images/             # 图片资产
├── notes/              # 演讲备注
├── templates/          # 项目模板
└── sources/            # 源材料（原始文件 + 转换后的 Markdown）
```

### 3. 核心方法

#### 3.1 `init_project(project_name, canvas_format, base_dir)`

- 创建目录结构
- 生成 README.md
- 返回项目路径

命名规范：`{project_name}_{format}_{YYYYMMDD}`，例如 `my_report_ppt169_20251201`

#### 3.2 `import_sources(project_path, source_items, move)`

**核心功能**：处理各种输入源（本地文件、URL），转换为 Markdown 并存放到 `sources/` 目录。

**输入类型自动识别**：
- URL → 调用 `web_to_md.py` 或 `web_to_md.cjs`（微信公众号用 Node 版本）
- PDF → 调用 `pdf_to_md.py` 转换
- DOCX/ODT/EPUB 等 → 调用 `doc_to_md.py` 转换
- Markdown/TXT → 直接复制，并自动处理关联的 `_files` 资产目录
- 其他格式 → 仅归档，不转换

**关键逻辑**：

- **去重**：对于 Markdown 文件，通过规范化内容（移除爬取时间、资产路径）比较是否已存在相同内容的 Markdown。
- **资产目录处理**：如果源文件附带 `_files` 目录（如 Pandoc 提取的媒体），会一并移动并重写 Markdown 中的引用路径。
- **自动转换抑制**：如果用户同时提供了 `.pdf` 和同名的 `.md`，则跳过 PDF 自动转换（避免重复）。
- **移动 vs 复制**：当源文件位于仓库内时，默认使用 `move`（归档后原文件消失），否则使用复制。

**返回摘要**：
```python
{
    'archived': [],   # 归档的原始文件/URL 记录
    'markdown': [],   # 生成的 Markdown 文件
    'assets': [],     # 移动的资产目录
    'notes': [],      # 备注信息（如跳过转换的原因）
    'skipped': []     # 跳过的项及原因
}
```

#### 3.3 `validate_project(project_path)`

调用 `project_utils.validate_project_structure` 并附加 SVG viewBox 校验。

#### 3.4 `get_project_info(project_path)`

返回项目的基本信息（名称、格式、创建日期、SVG 数量等）。

### 4. 辅助函数

| 函数 | 功能 |
|------|------|
| `is_url(s)` | 判断字符串是否为 HTTP/HTTPS URL |
| `sanitize_name(s)` | 清理字符串为文件系统安全名称 |
| `derive_url_basename(url)` | 从 URL 推导基础文件名 |
| `is_within_path(path, parent)` | 检查路径是否在父目录内 |

### 5. CLI 命令

```bash
# 初始化项目
python3 project_manager.py init my_project --format ppt169 --dir projects

# 导入源文件
python3 project_manager.py import-sources projects/my_project_ppt169_20251201 /path/to/doc.pdf --move

# 验证项目
python3 project_manager.py validate projects/my_project_ppt169_20251201

# 查看项目信息
python3 project_manager.py info projects/my_project_ppt169_20251201
```

### 6. 设计亮点

- **统一入口**：所有源文件通过同一个命令导入，自动识别类型并调用相应转换器。
- **幂等性**：重复导入相同内容会检测重复并跳过，避免冗余。
- **资产完整性**：自动处理 Markdown 关联的 `_files` 目录，保持引用有效。
- **智能转换抑制**：避免对已提供 Markdown 的 PDF 进行重复转换。
- **详细反馈**：返回结构化的导入摘要，便于用户了解处理结果。

---

## 四、project_utils.py —— 项目工具函数

### 1. 核心定位

提供 `project_manager.py` 和其他脚本可复用的底层函数，包括项目名解析、信息获取、结构验证、SVG viewBox 校验等。

### 2. 画布格式别名支持

```python
CANVAS_FORMAT_ALIASES = {
    'xhs': 'xiaohongshu',
    'wechat_moment': 'moments',
    '朋友圈': 'moments',
    '小红书': 'xiaohongshu',
}
```

`normalize_canvas_format()` 函数将别名转换为标准键名。

### 3. `parse_project_name(dir_name)` —— 核心解析函数

从项目目录名中提取三个信息：
- `name`: 项目名称（去除后缀）
- `format`: 画布格式（如 `ppt169`）
- `date`: 日期（如 `20251201`）

**解析优先级**：
1. 匹配 `name_format_YYYYMMDD` 标准格式
2. 若失败，查找目录名末尾是否包含已知画布格式
3. 最后提取日期后缀

**正则表达式**：
```python
full_match = re.match(r'^(?P<name>.+)_(?P<format>[a-z0-9_-]+)_(?P<date>\d{8})$', dir_name_lower)
```

### 4. `get_project_info(project_path)`

返回包含以下字段的字典：
- `path`, `dir_name`, `name`, `format`, `format_name`, `date`, `date_formatted`
- `exists`, `svg_count`, `has_spec`, `has_readme`, `has_source`, `source_count`
- `spec_file`, `svg_files`

### 5. `validate_project_structure(project_path, verbose)`

**检查项**：
- 目录是否存在
- `README.md` 是否存在（错误）
- 设计规范文件是否存在（警告）
- `svg_output` 目录是否存在（错误）
- `svg_output` 是否包含 SVG 文件（警告）
- SVG 文件命名是否符合规范（警告）
- 目录名是否包含日期后缀（警告）

**返回值**：`(is_valid, errors, warnings)`

### 6. `validate_svg_viewbox(svg_files, expected_format)`

- 读取每个 SVG 文件的前 2000 个字符，提取 `viewBox` 属性
- 如果指定了期望格式，检查 viewBox 是否匹配
- 收集所有不同的 viewBox，若存在多个则发出警告

### 7. 其他实用函数

| 函数 | 功能 |
|------|------|
| `find_all_projects(base_dir)` | 递归查找所有有效项目目录 |
| `format_file_size(size_bytes)` | 格式化文件大小（B/KB/MB/GB） |
| `get_project_stats(project_path)` | 统计项目文件数量及总大小 |

### 8. 设计亮点

- **容错性**：即使 `config.py` 不可用，也有后备配置保证基本功能。
- **正则健壮性**：项目名解析支持多种格式，兼容旧项目。
- **模块化**：验证函数可被其他脚本单独调用（如 CI 集成）。
- **多语言支持**：设计规范文件名支持中文（`设计规范与内容大纲.md`）和英文。

---

## 项目管理类脚本总结（架构与设计模式）

| 脚本 | 主要设计模式 | 核心特点 |
|------|-------------|----------|
| `config.py` | 单例配置 | 集中管理常量，支持 CLI 导出 |
| `error_helper.py` | 策略模式 | 错误类型到修复建议的映射 |
| `project_manager.py` | 外观模式 | 封装项目操作的复杂细节，提供简单 CLI |
| `project_utils.py` | 工具类 | 纯函数集合，无副作用 |

### 数据流

```
用户输入 (URL/PDF/DOCX/MD)
        ↓
project_manager.import_sources()
        ↓ (自动识别类型)
pdf_to_md.py / doc_to_md.py / web_to_md.py
        ↓
sources/*.md + sources/*_files/
        ↓
project_manager.validate_project()
        ↓ (后续由策略师、执行师处理)
```

### 工程智慧

1. **幂等导入**：重复导入相同内容不会产生重复文件，通过规范化内容哈希比较。
2. **资产目录跟随**：自动处理 Pandoc 生成的 `_files` 目录，保证 Markdown 图片引用有效。
3. **自动转换抑制**：同时提供 PDF 和同名 Markdown 时，跳过 PDF 转换，尊重用户已准备的内容。
4. **错误分类与友好提示**：`error_helper` 将技术错误转化为可操作的步骤。
5. **项目命名规范强制**：通过日期后缀和格式标识，确保项目目录可追溯、可排序。
