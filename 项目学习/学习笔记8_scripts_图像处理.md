## 图片处理脚本详细解析（analyze_images.py, gemini_watermark_remover.py, image_gen.py, rotate_images.py, backend_common.py, backend_siliconflow.py）

你提供的这些文件构成了 PPT Master 中**图片资产的全生命周期管理**：从分析已有图片、去除水印、生成新图片、修正图片方向，到多后端调用框架。以下逐文件解析。

---

# 一、analyze_images.py —— 图像尺寸分析工具

### 1. 核心定位

分析指定目录中所有图片的尺寸、宽高比、文件大小，并根据 PPT 16:9 画布给出布局建议。输出控制台报告、CSV 文件以及可供策略师直接复制的 Markdown 表格。

### 2. 关键常量

```python
PPT_WIDTH = 1280
PPT_HEIGHT = 720
FULL_SCREEN_MIN_RATIO = 1.5
FULL_SCREEN_MAX_RATIO = 2.0
```

- 全屏显示推荐的宽高比区间：1.5 ~ 2.0（即图片略宽于 16:9 画布比例 1.78）

### 3. 核心函数 `analyze_images(images_dir)`

- 遍历目录，使用 Pillow 打开每个图片文件
- 提取 `width`, `height`，计算 `aspect_ratio = width / height`
- 根据宽高比给出布局提示：
  - `> 1.5` → "Wide landscape"
  - `> 1.2` → "Standard landscape"
  - `> 0.8` → "Near square"
  - `> 0.6` → "Standard portrait"
  - 其他 → "Narrow portrait"

### 4. 输出内容

#### 4.1 控制台表格
包含序号、宽度、高度、宽高比、文件大小(KB)、布局提示、文件名。

#### 4.2 按宽高比分组统计
- Wide (>1.5)
- Standard (1.2-1.5)
- Square (0.8-1.2)
- Portrait (0.6-0.8)
- Narrow (<0.6)

#### 4.3 PPT 适配建议
统计适合全屏显示的图片数量（宽高比在 1.5~2.0 之间）。

#### 4.4 Markdown 片段（可直接复制到设计规范）
生成表格格式的图片资源清单，包含文件名、尺寸、比例、布局建议等。其中“用途”、“状态”、“生成描述”留空，由策略师填写。

```markdown
| Filename | Size | Ratio | Layout Suggestion | Usage | Status | Generation Description |
|----------|------|-------|-------------------|-------|--------|-----------------------|
| cover.png | 1920x1080 | 1.78 | Wide landscape (suitable for full-screen/illustration) | (to be filled) | Existing | - |
```

#### 4.5 CSV 文件
保存到图片目录的父目录，命名为 `image_analysis.csv`。

### 5. 设计亮点

- **一键生成资源清单草稿**：输出可直接粘贴到设计规范中，减少手工录入。
- **宽高比分组**：便于策略师快速了解图片类型分布，决定布局策略。
- **轻量级**：仅依赖 Pillow，无额外网络请求。

---

# 二、gemini_watermark_remover.py —— Gemini 图片水印去除工具

### 1. 核心定位

专门去除 Gemini 生成图片右下角的白色水印（“Made with Gemini”）。采用**逆向混合算法**，利用已知的背景水印模板还原原始像素。

### 2. 算法原理

Gemini 水印采用**半透明叠加**：`水印图片 = 原始图片 × (1 - α) + 白色 × α`。已知水印的 alpha 通道（从本地背景模板 `bg_48.png` / `bg_96.png` 提取），可以逆向求解原始图片：

```
原始 = (水印图 - α × 255) / (1 - α)
```

### 3. 关键常量

```python
ALPHA_THRESHOLD = 0.002      # 低于此阈值不处理（几乎透明）
MAX_ALPHA = 0.99             # 防止除零
LOGO_VALUE = 255             # 水印为白色
LARGE_IMAGE_THRESHOLD = 1024
LARGE_LOGO_SIZE = 96
SMALL_LOGO_SIZE = 48
LARGE_MARGIN = 64
SMALL_MARGIN = 32
```

