## 素材定制完整操作流程：从全局模板到项目专属资产

在 PPT Master 中，素材（布局、图表、图标）的定制分为两个层次：**全局模板库**（供所有项目复用）和**项目专属素材**（仅当前项目使用）。下面以 `google_style` 为例，详细说明如何实现定制。

---

### 一、全局模板库 vs 项目专属素材：选择指南

| 场景 | 推荐方式 | 理由 |
|------|----------|------|
| 你要为某个客户/品牌创建一套可重复使用的设计系统 | 添加全局模板（`templates/layouts/` 新目录） | 其他项目可复用，且会被 AI 自动发现和推荐 |
| 你只想为当前项目临时调整颜色、替换 logo，不改动整体结构 | 在项目目录中覆盖 `templates/` 或 `images/` | 不影响全局，快速定制 |
| 你需要一个现有模板没有的图表类型 | 添加全局图表模板（`templates/charts/` 新文件） | 可供所有项目使用 |
| 你只需要在当前项目中使用某个特定图表样式 | 执行师根据设计规范自由生成，或手动在项目 `templates/charts/` 下添加临时图表 | 无需污染全局库 |
| 你需要新的图标（全局可用） | 添加全局图标（`templates/icons/` 新文件）并注册 | 所有项目都能用 |
| 你只需要在当前项目中临时使用几个特殊图标 | 将图标 SVG 放在项目的 `images/` 目录，在 SVG 中用 `<image href="../images/my_icon.svg"/>` 直接引用 | 简单直接，不修改全局库 |

---

### 二、为项目指定现有全局模板（以 `google_style` 为例）

这是最常用的场景：你希望项目采用已有的 `google_style` 模板。

**操作流程（AI 自动完成，用户只需在对话中确认）**：

1. **用户启动 PPT 生成**：提供源文件或描述内容。
2. **AI 执行 Step 3（模板选择）**：
   - AI 读取 `templates/layouts/layouts_index.json`，列出可用模板。
   - AI 根据内容推荐模板（例如：*“根据你的科技主题，推荐使用 google_style 模板”*）。
   - 用户确认选择 **A) Use an existing template** → 指定 `google_style`。
3. **AI 自动复制模板文件到项目**：
   ```bash
   cp templates/layouts/google_style/*.svg <project_path>/templates/
   cp templates/layouts/google_style/design_spec.md <project_path>/templates/
   cp templates/layouts/google_style/*.png <project_path>/images/ 2>/dev/null || true
   ```
4. **后续执行师生成 SVG 时**，会参考项目 `templates/` 下的模板文件，继承其背景、页眉页脚、装饰等，并替换占位符内容。

**项目目录变化**：
```
<project_path>/
├── templates/
│   ├── 01_cover.svg
│   ├── 02_chapter.svg
│   ├── 02_toc.svg
│   ├── 03_content.svg
│   ├── 04_ending.svg
│   └── design_spec.md
├── images/          # 可能包含模板中的 logo 图片
└── ...
```

> **注意**：如果项目中已存在 `templates/` 目录，AI 会覆盖其中的模板文件（但不会影响 `svg_output/` 等）。

---

### 三、创建新的全局布局模板（为所有项目复用）

如果你希望为某个品牌（如“XX 公司”）创建一套全新的布局模板，并让它出现在 AI 的模板推荐列表中。

#### 步骤 1：创建模板目录和必需文件

```bash
cd skills/ppt-master/templates/layouts
mkdir my_company_style
cd my_company_style
```

创建以下文件：

- **`design_spec.md`**：参考 `google_style/design_spec.md` 的结构，填写你的品牌颜色、字体、布局规范。**必须包含**：
  - 画布格式（通常 `0 0 1280 720`）
  - 配色方案（主色、辅色、强调色、文字颜色）
  - 字体方案（字体栈）
  - 页面结构（页眉、内容区、页脚高度）
  - 页面类型（封面、章节、内容、结尾）的布局描述
