## `scripts/docs/` 文档目录详细解析

`scripts/docs/` 目录包含了 PPT Master 项目中所有脚本的**用户文档**，按功能分类：转换工具、图片工具、项目管理工具、SVG 流水线工具、故障排除。这些文档面向**使用脚本的 AI 或人类用户**，提供命令行示例、依赖说明和最佳实践。它们与之前解析的脚本一一对应，是理解“如何用”的重要补充。

---

# 一、conversion.md —— 源文件转换工具文档

### 对应脚本
- `pdf_to_md.py`
- `doc_to_md.py`
- `web_to_md.py` / `web_to_md.cjs`
- `rotate_images.py`

### 内容概要

#### `pdf_to_md.py`
- **推荐用于原生 PDF**（由 Word、PPT、LaTeX 等导出）。
- **使用场景**：隐私敏感文档、快速提取、作为 OCR 前的初筛。
- **不适用场景**：扫描件/图片型 PDF、多栏布局混乱、编码乱码（此时建议使用 MinerU 等 OCR 工具）。
- **依赖**：`PyMuPDF`（`pip install PyMuPDF`）。
- 命令示例：单文件、指定输出、批量转换目录。

#### `doc_to_md.py`
- 基于 Pandoc，支持 `.docx`, `.doc`, `.odt`, `.rtf`, `.epub`, `.html`, `.tex`, `.rst`, `.org`, `.ipynb`, `.typ`。
- 依赖：Pandoc（需单独安装，给出 macOS/Ubuntu 命令）。
- 命令示例：转换 Word、EPUB、LaTeX 等。

#### `web_to_md.py` / `web_to_md.cjs`
- Python 版本：普通网页。
- Node.js 版本：微信公众号等反爬网站（优先使用）。
- 支持批量 URL、从文件读取、指定输出。
- 依赖：Python 版需要 `requests`, `beautifulsoup4`；Node.js 版需要 Node.js 环境。

#### `rotate_images.py`
- 修复从网页或文档中提取的图片的 EXIF 方向问题。
- 子命令：`auto`（自动修复）、`gen`（生成 HTML 交互工具）、`fix`（应用 JSON 修复指令）。
- 使用场景：提取的照片出现侧向或倒置。

### 文档设计亮点
- 明确区分“首选”和“备选”方案（如 PDF 转换首选 `pdf_to_md.py`，扫描件建议外部工具）。
- 给出每个脚本的依赖安装命令。
- 强调 Node.js 版本对微信等站点的优先性。

---

# 二、image.md —— 图片工具文档

### 对应脚本
- `image_gen.py`
- `analyze_images.py`
- `gemini_watermark_remover.py`

### 内容概要

#### `image_gen.py`
- 统一图片生成入口。
- **支持后端分级**：
  - Core: `gemini`, `openai`, `qwen`, `zhipu`, `volcengine`
  - Extended: `stability`, `bfl`, `ideogram`
  - Experimental: `siliconflow`, `fal`, `replicate`
- 配置方式：
  - 当前进程环境变量（优先级高）
  - 仓库根目录 `.env`（后备）
- **必须显式设置 `IMAGE_BACKEND`**，并提供对应后端的 API Key（如 `GEMINI_API_KEY`）。
- **不支持** `IMAGE_API_KEY`, `IMAGE_MODEL`, `IMAGE_BACKEND_URL` 等全局变量。
- 命令示例：基本生成、指定宽高比/尺寸、输出目录、负向提示词、列出后端。
- 推荐：日常使用 Core 后端，Extended 用于特定风格，Experimental 为可选。

#### `analyze_images.py`
- 分析项目 `images/` 目录中的图片尺寸和宽高比。
- 在编写设计规范或排版前使用，替代直接打开图片文件。
- 输出分析报告（表格、分组统计、PPT 适配建议、Markdown 片段）。

#### `gemini_watermark_remover.py`
- 去除 Gemini 生成图片的水印（“Made with Gemini”）。
- 需要本地资产文件 `scripts/assets/bg_48.png` 和 `bg_96.png`。
- 最佳实践：先下载 Gemini “Full size” 图片，再运行去水印。
- 依赖：`Pillow`, `numpy`。

