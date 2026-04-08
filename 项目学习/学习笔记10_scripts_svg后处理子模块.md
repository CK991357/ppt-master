## `svg_finalize/` 目录 —— SVG 后处理子模块详细解析

`svg_finalize/` 下的六个脚本是 `finalize_svg.py` 统一入口调用的核心处理模块，每个脚本负责一类特定的 SVG 后处理任务。它们共同解决了从 AI 生成的原始 SVG 到 PowerPoint 兼容、高保真 SVG 的转换问题。

---

# 一、crop_images.py —— 智能图片裁剪

### 1. 解决的问题

SVG 中的 `<image>` 元素常使用 `preserveAspectRatio="xMidYMid slice"` 实现类似 CSS `object-fit: cover` 的效果。PowerPoint 在转换 SVG 为形状时会忽略该属性，导致图片被拉伸。此脚本根据 `slice` 模式**实际裁剪图片**，生成符合目标尺寸的新图片，并更新 SVG 引用。

### 2. 核心算法

- 解析 `preserveAspectRatio` 属性，提取对齐方式（如 `xMidYMid`）和模式（`meet`/`slice`）。
- 仅处理模式为 `slice` 的图片。
- 根据对齐方式计算裁剪锚点（`x_anchor`, `y_anchor`），取值 0（左/上）、0.5（中）、1（右/下）。
- 计算目标宽高比，在原图上裁剪出符合该宽高比的最大区域（保持原始分辨率，不缩放）。
- 将裁剪后的图片保存到 `images/cropped/` 目录，并更新 SVG 中的 `href` 路径。
- 移除原 `<image>` 的 `preserveAspectRatio` 属性（因为图片已精确适配）。

### 3. 关键函数

| 函数 | 作用 |
|------|------|
| `parse_preserve_aspect_ratio(attr)` | 解析对齐和模式 |
| `get_crop_anchor(align)` | 对齐字符串 → 锚点 (0~1) |
| `crop_image_to_size(img, target_w, target_h, x_anchor, y_anchor)` | 裁剪图片 |
| `process_svg_images(svg_file, output_dir, dry_run, verbose)` | 处理单个 SVG 中的所有图片 |

### 4. 使用方式

```bash
# 处理单个 SVG
python3 crop_images.py slide.svg

# 批量处理目录下的 SVG
python3 crop_images.py projects/xxx/svg_output

# 预览模式
python3 crop_images.py slide.svg --dry-run
```

### 5. 依赖

- `Pillow`（PIL）必须安装。

### 6. 设计亮点

- **无损裁剪**：不缩放图片，只裁剪，保持原始清晰度。
- **智能锚点**：支持 9 种对齐方式（`xMinYMin` ~ `xMaxYMax`）。
- **路径更新**：自动将图片引用改为 `../images/cropped/xxx.png`，并删除无用的 `preserveAspectRatio`。

---

# 二、embed_icons.py —— 图标占位符替换

### 1. 解决的问题

执行师在生成 SVG 时使用 `<use data-icon="rocket" .../>` 占位符引用图标库中的图标。此脚本将这些占位符替换为实际的图标 SVG 路径代码，并应用缩放和颜色。

### 2. 核心算法

- 正则匹配 `<use data-icon="..." .../>` 元素。
- 提取 `data-icon` 名称、`x`、`y`、`width`、`height`、`fill` 属性。
- 从 `templates/icons/` 目录读取对应的 `{icon_name}.svg` 文件，提取其中所有 `<path>` 元素（移除原 `fill` 属性）。
- 计算缩放比例：`scale = width / 16`（图标基准尺寸为 16×16）。
- 生成 `<g transform="translate(x, y) scale(scale)" fill="fill_color">` 包裹所有路径。
- 替换原 `<use>` 元素。

### 3. 关键函数

| 函数 | 作用 |
|------|------|
| `extract_paths_from_icon(icon_path)` | 从图标 SVG 中提取路径列表 |
| `parse_use_element(use_match)` | 解析 `<use>` 元素属性 |
| `generate_icon_group(attrs, paths)` | 生成最终的 `<g>` 代码 |
| `process_svg_file(svg_path, icons_dir, dry_run, verbose)` | 处理单个 SVG |

### 4. 使用方式

```bash
# 处理单个 SVG
python3 embed_icons.py slide.svg

# 批量处理
python3 embed_icons.py svg_output/*.svg

# 指定图标目录
python3 embed_icons.py --icons-dir my_icons/ slide.svg

# 预览
python3 embed_icons.py --dry-run *.svg
```

### 5. 依赖

- 仅标准库 + 图标库文件（`templates/icons/*.svg`）。

### 6. 设计亮点

