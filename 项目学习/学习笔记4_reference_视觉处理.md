## 设计与图像处理专题 — 逐项详细解析

按你给出的顺序，我们逐一深入解读这 6 个文件。它们共同构成了 PPT Master 中所有视觉资源（图片、画布、SVG 规范、模板）的规则体系。

---

# 一、图片生成流程核心：`image-generator.md`

**角色定位**：Image_Generator 的完整操作手册。负责将设计规范中的“图片资源清单”转化为真实图片文件。

## 1. 核心使命与触发条件

- **输入**：来自策略师的设计规范中的图片资源清单（包含文件名、尺寸、用途、类型、状态、生成描述）。
- **输出**：
  - 提示词文档 `project/images/image_prompts.md`（必须用文件写入工具保存，不能只在对话中输出）
  - 优化后的每条提示词（可直接用于 AI 生图工具，同时作为 alt 文本）
  - 图片文件（保存在 `project/images/` 目录）
  - 更新资源清单状态（“待生成” → “已生成”）

- **触发时机**：
  - **独立模式**：用户直接描述图片需求
  - **流水线模式**：`generate-ppt` 中选择了 AI 生成图片，策略师阶段完成后调用

## 2. 统一的提示词输出格式（强制）

每个图片必须按以下格式输出：

```markdown
### Image N: {filename}

| 属性 | 值 |
|------|-----|
| 用途 | {哪一页/什么功能} |
| 类型 | {背景/摄影/插画/图示/装饰} |
| 尺寸 | {宽}x{高} ({宽高比}) |
| 原始描述 | {清单中用户提供的描述} |

**Prompt**:
{主题描述}, {风格指令}, {色彩指令}, {构图指令}, {质量指令}

**Negative Prompt**:
{要排除的元素}

**Alt Text**:
> {无障碍描述，也可用作图片说明}
```

### 提示词组成部分

| 组件 | 说明 | 示例 |
|------|------|------|
| 主题描述 | 核心内容 | `抽象几何形状`、`团队协作场景` |
| 风格指令 | 视觉风格 | `扁平设计`、`3D 等距`、`水彩风格` |
| 色彩指令 | 配色方案 | `调色板：海军蓝(#1E3A5F)、金色(#D4AF37)` |
| 构图指令 | 布局比例 | `16:9 宽高比`、`居中构图` |
| 质量指令 | 分辨率质量 | `高质量`、`4K 分辨率`、`细节清晰` |
| 负向提示词 | 排除元素 | `文字、水印、模糊、低质量` |

### 风格关键词速查

| 设计风格 | 推荐图片风格 | 核心关键词 |
|----------|-------------|-----------|
| 通用灵活型 | 现代插画、扁平设计 | `modern`, `flat design`, `gradient`, `vibrant colors` |
| 通用咨询型 | 干净专业、企业风格 | `professional`, `clean`, `corporate`, `minimalist` |
| 顶级咨询型 | 高级极简、抽象几何 | `premium`, `sophisticated`, `geometric`, `abstract`, `elegant` |

### 色彩融入方法

从设计规范中提取颜色，转换为提示词指令：

```
主色: #1E3A5F (深海军蓝)  → "deep navy blue (#1E3A5F)"
次色: #F8F9FA (浅灰)      → "light gray (#F8F9FA)"
强调色: #D4AF37 (金色)    → "gold accent (#D4AF37)"

完整指令："color palette: deep navy blue (#1E3A5F), light gray (#F8F9FA), gold accent (#D4AF37)"
```

### 画布格式与宽高比

| 画布格式 | 背景宽高比 | 推荐分辨率 |
|----------|-----------|-----------|
| PPT 16:9 | 16:9 | 1920x1080 或 2560x1440 |
| PPT 4:3 | 4:3 | 1600x1200 |
| 小红书 | 3:4 | 1242x1660 |
| 朋友圈 | 1:1 | 1080x1080 |
| 故事 | 9:16 | 1080x1920 |

> 支持的宽高比：`1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`（Gemini 还支持 `1:4`, `1:8`, `4:1`, `8:1`）

## 3. 图片类型分类与处理（5 种）

每种类型有独立的判断流程和提示词模板。

### 3.1 背景（Background）

**识别特征**：封面或章节页的全页背景，必须支持文字叠加。

| 要点 | 说明 |
|------|------|
| 强调背景性质 | 添加 `background`, `backdrop` |
| 预留文字区域 | `negative space in center for text overlay` |
| 避免强主体 | 使用抽象、渐变、几何元素 |
| 低对比细节 | `subtle`, `soft`, `muted` |

**模板**：
```
Abstract {theme element} background, {style} style, {primary color} to {secondary color} gradient, subtle {decorative elements}, clean negative space in center for text overlay, {aspect ratio} aspect ratio, high resolution, professional presentation background
```

**负向提示词**：`text, letters, watermark, faces, busy patterns, high contrast details`

### 3.2 摄影（Photography）

**识别特征**：真实场景、人物、产品、建筑 — 照片级质量。

| 要点 | 说明 |
|------|------|
| 强调真实感 | `photography`, `photorealistic`, `real photo` |
| 光照效果 | `natural lighting`, `soft shadows`, `studio lighting` |
| 背景处理 | `white background` / `blurred background` / `contextual setting` |
| 人物多样性 | `diverse`, `professional attire` |

**模板**：
```
{subject description}, professional photography, {lighting type} lighting, {background type} background, color grading matching {color scheme}, high quality, sharp focus, 8K resolution
```

**负向提示词**：`watermark, text overlay, artificial, CGI, illustration, cartoon, distorted faces`

### 3.3 插画（Illustration）

**识别特征**：扁平设计、矢量风格、卡通、概念图。

| 要点 | 说明 |
|------|------|
| 指定风格 | `flat design`, `isometric`, `vector style`, `hand-drawn` |
| 简化细节 | `simplified`, `clean lines`, `minimal details` |
| 统一调色板 | 严格使用设计规范颜色 |
| 背景选择 | `white background` 或 `transparent background` |

**模板**：
```
{subject description}, {illustration style} illustration style, {detail level} with clean lines, color palette: {color list}, {background type} background, professional {purpose} illustration
```

**负向提示词**：`realistic, photography, 3D render, complex textures, watermark`

### 3.4 图示（Diagram）

**识别特征**：流程图、架构图、概念关系图、数据可视化。

| 要点 | 说明 |
|------|------|
| 清晰结构 | `clear structure`, `organized layout`, `logical flow` |
| 连接表示 | `arrows indicating flow`, `connecting lines` |
| 学术/专业感 | `suitable for academic publication`, `professional diagram` |
| 浅色背景 | `white background` 或 `light gray background` |

**模板**：
```
{diagram type} diagram showing {content description}, {component description} connected by {connection method}, {style} style with {color scheme}, white background, clear labels, professional technical diagram
```

**负向提示词**：`cluttered, messy, overlapping elements, dark background, realistic, photography`

### 3.5 装饰纹样（Decorative Pattern）

**识别特征**：局部装饰、纹理、边框、分割元素。

| 要点 | 说明 |
|------|------|
| 可重复性 | `seamless`, `tileable`, `repeatable`（如果需要） |
| 低调支持 | `subtle`, `understated`, `supporting element` |
| 透明友好 | `transparent background` 或 `isolated element` |
| 小尺寸可读 | 考虑在小尺寸下的清晰度 |

**模板**：
```
{pattern type} decorative pattern, {style} style, {color scheme}, {background type} background, subtle and elegant, suitable for {purpose}
```

**负向提示词**：`busy, cluttered, high contrast, distracting, photorealistic`

## 4. 图片生成工作流（4 个阶段）