### 文档设计亮点
- 清晰解释多后端配置逻辑和优先级。
- 强调 `IMAGE_BACKEND` 必须显式指定，避免歧义。
- 给出 `.env` 示例和进程环境变量示例。
- 明确各脚本的使用时机（如 `analyze_images.py` 在策略师阶段前运行）。

---

# 三、project.md —— 项目管理工具文档

### 对应脚本
- `project_manager.py`
- `project_utils.py`
- `batch_validate.py`
- `generate_examples_index.py`
- `pptx_template_import.py`
- `error_helper.py`

### 内容概要

#### `project_manager.py`
- 项目初始化、导入源文件、验证、信息查询。
- 常用格式：`ppt169`, `ppt43`, `xiaohongshu`, `moments`, `story`, `banner`, `a4`。
- 导入时文件处理规则：
  - 工作区外的文件默认**复制**到 `sources/`，使用 `--move` 则**移动**。
  - 已在工作区内的文件直接移动。
- 命令示例：`init`, `import-sources`, `validate`, `info`。

#### `project_utils.py`
- 被其他脚本共享的辅助模块。
- 可直接运行快速检查：`python3 project_utils.py <project_path>`。

#### `batch_validate.py`
- 批量检查项目结构。
- 用于发布前或清理时的仓库级健康检查。
- 支持导出报告。

#### `generate_examples_index.py`
- 自动重建 `examples/README.md`。
- 扫描 `examples/` 目录下的所有项目，生成索引。

#### `pptx_template_import.py`
- 轻量级 PPTX 模板导入工具。
- 提取媒体资产、幻灯片尺寸、主题颜色、字体元数据、背景图片继承关系。
- 输出 `manifest.json`, `analysis.md`, `assets/`。
- **不是** PPTX 到 SVG 的直接转换器，而是为模板重建提供参考。

#### `error_helper.py`
- 显示常见项目错误的标准化修复建议。
- 运行 `error_helper.py` 列出所有错误类型，或指定错误类型查看。

### 文档设计亮点
- 明确 `import-sources` 的移动/复制行为，避免意外丢失原文件。
- 强调 `pptx_template_import.py` 的有限目标，防止用户误以为它能完全转换 PPTX。
- 将 `error_helper.py` 定位为调试辅助工具。

---

# 四、svg-pipeline.md —— SVG 流水线工具文档

### 对应脚本
- `total_md_split.py`
- `finalize_svg.py`
- `svg_to_pptx.py`
- `svg_quality_checker.py`
- `svg_position_calculator.py`
- 以及 `svg_finalize/` 下的子工具（`embed_icons.py`, `crop_images.py`, `fix_image_aspect.py`, `embed_images.py`, `flatten_tspan.py`, `svg_rect_to_path.py`）

### 内容概要

#### 推荐流水线
```bash
python3 scripts/total_md_split.py <project_path>
python3 scripts/finalize_svg.py <project_path>
python3 scripts/svg_to_pptx.py <project_path> -s final
```

#### `finalize_svg.py`
- 统一后处理入口，聚合图标嵌入、图片裁剪、宽高比修复、图片嵌入、文本扁平化、圆角矩形转路径。
- 典型用法：`python3 scripts/finalize_svg.py <project_path>`。
- 仅在需要高级调试或单步修复时使用独立子工具。

#### `svg_to_pptx.py`
- 将项目 SVG 转换为 PPTX。
- 默认输出：原生可编辑 PPTX + SVG 参考版 PPTX。
- 推荐源目录：`svg_final/`（`-s final`）。
- 演讲备注默认嵌入，可用 `--no-notes` 禁用。
- 支持过渡效果、自动翻页、仅生成特定版本（`--only native` / `--only legacy`）。
- 依赖：`python-pptx`。

#### `total_md_split.py`
- 将 `total.md` 按页分割为独立的备注文件。
- 要求：每个章节以 `# ` 开头，标题与 SVG 文件名匹配，章节间用 `---` 分隔。

#### `svg_quality_checker.py`
- 检查 SVG 技术合规性（viewBox、禁止元素、宽高一致性、换行结构等）。
- 可单文件、目录、项目、全部示例批量检查。
- 支持指定期望格式（如 `--format ppt169`）以验证 viewBox 匹配。

