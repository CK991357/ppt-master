## 模板新增与SVG设计完整指南

你问的问题非常关键。简单回答：**是的，直接在 `templates/layouts/` 下新建文件夹，放入对应的 SVG 文件和 `design_spec.md`，并在 `layouts_index.json` 中注册，就可以被 AI 发现和使用。** 但要让模板真正可用，需要遵循一套严格的规范和设计方法。下面详细说明。

---

## 一、模板新增的操作步骤

### 1. 创建模板目录
```
skills/ppt-master/templates/layouts/my_template/
```

### 2. 准备必需文件
- `design_spec.md` —— 设计规范文档（必须）
- `01_cover.svg` —— 封面页（必须）
- `02_chapter.svg` —— 章节页（必须）
- `03_content.svg` —— 内容页（必须）
- `04_ending.svg` —— 结尾页（必须）
- `02_toc.svg` —— 目录页（可选，但强烈推荐）
- 任何模板中引用的图片（如 logo.png）放在同一目录下

### 3. 在 `layouts_index.json` 中注册
在 `layouts_index.json` 中添加条目（见之前章节的说明），确保 `meta.total` 和 `layouts` 对象更新。

### 4. 验证模板
```bash
python3 skills/ppt-master/scripts/svg_quality_checker.py templates/layouts/my_template --format ppt169
```

---

## 二、SVG 模板的设计与编制方法

### 2.1 基础规范（所有 SVG 必须遵守）

| 属性 | 值 | 说明 |
|------|-----|------|
| `viewBox` | `0 0 1280 720` | 固定画布（PPT 16:9） |
| `width` / `height` | `100%` / `100%` | 自适应 |
| 背景 | 使用 `<rect>` 填充 | 不用 `style` 或外部 CSS |
| 文本换行 | 使用 `<tspan>` | 禁止 `<foreignObject>` |
| 透明度 | 使用 `fill-opacity` / `stroke-opacity` | 禁止 `rgba()` |
| 阴影/发光 | 使用 `<filter>`（`feGaussianBlur` + `feOffset`） | 可转换为 PPT 效果 |
| 图标 | 使用 `<use data-icon="...">` 占位符 | 后处理自动嵌入 |
| 图片 | 使用 `<image href="../images/...">` | 相对路径，后处理嵌入 |
| 禁止元素 | `clipPath`, `mask`, `<style>`, `class`, `foreignObject`, `textPath`, `animate*`, `marker` | 详见 `shared-standards.md` |

### 2.2 占位符规范（新模板必须使用）

| 页面类型 | 占位符 | 说明 |
|----------|--------|------|
| 封面 | `{{TITLE}}`, `{{SUBTITLE}}`, `{{DATE}}`, `{{AUTHOR}}` | 主标题、副标题、日期、作者/机构 |
| 章节页 | `{{CHAPTER_NUM}}`, `{{CHAPTER_TITLE}}` | 章节编号、章节标题 |
| 目录页 | `{{TOC_ITEM_1_TITLE}}`, `{{TOC_ITEM_1_DESC}}` 等 | 索引占位符，支持多行 |
| 内容页 | `{{PAGE_TITLE}}`, `{{CONTENT_AREA}}`, `{{PAGE_NUM}}`, `{{SOURCE}}` | 页面标题、内容区、页码、数据来源 |
| 结尾页 | `{{THANK_YOU}}`, `{{ENDING_SUBTITLE}}`, `{{CLOSING_MESSAGE}}`, `{{CONTACT_INFO}}`, `{{COPYRIGHT}}` | 致谢、结语、联系方式、版权 |

**占位符在 SVG 中的写法**：直接放在 `<text>` 元素内，例如：
```xml
<text x="640" y="360" text-anchor="middle" font-size="48" fill="#333">{{TITLE}}</text>
```

执行师在生成 SVG 时会用实际内容替换这些占位符。

### 2.3 各页面类型的设计要点