- 图片宽或高 > 1024px 时使用 96px 水印，否则使用 48px。
- 水印位置固定在右下角，边距分别为 64px 或 32px。

### 4. 核心函数

#### 4.1 `detect_watermark_config(width, height)`
根据图片尺寸返回水印尺寸和边距。

#### 4.2 `calculate_alpha_map(bg_image)`
从背景 PNG 中提取 alpha 通道：
- 将背景图片转为 RGB，计算每个像素的 max(R,G,B) 作为透明度（因为背景是白色渐变，灰度值越高表示水印越不透明）。

#### 4.3 `remove_watermark(image, alpha_map, position)`
逐像素应用逆向混合公式，使用 NumPy 加速。

### 5. 使用方式

```bash
python3 gemini_watermark_remover.py input.png -o output.png
```

- 若不指定 `-o`，自动添加 `_unwatermarked` 后缀。
- 支持 JPG 输出（自动转为 RGB）。

### 6. 设计亮点

- **无 API 依赖**：纯本地算法，不消耗额外费用。
- **自适应尺寸**：根据图片分辨率自动选择 48px 或 96px 模板。
- **精确还原**：利用 Gemini 水印固定的位置和透明度特性，效果接近无损。
- **依赖最小**：需要 Pillow + NumPy。

---

# 三、image_gen.py —— 统一图像生成入口

### 1. 核心定位

屏蔽后端差异，提供统一的命令行接口生成 AI 图片。支持 11 种后端（Gemini, OpenAI, Stability, BFL, Ideogram, Qwen, Zhipu, Volcengine, SiliconFlow, fal, Replicate），通过环境变量 `IMAGE_BACKEND` 选择。

### 2. 配置加载逻辑

#### 2.1 `.env` 文件加载
- 搜索项目根目录的 `.env` 文件（`skills/ppt-master/` 的父目录的父目录的父目录，即仓库根目录）。
- 只加载以 `IMAGE_` 或各后端前缀开头的变量（如 `GEMINI_`, `OPENAI_` 等）。
- **拒绝全局变量**：`IMAGE_API_KEY`, `IMAGE_MODEL`, `IMAGE_BASE_URL` 已被废弃，遇到会报错并提示使用后端特定变量。

#### 2.2 后端注册表 `BACKEND_REGISTRY`

每个后端包含：
- `module`: 对应的 `backend_xxx.py` 文件名
- `tier`: `core`, `extended`, `experimental`
- `label`: 显示名称
- `default_model`: 默认模型
- `key_hint`: 需要的环境变量提示
- `aliases`: 别名列表（如 `google` → `gemini`）

### 3. 命令行参数

| 参数 | 说明 |
|------|------|
| `prompt` | 正向提示词 |
| `--negative_prompt` / `-n` | 负向提示词 |
| `--aspect_ratio` | 宽高比（支持 1:1, 16:9, 9:16, 4:3, 3:4, 2:3, 3:2, 4:5, 5:4, 1:4, 1:8, 4:1, 8:1, 21:9） |
| `--image_size` | 尺寸（512px, 1K, 2K, 4K） |
| `--output` / `-o` | 输出目录 |
| `--filename` / `-f` | 输出文件名（不含扩展名） |
| `--model` / `-m` | 覆盖模型 |
| `--backend` / `-b` | 临时覆盖后端 |
| `--list-backends` | 列出所有后端及支持级别 |

### 4. 执行流程

1. 加载 `.env` 并验证无废弃变量。
2. 解析命令行参数，若 `--backend` 则覆盖 `IMAGE_BACKEND`。
3. `_resolve_backend()`：根据 `IMAGE_BACKEND` 查找注册表，导入对应模块。
4. 调用后端的 `generate()` 函数。

### 5. 设计亮点

- **去中心化配置**：每个后端使用自己的 API key 变量，避免全局冲突。
- **可扩展性**：新增后端只需在 `BACKEND_REGISTRY` 注册并实现 `generate()`。
- **友好错误提示**：未配置时输出详细指导，列出可用后端和所需环境变量。
- **支持级别分组**：`--list-backends` 按 core / extended / experimental 显示，引导用户优先使用稳定的后端。

---

