## 辅助脚本详细解析

你已经完成了源文件转换、项目管理、图片处理、SVG 后处理、PPTX 导出等核心模块的解析。现在进入 `scripts/` 目录下剩余的**辅助脚本**。它们为项目提供批量验证、示例索引生成、动画 XML 生成、PPTX 模板导入、SVG 位置计算、质量检查、仓库更新等辅助功能。

以下逐文件解析。

---

# 一、batch_validate.py —— 批量项目验证工具

### 1. 核心定位

对多个项目目录（如 `examples/`、`projects/`）进行批量结构验证，输出统计摘要和问题列表。可用于 CI 或定期检查项目健康度。

### 2. 核心类 `BatchValidator`

#### 主要属性
- `results`: 存储每个项目的验证结果字典
- `summary`: 统计总数、有效数、有错误数、有警告数、缺少 README 数、缺少设计规范数、SVG 问题数

#### 主要方法

**`validate_directory(directory, recursive)`**
- 调用 `find_all_projects(directory)`（来自 `project_utils`）获取所有项目路径
- 对每个项目调用 `validate_project()`

**`validate_project(project_path)`**
- 调用 `get_project_info()` 获取项目基本信息
- 调用 `validate_project_structure()` 检查目录结构（README、设计规范、svg_output 等）
- 调用 `validate_svg_viewbox()` 检查 SVG 的 viewBox 是否与画布格式一致
- 聚合错误和警告，更新统计
- 打印单项目结果（状态、路径、格式、SVG 数量、日期、错误/警告）

**`print_summary()`**
- 输出总体统计：完全有效数、有警告数、有错误数（带百分比）
- 列出常见问题：缺少 README、缺少设计规范、SVG 格式问题
- 按画布格式分组统计项目分布
- 给出修复建议

**`export_report(output_file)`**
- 将所有验证结果写入文本文件，包括每个项目的详细错误/警告

### 3. 使用方式

```bash
# 验证 examples 目录下的所有项目
python3 batch_validate.py examples

# 验证多个目录
python3 batch_validate.py examples projects

# 验证 examples 和 projects（使用 --all）
python3 batch_validate.py --all

# 导出报告（默认 validation_report.txt，可用 --output 指定）
python3 batch_validate.py examples --export --output my_report.txt
```

### 4. 设计亮点

- **聚合验证**：一次性检查多个项目，节省时间。
- **统计分类**：清晰区分有效、警告、错误，便于优先级排序。
- **格式分布**：了解不同画布格式的使用频率。
- **CI 友好**：退出码：0（全部有效）、1（有错误）、2（有警告），便于集成到 CI 流水线。

---

# 二、generate_examples_index.py —— 示例索引生成器

### 1. 核心定位

自动扫描 `examples/` 目录下的所有项目，生成 `examples/README.md` 索引文件，供用户浏览示例。

### 2. 主要函数 `generate_examples_index(examples_dir)`

- 调用 `find_all_projects(examples_dir)` 获取所有项目
- 对每个项目调用 `get_project_info()` 获取名称、格式、日期、SVG 数量等
- 按日期排序（最新的在前）
- 按画布格式分组
- 生成 Markdown 内容，包括：
  - 总体统计（项目数、格式类型数、SVG 文件总数）
  - 格式分布
  - 最近更新的 5 个项目
  - 按格式分类的项目列表（链接到项目目录）
  - 使用说明（HTTP 服务器预览、直接打开 SVG）
  - 创建新项目指南
  - 贡献示例项目的要求和流程
  - 相关资源链接
- 返回 Markdown 字符串

### 3. 使用方式

```bash
# 默认扫描 examples/ 目录
python3 generate_examples_index.py

# 指定目录
python3 generate_examples_index.py /path/to/examples
```

### 4. 设计亮点

- **自动化文档**：避免手动维护示例列表。
- **分类清晰**：按格式分组，用户可快速找到特定画布格式的示例。
- **时效性**：按日期排序，突出最新项目。
- **可扩展**：易于添加更多统计信息或自定义模板。

---

# 三、pptx_animations.py —— PPTX 动画模块

### 1. 核心定位