#### **01_cover.svg —— 封面页**

**设计目标**：第一印象，包含主标题、副标题、日期、作者/机构。

**布局结构**：
- 全画幅背景（可以是纯色、渐变或图片）。
- 标题居中或左对齐，通常位于画布中上部。
- 副标题、日期、作者放在标题下方或底部。

**示例结构**：
```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1280 720">
  <defs>
    <linearGradient id="bg" x1="0" y1="0" x2="0" y2="1">
      <stop offset="0%" stop-color="#1E3A5F"/>
      <stop offset="100%" stop-color="#0A1A2E"/>
    </linearGradient>
  </defs>
  <rect width="1280" height="720" fill="url(#bg)"/>
  
  <!-- 主标题 -->
  <text x="640" y="320" text-anchor="middle" font-size="72" font-weight="bold" fill="#FFFFFF">{{TITLE}}</text>
  <!-- 副标题 -->
  <text x="640" y="380" text-anchor="middle" font-size="28" fill="#CCCCCC">{{SUBTITLE}}</text>
  <!-- 日期 -->
  <text x="640" y="650" text-anchor="middle" font-size="16" fill="#888888">{{DATE}}</text>
  <!-- 作者 -->
  <text x="640" y="680" text-anchor="middle" font-size="14" fill="#888888">{{AUTHOR}}</text>
</svg>
```

#### **02_toc.svg —— 目录页（可选）**

**设计目标**：列出各章节标题和页码。

**布局**：通常使用多行列表，每行有章节编号、标题、页码。

**占位符**：使用 `{{TOC_ITEM_1_TITLE}}`, `{{TOC_ITEM_1_PAGE}}` 等（注意：官方索引占位符是 `{{TOC_ITEM_1_TITLE}}` 和 `{{TOC_ITEM_1_DESC}}`，页码可能需要单独设计，或放在描述中）。建议使用 `{{TOC_ITEM_1_TITLE}}` 和 `{{TOC_ITEM_1_DESC}}`，页码可以放在描述里。

**示例**：
```xml
<text x="100" y="150" font-size="36" font-weight="bold">目录</text>
<text x="100" y="220" font-size="20">{{TOC_ITEM_1_TITLE}} ................ {{TOC_ITEM_1_PAGE}}</text>
<text x="100" y="260" font-size="20">{{TOC_ITEM_2_TITLE}} ................ {{TOC_ITEM_2_PAGE}}</text>
...
```

**注意**：由于目录页的条目数量不固定，执行师需要动态生成。模板只需给出**样式示例**（比如一行的高度、字体、对齐方式），实际内容由执行师根据章节数量生成。因此，`02_toc.svg` 通常只是一个**参考模板**，不是最终直接使用的文件。更常见的做法是：执行师根据内容大纲自行生成目录页，不依赖模板。如果一定要提供模板，应包含一个占位符块（如 `<!-- TOC_ITEMS -->`），执行师会替换成实际列表。

#### **02_chapter.svg —— 章节页**

**设计目标**：分隔章节，通常包含章节编号和章节标题，可能带有装饰背景。

**布局**：大号章节编号（如 `01`），章节标题，可能还有英文副标题或描述。

**占位符**：`{{CHAPTER_NUM}}`, `{{CHAPTER_TITLE}}`, `{{CHAPTER_TITLE_EN}}`（可选）。

**示例**：
```xml
<rect width="1280" height="720" fill="#F5F5F5"/>
<text x="640" y="360" text-anchor="middle" font-size="120" font-weight="bold" fill="#E0E0E0">{{CHAPTER_NUM}}</text>
<text x="640" y="420" text-anchor="middle" font-size="48" font-weight="bold" fill="#333">{{CHAPTER_TITLE}}</text>
```

#### **03_content.svg —— 内容页**

**设计目标**：最常用的页面，展示标题、要点、图表、图片。模板应定义**页眉、页脚、内容区**，内容区由执行师自由布局。