# 四、rotate_images.py —— 图像方向管理工具

### 1. 核心定位

解决图片因 EXIF 方向信息导致的显示方向错误问题。提供**自动 EXIF 修正**和**可视化手动旋转工具**两种方式。

### 2. 子命令

| 命令 | 功能 |
|------|------|
| `gen <images_dir>` | 生成一个 HTML 交互页面，让用户手动点击旋转图片，最终输出 JSON 修复指令 |
| `fix <fixes.json>` | 根据 JSON 指令执行批量旋转 |
| `auto <images_dir>` | 自动检测并修复 EXIF 方向（不生成 HTML） |

### 3. 核心类 `ImageRotator`

#### 3.1 `auto_fix_exif(target_dir)`
- 遍历目录下的 `.jpg`, `.jpeg`, `.webp` 文件。
- 读取 EXIF 的 `Orientation` 标签（ID 274）。
- 若值不为 1，根据值进行旋转/翻转操作，然后将 Orientation 重置为 1，并保存。

**支持的 Orientation 值**：
- 2: 左右翻转
- 3: 旋转 180°
- 4: 上下翻转
- 5: 旋转 270° + 翻转
- 6: 旋转 270°（顺时针 90° 逆操作）
- 7: 旋转 90° + 翻转
- 8: 旋转 90°

#### 3.2 `generate_html_tool(target_dir, output_filename)`
- 自动调用 `auto_fix_exif` 先修正 EXIF。
- 扫描目录下所有图片，生成一个独立的 HTML 文件（包含图片网格、点击旋转 90° 的交互）。
- 用户操作完成后点击“生成 Fix Code”，输出 JSON 修复指令。

#### 3.3 `apply_fixes(json_source)`
- 读取 JSON 指令（文件路径或 JSON 字符串）。
- 对每个条目，解析 `path` 和 `rotation`（90/180/270）。
- 执行旋转并保存。

### 4. 辅助功能

- `_save_in_place()`: 保持原始格式（JPEG/PNG/WebP）和质量（95%），同时保留 ICC 配置文件和清理后的 EXIF。
- `_normalize_task_path()`: 清理路径中的 `file://` 前缀和反斜杠，提高容错。
- `_natural_sort_key()`: 自然排序（`1.jpg, 2.jpg, 10.jpg`）。

### 5. 设计亮点

- **两阶段处理**：自动修正 EXIF（解决常见问题） + 手动微调（处理极端情况）。
- **可视化工具**：生成的 HTML 可脱离命令行使用，非技术人员也能操作。
- **无损保存**：旋转后保持原格式和质量，保留颜色配置文件。
- **路径解析健壮**：支持绝对路径、相对路径、仓库相对路径、`projects/` 下的自动查找。

---

# 五、backend_common.py —— 后端共享函数库

### 1. 核心定位

为所有图片生成后端提供公共函数：路径解析、图片格式检测、下载、保存、重试机制等。

### 2. 关键函数

#### 2.1 `resolve_output_path(prompt, output_dir, filename, ext)`
- 若指定 `filename`，使用它（去除扩展名）。
- 否则从 prompt 生成安全文件名（字母数字下划线，最多 30 字符）。
- 确保输出目录存在。

#### 2.2 `detect_image_extension(image_bytes, content_type)`
- 优先使用 `Content-Type` 头映射到扩展名。
- 否则通过魔数检测：PNG (`\x89PNG`), JPEG (`\xff\xd8\xff`), GIF, WebP, BMP, TIFF。
- 返回扩展名（如 `.png`）。

#### 2.3 `save_image_bytes(image_bytes, path, content_type)`
- 检测实际图片格式与目标扩展名是否一致。
- 若不一致且 Pillow 可用，自动转换格式（如 WebP → PNG，PNG → JPEG 时会转 RGB）。
- 若不一致且 Pillow 不可用，报错并提示安装 Pillow。
- 保存后调用 `report_resolution()` 打印图片尺寸。

#### 2.4 `download_image(url, path, headers, timeout)`
- 下载图片并调用 `save_image_bytes` 保存。