### 4.1 分析阶段
- 读取设计规范，理解整体项目风格
- 提取配色方案、画布格式、目标受众
- 逐个分析资源清单中的图片
- 确定每张图片的类型（参照第 3 节）

### 4.2 提示词生成阶段
对每个“待生成”状态的图片：
1. 确定类型 → 背景/摄影/插画/图示/装饰
2. 理解用途 → 哪一页？什么功能？
3. 分析原始描述 → 用户“生成描述”中的信息
4. 应用类型特定要点 → 参考对应类型表格
5. 生成优化后的提示词 → 使用 2.1 标准输出格式
6. 保存提示词文档 → **必须**写入 `project/images/image_prompts.md`

### 4.3 图片生成阶段

**前置条件**：4.2 完成，`images/image_prompts.md` 必须存在。

#### 方法 1：统一 CLI 工具（推荐）
```bash
python3 scripts/image_gen.py "your prompt" \
  --aspect_ratio 16:9 --image_size 1K \
  --output project/images --filename cover_bg
```

**参数说明**：

| 参数 | 短名 | 说明 | 默认值 |
|------|------|------|--------|
| `prompt` | - | 正向提示词（位置参数） | - |
| `--negative_prompt` | `-n` | 负向提示词 | None |
| `--aspect_ratio` | - | 图片宽高比 | `1:1` |
| `--image_size` | - | 尺寸（`1K`/`2K`/`4K`） | `1K` |
| `--output` | `-o` | 输出目录 | 当前目录 |
| `--filename` | `-f` | 输出文件名（无扩展名） | 自动命名 |
| `--backend` | `-b` | 覆盖后端 | None |
| `--model` | `-m` | 模型名称 | 后端默认 |
| `--list-backends` | - | 打印支持级别并退出 | `false` |

**配置来源**：
- 当前进程环境变量
- 项目根目录 `.env` 作为后备

**必需变量**：
| 变量 | 说明 |
|------|------|
| `IMAGE_BACKEND` | 必填：`gemini` / `openai` / `stability` / `bfl` / `ideogram` / `qwen` / `zhipu` / `volcengine` / `siliconflow` / `fal` / `replicate` |
| `{PROVIDER}_API_KEY` | 提供商特定的 API 密钥，如 `GEMINI_API_KEY`、`ZHIPU_API_KEY` |
| `{PROVIDER}_BASE_URL` | 可选，自定义端点 |
| `{PROVIDER}_MODEL` | 可选，覆盖模型 |

> 不支持 `IMAGE_API_KEY`、`IMAGE_MODEL`、`IMAGE_BASE_URL` 这种全局变量。

**支持级别**：
- 核心：`gemini`, `openai`, `qwen`, `zhipu`, `volcengine`
- 扩展：`stability`, `bfl`, `ideogram`
- 实验：`siliconflow`, `fal`, `replicate`

**生成节奏（强制）**：
- 一次只执行一个生成命令，等待文件确认后再进行下一个
- 建议图片之间间隔 2-5 秒，避免并发失败
- 如果失败/无输出，暂停队列，检查 `IMAGE_BACKEND`、提供商凭证和输出目录，然后恢复

#### 方法 2：自动生成
直接调用图片生成 API，下载并保存到 `project/images/` 目录。

#### 方法 3：Gemini Web 界面
1. 在 [Gemini](https://gemini.google.com/) 中生成图片
2. 选择 **Download full size** 获取高分辨率版本
3. 去除水印：`python3 scripts/gemini_watermark_remover.py <image_path>`
4. 将处理后的图片放入 `project/images/` 目录

#### 方法 4：手动生成（其他 AI 平台）
提示词保存在 `images/image_prompts.md` 中，告知用户文件位置。用户自行在 Midjourney、DALL-E、Stable Diffusion 等平台生成，并将图片放入 `project/images/` 目录。

### 4.4 验证阶段
- 确认所有图片已保存到 `images/` 目录
- 检查文件名与资源清单匹配
- 更新图片资源清单状态为“已生成”

## 5. 提示词文档模板

当创建 `project/images/image_prompts.md` 时，使用以下结构：

```markdown
# 图片生成提示词

> 项目：{project_name}
> 生成日期：{date}
> 配色方案：主色 {#HEX} | 次色 {#HEX} | 强调色 {#HEX}

---

## 图片列表概览

| # | 文件名 | 类型 | 尺寸 | 状态 |
|---|--------|------|------|------|
| 1 | cover_bg.png | 背景 | 1920x1080 | 待生成 |

---

## 详细提示词

### 图片 1：cover_bg.png

| 属性 | 值 |
|------|-----|
| 用途 | 封面背景 |
| 类型 | 背景 |
| 尺寸 | 1920x1080 (16:9) |
| 原始描述 | 现代科技抽象背景，深蓝色渐变 |

**Prompt**：
抽象的未来主义背景，带有流动的数字波浪...

**Alt Text**：
> 现代科技抽象背景，深蓝色渐变，数字波浪和粒子效果

---

## 使用说明

1. 将上面的“Prompt”复制到 AI 图片生成工具中
2. 推荐平台：Midjourney / DALL-E 3 / Gemini / Stable Diffusion
3. 将生成的图片重命名为相应的文件名
4. 放入 `images/` 目录
```

## 6. 负向提示词速查

### 按图片类型

| 类型 | 推荐负向提示词 |
|------|---------------|
| 背景 | `text, letters, watermark, faces, busy patterns, high contrast details` |
| 摄影 | `watermark, text overlay, artificial, CGI, illustration, cartoon, distorted faces` |
| 插画 | `realistic, photography, 3D render, complex textures, watermark` |
| 图示 | `cluttered, messy, overlapping elements, dark background, realistic` |
| 装饰纹样 | `busy, cluttered, high contrast, distracting, photorealistic` |

### 通用负向提示词

- **标准**：`text, watermark, signature, blurry, distorted, low quality`
- **扩展**（人物场景）：`text, watermark, signature, blurry, low quality, distorted, extra fingers, mutated hands, poorly drawn face, bad anatomy, extra limbs, disfigured, deformed`

## 7. 常见问题处理

### 当“生成描述”未提供时的默认推理

| 用途 | 默认推理 |
|------|----------|
| 封面背景 | 抽象渐变背景，预留中心文字区域 |
| 章节页背景 | 干净的几何图案，单色焦点 |
| 团队介绍页 | 团队协作场景插画（扁平风格） |
| 数据展示页 | 干净的几何图案或纯色背景 |
| 产品展示 | 产品摄影风格，白色或渐变背景 |

### 当图片不满意时

提供提示词变体供用户选择：变体 A（更抽象）、变体 B（更具体）、变体 C（不同色调）。

## 8. 角色协作

### 与策略师的交接

| 方向 | 内容 |
|------|------|
| 接收 | 设计规范与内容大纲（含图片资源清单） |
| 触发条件 | 用户在“图片使用方式”中选择了“C) AI 生成” |
| 关键信息 | 配色方案、设计风格、画布格式 |

### 与执行师的交接

| 方向 | 内容 |
|------|------|
| 交付 | 所有图片放入 `project/images/` 目录 |
| 执行师引用 | `<image href="../images/xxx.png" .../>` |
| 路径说明 | SVGs 在 `svg_output/`，图片在 `images/`；使用相对路径 `../images/` |

## 9. 任务完成检查点

### 必须完成项
- [ ] 创建提示词文档 `project/images/image_prompts.md`
- [ ] 每张图片都有：类型确定 + 优化提示词 + 负向提示词 + Alt Text
- [ ] 使用统一输出格式（2.1 标准格式）
- [ ] 阶段完成确认输出