**布局**：
- **页眉**：包含页面标题（`{{PAGE_TITLE}}`）、可能的分隔线或装饰。
- **内容区**：一个大矩形或区域，执行师会在其中放置文本、图片、图表等。可以用 `{{CONTENT_AREA}}` 占位符标记，但通常执行师会直接在该区域绘制内容，不需要占位符。
- **页脚**：页码（`{{PAGE_NUM}}`）、数据来源（`{{SOURCE}}`）、公司 logo 等。

**示例**：
```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1280 720">
  <defs>...</defs>
  <!-- 背景 -->
  <rect width="1280" height="720" fill="#FFFFFF"/>
  
  <!-- 页眉 -->
  <text x="80" y="80" font-size="32" font-weight="bold" fill="#333">{{PAGE_TITLE}}</text>
  <line x1="80" y1="100" x2="1200" y2="100" stroke="#CCCCCC" stroke-width="2"/>
  
  <!-- 内容区（执行师在这里绘制内容，模板只提供位置建议） -->
  <!-- 可放置一个参考框，但实际会被覆盖 -->
  <rect x="80" y="130" width="1120" height="520" fill="none" stroke="#E0E0E0" stroke-dasharray="4,4" rx="8"/>
  
  <!-- 页脚 -->
  <text x="80" y="690" font-size="12" fill="#999999">{{SOURCE}}</text>
  <text x="1200" y="690" text-anchor="end" font-size="12" fill="#999999">{{PAGE_NUM}}</text>
</svg>
```

#### **04_ending.svg —— 结尾页**

**设计目标**：致谢、联系方式、版权信息。

**布局**：居中或底部对齐，包含感谢语、联系信息、二维码等。

**占位符**：`{{THANK_YOU}}`, `{{ENDING_SUBTITLE}}`, `{{CLOSING_MESSAGE}}`, `{{CONTACT_INFO}}`, `{{COPYRIGHT}}`。

**示例**：
```xml
<rect width="1280" height="720" fill="#F8F9FA"/>
<text x="640" y="300" text-anchor="middle" font-size="56" font-weight="bold" fill="#333">{{THANK_YOU}}</text>
<text x="640" y="360" text-anchor="middle" font-size="24" fill="#666">{{ENDING_SUBTITLE}}</text>
<text x="640" y="420" text-anchor="middle" font-size="18" fill="#888">{{CLOSING_MESSAGE}}</text>
<text x="640" y="650" text-anchor="middle" font-size="14" fill="#999">{{CONTACT_INFO}}</text>
<text x="640" y="680" text-anchor="middle" font-size="12" fill="#CCCCCC">{{COPYRIGHT}}</text>
```

### 2.4 设计工具与技巧

- **使用矢量设计软件**：Inkscape（免费）、Adobe Illustrator、Figma。导出为纯 SVG（不要包含 Inkscape 或 Illustrator 专有元素，如 `sodipodi:` 命名空间）。
- **手动编写**：对于简单模板，可以直接用文本编辑器写 SVG。参考现有模板（如 `google_style`）的代码。
- **坐标与布局**：所有元素使用绝对坐标，不要依赖 CSS 布局。推荐使用 `text-anchor="middle"` 居中文本。
- **响应式**：不要使用百分比宽度/高度（除了最外层的 `width="100%"`），内部元素都用固定像素值。
- **测试**：在浏览器中打开 SVG 预览，或用 `python3 -m http.server` 从项目根目录启动服务器查看。

### 2.5 模板设计文档 `design_spec.md` 必须包含的内容

参考 `google_style/design_spec.md`，至少包括：
- 画布规范（viewBox、边距、安全区）
- 配色方案（主色、辅色、文字颜色、背景色）
- 字体方案（字体栈、字号层级）
- 页面结构（页眉、内容区、页脚高度）
- 页面类型（封面、目录、章节、内容、结尾）的布局描述
- 布局模式（推荐的单列、两栏、三栏等）
- 间距规范
- SVG 技术约束提醒
- 占位符规范