- **占位符简洁**：`<use data-icon="name">` 比直接嵌入长路径更易读、易维护。
- **自动缩放**：根据 `width` 自动计算缩放，保持图标原始比例。
- **颜色继承**：将 `fill` 属性设置在外层 `<g>`，内部路径无需重复定义。

---

# 三、embed_images.py —— 图片 Base64 嵌入

### 1. 解决的问题

SVG 中引用外部图片（如 `href="../images/photo.png"`）在预览时需要 HTTP 服务器，且打包分发时容易丢失依赖。此脚本将所有外部图片转为 Base64 内嵌，使 SVG 成为独立文件。

### 2. 核心算法

- 正则匹配 `href="xxx.png"`（排除已以 `data:` 开头的）。
- 解析相对路径，定位实际图片文件。
- 读取图片字节，Base64 编码。
- 根据文件扩展名或魔数确定 MIME 类型（`image/png`, `image/jpeg` 等）。
- 替换为 `href="data:image/png;base64,...."`。

### 3. 关键函数

| 函数 | 作用 |
|------|------|
| `get_mime_type(filename, file_bytes)` | 通过魔数或扩展名确定 MIME |
| `embed_images_in_svg(svg_path, dry_run)` | 处理单个 SVG |

### 4. 使用方式

```bash
python3 embed_images.py slide.svg
python3 embed_images.py --dry-run *.svg
```

### 5. 依赖

- 标准库（无需 PIL，但若需转换格式则建议安装）。

### 6. 设计亮点

- **无外部依赖**：纯标准库实现（`base64`, `re`）。
- **智能 MIME 检测**：优先使用魔数（PNG 头、JPEG 头等），回退扩展名。
- **大小对比**：输出原始文件大小与嵌入后大小，便于评估膨胀。

---

# 四、fix_image_aspect.py —— 修正图片宽高比（防止 PPT 拉伸）

### 1. 解决的问题

PowerPoint 将 SVG 转换为可编辑形状时，**完全忽略 `preserveAspectRatio`**，直接拉伸图片填满 `<image>` 的 `width`/`height` 区域。此脚本通过实际裁剪/留白的方式，重新计算 `<image>` 的 `x`, `y`, `width`, `height`，使图片以正确宽高比显示在原有区域内。

### 2. 核心算法

- 获取原始图片的实际宽高（支持外部文件或 Base64 内嵌）。
- 解析 `preserveAspectRatio` 获得对齐方式（如 `xMidYMid`）和模式（`meet`/`slice`）。
- 根据模式计算图片在目标框内的适配尺寸和偏移量：
  - `meet`：图片完全可见，可能有留白 → 计算居中后的 `new_width`, `new_height`。
  - `slice`：图片填满容器，可能裁剪 → 计算超出容器的尺寸，并通过偏移量实现居中裁剪。
- 更新 `<image>` 的 `x`, `y`, `width`, `height`，并删除 `preserveAspectRatio`（因为尺寸已精确适配）。

### 3. 关键函数

| 函数 | 作用 |
|------|------|
| `get_image_dimensions_*` | 获取图片尺寸（支持 PIL 或基础解析） |
| `calculate_fitted_dimensions(img_w, img_h, box_w, box_h, mode)` | 计算适配后的尺寸和偏移 |
| `fix_image_aspect_in_svg(svg_path, dry_run, verbose)` | 处理单个 SVG |

### 4. 使用方式

```bash
python3 fix_image_aspect.py slide.svg
python3 fix_image_aspect.py --dry-run *.svg
```

### 5. 依赖

- 推荐 Pillow，但未安装时可用基础解析（JPEG/PNG 头）。

### 6. 设计亮点

- **彻底解决拉伸问题**：通过直接修改几何属性，不依赖 PPT 对 `preserveAspectRatio` 的支持。
- **支持内嵌 Base64 图片**：能解析 `data:` URI 中的图片尺寸。
- **容错强**：PIL 缺失时仍能处理常见格式。

---

# 五、flatten_tspan.py —— 文本扁平化

### 1. 解决的问题

SVG 中使用 `<tspan>` 实现多行文本或样式变化。某些 PPT 转换器或渲染器对嵌套 `<tspan>` 支持不佳。此脚本将 `<tspan>` 扁平化为独立的 `<text>` 元素，每行一个，保留样式和位置。

### 2. 核心算法

- 遍历 SVG 中的 `<text>` 元素，检查是否有需要换行的 `<tspan>`（即 `y` 属性不同或 `dy` 非零）。
- 按行分组：收集属于同一行的连续 `<tspan>` 以及 `<text>` 的直接文本。
- 对每一行，计算绝对位置（基于当前 `x`, `y` 和 `<tspan>` 的 `dx`, `dy`）。
- 创建新的 `<text>` 元素，复制父级样式，设置 `x` 和 `y` 为计算后的值。
- 如果一行只有一个 `<tspan>` 且无前缀文本，直接将该 `<tspan>` 的内容和样式合并到新 `<text>`；否则保留 `<tspan>` 结构。
- 删除原始 `<text>`，插入新生成的 `<text>` 列表。