### 图片就绪（至少满足一项）
- [ ] 所有图片已保存到 `project/images/` 目录
- [ ] 或：用户明确被告知使用 `image_prompts.md` 自行生成

### 流水线流程
- [ ] 提示用户进入下一步（切换到执行师角色）

> **关键检查**：如果 `images/image_prompts.md` 未创建，或输出格式不符合 2.1 标准，任务不算完成。

### 完成确认输出格式

```markdown
## Image_Generator Phase Complete

- [x] 创建提示词文档 `project/images/image_prompts.md`
- [x] 为 X 张图片生成优化提示词
- [x] 所有图片已保存到 `images/` 目录
- [x] 更新图片资源清单状态

**图片状态摘要**：

| 文件名 | 类型 | 尺寸 | 状态 |
|--------|------|------|------|
| cover_bg.png | 背景 | 1920x1080 | 已生成 |

**下一步**：切换到 Executor 角色开始 SVG 生成
```

---

# 二、图片布局强制规范：`image-layout-spec.md`

**核心原则**：根据图片原始宽高比动态计算布局，确保图片完整显示，无多余留白或裁剪。

## 1. 布局决策流程

```
1. 获取图片原始尺寸 → 计算宽高比（宽/高）
2. 根据宽高比选择布局类型
3. 计算图片的最大显示尺寸
4. 剩余空间分配给文字区
5. 将结果填入设计规范的图片资源清单
```

**执行时机**：如果图片方式包含“B) 用户提供”，则策略师完成八项确认后、内容分析和大纲输出前，必须运行扫描并填充图片资源清单。

## 2. 布局类型选择（强制）

| 图片宽高比 | 布局类型 | 图片位置 | 说明 |
|-----------|----------|----------|------|
| > 2.0（超宽） | 上下分割 | 顶部通栏 | 图片横跨画布宽度，高度按比例 |
| 1.5-2.0（宽幅） | 上下分割 | 顶部 | 图片宽度 = 内容区宽度，高度按比例 |
| 1.2-1.5（标准横图） | 左右分割 | 左侧 | 图片高度优先适配，宽度按比例 |
| 0.8-1.2（方形） | 左右分割 | 左侧 | 图片占内容区高度，宽度按比例 |
| < 0.8（竖图） | 左右分割 | 左侧 | 图片高度 = 内容区高度，宽度按比例 |

> 边界情况：当宽高比恰好在边界（如 1.5），根据文字量决定。文字多 → 左右分割；文字少 → 上下分割。

## 3. 尺寸计算公式（PPT 16:9 画布）

### 画布参数
```
画布：1280 x 720 px
内容区：1160 x 640 px（左右边距 60px，上下边距 40px）
标题区高度：60 px
内容起始 y = 80 px（标题 + 间距）
```

### 上下分割布局计算
```
图片宽度 = W = 1160 px
图片高度 = W / R = 1160 / R px
文字区高度 = H - 图片高度 - 间隙(20px)

验证：文字区高度 >= 150px（至少 3-4 行文字）
若不满足 → 切换到左右分割布局
```

### 左右分割布局计算

**方法 1（高度优先，适用于竖图）**：
```
图片高度 = H = 600 px
图片宽度 = H × R = 600 × R px
文字区宽度 = W - 图片宽度 - 间隙(20px)
```

**方法 2（宽度约束，适用于转为左右分割的宽图）**：
```
图片宽度 = W × 0.7 = 812 px
图片高度 = 图片宽度 / R
文字区宽度 = W - 图片宽度 - 间隙(20px)
```

**验证**：文字区宽度 >= 280px；否则减少图片区域宽度。

## 4. 布局示例

### 超宽图片（宽高比 2.45）
```
原始：1960x800，R=2.45 → 上下分割
图片：1160x473，文字区：1160x147 → 7:3 上下分割
```

### 标准横图（宽高比 1.38）
```
原始：1614x1171，R=1.38 → 左右分割
图片：773x560（左侧），文字区：367x560（右侧）→ 7:3 左右分割
```

### 宽图边界情况（宽高比 1.75）
```
原始：1820x1040，R=1.75
尝试上下分割：图片高度=663，文字区高度=-43 ❌
切换到左右分割：图片 780x446（左侧），文字区 360x600（右侧）→ 7:3 左右分割
```

## 5. 禁止做法

| 禁止 | 正确做法 |
|------|----------|
| 固定 50:50 或任意比例 | 基于图片宽高比动态计算 |
| 将宽图强行放入方形容器 | 使用上下分割布局，或增加图片区域宽度 |
| 将竖图放入狭窄横条 | 使用左右分割布局，图片靠左 |
| 图片留白超过 10% | 重新计算布局或选择替代方案 |
| 裁剪图片关键内容 | 使用 `preserveAspectRatio="xMidYMid meet"` |
| 文字区太小无法阅读 | 确保文字区高度 >= 150px（上下分割）或宽度 >= 280px（左右分割） |

## 6. SVG 图片嵌入代码

### 完整显示（推荐用于数据图表）
```xml
<image href="../images/xxx.png"
       x="60" y="80" width="780" height="446"
       preserveAspectRatio="xMidYMid meet"/>
```

### 裁剪填充（仅用于背景）
```xml
<image href="../images/bg.png"
       x="0" y="0" width="1280" height="720"
       preserveAspectRatio="xMidYMid slice"/>
```

## 7. 图片资源清单模板

在设计规范与内容大纲中，图片资源清单必须包含：

| 字段 | 说明 | 示例 |
|------|------|------|
| 文件名 | 图片文件名 | `p12_0.png` |
| 原始尺寸 | 宽 x 高 | 1524x968 |
| 宽高比 | 宽 / 高 | 1.57 |
| 页面 | 使用页码 | 第 5 页 |
| 类型 | 视觉类型 | 背景 / 摄影 / 插画 / 图示 / 装饰 |
| 布局方案 | 上下/左右分割 + 分割比例 | 上下分割 6:4 或 左右分割 7:3 |
| 图片区域 | 图片显示尺寸 | 1160x420 或 780x446 |
| 文字区域 | 剩余空间尺寸 | 1160x200 或 360x600 |

**类型字段用于 Image_Generator 选择适当的提示词策略。**

## 8. 自动化工具

```bash
python3 scripts/analyze_images.py <project_path>/images
```

输出包括尺寸、宽高比和布局建议（Markdown 表格），可直接用于填充图片资源清单。

## 9. 角色职责

| 角色 | 职责 |
|------|------|
| **策略师** | 运行 analyze_images.py，根据本规范计算布局，填充图片资源清单 |
| **执行师** | 生成 SVG 时严格遵守图片资源清单中的布局方案和尺寸 |

---

# 三、画布格式速查：`canvas-formats.md`

**作用**：定义所有支持的输出画布格式，是所有 SVG 页面的坐标系基础。

## 格式速查表

| 格式 | viewBox | 宽高比 | 使用场景 |
|------|---------|--------|----------|
| PPT 16:9 | `0 0 1280 720` | 16:9 | 商务演示、会议 |
| PPT 4:3 | `0 0 1024 768` | 4:3 | 传统投影仪、学术演讲 |
| 小红书 | `0 0 1242 1660` | 3:4 | 图文分享、知识帖 |
| 朋友圈 / IG | `0 0 1080 1080` | 1:1 | 方形海报、品牌展示 |
| 故事 / 抖音 | `0 0 1080 1920` | 9:16 | 竖屏故事、短视频封面 |
| 公众号文章头图 | `0 0 900 383` | 2.35:1 | 微信公众号文章封面 |
| 横版横幅 | `0 0 1920 1080` | 16:9 | 网页横幅、数字屏幕 |
| 竖版海报 | `0 0 1080 1920` | 9:16 | 手机屏幕、电梯广告 |
| A4 打印 | `0 0 1240 1754` | 1:1.414 | 打印海报、传单 |

