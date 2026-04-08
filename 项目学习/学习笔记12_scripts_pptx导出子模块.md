## pptx 导出子模块完整解析

`svg_to_pptx` 包是 PPT Master 的**核心转换引擎**，负责将后处理后的 SVG 文件（`svg_final/`）转换为真正的 PowerPoint 原生形状（DrawingML）。整个包分为以下几个逻辑组：

| 模块 | 职责 |
|------|------|
| `drawingml_context.py` | 共享上下文，传递变换矩阵、继承样式、计数器 |
| `drawingml_utils.py` | 坐标转换、颜色解析、字体映射、XML 转义等工具函数 |
| `drawingml_styles.py` | 填充、描边、阴影、渐变、效果（发光/阴影）XML 构建 |
| `drawingml_paths.py` | SVG 路径解析、标准化、弧转贝塞尔、生成 DrawingML 路径命令 |
| `drawingml_elements.py` | 各类 SVG 元素（rect, circle, line, path, polygon, polyline, text, image, ellipse）到 DrawingML 的转换器 |
| `drawingml_converter.py` | 顶层调度器，处理 `<g>` 分组、`<defs>` 收集，协调各元素转换 |
| `pptx_builder.py` | 核心构建器：创建 PPTX 文件结构，组装幻灯片 XML、媒体文件、关系文件 |
| `pptx_cli.py` | 命令行入口，参数解析，生成 native 和 legacy 两个版本 |
| `pptx_dimensions.py` | 画布尺寸、EMU 转换、格式检测 |
| `pptx_discovery.py` | 查找项目中的 SVG 文件和演讲备注文件 |
| `pptx_media.py` | SVG 转 PNG（兼容模式），支持 CairoSVG 和 svglib |
| `pptx_notes.py` | Markdown 备注转纯文本，生成备注幻灯片 XML |
| `pptx_slide_xml.py` | 生成幻灯片 XML 和关系文件（兼容模式 / 纯 SVG 模式） |

---

## 一、上下文与工具类

### 1. `drawingml_context.py` – `ConvertContext`

**核心职责**：在递归遍历 SVG 树时传递状态信息，包括：
- `defs`：已收集的滤镜、渐变定义（id → ET.Element）
- `translate_x, translate_y, scale_x, scale_y`：累计的变换矩阵
- `filter_id`：当前生效的滤镜 ID（继承自父级）
- `inherited_styles`：从父 `<g>` 继承的样式（fill, stroke, font 等）
- `id_counter`, `rel_id_counter`：分配形状 ID 和关系 ID
- `media_files`, `rel_entries`：收集的媒体文件和关系条目

**关键方法**：
- `next_id()` / `next_rel_id()`：分配唯一标识符
- `child(dx, dy, sx, sy, filter_id, style_overrides)`：创建子上下文，更新变换和样式（opacity 会累乘，其他样式覆盖）
- `sync_from_child()`：将子上下文的计数器同步回父级

**设计亮点**：
- 透明度累乘逻辑：子元素 `opacity` 与父元素 `opacity` 相乘，符合 SVG 合成规则。
- 变换累积：`translate_x += dx`, `scale_x *= sx`，正确实现嵌套 `<g>` 的坐标变换。

### 2. `drawingml_utils.py` – 工具函数集

#### 常量
- `EMU_PER_PX = 9525`：1 SVG 像素 = 9525 EMU（基于 96 DPI）
- `FONT_PX_TO_HUNDREDTHS_PT = 75`：1px = 0.75pt = 75 百分之一 pt
- `ANGLE_UNIT = 60000`：DrawingML 角度单位（1° = 60000）
- `INHERITABLE_ATTRS`：可从父元素继承的 SVG 属性列表
- `EA_FONTS`：东亚字体集合
- `FONT_FALLBACK_WIN`：macOS/Linux 字体到 Windows 字体的映射表
- `DASH_PRESETS`：常用 stroke-dasharray 到 DrawingML 预设虚线类型的映射