为导出的 PPTX 添加幻灯片过渡效果和进入动画的 XML 生成器。**纯 XML 生成，无外部依赖**。

### 2. 过渡效果定义 `TRANSITIONS`

| 效果键 | 名称 | XML 元素 | 属性 |
|--------|------|----------|------|
| `fade` | 淡入淡出 | `fade` | 无 |
| `push` | 推送 | `push` | `dir="r"`（从右推） |
| `wipe` | 擦除 | `wipe` | `dir="r"` |
| `split` | 分裂 | `split` | `orient="horz" dir="out"` |
| `strips` | 条纹 | `strips` | `dir="rd"`（右下角对角） |
| `cover` | 覆盖 | `cover` | `dir="r"` |
| `random` | 随机 | `random` | 无 |

### 3. 核心函数 `create_transition_xml(effect, duration, advance_after)`

- 生成 `<p:transition>` 元素，包含 `dur`（毫秒）和可选的 `advTm`（自动切换时间）。
- 内部添加对应的效果元素（如 `<p:fade/>`）。
- 返回 XML 字符串，可嵌入幻灯片 XML 的 `</p:sld>` 之前。

### 4. 进入动画定义 `ANIMATIONS`

| 动画 | 名称 | 过滤器 | 备注 |
|------|------|--------|------|
| `fade` | 淡入 | `fade` | |
| `fly` | 飞入 | `fly` | `prLst="from(b)"`（从底部） |
| `zoom` | 缩放 | `zoom` | `prLst="in"` |
| `appear` | 出现 | 无 | 直接设置可见性 |

### 5. 核心函数 `create_timing_xml(animation, duration, delay, shape_id)`

- 生成复杂的 `<p:timing>` 结构，包含设置可见性和动画效果。
- 对于 `appear` 只设置可见性；其他动画在可见性设置后附加 `animEffect`。
- 使用 `spTgt spid` 指定目标形状 ID（通常 SVG 图片的形状 ID 为 2）。

### 6. 辅助函数

- `get_available_transitions()` / `get_available_animations()`：列出可用效果。
- `get_transition_help()` / `get_animation_help()`：返回帮助文本。

### 7. 设计亮点

- **无依赖**：直接生成 XML，不依赖 `python-pptx` 的动画 API（后者支持有限）。
- **兼容性**：生成的 XML 符合 Office Open XML 标准。
- **可扩展**：添加新效果只需在字典中定义。

---

# 四、pptx_template_import.py —— PPTX 模板导入工具

### 1. 核心定位

从现有的 `.pptx` 文件中提取**轻量级模板资产和样式元数据**，生成 `manifest.json`、`analysis.md` 和 `assets/` 目录，供后续模板重建使用。

> **重要**：此工具**不**尝试将任意 PPTX 形状转换为 SVG 模板，而是提取可复用的背景、图片、主题颜色、字体等，并分析每页的类型（封面、目录、章节、内容、结尾候选），为 `template-designer` 提供参考。

### 2. 提取内容

- **幻灯片尺寸**：宽度/高度（EMU 和像素）
- **主题颜色**：从 `<a:clrScheme>` 提取
- **主题字体**：从 `<a:fontScheme>` 提取（majorLatin, minorLatin, minorEastAsia）
- **媒体资产**：`ppt/media/` 下的所有图片，复制到 `assets/` 目录，重名冲突自动加后缀
- **幻灯片分析**：对每页提取背景图片（从 slide/layout/master 继承）、图片引用、文本样本、形状数量，并分类页面类型（`cover_candidate`, `toc_candidate`, `chapter_candidate`, `ending_candidate`, `content_candidate`）

### 3. 页面类型分类规则

| 类型 | 判定条件 |
|------|----------|
| `ending_candidate` | 文本包含 thanks/致谢/联系方式 等关键词，或最后一页且文本数 ≤ 6 |
| `toc_candidate` | 文本包含 agenda/contents/目录/议程 等关键词 |
| `chapter_candidate` | 文本包含 chapter/part/章节 等关键词，或文本数 ≤ 3 且形状数 ≤ 12 |
| `cover_candidate` | 第一页且图片数 ≤ 3 |
| `content_candidate` | 其他情况 |