## 格式选择决策树

```
内容用途？
├── 演示文稿
│   ├── 现代设备 → PPT 16:9 (1280x720)
│   └── 传统设备 → PPT 4:3 (1024x768)
├── 社交分享
│   ├── 小红书 → 1242x1660
│   ├── 朋友圈 / IG → 1080x1080
│   └── 故事 / 抖音 → 1080x1920
└── 营销材料
    ├── 公众号文章头图 → 900x383
    ├── 横幅 → 1920x1080
    └── 打印 → 1240x1754
```

## 布局原则

### 横版（16:9, 4:3, 2.35:1）
- 视觉流向：Z 字形，从左到右
- 边距：40-80px
- 布局方式：多列、左右分割、网格
- 卡片尺寸（16:9）：单行 530-600px，双行 265-295px

### 竖版（3:4, 9:16）
- 视觉流向：从上到下
- 边距：60-120px
- 布局方式：单列、上下分割、卡片堆叠
- 卡片尺寸（3:4）：高度 400-600px，间隙 40-60px

### 方形（1:1）
- 视觉流向：中心放射
- 边距：60-100px
- 核心区域：约 800x800px

## 格式特定设计

| 格式 | 标题区 | 内容区 | 特殊说明 |
|------|--------|--------|----------|
| PPT | 80-100px | 充分利用宽度 | 页码在右下角 |
| 小红书 | 180-240px（粗体） | 上下留白充足 | 底部品牌区 120-160px |
| 朋友圈 | 200-280px | 居中 500-600px | 底部二维码区 150-200px |
| 故事 | — | 中部 1500px | 顶部安全区 120px，底部 180px |
| 公众号头图 | 居中/左对齐 48-72px | — | 图片靠右或作为背景 |

## viewBox 示例

```xml
<svg width="1280" height="720" viewBox="0 0 1280 720">   <!-- PPT 16:9 -->
<svg width="1242" height="1660" viewBox="0 0 1242 1660"> <!-- 小红书 -->
<svg width="1080" height="1080" viewBox="0 0 1080 1080"> <!-- 朋友圈 -->
<svg width="1080" height="1920" viewBox="0 0 1080 1920"> <!-- 故事 -->
<svg width="900" height="383" viewBox="0 0 900 383">     <!-- 公众号头图 -->
```

---

# 四、全局技术约束：`shared-standards.md`

**作用**：全角色共享的基础规范，是所有 SVG 生成和处理的“宪法”。

## 1. SVG 禁用特性黑名单

以下特性**绝对禁止**使用 — PPT 导出会失败：

| 禁用特性 | 说明 |
|----------|------|
| `clipPath` | 裁剪路径 |
| `mask` | 蒙版 |
| `<style>` | 内嵌样式表 |
| `class` | CSS 选择器属性（`<defs>` 内的 `id` 是合法引用，不禁用） |
| 外部 CSS | 外部样式表链接 |
| `<foreignObject>` | 嵌入外部内容 |
| `<symbol>` + `<use>` | 符号引用复用 |
| `textPath` | 沿路径文本 |
| `@font-face` | 自定义字体声明 |
| `<animate*>` / `<set>` | SVG 动画 |
| `<script>` / 事件属性 | 脚本和交互 |
| `marker` / `marker-end` | 线条端点标记 |
| `<iframe>` | 嵌入框架 |

## 2. PPT 兼容替代方案

| 禁用语法 | 正确替代 |
|----------|----------|
| `fill="rgba(255,255,255,0.1)"` | `fill="#FFFFFF" fill-opacity="0.1"` |
| `<g opacity="0.2">...</g>` | 在每个子元素上单独设置 `fill-opacity` / `stroke-opacity` |
| `<image opacity="0.3"/>` | 在图片后叠加一个 `<rect fill="background-color" opacity="0.7"/>` 蒙版层 |
| `marker-end` 箭头 | 用 `<polygon>` 绘制三角形箭头 |

**助记**：PPT 不识别 rgba、组透明度、图片透明度、标记。

## 3. 基础 SVG 规则

- **viewBox** 必须与画布尺寸匹配（`width`/`height` 必须与 `viewBox` 一致）
- **背景**：使用 `<rect>` 定义页面背景色
- **换行**：使用 `<tspan>` 手动换行；`<foreignObject>` 禁止
- **字体**：仅使用系统字体（微软雅黑、Arial、Calibri 等）；`@font-face` 禁止
- **样式**：仅使用内联样式（`fill="..."` `font-size="..."`）；`<style>` / `class` 禁止（`<defs>` 内的 `id` 合法）
- **颜色**：使用 HEX 值；透明度使用 `fill-opacity` / `stroke-opacity`
- **图片引用**：`<image href="../images/xxx.png" preserveAspectRatio="xMidYMid slice"/>`
- **图标占位符**：`<use data-icon="icon-name" x="" y="" width="48" height="48" fill="#HEX"/>`（后处理时自动嵌入）

### 元素分组（强制）

逻辑相关的元素**必须**用 `<g>` 标签包裹。这会在导出的 PPTX 中生成 PowerPoint 组，使幻灯片更易于选择、移动和编辑。

> ⚠️ **只有 `<g opacity="...">` 被禁止**（见第 2 节）。纯 `<g>` 用于结构分组是必需的。

**什么需要分组**：

| 分组单元 | 包含内容 |
|----------|----------|
| 卡片/面板 | 背景矩形 + 阴影 + 图标 + 标题 + 正文 |
| 流程步骤 | 数字圆圈 + 图标 + 标签 + 描述 |
| 列表项 | 项目符号/数字 + 图标 + 标题 + 描述 |
| 图标-文字组合 | 图标元素 + 相邻标签 |
| 页眉 | 标题 + 副标题 + 装饰 |
| 页脚 | 页码 + 品牌 |
| 装饰簇 | 相关的装饰形状（圆环、球体、点） |

**示例**：
```xml
<g id="card-benefits-1">
  <rect x="60" y="115" width="565" height="260" rx="20" fill="#FFFFFF" filter="url(#shadow)"/>
  <use data-icon="bolt" x="108" y="163" width="44" height="44" fill="#0071E3"/>
  <text x="105" y="270" font-size="56" font-weight="bold" fill="#0071E3">10×</text>
  <text x="250" y="270" font-size="30" font-weight="bold" fill="#1D1D1F">更快</text>
  <text x="105" y="310" font-size="18" fill="#6E6E73">生产时间从天缩短到小时。</text>
</g>
```

**命名约定**：在 `<g>` 标签上使用描述性 `id` 属性（如 `card-1`、`step-discover`、`header`、`footer`）。ID 可选但推荐用于可读性。

## 4. 后处理流水线（3 步）

必须按顺序执行 — 跳过或添加额外标志是禁止的：

```bash
# 1. 将演讲备注分割成每页备注文件
python3 scripts/total_md_split.py <project_path>

# 2. SVG 后处理（图标嵌入、图片裁剪/嵌入、文本扁平化、圆角矩形转路径）
python3 scripts/finalize_svg.py <project_path>

# 3. 导出 PPTX（从 svg_final/ 导出，默认嵌入演讲备注）
python3 scripts/svg_to_pptx.py <project_path> -s final
# 默认生成：原生形状 (.pptx) + SVG 参考版 (_svg.pptx)
```

**禁止**：
- 绝不能用 `cp` 代替 `finalize_svg.py`
- 绝不能直接从 `svg_output/` 导出 — 必须从 `svg_final/` 导出（使用 `-s final`）
- 绝不能添加额外标志如 `--only`