- **`01_cover.svg`**：封面 SVG。使用占位符 `{{TITLE}}`, `{{SUBTITLE}}`, `{{DATE}}`, `{{AUTHOR}}` 等。
- **`02_chapter.svg`**：章节页 SVG。使用 `{{CHAPTER_NUM}}`, `{{CHAPTER_TITLE}}` 等。
- **`03_content.svg`**：内容页 SVG。必须包含页眉和页脚，内容区留空或使用 `{{CONTENT_AREA}}` 占位符。
- **`04_ending.svg`**：结尾页 SVG。使用 `{{THANK_YOU}}`, `{{CONTACT_INFO}}` 等。
- **（可选）`02_toc.svg`**：目录页 SVG。使用 `{{TOC_ITEM_1_TITLE}}`, `{{TOC_ITEM_2_TITLE}}` 等索引占位符。
- **（可选）品牌图片**：如 logo，放在同目录下（例如 `logo.png`），在 SVG 中用相对路径引用（如 `href="logo.png"`）。实际使用时会被复制到项目的 `images/` 目录。

#### 步骤 2：遵守 SVG 技术规范

- `viewBox="0 0 1280 720"`
- 无禁用元素（`clipPath`, `mask`, `<style>`, `class`, `foreignObject`, `textPath`, `marker`, `rgba`, `<g opacity>` 等）
- 使用 `<text>` + `<tspan>` 实现换行
- 使用 `fill-opacity` / `stroke-opacity` 实现透明度
- 占位符格式：`{{PLACEHOLDER_NAME}}`

#### 步骤 3：注册到 `layouts_index.json`

编辑 `templates/layouts/layouts_index.json`：

1. **在 `categories` 中添加分类**（或使用现有分类如 `brand`）：
   ```json
   "brand": {
     "label": "Brand Style Templates",
     "layouts": ["google_style", "mckinsey", "my_company_style"]
   }
   ```

2. **在 `quickLookup` 中添加关键词**（可选）：
   ```json
   "my_industry": ["my_company_style"]
   ```

3. **在 `layouts` 对象中添加模板元数据**：
   ```json
   "my_company_style": {
     "label": "My Company Style Template",
     "summary": "Corporate template for My Company, suitable for annual reports and board presentations",
     "tone": "Professional, modern, brand-focused",
     "themeMode": "Light theme (white background + brand blue/gold accents)",
     "keywords": ["corporate", "mycompany", "annual report", "board"],
     "assets": ["logo.png"]
   }
   ```

#### 步骤 4：验证模板

```bash
python3 scripts/svg_quality_checker.py templates/layouts/my_company_style --format ppt169
```

如果出现错误，根据提示修复 SVG。

#### 步骤 5：使用新模板

之后任何项目在策略师阶段，AI 都会读取索引并推荐你的新模板。

---

### 四、为单个项目临时定制布局（不注册全局）

如果你只需要为当前项目修改某个模板的配色或 logo，而不想创建全局模板：

#### 方法 A：复制全局模板到项目并修改

```bash
# 复制 google_style 模板到项目
cp -r skills/ppt-master/templates/layouts/google_style projects/my_project/templates/

# 修改 projects/my_project/templates/design_spec.md 中的颜色值
# 修改 projects/my_project/templates/01_cover.svg 中的 logo 路径或颜色
```

然后，在策略师阶段选择“不使用模板”，或者让 AI 知道项目 `templates/` 目录下已有模板（AI 会自动检测）。但注意：AI 在模板选择阶段只会查询全局索引，不会自动扫描项目目录。你可以手动告诉 AI：“项目 templates/ 目录下有一套自定义模板，请使用它。”

#### 方法 B：直接在项目 `images/` 中放置图片，手动编辑 SVG

如果你只想替换封面背景图或添加 logo，可以在项目 `images/` 中放入图片，然后让执行师在生成 SVG 时引用它们（通过设计规范的图片资源清单）。这不需要模板。

---

### 五、定制图表（全局或项目级）

#### 全局图表模板（推荐）

1. **准备图表 SVG**（如 `my_bar_chart.svg`），放在 `templates/charts/` 下。
2. **在 `charts_index.json` 中注册**：
   ```json
   "my_bar_chart": {
     "label": "My Custom Bar Chart",
     "summary": "Bar chart with rounded corners and gradient fill",
     "bestFor": ["Comparing 3-6 categories", "Highlighting top performer"],
     "avoidFor": ["Time series", "More than 8 categories"],
     "keywords": ["bar chart", "rounded", "gradient"]
   }
   ```
3. **使用**：策略师在设计规范中引用 `my_bar_chart`，执行师查阅其结构后生成实际图表。