#### 坐标辅助
- `px_to_emu(px)`：像素转 EMU
- `_f(val, default)`：安全浮点数转换
- `ctx_x, ctx_y, ctx_w, ctx_h`：应用当前上下文的缩放/平移

#### 颜色与样式
- `parse_hex_color`：`#RRGGBB` → `RRGGBB`
- `parse_stop_style`：解析渐变停止点的样式字符串
- `resolve_url_id`：从 `url(#id)` 中提取 id
- `get_effective_filter_id`：获取元素自身的滤镜或继承的滤镜

#### 字体处理
- `parse_font_family(font_family_str)`：解析 CSS `font-family`，返回 `{'latin': '...', 'ea': '...'}`。将 macOS/Linux 特有字体映射到 Windows 可用字体（如 PingFang SC → Microsoft YaHei）。
- `is_cjk_char(ch)`：判断是否为 CJK 字符（基于 Unicode 码位范围）。
- `estimate_text_width(text, font_size, font_weight)`：估算文本宽度（用于文本框自动调整大小）。中文字符宽度 ≈ `font_size`，英文字符按比例估算，粗体加 5% 宽度。

#### XML 转义
- `_xml_escape`：替换 `&`, `<`, `>`, `"`。

---

## 二、样式与效果构建

### `drawingml_styles.py` – 填充、描边、阴影、渐变

#### 填充
- `build_solid_fill(color, opacity)`：生成 `<a:solidFill><a:srgbClr val="..."><a:alpha val="..."/></a:srgbClr></a:solidFill>`。
- `build_gradient_fill(grad_elem, opacity)`：将 SVG 的 `<linearGradient>` 或 `<radialGradient>` 转换为 DrawingML `<a:gradFill>`。线性渐变计算角度（`atan2(y2-y1, x2-x1)`），径向渐变使用 `path="circle"`。停止点颜色和透明度通过 `<a:gs>` 表示。
- `build_fill_xml(elem, ctx, opacity)`：根据元素 `fill` 属性（可能是 `none`、颜色值、`url(#gradient)`）选择填充方式。

#### 描边
- `build_stroke_xml(elem, ctx, opacity)`：处理 `stroke`、`stroke-width`、`stroke-dasharray`、`stroke-linecap`、`stroke-linejoin`。虚线预设（`4,4` → `dash`）或自定义 `custDash`（按比例计算）。线帽映射：`round` → `rnd`, `square` → `sq`, `butt` → `flat`。线连接：`round` → `<a:round/>`，`bevel` → `<a:bevel/>`，`miter` → `<a:miter lim="800000"/>`。

#### 效果（阴影/发光）
- `_parse_filter_params(filter_elem)`：从 SVG `<filter>` 中提取 `stdDeviation`, `dx`, `dy`, `opacity`, `color`, `has_offset`。
- `build_shadow_xml(filter_elem)`：生成 `<a:outerShdw>`。计算模糊半径（`blurRad = px_to_emu(std_dev * 2)`）、距离（`dist = px_to_emu(sqrt(dx²+dy²))`）、方向角度。
- `build_glow_xml(filter_elem)`：生成 `<a:glow>`（无偏移的模糊效果）。
- `build_effect_xml(filter_elem)`：根据是否存在 `feOffset` 自动选择阴影或发光。

#### 透明度辅助
- `get_element_opacity(elem)`：获取元素 `opacity` 属性。
- `get_fill_opacity(elem, ctx)`：结合 `opacity` 和 `fill-opacity` 相乘。
- `get_stroke_opacity(elem, ctx)`：结合 `opacity` 和 `stroke-opacity` 相乘。

---

## 三、路径处理

### `drawingml_paths.py` – SVG 路径解析与转换

#### 数据结构
- `PathCommand(cmd, args)`：表示一个路径命令（`M`, `L`, `C`, `Z` 等）。