**重新运行规则**：后处理完成后对 `svg_output/` 的任何修改（包括页面修订、添加或删除）都需要重新运行步骤 2 和 3。步骤 1 仅在 `notes/total.md` 也被修改时需要重新运行。

## 5. 阴影与叠加技术

> `<mask>` 元素和 `<image opacity="...">` 被禁止。始终使用堆叠的 `<rect>` 或渐变叠加层代替（见第 2 节）。

### 滤镜软阴影 — 推荐
最适合：卡片、浮动面板、提升元素。`svg_to_pptx` 转换器自动将 `feGaussianBlur` + `feOffset` 转换为原生 PPTX `<a:outerShdw>`。

```xml
<defs>
  <filter id="softShadow" x="-15%" y="-15%" width="140%" height="140%">
    <feGaussianBlur in="SourceAlpha" stdDeviation="12"/>
    <feOffset dx="0" dy="6" result="offsetBlur"/>
    <feFlood flood-color="#000000" flood-opacity="0.15" result="shadowColor"/>
    <feComposite in="shadowColor" in2="offsetBlur" operator="in" result="shadow"/>
    <feMerge>
      <feMergeNode in="shadow"/>
      <feMergeNode in="SourceGraphic"/>
    </feMerge>
  </filter>
</defs>
<rect x="60" y="60" width="400" height="240" rx="12" fill="#FFFFFF" filter="url(#softShadow)"/>
```

推荐参数：
```
stdDeviation:   10–16    （越小越锐利，越大越柔和）
flood-opacity:  0.12–0.20 （太低在 PPTX 中不可见）
dy:             4–8      （垂直 > 水平，符合顶部光源自然感）
dx:             0–2
```

### 彩色阴影
最适合：强调按钮、品牌色卡片。使用元素自身的颜色系列而非黑色。

```xml
<filter id="colorShadow" x="-15%" y="-15%" width="140%" height="140%">
  <feGaussianBlur in="SourceAlpha" stdDeviation="10"/>
  <feOffset dx="0" dy="6" result="offsetBlur"/>
  <feFlood flood-color="#1A73E8" flood-opacity="0.20" result="shadowColor"/>
  <feComposite in="shadowColor" in2="offsetBlur" operator="in" result="shadow"/>
  <feMerge>
    <feMergeNode in="shadow"/>
    <feMergeNode in="SourceGraphic"/>
  </feMerge>
</filter>
```

### 发光效果
最适合：标题高亮、关键指标、英雄文本。转换器自动将不带 `feOffset` 的 `feGaussianBlur` 转换为原生 PPTX `<a:glow>`。

```xml
<defs>
  <filter id="titleGlow" x="-30%" y="-30%" width="160%" height="160%">
    <feGaussianBlur in="SourceAlpha" stdDeviation="6" result="blur"/>
    <feFlood flood-color="#1A73E8" flood-opacity="0.45" result="glowColor"/>
    <feComposite in="glowColor" in2="blur" operator="in" result="glow"/>
    <feMerge>
      <feMergeNode in="glow"/>
      <feMergeNode in="SourceGraphic"/>
    </feMerge>
  </filter>
</defs>
<text x="640" y="360" text-anchor="middle" font-size="48" fill="#1A73E8" filter="url(#titleGlow)">关键洞察</text>
```

推荐参数：
```
stdDeviation:   4–8      （越小越微妙，越大越突出）
flood-color:    品牌色或强调色（不是黑色）
flood-opacity:  0.35–0.55 （比阴影更强以确保可见）
```

**与阴影的关键区别**：没有 `<feOffset>` 元素（或 dx=0/dy=0）。转换器用此区分发光和阴影。

### 分层矩形阴影 — 高兼容性后备
最适合：与旧版 PowerPoint 的最大兼容性。在主卡片后方堆叠 2-3 个半透明矩形：

```xml
<!-- 阴影层（从后到前，偏移量最大的先） -->
<rect x="68" y="72" width="400" height="240" rx="16" fill="#000000" fill-opacity="0.03"/>
<rect x="65" y="69" width="400" height="240" rx="14" fill="#000000" fill-opacity="0.05"/>
<rect x="62" y="66" width="400" height="240" rx="12" fill="#1A73E8" fill-opacity="0.04"/>
<!-- 主卡片 -->
<rect x="60" y="60" width="400" height="240" rx="12" fill="#FFFFFF"/>
```

### 图片叠加

#### 线性渐变叠加 — 最常用
最适合：图文页面。渐变方向应与文字位置匹配（文字在左侧 → 渐变向左加深）。

```xml
<image href="..." x="0" y="0" width="1280" height="720" preserveAspectRatio="xMidYMid slice"/>
<defs>
  <linearGradient id="imgOverlay" x1="0" y1="0" x2="1" y2="0">
    <stop offset="0%"   stop-color="#1A1A2E" stop-opacity="0.85"/>
    <stop offset="55%"  stop-color="#1A1A2E" stop-opacity="0.30"/>
    <stop offset="100%" stop-color="#1A1A2E" stop-opacity="0"/>
  </linearGradient>
</defs>
<rect x="0" y="0" width="1280" height="720" fill="url(#imgOverlay)"/>
```

#### 底部渐变条
最适合：封面幻灯片和底部标题的全图页面。

```xml
<defs>
  <linearGradient id="bottomBar" x1="0" y1="0" x2="0" y2="1">
    <stop offset="0%"   stop-color="#000000" stop-opacity="0"/>
    <stop offset="100%" stop-color="#000000" stop-opacity="0.72"/>
  </linearGradient>
</defs>
<rect x="0" y="380" width="1280" height="340" fill="url(#bottomBar)"/>
```

#### 径向渐变叠加 — 暗角效果
最适合：全屏氛围幻灯片；将注意力吸引到中心。

```xml
<defs>
  <radialGradient id="vignette" cx="50%" cy="50%" r="70%">
    <stop offset="0%"   stop-color="#000000" stop-opacity="0"/>
    <stop offset="100%" stop-color="#000000" stop-opacity="0.58"/>
  </radialGradient>
</defs>
<rect x="0" y="0" width="1280" height="720" fill="url(#vignette)"/>
```

#### 品牌色叠加
最适合：需要强烈视觉品牌识别的幻灯片。

```xml
<defs>
  <linearGradient id="brandOverlay" x1="0" y1="0" x2="1" y2="0">
    <stop offset="0%"   stop-color="#005587" stop-opacity="0.80"/>
    <stop offset="100%" stop-color="#005587" stop-opacity="0.10"/>
  </linearGradient>
</defs>
<rect x="0" y="0" width="1280" height="720" fill="url(#brandOverlay)"/>
```

### 快速参考表

| 场景 | 推荐技术 | 避免 |
|------|----------|------|
| 卡片/面板阴影 | 滤镜软阴影（`flood-opacity` ≤ 0.12） | 硬黑阴影 |
| 强调/CTA 按钮 | 彩色阴影（同色系） | 通用灰色阴影 |
| 标题/指标高亮 | 发光滤镜（品牌色，无偏移） | 在正文上过度使用 |
| 文字在图片上 | 线性渐变叠加（方向匹配文字侧） | 整个图片均匀平铺透明度 |
| 封面/全图幻灯片 | 底部渐变条 + 品牌色 | 纯黑色叠加 |
| 氛围/主视觉幻灯片 | 径向暗角 | 未处理的原始图片 |
| 需要最大 PPT 兼容性 | 分层矩形阴影 | 基于滤镜的阴影 |

## 6. 描边、文本与形状效果