### 4. 输出文件

- `manifest.json`：包含源信息、幻灯片尺寸、主题、资产列表、页面类型候选、每页详细数据
- `analysis.md`：人类可读的分析报告
- `assets/`：提取的图片文件

### 5. 使用方式

```bash
python3 pptx_template_import.py template.pptx
python3 pptx_template_import.py template.pptx -o ./output_dir
```

### 6. 设计亮点

- **有限目标**：不尝试完全逆向工程 PPTX，只提取模板所需的关键元素。
- **继承链追踪**：背景图片可能来自 slide → layout → master，正确找到最终来源。
- **页面类型启发**：为模板设计师提供页面角色建议。
- **资产规范化**：重命名、去重，避免文件名冲突。

---

# 五、svg_position_calculator.py —— SVG 位置计算与验证工具

### 1. 核心定位

为**图表设计师**或**AI 生成图表**提供精确的坐标计算和验证。支持柱状图、饼图、雷达图、折线图、网格布局等，可交互或批量计算，并验证生成的 SVG 坐标是否符合预期。

### 2. 核心类

#### `CoordinateSystem`
- 管理画布尺寸和图表区域（`ChartArea`）。
- 提供 `data_to_svg_x/y` 将数据域映射到 SVG 坐标（Y 轴反转）。

#### `BarChartCalculator`
- 计算垂直/水平柱状图每个柱子的位置、宽度、高度、标签位置、数值位置。
- 支持自动布局（根据柱数、间隙比例计算起始 X）。
- `format_table()` 输出表格。

#### `PieChartCalculator`
- 计算饼图/环形图的扇区，生成 SVG 路径 `d` 属性。
- 支持内半径（环形图）和起始角度。
- 计算标签位置（在半径 70% 处）。

#### `RadarChartCalculator`
- 计算雷达图各维度顶点坐标，支持网格层生成。
- 输出顶点绝对坐标和相对坐标。

#### `LineChartCalculator`
- 将数据点映射到 SVG 坐标，生成折线路径。
- 支持自定义 X/Y 轴范围。

#### `GridLayoutCalculator`
- 在图表区域内生成 N×M 网格布局，计算每个单元格的位置和尺寸。

#### `SVGPositionValidator`
- 从 SVG 文件中提取元素坐标，与期望值比较，允许偏差（tolerance）。
- 可提取所有元素位置。

### 3. 使用方式

```bash
# 分析 SVG 文件中的所有图表元素
python3 svg_position_calculator.py analyze chart.svg

# 快速计算柱状图
python3 svg_position_calculator.py calc bar --data "East:185,South:142,West:128"

# 计算饼图
python3 svg_position_calculator.py calc pie --data "A:35,B:25,C:20" --center 420,400 --radius 200

# 计算雷达图
python3 svg_position_calculator.py calc radar --data "Speed:90,Power:85,Accuracy:75"

# 交互模式（引导式输入）
python3 svg_position_calculator.py interactive

# 从 JSON 配置文件读取
python3 svg_position_calculator.py from-json config.json
```

### 4. 设计亮点

- **无绘图依赖**：只计算坐标，不依赖 matplotlib 等绘图库。
- **灵活数据输入**：支持 `label:value` 格式，易于解析。
- **精确路径生成**：饼图/环形图使用贝塞尔近似弧段，生成可直接用于 SVG 的 `d` 属性。
- **验证功能**：可对比期望坐标与实际 SVG 坐标，用于测试自动化。
- **交互式引导**：适合不熟悉命令行的用户。

---

# 六、svg_quality_checker.py —— SVG 质量检查工具

### 1. 核心定位

检查 SVG 文件是否符合 PPT Master 技术规范（禁止元素、字体、viewBox 等）。可单文件、目录或批量检查所有示例项目。

### 2. 检查项