#### 项目级临时图表

如果图表只用于当前项目，可以直接让执行师根据设计规范自由生成，无需模板。或者将图表 SVG 放在项目的 `templates/charts/` 目录（需手动创建），并在 `design_spec.md` 中注明引用路径。但 AI 不会自动索引项目级图表，需要你明确指示。

---

### 六、定制图标（全局 vs 项目级）

#### 全局图标（推荐，可复用）

1. **准备图标 SVG**：必须是 16×16，`viewBox="0 0 16 16"`，纯路径（无填充或填充为 `currentColor`）。例如 `my_icon.svg`。
2. **放入 `templates/icons/`**。
3. **注册到 `icons_index.json`**：
   - 在对应 `categories` 下的 `icons` 数组中添加 `"my_icon"`。
   - 可选：在 `quickLookup` 中添加语义映射。
4. **使用**：执行师在 SVG 中写 `<use data-icon="my_icon" .../>`，后处理 `embed_icons.py` 会自动嵌入。

#### 项目级临时图标

如果图标只用于当前项目且不希望污染全局库：

1. 将图标 SVG 文件放在项目的 `images/` 目录（如 `projects/my_project/images/my_icon.svg`）。
2. 在生成 SVG 时，直接使用 `<image href="../images/my_icon.svg" x="..." y="..." width="..." height="..."/>` 引用（注意：这样不会应用后处理的图标嵌入逻辑，但可以正常工作）。或者，你也可以让执行师将图标路径嵌入为 `data:image/svg+xml;base64,...`，但这会增加复杂度。
3. 缺点：无法使用 `<use data-icon>` 的简洁语法，且不会自动缩放和着色。

---

### 七、总结：不同场景的操作一览表

| 目标 | 操作位置 | 是否需注册索引 | AI 自动发现 | 复杂度 |
|------|----------|----------------|--------------|--------|
| 为项目选择已有全局模板 | 对话中确认 | 已注册 | ✅ 是 | 低 |
| 创建新全局布局模板 | `templates/layouts/新目录` | 需要编辑 `layouts_index.json` | ✅ 是 | 中 |
| 为项目临时修改模板 | 复制到项目 `templates/` 并手动修改 | 否 | ❌ 否（需手动告知 AI） | 低 |
| 创建新全局图表模板 | `templates/charts/新文件` | 需要编辑 `charts_index.json` | ✅ 是 | 中 |
| 为项目临时设计图表 | 执行师自由生成 | 否 | ❌ 否 | 低 |
| 添加全局图标 | `templates/icons/新文件` | 需要编辑 `icons_index.json` | ✅ 是（后处理时） | 低 |
| 为项目临时使用图标 | 项目 `images/` 目录 | 否 | ❌ 否 | 低 |

---

### 八、实际操作示例：为“我的公司”创建一套完整定制

假设你要为“XX 科技”公司创建一套包含布局、图表样式和品牌图标的全局模板。

1. **创建布局模板**：
   - `templates/layouts/xxtech_style/design_spec.md`（定义品牌色 `#0055A4`，字体使用 `Inter`，页眉高度 80px）
   - 四个核心 SVG 文件（`01_cover.svg` 等），使用占位符，背景使用品牌渐变。
   - 添加 logo 图片 `xxtech_logo.png`，在 `01_cover.svg` 和 `03_content.svg` 中引用 `href="xxtech_logo.png"`。

2. **注册布局模板**：更新 `layouts_index.json`，添加 `xxtech_style` 条目，分类为 `brand`。

3. **创建图表模板**（可选）：
   - 如果你希望公司报告中常用“圆角柱状图”，创建 `templates/charts/rounded_bar_chart.svg`。
   - 注册到 `charts_index.json`，`bestFor` 写“公司内部 KPI 对比”。

4. **添加品牌图标**：
   - 将公司专属图标（如产品图标）放入 `templates/icons/`，如 `product_icon.svg`。
   - 注册到 `icons_index.json` 的 `business` 分类。

5. **使用模板**：后续任何“XX 科技”相关项目，AI 在模板选择时会推荐 `xxtech_style`，并自动复制到项目；执行师可在内容中使用新图标。

通过以上流程，你可以实现从全局到项目级的全链路素材定制。