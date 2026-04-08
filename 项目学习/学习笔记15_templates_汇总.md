## `templates/` 目录完整解析：架构、作用、扩展流程

`templates/` 目录是 PPT Master 的**设计资源库**，为 AI 角色（策略师、执行师）提供可复用的布局模板、图表模板和图标库。它使得生成的 SVG 能够遵循一致的设计语言，而无需从零设计每个元素。

下面从**架构**、**作用**、**扩展规范与流程**三个方面详细解析，并回答你的三个问题。

---

### 一、`templates/` 目录架构

```
templates/
├── design_spec_reference.md   # 设计规范参考模板（供策略师输出 design_spec.md 使用）
├── README.md                   # 模板库总体说明
├── layouts/                    # 页面布局模板（20+ 套）
│   ├── layouts_index.json      # 机器可读的布局索引（AI 优先使用）
│   ├── README.md               # 人类可读的布局说明
│   ├── google_style/           # 每个模板是一个子目录
│   │   ├── design_spec.md      # 该模板的设计规范
│   │   ├── 01_cover.svg        # 封面页 SVG 模板
│   │   ├── 02_toc.svg          # 目录页 SVG 模板（可选）
│   │   ├── 02_chapter.svg      # 章节页 SVG 模板
│   │   ├── 03_content.svg      # 内容页 SVG 模板
│   │   └── 04_ending.svg       # 结尾页 SVG 模板
│   └── ...（其他模板目录）
├── charts/                     # 图表模板库（33 种）
│   ├── charts_index.json       # 机器可读的图表索引
│   ├── README.md               # 图表说明
│   └── *.svg                   # 每个图表一个 SVG 文件（如 bar_chart.svg）
└── icons/                      # 图标库（640+ 个）
    ├── icons_index.json        # 机器可读的图标索引
    ├── README.md               # 图标使用说明
    ├── FULL_INDEX.md           # 完整图标列表（人类浏览）
    └── *.svg                   # 每个图标一个 SVG 文件（如 rocket.svg）
```

---

### 二、各子目录的作用与设计理念

#### 1. `layouts/` —— 页面布局模板

**作用**：提供**整套演示文稿的视觉框架**，包括封面、目录、章节页、内容页、结尾页的 SVG 布局。模板定义了背景、页眉页脚、配色、字体、装饰元素等，但**内容区保留灵活性**（AI 根据实际内容自由布局）。

**设计理念**：
- **固定结构 + 灵活内容**：模板只规定页眉、页脚、装饰，内容区由执行师自由发挥。
- **一致性**：同一模板下所有页面共享视觉主题（颜色、字体、间距）。
- **可组合**：模板是“起点而非终点”，策略师可根据需要调整颜色、布局比例。

**标准文件清单**（每个模板目录必须包含）：
| 文件 | 必需 | 说明 |
|------|------|------|
| `design_spec.md` | ✅ | 该模板的完整设计规范（颜色、字体、布局原则） |
| `01_cover.svg` | ✅ | 封面页模板 |
| `02_chapter.svg` | ✅ | 章节页模板（章节分隔） |
| `03_content.svg` | ✅ | 内容页模板（最常用） |
| `04_ending.svg` | ✅ | 结尾页模板 |
| `02_toc.svg` | ⭕ | 目录页模板（可选，若无则 AI 可自行设计目录） |

**索引文件 `layouts_index.json`**：
- 供 AI 程序化查询，包含 `meta`（总数、默认 viewBox）、`categories`（按品牌/通用/场景/政府/特殊分类）、`quickLookup`（按使用场景快速查找，如 `strategy`, `academic`）、`layouts`（每个模板的详细元数据：标签、摘要、调性、主题模式、关键词、关联资源）。

#### 2. `charts/` —— 图表模板库

**作用**：提供**33 种标准图表 SVG 模板**，供执行师在生成数据可视化页面时参考结构、样式和布局。**不是直接复制粘贴**，而是作为**样式和结构参考**，执行师需根据实际数据调整坐标、颜色、标签。

**设计理念**：
- **参考而非直接使用**：图表模板展示某种图表的典型布局（轴、图例、数据系列位置），但执行师必须根据设计规范的配色和数据范围重新生成。
- **分类清晰**：按用途分为比较、趋势、构成、指标、分析、项目管理/关系、战略框架等七大类。
- **机器可读索引**：`charts_index.json` 包含每个图表的 `bestFor`（适用场景）、`avoidFor`（避免场景）、`keywords`，帮助 AI 选择。