### stroke-dasharray — 虚线/点线

转换为原生 PPTX `<a:prstDash>`。使用预设模式以获得最佳效果：

| SVG 值 | PPTX 预设 | 最适合 |
|--------|-----------|--------|
| `4,4` | Dash | 通用虚线、分隔线 |
| `2,2` | Dot (sysDot) | 微妙点状边框、占位符轮廓 |
| `8,4` | Long dash | 时间线连接器、流程箭头 |
| `8,4,2,4` | Long dash-dot | 技术图纸、尺寸线 |

```xml
<rect x="60" y="60" width="400" height="240" rx="12"
  fill="none" stroke="#999999" stroke-width="2" stroke-dasharray="4,4"/>

<line x1="100" y1="360" x2="1180" y2="360"
  stroke="#CCCCCC" stroke-width="1" stroke-dasharray="2,2"/>
```

### stroke-linejoin

控制线段在角点的连接方式。支持的值转换为原生 PPTX 线连接类型：

| SVG 值 | PPTX 等效 | 最适合 |
|--------|-----------|--------|
| `round` | Round join | 平滑折线图、有机形状 |
| `bevel` | Bevel join | 技术图表 |
| `miter` | Miter join（默认） | 尖角矩形、箭头 |

```xml
<polyline points="100,200 200,100 300,200" fill="none"
  stroke="#1A73E8" stroke-width="3" stroke-linejoin="round"/>
```

### text-decoration

支持的文本装饰转换为原生 PPTX 文本格式：

| SVG 值 | PPTX 等效 | 最适合 |
|--------|-----------|--------|
| `underline` | 单下划线 | 强调、链接、关键术语 |
| `line-through` | 删除线 | 已移除项目、前后对比 |

```xml
<text x="100" y="200" font-size="20" fill="#333333" text-decoration="underline">重要术语</text>

<!-- 每个 tspan 的装饰 -->
<text x="100" y="240" font-size="18" fill="#333333">
  常规文本 <tspan text-decoration="line-through" fill="#999999">旧值</tspan> 新值
</text>
```

### 渐变填充 — linearGradient 和 radialGradient

在 `<defs>` 中定义渐变，通过 `fill="url(#id)"` 引用，转换为原生 PPTX `<a:gradFill>`。将其用作形状填充（不仅仅是叠加）以获得抛光表面。

**线性渐变** — 最适合按钮、标题栏、背景面板：
```xml
<defs>
  <linearGradient id="btnGrad" x1="0" y1="0" x2="1" y2="0">
    <stop offset="0%" stop-color="#1A73E8"/>
    <stop offset="100%" stop-color="#0D47A1"/>
  </linearGradient>
</defs>
<rect x="540" y="600" width="200" height="48" rx="24" fill="url(#btnGrad)"/>
```

**径向渐变** — 最适合聚光灯背景、圆形装饰：
```xml
<defs>
  <radialGradient id="spotBg" cx="50%" cy="50%" r="70%">
    <stop offset="0%" stop-color="#1A73E8" stop-opacity="0.15"/>
    <stop offset="100%" stop-color="#1A73E8" stop-opacity="0"/>
  </radialGradient>
</defs>
<circle cx="640" cy="360" r="300" fill="url(#spotBg)"/>
```

### transform: rotate — 元素旋转

旋转转换为原生 PPTX `<a:xfrm rot="...">`。支持所有元素类型：`rect`, `circle`, `ellipse`, `line`, `path`, `polygon`, `polyline`, `image`, 和 `text`。

```xml
<!-- 旋转装饰元素 -->
<rect x="100" y="100" width="60" height="60" fill="#1A73E8" fill-opacity="0.1"
  transform="rotate(45, 130, 130)"/>

<!-- 旋转文本标签 -->
<text x="50" y="400" font-size="14" fill="#999999"
  transform="rotate(-90, 50, 400)">Y 轴标签</text>
```

**语法**：`rotate(angle)` 或 `rotate(angle, cx, cy)`，其中 `cx,cy` 是旋转中心。正角度顺时针旋转。

### 圆弧路径 — 环形图/饼图

绘制环形图或饼图扇区时，必须精确使用三角函数计算圆弧端点坐标。**绝不估算或近似圆弧端点** — 即使小误差也会产生完全错误的形状。

**计算公式**（中心 `cx,cy`，半径 `r`，角度 `θ` 以度为单位）：
```
x = cx + r × cos(θ × π / 180)
y = cy + r × sin(θ × π / 180)
```

**关键规则**：
1. 从 **-90°**（12 点钟位置）开始，顺时针方向
2. 每个扇区跨度为 `百分比 × 360°`
3. 当扇区 > 180° 时使用**大弧标志 = 1**，否则为 **0**
4. sweep-direction = 1（顺时针）用于外弧，0（逆时针）用于返回内弧
5. **始终验证**所有扇区角度之和等于 360°，且最后一个扇区的终点与第一个扇区的起点匹配

**示例 — 75% 环形图扇区**（中心 400,400，外半径 180，内半径 100）：
```
起始角：-90°    → 外点(400, 220)，内点(400, 300)
结束角：-90+270=180° → 外点(220, 400)，内点(300, 400)
大弧标志：1（270° > 180°）

<path d="M 400,220 A 180,180 0 1,1 220,400 L 300,400 A 100,100 0 1,0 400,300 Z"/>
```

### 对角线上的多边形箭头

使用 `<polygon>` 三角形作为箭头时（因为 `marker-end` 被禁用），**水平或垂直线**上的箭头可以使用简单的点偏移。但**对角线**上的箭头必须旋转三角形顶点以匹配线方向。

**方法**：使用线的方向向量计算三角形点：

```
给定从 (x1,y1) 到 (x2,y2) 的线：
1. 方向向量：dx = x2-x1, dy = y2-y1
2. 归一化：len = √(dx²+dy²), ux = dx/len, uy = dy/len
3. 垂直向量：px = -uy, py = ux
4. 箭头尖端 = (x2, y2)
5. 后点 1 = (x2 - ux×12 + px×5,  y2 - uy×12 + py×5)
6. 后点 2 = (x2 - ux×12 - px×5,  y2 - uy×12 - py×5)
```

**示例 — 对角线**从 (260,310) 到 (370,430)：
```
dx=110, dy=120, len≈162.8, ux=0.676, uy=0.737
px=-0.737, py=0.676
尖端：(370, 430)
后点1：(370-8.1-3.7, 430-8.8+3.4) = (358.2, 424.6)
后点2：(370-8.1+3.7, 430-8.8-3.4) = (365.6, 417.8)

<polygon points="370,430 365.6,417.8 358.2,424.6" fill="#C8A96E"/>
```

⚠️ **绝不要在对角线上使用固定的向下/向右三角形** — 箭头会指向错误的方向。

## 7. 项目目录结构

```
project/
├── svg_output/    # 原始 SVG（执行师输出，包含占位符）
├── svg_final/     # 后处理后的最终 SVG（finalize_svg.py 输出）
├── images/        # 图片资产（用户提供 + AI 生成）
├── notes/         # 演讲备注（.md 文件，匹配 SVG 名称）
│   └── total.md   # 完整的演讲备注文档（分割前）
├── templates/     # 项目模板（如果有）
└── *.pptx         # 导出的 PPT 文件
```

---

# 五、SVG 图片嵌入指南：`svg-image-embedding.md`

**作用**：在 SVG 中引用或嵌入图片的技术指南。

## 1. 图片资源清单格式

在设计规范中定义，每个图片有状态注释。如果图片方式包含“B) 用户提供”，则策略师在完成八项确认后必须立即运行 `analyze_images.py`，并在输出设计规范前完成清单。