#### `svg_position_calculator.py`
- 分析或预计算图表坐标（柱状图、饼图、雷达图、折线图、网格布局）。
- 交互模式、快速计算、JSON 配置。
- 用于 AI 生成图表前的坐标验证。

#### 高级独立工具
- `flatten_tspan.py`：展开 `<tspan>` 为独立 `<text>`。
- `svg_rect_to_path.py`：将带圆角的 `<rect>` 转换为 `<path>`。
- `fix_image_aspect.py`：修复嵌入图片的宽高比（防止 PPT 形状转换时拉伸）。
- `embed_icons.py`：手动嵌入图标（通常由 `finalize_svg.py` 自动完成）。

#### PPT 兼容规则速查表
| 避免 | 使用 |
|------|------|
| `fill="rgba(...)"` | `fill="#hex"` + `fill-opacity` |
| `<g opacity="...">` | 在每个子元素上设置 opacity |
| `<image opacity="...">` | 叠加一个 mask 层 |

另外，PPT 对基于 marker 的箭头、不支持的滤镜、未映射到 DrawingML 的 SVG 特性也有问题。

### 文档设计亮点
- 明确推荐流水线顺序，强调使用 `svg_final/` 而非 `svg_output/`。
- 将高级独立工具归类为“仅在调试时使用”，引导用户使用统一入口。
- 提供 PPT 兼容性替代语法表格，简洁实用。

---

# 五、troubleshooting.md —— 故障排除文档

### 内容概要

#### 验证失败
1. 运行 `project_manager.py validate <project_path>`
2. 修复验证器报告的问题
3. 在后处理或导出前重新验证

#### SVG 预览异常
- 检查文件路径和命名。
- 通过本地 HTTP 服务器预览（解决浏览器直接打开时的跨域限制）：
  ```bash
  python3 -m http.server --directory <svg_output_path> 8000
  ```

#### 演讲备注无法分割
- 检查 `total.md`：
  - 标题必须以 `# ` 开头
  - 标题文本必须与 SVG 文件名匹配
  - 章节间用 `---` 分隔
- 重新运行 `total_md_split.py`

#### PPT 导出质量问题
- 严格执行推荐流水线顺序（先分割、再 `finalize_svg.py`、最后 `svg_to_pptx.py -s final`）。
- 不要从 `svg_output/` 直接导出。

#### 依赖检查清单
- 大多数工具使用标准库。
- 按需安装额外依赖：`pip install -r requirements.txt`。
- 重要可选包：
  - `python-pptx`：PPTX 导出
  - `Pillow`：图片工具
  - `numpy`：水印去除
  - `PyMuPDF`：PDF 转换
  - `google-genai` / `openai`：图片生成后端

### 文档设计亮点
- 聚焦最常见的问题，给出具体检查点和命令。
- 强调使用 HTTP 服务器预览 SVG，避免新手直接双击 SVG 导致的路径问题。
- 明确推荐流水线顺序，防止用户跳步。

---

## 文档目录总结

| 文档 | 覆盖脚本 | 核心价值 |
|------|----------|----------|
| `conversion.md` | 源文件转换类 | 指导如何将各种输入转为 Markdown |
| `image.md` | 图片生成与分析类 | 配置多后端、生成图片、去水印、分析图片 |
| `project.md` | 项目管理类 | 创建项目、导入源文件、批量验证、生成示例索引、导入 PPTX 模板、错误帮助 |
| `svg-pipeline.md` | 后处理与导出类 | 统一流水线、SVG 质量检查、坐标计算、高级独立工具 |
| `troubleshooting.md` | 综合 | 常见问题排查步骤和依赖说明 |

这些文档构成了 PPT Master 的**用户手册**，与之前解析的脚本代码形成互补：代码展示“如何实现”，文档展示“如何使用”。对于 AI 角色（策略师、执行师）来说，这些文档是执行任务时的参考，也是用户在命令行环境操作的指南。

---

如果你已完成全部脚本和文档的解析，我们可以进入下一阶段：总结整个项目的架构、设计模式、核心算法和工程智慧。请告知。