#### 解析
- `parse_svg_path(d)`：正则分词，按命令分割参数，返回 `PathCommand` 列表。支持连续的隐含命令（如 `M 10 20 30 40` 后续为 `L`）。

#### 标准化
- `svg_path_to_absolute(commands)`：将所有相对命令（`m`, `l`, `c` 等）转换为绝对坐标。维护当前点 `(cx, cy)` 和子路径起点 `(sx, sy)`。
- `normalize_path_commands(commands)`：将 `S`, `Q`, `T`, `A` 转换为 `C` 命令（三次贝塞尔）。具体包括：
  - `S`：反射上一个控制点，生成 `C`。
  - `Q`：二次贝塞尔转三次贝塞尔（`_quad_to_cubic`）。
  - `T`：反射上一个二次控制点，再转三次。
  - `A`：椭圆弧转多段三次贝塞尔（`_arc_to_cubic_beziers`，采用 SVG 规范 F.6.5 算法）。

#### 弧转贝塞尔
- `_arc_to_cubic_beziers`：将端点参数化的弧转换为中心参数化，再分割成最多 90° 的小弧，每个小弧用三次贝塞尔近似（`α = 4/3 * tan(Δθ/4)`）。返回一系列 `C` 命令。

#### 生成 DrawingML 路径
- `path_commands_to_drawingml(commands, offset_x, offset_y, scale_x, scale_y)`：
  1. 计算所有点的包围盒 `(min_x, min_y, max_x, max_y)`。
  2. 将每个命令的坐标转换为相对于包围盒左上角的 EMU 值。
  3. 生成 DrawingML 路径命令：`<a:moveTo>`, `<a:lnTo>`, `<a:cubicBezTo>`, `<a:close/>`。
  4. 返回 `(path_xml, min_x, min_y, width, height)`。

---

## 四、元素转换器

### `drawingml_elements.py` – 单个 SVG 元素 → DrawingML 形状

每个转换器接收 `ET.Element` 和 `ConvertContext`，返回一段 DrawingML XML 字符串。

#### 通用包装函数 `_wrap_shape`
生成 `<p:sp>` 结构，包括：
- `<p:nvSpPr>`：形状 ID 和名称
- `<p:spPr>`：变换 (`<a:xfrm>`)、几何 (`<a:prstGeom>` 或 `<a:custGeom>`)、填充、描边、效果
- 可选的 `extra_xml`（如文本框内容）

#### 各元素转换细节

**`rect`**：
- 获取 `x, y, width, height`，应用上下文变换。
- 使用 `prstGeom prst="rect"`（预定义矩形几何）。
- 支持 `transform` 中的 `rotate`。

**`circle` / `ellipse`**：
- 标准圆/椭圆使用 `prstGeom prst="ellipse"`。
- **环形图弧段检测**：当 `<circle>` 有 `stroke-dasharray` 且 `stroke-width / r >= 0.15` 时，视为环形图扇区。调用 `_build_arc_ring_path` 生成自定义路径（外弧→内弧→闭合），并用描边颜色作为填充。这是因为 PowerPoint 无法直接渲染带虚线描边的环形弧段。

**`line`**：
- 转换为 `custGeom`，包含两个点 `moveTo` 和 `lnTo`。无填充，只有描边。

**`path`**：
- 解析 `d` 属性，标准化命令，生成 `custGeom`。支持填充和描边。

**`polygon` / `polyline`**：
- 解析 `points` 字符串，生成 `M L ... Z`（多边形）或 `M L ...`（折线）。多边形有填充，折线无填充。

