## SVG 后处理流水线（核心三兄弟）详细解析

你提供的这三个脚本是 PPT Master 后处理阶段的核心，它们按顺序执行：**分割演讲备注 → 最终处理 SVG → 导出 PPTX**。以下逐一深入解析。

---

# 一、total_md_split.py —— 演讲备注分割工具

### 1. 核心定位

将项目 `notes/total.md` 中的完整演讲备注文档，按一级标题（`#`）分割成多个独立文件，每个文件对应一个 SVG 页面。分割后的文件保存在 `notes/` 目录下，文件名与 SVG 文件名一致（如 `01_cover.md`）。

### 2. 关键常量

```python
HEADING_RE = re.compile(r'^(#{1,6})\s*(.+?)\s*$')   # 匹配任意级别标题
HR_RE = re.compile(r'^\s*[-*]{3,}\s*$')            # 匹配水平分隔线（---）
```

### 3. 核心函数解析

#### 3.1 `normalize_title(title)` —— 标题规范化

用于模糊匹配 SVG 文件名与备注标题。

```python
def normalize_title(title: str) -> str:
    # 移除标点符号，替换为下划线，转为小写
    text = re.sub(r'[^0-9A-Za-z\u4e00-\u9fff]+', '_', title)
    text = re.sub(r'_+', '_', text).strip('_')
    return text.lower()
```

- 保留字母、数字、中文字符，其他转为下划线。
- 连续下划线压缩为单个。

#### 3.2 `extract_leading_number(text)` —— 提取前导数字

从文本中提取开头的数字（1-3 位），支持以下模式：
- `"01_cover"` → 1
- `"Slide 5"` → 5
- `"第3页"` → 3

用于将备注标题中的编号与 SVG 文件名中的编号匹配。

#### 3.3 `build_match_maps(svg_stems)` —— 构建匹配映射

为所有 SVG 文件名（无扩展名）建立三种映射：
- `exact`: 精确匹配集合（原样）
- `norm_map`: 规范化后 → 原始文件名列表
- `num_map`: 提取的数字 → 原始文件名列表

```python
exact, norm_map, num_map = build_match_maps(svg_stems)
```

#### 3.4 `match_title(raw_title, exact, norm_map, num_map, svg_stems)` —— 标题匹配

匹配优先级（从高到低）：
1. 精确匹配（`raw_title in exact`）
2. 规范化后唯一匹配（`norm_map` 中只有一个候选）
3. 数字提取后唯一匹配（`num_map` 中只有一个候选）
4. 模糊包含匹配（规范化后的 `norm` 包含在某个 SVG 文件名中，且唯一）

#### 3.5 `parse_total_md(md_path, svg_stems, verbose)` —— 解析 total.md

- 逐行读取 `total.md`。
- 遇到标题行时，尝试匹配到 SVG 文件名。
- 若匹配成功，则切换当前备注段落。
- 忽略不匹配的标题（发出警告）。
- 跳过 `---` 分隔线。
- 返回字典 `{svg_stem: notes_content}`。

#### 3.6 `check_svg_note_mapping(svg_files, notes)` —— 检查一对一映射

遍历所有 SVG 文件，检查是否有 SVG 文件在 `notes` 字典中找不到对应备注。若有缺失，返回缺失列表。

#### 3.7 `split_notes(notes, output_dir, verbose)` —— 写入文件

将字典中的每个条目写入 `output_dir/{svg_stem}.md`。

### 4. 使用方式

```bash
python3 total_md_split.py projects/my_project
```

- 默认输出目录：`<project>/notes/`
- 可选 `-o` 指定输出目录。
- 可选 `-q` 安静模式。

### 5. 设计亮点

- **智能匹配**：不要求备注标题与 SVG 文件名完全相同，支持规范化、数字提取、模糊包含等多种匹配策略。
- **严格校验**：必须保证每个 SVG 都有对应的备注文件，否则报错并退出（避免后处理遗漏）。
- **忽略无关标题**：`total.md` 中可以包含其他标题（如元数据），不匹配的会被忽略并警告。
- **无额外依赖**：仅使用 Python 标准库。

---

# 二、finalize_svg.py —— SVG 最终处理工具（统一入口）

### 1. 核心定位

对 `svg_output/` 中的原始 SVG 文件执行一系列后处理操作，输出到 `svg_final/`。这些操作包括：图标嵌入、图片智能裁剪、图片宽高比修正、图片 Base64 嵌入、文本扁平化、圆角矩形转路径。

### 2. 处理选项（6 个）

| 选项 | 对应函数 | 作用 |
|------|----------|------|
| `embed-icons` | `embed_icons_in_file` | 将 `<use data-icon="..."/>` 替换为实际图标 SVG 代码 |
| `crop-images` | `crop_images_in_svg` | 根据 `preserveAspectRatio="slice"` 智能裁剪图片 |
| `fix-aspect` | `fix_image_aspect_in_svg` | 修正图片宽高比，防止 PPT 形状转换时拉伸 |
| `embed-images` | `embed_images_in_svg` | 将外部图片引用转为 Base64 嵌入 |
| `flatten-text` | `flatten_text_with_tspans` | 将 `<tspan>` 展开为独立的 `<text>`（针对特殊渲染器） |
| `fix-rounded` | `process_svg` | 将 `<rect rx="...">` 转换为 `<path>`（PPT 形状转换需要） |

### 3. 执行流程

#### 3.1 主函数 `finalize_project`