#### 2.5 `require_api_key(*candidates, message)`
- 按顺序查找环境变量，返回第一个非空值，否则抛出错误。

#### 2.6 `poll_json(url, headers, interval_seconds, timeout_seconds, status_label, ready_values, failed_values)`
- 用于异步任务的后端（如某些 API 需要轮询状态）。
- 轮询直到状态变为 ready 或失败。

#### 2.7 `is_rate_limit_error(exc)`
- 检查异常是否包含 429 或 rate/quota 关键字。
- `retry_delay()` 针对限流使用指数退避（基础 10 秒），普通错误退避 5 秒。

### 3. 设计亮点

- **格式自动转换**：避免因扩展名与实际图片类型不匹配导致的无法打开问题。
- **健壮的路径解析**：支持中文 prompt 生成合法文件名。
- **可配置的重试**：统一处理限流和临时错误。
- **轮询抽象**：简化异步 API 的实现。

---

# 六、backend_siliconflow.py —— SiliconFlow 后端实现示例

### 1. 核心定位

实现 `generate()` 函数，调用 SiliconFlow 的图片生成 API（兼容 OpenAI 格式但略有不同）。

### 2. 配置项

- `SILICONFLOW_API_KEY`（必需）
- `SILICONFLOW_BASE_URL`（可选，默认为 `https://api.siliconflow.cn/v1/images/generations`）
- `SILICONFLOW_MODEL`（可选，默认为 `Qwen/Qwen-Image`）

### 3. 宽高比与尺寸映射表 `ASPECT_RATIO_SIZE_MAP`

SiliconFlow API 要求传入具体的 `image_size` 字符串（如 `"1024x1024"`），而非宽高比。该映射表将 `(image_size, aspect_ratio)` 组合转换为具体分辨率。

示例：
- `image_size="1K", aspect_ratio="16:9"` → `"1664x928"`
- `image_size="2K", aspect_ratio="1:1"` → `"2048x2048"`

### 4. 核心函数 `_generate_image`

- 构建请求 payload：`{"model", "prompt", "image_size", "negative_prompt"}`
- 发送 POST 请求，打印生成耗时。
- 从响应中提取 `images[0].url`，调用 `download_image` 保存。

### 5. 重试机制

继承 `backend_common` 的重试逻辑：最多重试 `MAX_RETRIES`（3 次），遇到限流使用指数退避。

### 6. 设计亮点

- **适配非标准 API**：将统一参数（`aspect_ratio`, `image_size`）转换为 SiliconFlow 所需的 `image_size` 字符串。
- **清晰的日志**：打印模型、prompt 摘要、分辨率、生成耗时。
- **错误处理**：捕获 HTTP 错误并转换为友好消息。

---

## 图片处理脚本总结（架构与数据流）

```
用户图片（已有）                 AI 生成图片需求
      │                              │
      ▼                              ▼
analyze_images.py            image_gen.py
      │                              │
      │ (输出分析报告、Markdown 表格)   │ (调用后端)
      │                              │
      ▼                              ▼
策略师填写设计规范               backend_common.py
      │                              │
      │ (图片资源清单)                 │ (下载、格式转换、保存)
      │                              │
      ▼                              ▼
用户准备图片至 images/           images/ 目录
      │
      ▼
（可选）gemini_watermark_remover.py  去除水印
      │
      ▼
（可选）rotate_images.py  修正方向
      │
      ▼
images/ 目录（最终资产）
      │
      ▼
执行师在 SVG 中引用 <image href="../images/..."/>
```

### 工程智慧

1. **分析 → 生成 → 后处理** 流水线清晰，各脚本职责单一。
2. **统一的图片保存逻辑**（`save_image_bytes`）自动处理格式转换，避免兼容性问题。
3. **多后端抽象**：`image_gen.py` 作为门面，实际逻辑分布在各 `backend_*.py`，新增后端无需修改主入口。
4. **用户友好**：
   - `analyze_images.py` 直接输出可复制的 Markdown。
   - `rotate_images.py` 生成 HTML 交互工具。
   - `gemini_watermark_remover.py` 省去手动去水印的麻烦。
5. **健壮性**：重试机制、路径容错、格式自动检测。

---