**`text`**：
- 最复杂的转换器。流程：
  1. 获取 `x, y, font-size, font-family, text-anchor, fill, opacity` 等属性。
  2. 递归处理 `<tspan>` 子元素，为每个文本片段生成独立的文本运行（run）。
  3. 估算文本宽度和高度，创建文本框形状（`<p:sp>` with `txBox="1"`）。
  4. 根据 `text-anchor`（`start`/`middle`/`end`）调整文本框位置。
  5. 生成 `<a:r>`（文本运行），每个运行包含 `<a:rPr>`（字体、大小、粗体、斜体、下划线、删除线、颜色/渐变）和 `<a:t>`（文本内容）。
  6. 支持文本阴影/发光（通过 `filter` 属性）。

**`image`**：
- 处理 `<image>` 的 `href` 属性（支持 `data:image/...;base64` 和外部文件路径）。
- 保存图片字节到 `ctx.media_files`，生成关系 ID 和关系条目。
- 输出 `<p:pic>`（图片形状），使用 `blipFill` 拉伸填充。

**`ellipse`**：同 `circle` 类似。

---

## 五、顶层调度与分组

### `drawingml_converter.py` – 核心转换调度

#### `collect_defs(root)`
遍历 SVG 根元素，收集所有 `<defs>` 下的子元素，以 `id` 为键存入字典。

#### `parse_transform(transform_str)`
提取 `translate(dx, dy)` 和 `scale(sx, sy)`，返回 `(dx, dy, sx, sy)`。

#### `convert_g(elem, ctx)` – 分组处理
- 解析 `<g>` 的 `transform`，创建子上下文。
- 递归转换所有子元素。
- **扁平化优化**：如果只有一个子元素，直接返回子元素 XML（避免多余分组）。
- 如果有多个子元素，计算所有子形状的包围盒，生成 `<p:grpSp>`（组形状），设置 `chOff` 和 `chExt` 为组本身的偏移和尺寸。这样 PowerPoint 中可以将整个组作为单一对象移动。

#### `convert_element(elem, ctx)`
根据标签名（移除命名空间）派发到对应的转换器（`convert_rect`, `convert_circle`, ...）。未知标签或 `defs`/`title` 等忽略。

#### `convert_svg_to_slide_shapes(svg_path, slide_num, verbose)`
- 解析 SVG 文件，收集 `<defs>`。
- 创建初始 `ConvertContext`。
- 遍历根元素，调用 `convert_element` 收集所有形状 XML。
- 组装完整的幻灯片 XML（包含 `<p:sld>` 根、`<p:spTree>` 容器、`<p:grpSpPr>` 等）。
- 返回 `(slide_xml, media_files, rel_entries)`。

---

## 六、PPTX 构建与组装

### `pptx_builder.py` – `create_pptx_with_native_svg`

这是最顶层的构建函数，负责生成最终的 `.pptx` 文件。

**主要流程**：
1. **参数处理**：判断是否使用原生形状模式（`use_native_shapes=True`）或兼容模式（PNG+SVG）。兼容模式需要检测 PNG 渲染器（cairosvg / svglib）。
2. **尺寸确定**：根据 `canvas_format` 或从第一个 SVG 的 `viewBox` 检测画布尺寸，转换为 EMU。
3. **创建临时目录**：使用 `tempfile.mkdtemp()`。
4. **生成基础 PPTX**：用 `python-pptx` 创建空白演示文稿，添加与 SVG 数量相同的空白幻灯片，保存后解压。
5. **逐页处理**：
   - **原生形状模式**：调用 `convert_svg_to_slide_shapes` 获得幻灯片 XML、媒体文件和关系条目。将媒体文件写入 `ppt/media/`，幻灯片 XML 写入 `ppt/slides/slideN.xml`，关系文件写入 `ppt/slides/_rels/slideN.xml.rels`。同时更新 `[Content_Types].xml` 添加媒体类型。
   - **兼容模式**：将 SVG 文件复制到 `ppt/media/`，调用 `convert_svg_to_png` 生成 PNG 回退图。使用 `create_slide_xml_with_svg` 生成幻灯片 XML（嵌入 PNG 作为主图片，SVG 作为扩展），关系文件引用 PNG 和 SVG。
   - **演讲备注**：如果启用，读取 `notes` 字典（`svg_stem` → 备注内容），调用 `markdown_to_plain_text` 转换，生成备注幻灯片 XML（`ppt/notesSlides/notesSlideN.xml`）和对应的关系文件，并更新幻灯片的关系文件添加备注关联。