```markdown
| 文件名 | 尺寸 | 用途 | 状态 | 生成描述 |
|--------|------|------|------|----------|
| cover_bg.png | 1280x720 | 封面背景 | 待生成 | 现代科技抽象背景，深蓝色渐变 |
| product.png | 600x400 | 第 3 页 | 已存在 | - |
| team.png | 600x400 | 第 5 页 | 占位符 | 团队协作场景（后续添加） |
```

### 三种状态类型

| 状态 | 含义 | 执行师处理 |
|------|------|-----------|
| **待生成** | 需要 AI 生成，有描述 | 先生成图片放入 `images/`，然后用 `<td>` 引用 |
| **已存在** | 用户已有图片 | 放入 `images/`，用 `<td>` 引用 |
| **占位符** | 尚未处理 | 使用虚线边框占位符；后续替换 |

## 2. 工作流程

```
1. 策略师定义图片需求 → 添加图片资源清单，标注每个状态
2. 图片准备（待生成/已存在） → 放入 project/images/
3. 执行师生成 SVG（svg_output/）
   ├── 已存在/待生成 → <image href="../images/xxx.png" .../>
   └── 占位符 → 虚线边框 + 描述文字
4. 预览：python3 -m http.server -d <project_path> 8000 → /svg_output/<filename>.svg
5. 后处理与导出
   ├── python3 scripts/finalize_svg.py <project_path>
   └── python3 scripts/svg_to_pptx.py <project_path> -s final
```

> 推荐：生成期间在 `svg_output/` 中保持外部引用。通过 `finalize_svg.py` 后处理自动将图片嵌入 `svg_final/`，然后从 `svg_final/` 导出 PPTX。

## 3. 外部引用 vs Base64 嵌入

| 方法 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **外部引用** | 文件小，迭代快，易于替换 | 预览需要从项目根目录启动 HTTP 服务器 | `svg_output/` 开发阶段 |
| **Base64 嵌入** | 文件自包含，导出稳定 | 文件大 | `svg_final/` 交付阶段 |

## 4. 方法 1：外部引用（推荐用于生成阶段）

### 语法
```xml
<image href="../images/image.png" x="0" y="0" width="1280" height="720"
       preserveAspectRatio="xMidYMid slice"/>
```

### 关键属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `href` | 图片路径（相对或绝对） | `"../images/cover.png"` |
| `x`, `y` | 图片左上角位置 | `x="0" y="0"` |
| `width`, `height` | 图片显示尺寸 | `width="1280" height="720"` |
| `preserveAspectRatio` | 缩放模式 | `"xMidYMid slice"` |

### preserveAspectRatio 常用值

| 值 | 效果 |
|----|------|
| `xMidYMid slice` | 中心裁剪（类似 CSS `cover`） |
| `xMidYMid meet` | 完整显示（类似 CSS `contain`） |
| `none` | 拉伸填充，不保持宽高比 |

### 预览方法

浏览器安全限制阻止从直接打开的 SVG 加载外部图片。从项目根目录启动 HTTP 服务器：

```bash
python3 -m http.server -d <project_path> 8000
# 访问 http://localhost:8000/svg_output/your_file.svg
```

## 5. 方法 2：Base64 嵌入（推荐用于交付阶段）

### 语法
```xml
<image href="data:image/png;base64,iVBORw0KGgo..." x="0" y="0" width="1280" height="720"/>
```

### MIME 类型

| MIME 类型 | 文件格式 |
|-----------|----------|
| `image/png` | PNG |
| `image/jpeg` | JPG/JPEG |
| `image/gif` | GIF |
| `image/webp` | WebP |
| `image/svg+xml` | SVG |

## 6. 转换过程

### 推荐：使用 finalize_svg.py（统一流水线）

```bash
python3 scripts/finalize_svg.py <project_path>         # 图标、图片、文本、圆角矩形 — 一次完成
python3 scripts/svg_to_pptx.py <project_path> -s final  # 从最终版本导出 PPTX
```

### 独立使用：embed_images.py（高级用法）

用于在不运行完整流水线的情况下处理特定 SVG：

```bash
python3 scripts/svg_finalize/embed_images.py <svg_file>                         # 单个文件
python3 scripts/svg_finalize/embed_images.py <project_path>/svg_output/*.svg    # 批量
python3 scripts/svg_finalize/embed_images.py --dry-run <project_path>/svg_output/*.svg  # 预览
```

## 7. 最佳实践

### 图片优化

在嵌入前压缩图片以减少文件大小：

```bash
convert input.png -quality 85 -resize 1920x1080\> output.png  # ImageMagick
pngquant --quality=65-80 input.png -o output.png               # pngquant（推荐）
```

### 文件组织

```
project/
├── images/            # 图片资产
├── sources/           # 源文件及其附带图片
│   └── article_files/
├── svg_output/        # 原始版本（外部引用）
└── svg_final/         # 最终版本（图片已嵌入）
```

### 圆角处理（clipPath 被禁止）

由于 `clipPath` 与 PPT 不兼容，禁止使用裁剪路径实现图片圆角。替代方案：
- 在图片生成时处理圆角（导出带圆角的 PNG）
- 或在边缘叠加一个相同大小的圆角矩形（视觉模拟）

## 8. FAQ

**问：直接打开 SVG 看不到图片？**
浏览器安全策略阻止跨目录请求。从项目根目录启动 HTTP 服务器，或先运行 `finalize_svg.py` 然后从 `svg_final/` 查看。

**问：Base64 文件太大？**
压缩原始图片，使用 JPEG 格式，降低分辨率（匹配实际显示尺寸）。

**问：如何反向提取 Base64 图片？**
```bash
base64 -d image.b64 > image.png
```

---

# 六、模板设计师：`template-designer.md`

**作用**：独立角色，用于通过 `/create-template` 工作流创建全局布局模板。**与图片处理无关**，属于设计资源建设。

## 1. 核心使命

基于最终确定的模板简报，为**全局模板库**生成可复用的页面模板。

> 这是一个独立角色：仅在 `/create-template` 工作流中触发。**不是**主 PPT 生成流水线中的项目级模板选择/自定义步骤。

## 2. 使用方式

- **触发**：`/create-template` 工作流
- **输出位置**：`templates/layouts/<template_name>/`
- **输入**：最终确定的模板简报（模板 ID、显示名称、分类、适用场景、调性、主题模式、画布格式、可选的参考资产）

当工作流提供 PPTX 导入输出时，有效输入包成为：
- 最终确定的模板简报
- `manifest.json`
- `analysis.md`
- 导出的 `assets/`
- 可选的截图用于视觉交叉检查

PPTX 支持的模板创建的输入优先级：
1. `manifest.json` 用于事实元数据
2. 导出的 `assets/` 用于可重用视觉资源
3. `analysis.md` 用于页面类型指导
4. 截图/原始 PPTX 仅用于样式验证

## 3. 核心模板清单

| # | 文件名 | 用途 | 描述 |
|---|--------|------|------|
| 01 | `01_cover.svg` | 封面 | 固定结构：标题、副标题、日期、组织 |
| 02 | `02_chapter.svg` | 章节页 | 固定结构：章节编号、章节标题 |
| 03 | `03_content.svg` | 内容页 | 灵活结构：只定义页眉/页脚；内容区由 AI 自由布局 |
| 04 | `04_ending.svg` | 结尾页 | 固定结构：致谢信息、联系方式 |
| -- | `02_toc.svg` | 目录 | 可选：目录标题、章节列表（编号+标题） |

**设计理念**：模板定义视觉一致性和结构页面；内容页保持最大灵活性。