**使用流程**：
1. 策略师在内容大纲中注明某页需要“柱状图” → 在 `design_spec` 的 **VII. Chart Reference List** 中列出图表类型。
2. 执行师查阅 `charts/bar_chart.svg`，理解其基本结构（X/Y 轴、柱宽、间距等）。
3. 执行师根据实际数据和设计规范中的颜色，生成新的柱状图 SVG（不复制原文件）。

#### 3. `icons/` —— 图标库

**作用**：提供 **640+ 矢量图标**，供执行师在 SVG 中通过 `<use data-icon="...">` 占位符引用，后处理时自动嵌入实际路径。

**设计理念**：
- **占位符驱动**：生成时使用轻量级 `<use data-icon="name">`，后处理时替换为真实 SVG 路径（避免生成阶段嵌入大量代码）。
- **统一基准**：所有图标原始尺寸为 16×16，`viewBox="0 0 16 16"`，通过 `transform="scale(...)"` 缩放。
- **机器可读索引**：`icons_index.json` 按分类（导航、数据、用户、状态等）组织，并提供 `quickLookup`（按语义快速查找，如 `growth`, `success`, `error`）。

**使用方式**：
```xml
<use data-icon="rocket" x="100" y="200" width="48" height="48" fill="#0076A8"/>
```
后处理命令：`python3 scripts/embed_icons.py svg_output/*.svg`

---

### 三、扩展规范与流程（回答你的三个问题）

#### 问题 1：layout 是否可以自己添加新的专属 layout？是什么规范，什么流程？

**答：可以。** 你可以为特定客户、品牌或项目创建专属布局模板，并添加到 `templates/layouts/` 目录中。标准流程如下：

**步骤 1：创建模板目录**
```
templates/layouts/my_custom_template/
```

**步骤 2：准备必需文件**
- `design_spec.md`：必须遵循标准章节结构（参考 `design_spec_reference.md` 或现有模板如 `google_style/design_spec.md`）。至少包含：画布规范、配色方案、字体系统、页面结构、页面类型、布局模式、间距规范、SVG 技术约束、占位符规范。
- SVG 模板文件（至少 `01_cover.svg`, `02_chapter.svg`, `03_content.svg`, `04_ending.svg`）。推荐也提供 `02_toc.svg`。
- 可选：模板中引用的图片资源（如 logo）应放在模板目录下，但实际使用时需复制到项目的 `images/` 目录。

**步骤 3：遵循 SVG 技术规范**
- `viewBox="0 0 1280 720"`（PPT 16:9）。
- 禁止使用 `clipPath`, `mask`, `<style>`, `class`, `foreignObject`, `textPath`, `animate*`, `marker`, `rgba()`, `<g opacity>` 等（参考 `shared-standards.md`）。
- 使用占位符 `{{PLACEHOLDER}}` 标记可替换内容（如 `{{TITLE}}`, `{{PAGE_TITLE}}`, `{{CHAPTER_NUM}}` 等）。占位符列表参见 `layouts/README.md` 中的规范。

**步骤 4：注册到 `layouts_index.json`**
在 `layouts_index.json` 中添加条目：
- 在 `categories` 的适当分类下（如 `brand`, `general`, `scenario`, `government`, `special`）添加 `layouts` 数组中的模板名称。
- 在 `quickLookup` 中添加关键词映射（如果适用）。
- 在 `layouts` 对象中添加模板的元数据：
```json
"my_custom_template": {
  "label": "My Custom Template",
  "summary": "One-line description",
  "tone": "Design tone",
  "themeMode": "Light/Dark/Hybrid",
  "keywords": ["keyword1", "keyword2"],
  "assets": ["logo.png"]   // 如果有额外资产
}
```

**步骤 5：验证模板**
```bash
# 检查 SVG 合规性
python3 scripts/svg_quality_checker.py templates/layouts/my_custom_template --format ppt169
```

**步骤 6：使用模板**
策略师在 Step 3（模板选择）时，AI 会读取 `layouts_index.json`，用户可选择你的自定义模板。

> **注意**：模板一旦注册，AI 在生成 PPT 时可能会推荐它。确保模板设计质量高且通用。

#### 问题 2：`charts/` 是直接添加样式么？

**答：不是直接“添加样式”，而是添加新的图表 SVG 模板文件并注册到索引。**