6. **更新 `[Content_Types].xml`**：添加 SVG、PNG、JPEG 等扩展名的默认内容类型，以及备注幻灯片的覆盖类型。
7. **重新打包**：将临时目录中的所有文件打包为 `.pptx` 文件。
8. **清理临时目录**。

**关键细节**：
- 原生形状模式下，使用 `use_native_shapes=True`，禁用兼容模式。
- 兼容模式需要 `cairosvg` 或 `svglib`，否则降级为纯 SVG（仅 Office 2019+ 支持）。
- 过渡效果通过 `create_transition_xml` 生成（如果 `pptx_animations` 可用）。
- 演讲备注：`markdown_to_plain_text` 将 `#` 标题、列表 `-` 转换为纯文本段落，合并空行。

### `pptx_cli.py` – 命令行接口

**主要功能**：
- 解析命令行参数：项目路径、输出文件、SVG 源目录（`-s final`）、画布格式（`-f`）、安静模式、兼容模式开关、过渡效果、演讲备注开关等。
- 确定生成哪些版本：
  - `--only native`：只生成原生形状版（可编辑）。
  - `--only legacy`：只生成 SVG 图片版（不可编辑）。
  - 默认生成两个版本（原生 + SVG 参考）。
- 调用 `find_svg_files` 和 `find_notes_files` 获取输入。
- 调用 `create_pptx_with_native_svg` 生成 PPTX。

### `pptx_dimensions.py` – 尺寸与格式

- `get_slide_dimensions(format, custom_pixels)`：返回 `(width_emu, height_emu)`。
- `get_pixel_dimensions`：返回像素尺寸。
- `get_viewbox_dimensions(svg_path)`：从 SVG 中提取 `viewBox` 的宽高。
- `detect_format_from_svg(svg_path)`：匹配已知画布格式的 `viewBox`。

### `pptx_discovery.py` – 文件查找

- `find_svg_files(project_path, source)`：根据 `source` 参数（`output`/`final`/自定义）查找 SVG 文件列表。
- `find_notes_files(project_path, svg_files)`：扫描 `notes/` 目录下的 `.md` 文件，通过文件名或索引（`slide01.md`）匹配到 SVG 文件，返回 `{svg_stem: content}`。

### `pptx_media.py` – SVG 转 PNG

- 优先使用 `cairosvg`（质量高，支持渐变），回退到 `svglib`。
- `convert_svg_to_png(svg_path, png_path, width, height)`：根据选中的渲染器执行转换。

### `pptx_notes.py` – 演讲备注处理

- `markdown_to_plain_text(md_content)`：将 Markdown 格式的备注转换为纯文本，保留基本结构（标题、列表项），移除加粗标记。
- `create_notes_slide_xml(slide_num, notes_text)`：生成备注幻灯片 XML，包含文本占位符和内容段落。
- `create_notes_slide_rels_xml(slide_num)`：生成备注幻灯片的关系文件，关联到笔记母版和对应的幻灯片。

### `pptx_slide_xml.py` – 幻灯片 XML 生成（兼容模式）

- `create_slide_xml_with_svg(...)`：生成幻灯片 XML，根据 `use_compat_mode` 决定是否嵌入 PNG 作为主图片并附加 SVG 扩展（`<asvg:svgBlip>`）。
- `create_slide_rels_xml(...)`：生成幻灯片关系文件，引用 PNG 和 SVG（兼容模式）或仅 SVG（纯 SVG 模式）。

---