**命名说明**：目录页保持 `02_toc.svg` 命名以保持模板库兼容性和排序顺序。

### 可选的扩展页面（按需）
- 过渡/子章节页（如 `05_section_break.svg`）
- 附录页（如 `06_appendix.svg`）
- 免责/保密页（如 `07_disclaimer.svg`）

## 4. 模板设计规范

### 4.1 必须生成 design_spec.md

创建全局模板时，必须生成 `design_spec.md`，包含：

```markdown
# [模板名称] - 设计规范

## I. 模板概述（名称、使用场景、设计调性）
## II. 画布规范（16:9, 1280x720, viewBox）
## III. 配色方案（主色、次色、强调色 HEX 值）
## IV. 字体系统（字体栈、字号层级）
## V. 页面结构（通用布局、装饰设计）
## VI. 页面类型（4 种核心页面类型）
## VII. 布局模式（推荐）
## VIII. 间距规范
## IX. SVG 技术约束
## X. 占位符规范
```

### 4.2 继承设计规范

模板必须严格遵循最终确定的模板简报和生成的 `design_spec.md`：
- **画布尺寸**：viewBox 与设计规范匹配
- **配色方案**：使用规范中的主色、次色、强调色
- **字体方案**：使用规范中的字体预设
- **布局原则**：边距和间距符合规范

如果存在 PPTX 导入输出：
- 优先使用导入的主题颜色和字体，而非视觉猜测值
- 在模板全局有意义的地方重用提取的背景和标志
- 将 `analysis.md` 中的页面类型候选视为提示，而非保证

### 4.3 PPTX 导入简化规则

导入的 PPTX 是**参考源**，而非直接转换目标。

**要做**：
- 保留品牌资产、重复背景和稳定的结构主题
- 将布局重建为符合 PPT Master 约束的干净 SVG 结构
- 将重复的装饰片段简化为更少数量的可维护 SVG 元素
- 当原始装饰层过于复杂无法干净重建时，使用背景图片资产

**不要做**：
- 尝试 1:1 翻译每个 PowerPoint 形状、组、阴影或装饰片段
- 镜像 PPT 特定的复杂性，使生成的 SVG 脆弱或难以编辑
- 引入密集的低价值矢量细节，不会实质改善模板复用

### 4.4 占位符标记

对可替换内容使用清晰的占位符标记：

```xml
<!-- 文本占位符 -->
<text x="80" y="320" fill="#FFFFFF" font-size="48" font-weight="bold">
  {{TITLE}}
</text>

<!-- 内容区占位符（仅内容页） -->
<rect x="40" y="90" width="1200" height="550" fill="#FFFFFF" rx="8"/>
<text x="640" y="365" text-anchor="middle" fill="#CBD5E1" font-size="16">
  {{CONTENT_AREA}}
</text>
```

### 4.5 占位符参考

| 占位符 | 用途 | 适用模板 |
|--------|------|----------|
| `{{TITLE}}` | 主标题 | 封面 |
| `{{SUBTITLE}}` | 副标题 | 封面 |
| `{{DATE}}` | 日期 | 封面 |
| `{{AUTHOR}}` | 作者/组织 | 封面 |
| `{{CHAPTER_NUM}}` | 章节编号 | 章节页 |
| `{{CHAPTER_TITLE}}` | 章节标题 | 章节页 |
| `{{CHAPTER_DESC}}` | 章节描述 | 章节页 |
| `{{PAGE_TITLE}}` | 页面标题 | 内容页 |
| `{{KEY_MESSAGE}}` | 关键结论 | 内容页（咨询风格） |
| `{{CONTENT_AREA}}` | 内容区 | 内容页 |
| `{{SECTION_NAME}}` | 章节名称 | 内容页页脚 |
| `{{SOURCE}}` | 数据来源 | 内容页页脚 |
| `{{PAGE_NUM}}` | 页码 | 内容页、结尾页 |
| `{{THANK_YOU}}` | 致谢信息 | 结尾页 |
| `{{ENDING_SUBTITLE}}` | 结尾副标题 | 结尾页 |
| `{{CLOSING_MESSAGE}}` | 结语 | 结尾页 |
| `{{CONTACT_INFO}}` | 联系方式 | 结尾页 |
| `{{COPYRIGHT}}` | 版权 | 结尾页 |

对于**新创建的库模板**中的目录页，使用索引占位符：
- `{{TOC_ITEM_1_TITLE}}`, `{{TOC_ITEM_1_DESC}}`
- `{{TOC_ITEM_2_TITLE}}`, `{{TOC_ITEM_2_DESC}}`
- ...

**不要**为新模板创建其他目录占位符家族，如 `{{CHAPTER_01_TITLE}}`。现有模板可能包含遗留占位符变体，但新的库资产应收敛到索引目录契约。

当从导入的 PPTX 参考重建时，占位符插入优先于视觉模仿。如果原始布局没有为规范占位符留出足够空间，调整布局而不是发明一次性的占位符家族。

## 5. 输出要求

### 文件保存位置

```
templates/layouts/<template_name>/
├── design_spec.md     # 设计规范（必需）
├── 01_cover.svg
├── 02_chapter.svg
├── 02_toc.svg          # 可选
├── 03_content.svg
├── 04_ending.svg
└── *.png / *.jpg       # 图片资产（如果有）
```

### 模板预览

每个模板生成后，提供简短的摘要表列出每个模板的状态。

如果模板基于 PPTX 导入输出，简要说明：
- 哪些提取的资产被直接重用
- 哪些原始复杂装饰被有意简化
- 是否有任何页面类型映射需要超出导入启发式的判断

## 6. 使用预建模板库（可选）

如果已有合适的模板资源，直接使用而非生成新模板：

1. **复制模板**：将模板文件复制到项目的 `templates/` 目录
2. **调整颜色**：根据项目设计规范修改颜色
3. **自定义**：进行项目特定的调整

本节描述下游重用。`Template_Designer` 角色本身负责首先创建或规范化可重用的库资产。

**示例库结构**（查询 `templates/layouts/layouts_index.json`）：
```
templates/layouts/
├── exhibit/           # 展览风格（结论先行，数据驱动）
├── 科技蓝商务/         # 科技蓝商务风格
└── smart_red/         # 智能红橙色风格
```

## 7. 阶段完成检查点

```markdown
## Template_Designer Phase Complete

- [x] 读取 `references/template-designer.md`
- [x] 生成 4 个核心页面模板
- [ ] 目录页模板（可选）
- [ ] 可选扩展页面（如果需要）
- [x] 所有模板保存到 `templates/layouts/<template_name>/`
- [x] 模板遵循设计规范（颜色、字体、布局）
- [x] 占位符标记清晰且标准化
- [ ] **下一步**：验证资产并在 `layouts_index.json` 中注册模板
```

---

## 专题总结

你现在已经完成了“设计与图像处理”专题的 6 个核心文件的逐项解析：

| 文件 | 核心职能 |
|------|----------|
| `image-generator.md` | 图片生成的全流程操作手册（提示词、类型分类、后端调用） |
| `image-layout-spec.md` | 根据图片宽高比强制计算布局尺寸的规范 |
| `canvas-formats.md` | 10 种画布格式的定义和布局原则 |
| `shared-standards.md` | 全角色共享的 SVG 技术约束、后处理流水线、视觉效果写法 |
| `svg-image-embedding.md` | SVG 中引用/嵌入图片的技术指南 |
| `template-designer.md` | 创建全局布局模板的独立角色规范 |

这些文件共同构成了 PPT Master 中所有**视觉资源生成与布局**的规则体系。后续如果你想深入其他专题（如执行师的具体 SVG 生成技巧、后处理脚本的内部实现等），请告诉我。