1. **检查目录**：验证 `svg_output` 存在且包含 SVG 文件。
2. **复制目录**：将 `svg_output/` 完整复制到 `svg_final/`，所有后续操作都在 `svg_final/` 上进行（保留原始文件）。
3. **顺序执行各步骤**（根据 `options` 开关）：
   - 嵌入图标
   - 智能裁剪图片
   - 修正图片宽高比
   - 嵌入图片（Base64）
   - 文本扁平化
   - 圆角矩形转路径
4. **输出提示**：显示处理的文件数、嵌入的图标数、裁剪的图片数等。

#### 3.2 辅助函数

- `process_flatten_text()`: 调用 `svg_finalize.flatten_tspan.flatten_text_with_tspans` 修改 SVG 树。
- `process_rounded_rect()`: 调用 `svg_finalize.svg_rect_to_path.process_svg` 处理圆角矩形。

### 4. 使用方式

```bash
# 执行所有步骤（推荐）
python3 finalize_svg.py projects/my_project

# 只执行特定步骤
python3 finalize_svg.py projects/my_project --only embed-icons fix-rounded

# 预览模式（不实际修改）
python3 finalize_svg.py projects/my_project --dry-run

# 安静模式
python3 finalize_svg.py projects/my_project -q
```

### 5. 设计亮点

- **统一入口**：将多个独立功能（原为独立脚本）整合到一个命令，简化调用。
- **可选择性执行**：`--only` 参数允许跳过某些步骤（例如调试时只需处理图标）。
- **非破坏性**：操作在 `svg_final/` 副本上进行，原始 `svg_output/` 不变。
- **详细统计**：每个步骤结束后报告处理数量。
- **平台兼容**：`safe_print` 处理 Windows 终端无法显示的 emoji。

---

# 三、svg_to_pptx.py —— SVG 转 PPTX 工具（薄包装）

### 1. 核心定位

这是一个**薄包装器**，实际功能委托给 `svg_to_pptx` 包（位于 `scripts/svg_to_pptx/` 目录）。该文件保持向后兼容性，用户仍然可以使用原来的命令。

### 2. 关键代码

```python
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent))
from svg_to_pptx import main

if __name__ == '__main__':
    main()
```

### 3. 实际功能

真正的转换逻辑在 `svg_to_pptx` 包中，包含以下模块（已在目录树中列出）：
- `pptx_builder.py`: 主构建器
- `drawingml_converter.py`: SVG → DrawingML 转换
- `drawingml_elements.py`, `drawingml_paths.py`, `drawingml_styles.py`, `drawingml_utils.py`: 元素、路径、样式、工具
- `pptx_cli.py`: 命令行接口
- `pptx_dimensions.py`: 尺寸计算
- `pptx_notes.py`: 演讲备注嵌入
- `pptx_media.py`: 媒体资源处理
- `pptx_slide_xml.py`: XML 底层操作

### 4. 典型用法

```bash
python3 svg_to_pptx.py projects/my_project -s final
```

- `-s final` 表示从 `svg_final/` 读取（而非 `svg_output/`）。
- 默认生成两个文件：`<project_name>.pptx`（原生形状）和 `<project_name>_svg.pptx`（SVG 参考版）。

### 5. 设计亮点

- **薄包装**：保持 CLI 稳定，便于未来重构。
- **模块化**：转换逻辑拆分为多个职责单一的模块。

---

## 后处理三兄弟的协作关系

```
Step 1: total_md_split.py
  输入: notes/total.md
  输出: notes/*.md (每个 SVG 一个)
  校验: 确保 SVG 与 notes 一对一

Step 2: finalize_svg.py
  输入: svg_output/*.svg
  输出: svg_final/*.svg
  操作: 图标嵌入、图片裁剪/嵌入、文本扁平化、圆角转路径

Step 3: svg_to_pptx.py
  输入: svg_final/*.svg + notes/*.md
  输出: *.pptx (原生形状 + SVG 参考版)
```

### 强制串行规则（来自 SKILL.md）

> 三个命令必须**逐个单独执行**，每个完成并确认成功后才能运行下一个。  
> ❌ 绝不可将三个命令放在同一个代码块或同一个 shell 调用中。

原因：
1. 每个步骤依赖前一步的输出。
2. 中间步骤可能产生错误（如 notes 与 SVG 不匹配），需要人工介入。
3. 后处理涉及文件 I/O 和转换，并行可能导致竞态条件。

---

## 工程智慧总结

| 脚本 | 核心价值 | 关键技术 |
|------|----------|----------|
| `total_md_split.py` | 确保每页都有演讲备注 | 智能标题匹配、多策略映射 |
| `finalize_svg.py` | 将 SVG 转换为 PPT 兼容格式 | 图标嵌入、图片处理、文本扁平化、圆角转路径 |
| `svg_to_pptx.py` | 将 SVG 转换为真正的 PPTX 形状 | DrawingML 转换、python-pptx 集成 |

### 设计原则

1. **关注点分离**：每个脚本只做一件事，通过组合形成流水线。
2. **可重复性**：操作是确定性的，相同输入总是产生相同输出。
3. **容错性**：`total_md_split.py` 会检测缺失备注并报错，避免后期问题。
4. **非破坏性**：`finalize_svg.py` 在副本上操作，保留原始 SVG。
5. **用户友好**：详细的进度输出和统计信息。

---

如果需要继续深入解析 `svg_to_pptx/` 包内的具体模块（如 `drawingml_converter.py` 或 `pptx_builder.py`），请告知。