## 七、整体架构与数据流

```
svg_final/*.svg
       │
       ▼
pptx_cli.py (解析参数)
       │
       ├── find_svg_files → svg_files
       ├── find_notes_files → notes dict
       │
       ▼
pptx_builder.create_pptx_with_native_svg
       │
       ├── 创建空白 PPTX 并解压
       │
       ├── 对每个 SVG:
       │   │
       │   ├── use_native_shapes = True
       │   │       │
       │   │       ▼
       │   │   drawingml_converter.convert_svg_to_slide_shapes
       │   │       │
       │   │       ├── collect_defs
       │   │       ├── 遍历根元素 → convert_element
       │   │       │       ├── rect / circle / line / path / polygon / polyline / text / image / ellipse
       │   │       │       └── 内部调用 drawingml_paths, drawingml_styles, drawingml_utils
       │   │       └── 返回 slide_xml, media_files, rel_entries
       │   │
       │   ├── use_native_shapes = False
       │   │       │
       │   │       ├── 复制 SVG 到 media/
       │   │       ├── convert_svg_to_png (兼容模式)
       │   │       └── pptx_slide_xml.create_slide_xml_with_svg
       │   │
       │   └── 写入 slideN.xml, media/, _rels/slideN.xml.rels
       │
       ├── 更新 [Content_Types].xml
       ├── 写入备注幻灯片（如果启用）
       │
       └── 重新打包为 .pptx
```

---

## 八、设计亮点与工程智慧

1. **原生形状转换**：将 SVG 元素转换为 PowerPoint 的 DrawingML 原生形状（`<p:sp>`, `<p:pic>`），使得生成的 PPTX 中每个元素都可以在 PowerPoint 中编辑（移动、缩放、改颜色、改字体等），而不是作为一张图片。

2. **环形图弧段特判**：PowerPoint 不支持带虚线描边的圆，但 PPT Master 通过将每个弧段转换为填充的环形路径（自定义几何）实现了环形图效果，这是 SVG 到 PPT 转换中的精巧 hack。

3. **文本估算与自动换行**：通过 `estimate_text_width` 估算文本宽度，为文本框设置合适的大小，避免文本溢出或裁剪。支持 `text-anchor` 对齐。

4. **字体跨平台映射**：`parse_font_family` 将 macOS/Linux 特有字体（如 PingFang SC）映射到 Windows 可用字体（Microsoft YaHei），确保在不同操作系统上生成的 PPTX 打开后字体正确。

5. **继承样式处理**：`ConvertContext` 维护 `inherited_styles`，子元素自动继承父 `<g>` 的 `fill`、`stroke`、`font-family` 等属性，同时 opacity 累乘，符合 SVG 规范。

6. **滤镜转换为 PPT 效果**：将 SVG 的 `<filter>`（`feGaussianBlur` + `feOffset`）转换为 PowerPoint 的 `<a:outerShdw>`（阴影）或 `<a:glow>`（发光），保留视觉深度。

7. **分组保留**：将 `<g>` 转换为 `<p:grpSp>`，使得 PowerPoint 中可以将一组形状作为整体移动和编辑，而不是打散。

8. **兼容模式**：对于不支持 SVG 的旧版 Office，提供 PNG 回退图片，同时在新版 Office 中仍然显示可编辑的 SVG（通过 `<asvg:svgBlip>` 扩展）。这通过 `pptx_slide_xml` 中的特殊命名空间实现。

9. **媒体与关系管理**：统一收集图片字节、生成关系 ID，自动维护 `_rels` 文件，确保 PPTX 内部引用正确。

10. **错误容忍**：单个元素转换失败不会导致整个幻灯片崩溃，会输出警告并跳过该元素。

---

通过以上模块的协作，`svg_to_pptx` 成功地将符合 PPT Master 规范的 SVG 文件转换为高质量的、原生可编辑的 PowerPoint 演示文稿。