### 3. 关键函数

| 函数 | 作用 |
|------|------|
| `compute_line_positions(text_el, tspan_el, cur_x, cur_y)` | 计算 `<tspan>` 的绝对位置 |
| `is_new_line_tspan(tspan)` | 判断 `<tspan>` 是否表示新行（有 `y` 或 `dy≠0` 或 `x`） |
| `flatten_text_with_tspans(tree)` | 主处理逻辑 |
| `_create_text_element_from_line(...)` | 根据一行内容生成新的 `<text>` 元素 |

### 4. 使用方式

```bash
# 处理单个文件（输出带 _flattext 后缀）
python3 flatten_tspan.py input.svg output.svg

# 处理目录
python3 flatten_tspan.py svg_output/ -o svg_flat/

# 交互模式
python3 flatten_tspan.py -i
```

### 5. 依赖

- 仅标准库（`xml.etree.ElementTree`）。

### 6. 设计亮点

- **精确位置计算**：支持 `x`, `y`, `dx`, `dy` 组合，符合 SVG 规范。
- **样式合并**：正确处理 `style` 属性和单独样式属性（`fill`, `font-size` 等）的继承与覆盖。
- **最小化破坏**：只对需要换行的 `<tspan>` 进行扁平化，避免过度处理。

---

# 六、svg_rect_to_path.py —— 圆角矩形转路径

### 1. 解决的问题

PowerPoint 的“转换为形状”功能会丢失 `<rect rx="...">` 的圆角，将其变为直角矩形。此脚本将带 `rx`/`ry` 的 `<rect>` 转换为等效的 `<path>`，使用圆弧命令绘制圆角，确保圆角在 PPT 中保留。

### 2. 核心算法

- 遍历 SVG 中的所有 `<rect>` 元素。
- 读取 `x`, `y`, `width`, `height`, `rx`, `ry`。
- 如果 `rx` 或 `ry` 未指定，则两者取相同值；限制半径不超过宽/高的一半。
- 生成路径字符串：从左上角起点开始，绘制直线到右上角起点，圆弧到右上角，直线到右下角，圆弧到右下角，直线到左下角，圆弧到左下角，直线到左上角起点，闭合。
- 将 `<rect>` 标签替换为 `<path>`，设置 `d` 属性，删除 `x`, `y`, `width`, `height`, `rx`, `ry` 属性，保留其他属性（`fill`, `stroke` 等）。

### 3. 关键函数

| 函数 | 作用 |
|------|------|
| `rect_to_rounded_path(x, y, w, h, rx, ry)` | 生成路径字符串 |
| `process_svg(content, verbose)` | 处理 SVG 内容，返回新内容及转换数量 |
| `process_svg_file(input_path, output_path, verbose)` | 处理文件 |

### 4. 使用方式

```bash
# 处理项目（默认从 svg_output 读取，输出到 svg_rounded/）
python3 svg_rect_to_path.py projects/xxx

# 指定源目录（如 final）
python3 svg_rect_to_path.py projects/xxx -s final

# 处理单个文件
python3 svg_rect_to_path.py slide.svg
```

### 5. 依赖

- 标准库 + `xml.etree.ElementTree`。

### 6. 设计亮点

- **精确的路径生成**：使用 `A`（椭圆弧）命令，支持不同 `rx` 和 `ry`。
- **保留所有样式**：`fill`, `stroke`, `filter` 等属性原样保留。
- **批处理友好**：可处理整个项目，自动查找 `svg_output` 或 `svg_final`。

---

## 总结：`svg_finalize` 子模块在流水线中的角色

```
原始 SVG (svg_output/)
    │
    ├── embed_icons.py      → 替换图标占位符为实际路径
    ├── crop_images.py      → 根据 slice 模式裁剪图片
    ├── fix_image_aspect.py → 修正图片宽高比（防止 PPT 拉伸）
    ├── embed_images.py     → 图片 Base64 内嵌
    ├── flatten_tspan.py    → 多行文本扁平化
    └── svg_rect_to_path.py → 圆角矩形转路径
    │
    ▼
最终 SVG (svg_final/)
```

这些工具各司其职，共同将 AI 生成的“友好但含占位符”的 SVG 转换为“PPT 兼容、自包含、无特殊属性”的最终 SVG，从而保证 `svg_to_pptx.py` 能够稳定生成高质量可编辑的 PPTX。`finalize_svg.py` 按顺序调用它们，用户通常无需单独运行这些子脚本。