| 类别 | 检查内容 |
|------|----------|
| viewBox | 是否存在、格式是否正确、是否匹配期望格式 |
| 禁止元素 | `clipPath`, `mask`, `<style>`, `class`, `foreignObject`, `symbol`+`use`, `marker`, `marker-end`, `textPath`, `@font-face`, `<animate*`, `<set`, `<script`, 事件属性, `<iframe`, `rgba()`, `<g opacity`, `<image opacity` |
| 字体 | 是否使用系统 UI 字体栈（`system-ui`, `-apple-system`, `BlinkMacSystemFont`, `Segoe UI`） |
| 尺寸 | `width`/`height` 是否与 viewBox 一致 |
| 文本 | 是否存在过长单行文本（>100 字符），提示使用 `<tspan>` 换行 |

### 3. 核心类 `SVGQualityChecker`

- `check_file()`: 检查单个文件，返回结果字典（包含错误、警告、信息）。
- `check_directory()`: 检查目录下所有 SVG 文件。
- `print_summary()`: 输出统计摘要和常见修复建议。
- `export_report()`: 导出详细报告。

### 4. 使用方式

```bash
# 检查单个 SVG 文件
python3 svg_quality_checker.py slide.svg

# 检查目录（如 svg_output）
python3 svg_quality_checker.py examples/project/svg_output

# 检查整个项目（自动定位 svg_output）
python3 svg_quality_checker.py examples/project

# 检查所有示例项目
python3 svg_quality_checker.py --all examples

# 指定期望格式（用于 viewBox 匹配）
python3 svg_quality_checker.py slide.svg --format ppt169

# 导出报告
python3 svg_quality_checker.py examples --export --output my_report.txt
```

### 5. 设计亮点

- **全面覆盖**：检查所有 SKILL.md 中禁止的 SVG 特性。
- **上下文感知**：`id` 属性仅在 `<style>` 存在时报错（因为 `id` 本身无害，但与 CSS 选择器组合才禁止）。
- **集成 `error_helper`**：如果可用，错误信息会附带修复建议。
- **退出码**：有错误时退出 1，便于 CI 集成。

---

# 七、update_repo.py —— 仓库更新脚本

### 1. 核心定位

一键拉取最新代码，并在 `requirements.txt` 发生变化时自动同步 Python 依赖。**要求工作区干净（无未提交的跟踪文件修改）**。

### 2. 执行流程

1. 检查 `git` 是否可用。
2. 检查是否有未提交的跟踪文件修改（`git status --porcelain --untracked-files=no`），有则报错退出。
3. 记录当前 HEAD 和 `requirements.txt` 的 SHA-256。
4. 执行 `git pull --ff-only`（仅快进合并）。
5. 比较 HEAD 和 requirements 哈希。
6. 如果 requirements 变化且未指定 `--skip-pip`，则运行 `pip install -r requirements.txt`。
7. 提示用户仍需手动安装系统依赖（Node.js, Pandoc 等）。

### 3. 使用方式

```bash
# 正常更新
python3 update_repo.py

# 跳过 pip 安装
python3 update_repo.py --skip-pip
```

### 4. 设计亮点

- **安全第一**：要求工作区干净，避免冲突或丢失更改。
- **智能依赖同步**：只在 requirements 变化时安装，节省时间。
- **仅快进合并**：避免非快进合并导致的复杂冲突。
- **轻量级**：只依赖 Python 标准库。

---

## 辅助脚本总结

| 脚本 | 主要用途 | 使用场景 |
|------|----------|----------|
| `batch_validate.py` | 批量验证项目结构 | 维护质量、CI |
| `generate_examples_index.py` | 自动生成示例索引 | 文档维护 |
| `pptx_animations.py` | 生成过渡/动画 XML | PPTX 导出增强 |
| `pptx_template_import.py` | 从 PPTX 提取模板资产 | 模板设计辅助 |
| `svg_position_calculator.py` | 计算图表坐标 | 图表生成辅助 |
| `svg_quality_checker.py` | 检查 SVG 合规性 | 质量保证 |
| `update_repo.py` | 更新仓库并同步依赖 | 日常维护 |

这些脚本虽不直接参与主流水线，但为**开发、维护、质量保证**提供了重要支持，体现了项目的工程成熟度。

---

如果需要继续解析 `scripts/docs/` 目录下的文档文件，请告知。