这份文档会被策略师读取，用于生成项目的 `design_spec.md`，并指导执行师生成页面。

---

## 三、示例：创建一个简单的自定义模板

假设你要创建一个“极简商务蓝”模板。

**1. 创建目录** `templates/layouts/business_blue/`

**2. 创建 `design_spec.md`**（简化版）：
```markdown
# Business Blue Template

## Color Scheme
- Primary: `#003366`
- Secondary: `#4A90D9`
- Accent: `#FFA500`
- Text: `#333333`
- Background: `#FFFFFF`

## Typography
- Font: system-ui, -apple-system, sans-serif
- Title: 48px bold
- Body: 20px regular

## Page Structure
- Header height: 100px
- Footer height: 60px
- Content area: y=120 to 660
```

**3. 创建 `01_cover.svg`**：
```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1280 720">
  <rect width="1280" height="720" fill="#003366"/>
  <text x="640" y="360" text-anchor="middle" font-size="64" fill="#FFFFFF" font-family="system-ui">{{TITLE}}</text>
  <text x="640" y="420" text-anchor="middle" font-size="28" fill="#CCCCCC">{{SUBTITLE}}</text>
  <text x="640" y="650" text-anchor="middle" font-size="16" fill="#888888">{{DATE}} | {{AUTHOR}}</text>
</svg>
```

**4. 创建 `03_content.svg`**：
```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1280 720">
  <rect width="1280" height="720" fill="#FFFFFF"/>
  <rect x="0" y="0" width="1280" height="6" fill="#003366"/>
  <text x="80" y="70" font-size="32" font-weight="bold" fill="#003366">{{PAGE_TITLE}}</text>
  <line x1="80" y1="90" x2="1200" y2="90" stroke="#4A90D9" stroke-width="3"/>
  <!-- 内容区 -->
  <rect x="80" y="120" width="1120" height="520" fill="#F8F9FA" rx="8"/>
  <text x="640" y="380" text-anchor="middle" font-size="18" fill="#999999">{{CONTENT_AREA}}</text>
  <text x="80" y="690" font-size="12" fill="#999999">{{SOURCE}}</text>
  <text x="1200" y="690" text-anchor="end" font-size="12" fill="#999999">{{PAGE_NUM}}</text>
</svg>
```

**5. 注册到 `layouts_index.json`**（添加 `business_blue` 条目）。

**6. 验证**：
```bash
python3 skills/ppt-master/scripts/svg_quality_checker.py templates/layouts/business_blue --format ppt169
```

---

## 四、常见问题

**Q: 模板中的图片（如 logo）如何引用？**  
A: 将图片文件放在模板目录下，在 SVG 中用相对路径 `href="logo.png"`。执行师复制模板到项目时，图片会被复制到项目的 `images/` 目录，SVG 中的路径会被自动修正为 `../images/logo.png`。

**Q: 内容页的内容区应该留空还是放占位符？**  
A: 建议放一个浅色的虚线框和 `{{CONTENT_AREA}}` 文本，表示这是内容区域。实际执行时，执行师会忽略这个框，直接在该区域绘制内容。

**Q: 如何确保模板被 AI 正确推荐？**  
A: 在 `layouts_index.json` 中填写准确的 `summary`, `tone`, `keywords`，AI 在推荐时会根据用户内容和这些元数据进行匹配。

**Q: 是否支持多页目录？**  
A: 目录页的条目数不固定，模板只能提供样式。执行师会根据实际章节数生成目录，所以 `02_toc.svg` 只是一个样式参考，不会被直接复制使用。更好的做法是不提供 `02_toc.svg`，让执行师自动生成目录。

---

通过以上方法，你可以轻松创建符合 PPT Master 规范的全局模板，并让 AI 在生成 PPT 时自动发现和使用它们。