图表模板是**结构参考**，不是直接可复用的样式。扩展流程：

1. **创建新图表 SVG 文件**（如 `my_chart.svg`），放在 `templates/charts/` 下。
2. **设计图表**：使用 PPT 兼容的 SVG 语法（无禁用特性），尺寸 `1280×720`，展示典型数据（如示例柱状图、折线图）。图表应包含轴、标签、图例等元素，但数据点用占位符（如 `{{DATA}}`）或示例数据。
3. **编写元数据**：在 `charts_index.json` 中注册：
   - 在 `categories` 下选择合适的分类（如 `comparison`, `trend` 等），将图表名加入 `charts` 数组。
   - 在 `quickLookup` 中添加关键词映射（如 `ranking` → `["my_chart"]`）。
   - 在 `charts` 对象中添加条目：
```json
"my_chart": {
  "label": "My Chart Name",
  "summary": "What it shows",
  "bestFor": ["Use case 1", "Use case 2"],
  "avoidFor": ["Not suitable for"],
  "keywords": ["keyword1", "keyword2"]
}
```
4. **使用**：策略师在设计规范中引用该图表类型，执行师查看 `templates/charts/my_chart.svg` 的结构，然后根据实际数据生成新图表。

#### 问题 3：同样 icon 也是这个？整个 templates/ 的架构、作用、流程、后续每一项的扩展等等。

**答：是的，图标库的扩展流程类似，但更简单。**

**扩展流程**：
1. **准备图标 SVG**：必须是 16×16 的纯路径图标，`viewBox="0 0 16 16"`，无填充（或填充为 `currentColor`，但实际使用时会由外层 `fill` 覆盖）。建议从 [SVG Repo](https://www.svgrepo.com/) 下载，确保开源许可。
2. **放置文件**：将 `my_icon.svg` 放入 `templates/icons/` 目录。
3. **注册到 `icons_index.json`**：
   - 在适当的 `categories` 下（如 `navigation`, `data`, `status` 等）将图标名加入 `icons` 数组。
   - 可选：在 `quickLookup` 中添加语义映射（如 `"rocket": ["my_icon"]` 或合并到现有关键词）。
4. **使用**：执行师在 SVG 中使用 `<use data-icon="my_icon" .../>`。

**注意事项**：
- 图标名称必须与文件名一致（不含 `.svg`）。
- 图标库不依赖后处理以外的脚本，只需确保 `embed_icons.py` 能找到该文件。
- 现有 640+ 图标已覆盖大部分场景，通常无需新增。

---

### 四、`templates/` 在整个工作流中的角色

```
策略师阶段
    │
    ├── 读取 layouts_index.json → 向用户推荐模板
    │
    ├── 用户选择模板后，将模板文件复制到项目 templates/ 目录
    │
    ├── 策略师输出 design_spec.md（可能引用 charts/ 中的图表类型）
    │
    ▼
执行师阶段
    │
    ├── 从项目 templates/ 读取布局模板（如 01_cover.svg） → 继承背景、页眉页脚，替换占位符
    │
    ├── 根据 design_spec 中的图表类型，查阅 charts/ 对应 SVG 了解结构 → 生成新图表
    │
    ├── 在内容中使用 <use data-icon="..."> 引用图标
    │
    ▼
后处理阶段
    │
    └── embed_icons.py 从 templates/icons/ 读取图标文件，替换占位符为实际路径
```

---

### 五、工程智慧总结

| 资源类型 | 扩展难度 | 是否需要修改代码 | 主要维护文件 |
|----------|----------|------------------|--------------|
| Layout 模板 | 中等 | 否（仅添加目录和 JSON 条目） | `layouts_index.json`, 模板目录 |
| Chart 模板 | 中等 | 否 | `charts_index.json`, 图表 SVG |
| Icon | 低 | 否 | `icons_index.json`, 图标 SVG |

**核心设计原则**：
- **声明式索引**：所有资源通过 JSON 索引供 AI 查询，避免 AI 直接遍历文件系统。
- **占位符驱动**：图标和模板内容使用占位符，后处理时具体化，降低生成阶段的复杂度。
- **分层抽象**：布局模板定义整体框架，图表模板定义数据呈现模式，图标提供细粒度视觉元素。
- **向后兼容**：添加新资源不会破坏现有流程，AI 会根据索引自动发现。

通过遵循上述规范，你可以轻松为 PPT Master 扩展自定义模板、图表和图标，而无需修改任何核心脚本。