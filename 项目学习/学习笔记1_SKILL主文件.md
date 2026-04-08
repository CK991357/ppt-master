## SKILL.md 完整逐行解析

### 文件头 (Frontmatter)

```yaml
---
name: ppt-master
description: >-
  AI-driven multi-format SVG content generation system. Converts source documents
  (PDF/DOCX/URL/Markdown) into high-quality SVG pages and exports to PPTX through
  multi-role collaboration. Use when user asks to "create PPT", "make presentation",
  "生成PPT", "做PPT", "制作演示文稿", or mentions "ppt-master".
---
```

- `name`：技能的唯一标识符，供 AI 系统（如 Claude Code、Cursor）识别此技能。
- `description`：简短描述，告诉 AI 何时应激活此技能。触发关键词包括：`create PPT`、`make presentation`、`生成PPT`、`做PPT`、`制作演示文稿`，或提及 `ppt-master`。多语言支持（中英文）。

---

### 核心流水线概述

> **Core Pipeline**: `Source Document → Create Project → Template Option → Strategist → [Image_Generator] → Executor → Post-processing → Export`

- 整个系统的 8 个阶段，箭头表示串行顺序。
- 方括号 `[Image_Generator]` 表示该步骤是条件性的（仅在需要 AI 生成图片时执行）。

---

### 全局执行纪律（强制）

这是一个警告块（`> [!CAUTION]`），包含 7 条最高优先级规则。违反任何一条都算执行失败。

#### 1. 串行执行
> Steps MUST be executed in order; the output of each step is the input for the next. Non-BLOCKING adjacent steps may proceed continuously once prerequisites are met, without waiting for the user to say "continue"

- **严格串行**：每一步的输出是下一步的输入。
- **非阻塞步骤可连续执行**：如果两个相邻步骤都不是阻塞点（BLOCKING），一旦前置条件满足，可以自动继续，无需用户说“继续”。

#### 2. 阻塞 = 硬停止
> Steps marked ⛔ BLOCKING require a full stop; the AI MUST wait for an explicit user response before proceeding and MUST NOT make any decisions on behalf of the user

- 标记为 `⛔ BLOCKING` 的步骤必须**硬停止**，AI 必须等待用户明确回复，不能替用户做决定。

#### 3. 禁止跨阶段捆绑
> Cross-phase bundling is FORBIDDEN. (Note: the Eight Confirmations in Step 4 are ⛔ BLOCKING — the AI MUST present recommendations and wait for explicit user confirmation before proceeding. Once the user confirms, all subsequent non-BLOCKING steps — design spec output, SVG generation, speaker notes, and post-processing — may proceed automatically without further user confirmation)

- **禁止跨阶段捆绑**：不能把多个步骤合并执行。
- **特殊说明**：Step 4 的“八项确认”是阻塞点，AI 必须给出建议并等待用户确认。一旦确认后，后续的非阻塞步骤（设计规范输出、SVG 生成、演讲备注、后处理）可以自动进行，无需再次确认。

#### 4. 进入前门禁
> Each Step has prerequisites (🚧 GATE) listed at the top; these MUST be verified before starting that Step

- 每个步骤开头都有 **🚧 GATE**（门禁），开始该步骤前必须验证这些前置条件。

#### 5. 禁止投机执行
> "Pre-preparing" content for subsequent Steps is FORBIDDEN (e.g., writing SVG code during the Strategist phase)

- **禁止投机执行**：不能提前准备后续步骤的内容。例如，在策略师阶段不能写 SVG 代码。

#### 6. 禁止子代理生成 SVG
> Executor Step 6 SVG generation is context-dependent and MUST be completed by the current main agent end-to-end. Delegating page SVG generation to sub-agents is FORBIDDEN

- **禁止子代理生成 SVG**：Step 6 的 SVG 生成必须由当前主代理（AI）从头到尾完成，不能委托给子代理。这是因为 SVG 设计依赖于完整的上下文（源内容、设计规范、模板映射、图片决策、跨页一致性）。

#### 7. 仅限顺序页面生成
> In Executor Step 6, after the global design context is confirmed, SVG pages MUST be generated sequentially page by page in one continuous pass. Grouped page batches (for example, 5 pages at a time) are FORBIDDEN

- **仅限顺序页面生成**：在 Step 6 中，确认全局设计参数后，必须一页一页顺序生成 SVG，不能分组批量生成（如一次 5 页）。

---

### 语言与沟通规则

```markdown
> [!IMPORTANT]
> ## 🌐 Language & Communication Rule
> 
> - **Response language**: Always match the language of the user's input and provided source materials.
> - **Explicit override**: If the user explicitly requests a specific language, use that language instead.
> - **Template format**: The `design_spec.md` file MUST always follow its original English template structure...
```

- **响应语言**：与用户输入和源材料语言一致。
- **显式覆盖**：如果用户要求特定语言，则使用该语言。
- **模板格式**：`design_spec.md` 必须保持英文模板结构（章节标题、字段名），但内容值可以用用户语言。

---

### 兼容性说明

```markdown
> [!IMPORTANT]
> ## 🔌 Compatibility With Generic Coding Skills
> 
> - `ppt-master` is a repository-specific workflow skill, not a general application scaffold
> - Do NOT create or require `.worktrees/`, `tests/`, branch workflows, or other generic engineering structure by default
> - If another generic coding skill suggests repository conventions that conflict with this workflow, follow this skill first unless the user explicitly asks otherwise
```

- 说明这是一个**仓库特定的工作流技能**，不是通用应用脚手架。
- 不要默认创建 `.worktrees/`、`tests/`、分支工作流等通用工程结构。
- 如果其他通用编码技能与此工作流冲突，优先遵循此技能，除非用户明确要求。

---

### 主流水线脚本表

| 脚本 | 用途 |
|------|------|
| `pdf_to_md.py` | PDF 转 Markdown |
| `doc_to_md.py` | 文档转 Markdown（通过 Pandoc 支持 DOCX, EPUB, HTML, LaTeX, RST 等） |
| `web_to_md.py` | 普通网页转 Markdown |
| `web_to_md.cjs` | 微信公众号/高安全性站点转 Markdown |
| `project_manager.py` | 项目初始化、验证、管理 |
| `analyze_images.py` | 图像分析 |
| `image_gen.py` | AI 图像生成（多提供商） |
| `svg_quality_checker.py` | SVG 质量检查 |
| `total_md_split.py` | 演讲备注分割 |
| `finalize_svg.py` | SVG 后处理（统一入口） |
| `svg_to_pptx.py` | 导出 PPTX |

- 变量 `${SKILL_DIR}` 表示技能目录的根路径（即 `skills/ppt-master/`）。

---

### 模板索引

| 索引 | 路径 | 用途 |
|------|------|------|
| 布局模板 | `templates/layouts/layouts_index.json` | 查询可用的页面布局模板 |
| 图表模板 | `templates/charts/charts_index.json` | 查询可用的图表 SVG 模板 |
| 图标库 | `templates/icons/icons_index.json` | 查询可用的图标名称和分类 |

---

### 独立工作流

| 工作流 | 路径 | 用途 |
|--------|------|------|
| `create-template` | `workflows/create-template.md` | 独立的模板创建工作流 |

---

## 详细工作流（Step 1 - Step 7）

### Step 1: 源内容处理

**🚧 GATE**: 用户已提供源材料（PDF / DOCX / EPUB / URL / Markdown 文件 / 文本描述 / 对话内容 — 任何形式均可）。

**转换规则**：

| 用户提供 | 命令 |
|----------|------|
| PDF 文件 | `python3 ${SKILL_DIR}/scripts/pdf_to_md.py <file>` |
| DOCX / Word / Office 文档 | `python3 ${SKILL_DIR}/scripts/doc_to_md.py <file>` |
| EPUB / HTML / LaTeX / RST / 其他 | `python3 ${SKILL_DIR}/scripts/doc_to_md.py <file>` |
| 网页链接 | `python3 ${SKILL_DIR}/scripts/web_to_md.py <URL>` |
| 微信公众号/高安全性站点 | `node ${SKILL_DIR}/scripts/web_to_md.cjs <URL>` |
| Markdown | 直接读取 |

**✅ Checkpoint**: 确认源内容准备就绪，进入 Step 2。

---

### Step 2: 项目初始化

**🚧 GATE**: Step 1 完成；源内容已准备。

**初始化命令**：
```bash
python3 ${SKILL_DIR}/scripts/project_manager.py init <project_name> --format <format>
```
- `<format>` 可选：`ppt169`（默认）、`ppt43`、`xhs`、`story` 等。完整列表见 `references/canvas-formats.md`。

**导入源内容**：
- 如果有源文件（PDF/MD 等）：
  ```bash
  python3 ${SKILL_DIR}/scripts/project_manager.py import-sources <project_path> <source_files...> --move
  ```
- 如果用户直接在对话中提供了文本：无需导入，内容已在对话上下文中。

> ⚠️ **必须使用 `--move`**：所有源文件必须**移动**（不是复制）到 `sources/` 中归档。中间产物（如 `_files/` 目录）由 `import-sources` 自动处理。执行后，源文件不再存在于原位置。

**✅ Checkpoint**: 确认项目结构创建成功，`sources/` 包含所有源文件，转换材料准备就绪。进入 Step 3。

---

### Step 3: 模板选择

**🚧 GATE**: Step 2 完成；项目目录结构已准备。

**⛔ BLOCKING**: 如果用户尚未明确表达是否使用模板，AI 必须提供选项并**等待用户明确回复**。如果用户之前已说过“不使用模板”或指定了某个模板，则跳过此提示直接进入 Step 4。

**⚡ Early-exit**: 如果用户在任何时候已说过“no template”/“不使用模板”/“自由设计”，则**不要查询** `layouts_index.json`，直接跳到 Step 4。

**模板推荐流程**（仅当用户尚未决定时）：
1. 查询 `${SKILL_DIR}/templates/layouts/layouts_index.json` 列出可用模板及其风格描述。
2. **提供专业推荐**：基于当前 PPT 主题和内容，推荐一个具体模板或自由设计，并给出理由。
3. 询问用户：
   > 💡 **AI Recommendation**: Based on your content topic (brief summary), I recommend **[specific template / free design]** because...
   >
   > Which approach would you prefer?
   > **A) Use an existing template** (please specify template name or style preference)
   > **B) No template** — free design

**用户确认选项 A 后**，复制模板文件到项目目录：
```bash
cp ${SKILL_DIR}/templates/layouts/<template_name>/*.svg <project_path>/templates/
cp ${SKILL_DIR}/templates/layouts/<template_name>/design_spec.md <project_path>/templates/
cp ${SKILL_DIR}/templates/layouts/<template_name>/*.png <project_path>/images/ 2>/dev/null || true
cp ${SKILL_DIR}/templates/layouts/<template_name>/*.jpg <project_path>/images/ 2>/dev/null || true
```

**用户确认选项 B 后**，直接进入 Step 4。

**✅ Checkpoint**: 用户已回复模板选择，模板文件已复制（如果是选项 A）。进入 Step 4。

---

### Step 4: 策略师阶段（强制 — 不可跳过）

**🚧 GATE**: Step 3 完成；用户已确认模板选择。

**首先读取角色定义**：
```
Read references/strategist.md
```

**必须完成八项确认**（参考 `templates/design_spec_reference.md` 的模板结构）：

⛔ **BLOCKING**: 八项确认必须作为一个打包的建议集呈现给用户，AI 必须**等待用户确认或修改**后才能输出设计规范和内容大纲。这是工作流中仅有的两个核心确认点之一（另一个是模板选择）。一旦确认，后续所有脚本执行和幻灯片生成都应全自动进行。

**八项确认内容**：
1. Canvas format（画布格式）
2. Page count range（页数范围）
3. Target audience（目标受众）
4. Style objective（风格目标）
5. Color scheme（配色方案）
6. Icon usage approach（图标使用方式）
7. Typography plan（字体计划）
8. Image usage approach（图片使用方式）

**如果用户提供了图片**，在输出设计规范前运行分析脚本（不要直接读取/打开图片文件，只使用脚本输出）：
```bash
python3 ${SKILL_DIR}/scripts/analyze_images.py <project_path>/images
```

> ⚠️ **图片处理规则**：AI 绝对不能直接读取、打开或查看图片文件（`.jpg`、`.png` 等）。所有图片信息必须来自 `analyze_images.py` 脚本输出或设计规范的图片资源列表。

**输出**：`<project_path>/design_spec.md`

**✅ Checkpoint**: 阶段交付物完成，自动进入下一步。输出格式：
```markdown
## ✅ Strategist Phase Complete
- [x] Eight Confirmations completed (user confirmed)
- [x] Design Specification & Content Outline generated
- [ ] **Next**: Auto-proceed to [Image_Generator / Executor] phase
```

---

### Step 5: 图片生成阶段（条件性）

**🚧 GATE**: Step 4 完成；设计规范和内容大纲已生成且用户确认。

> **触发条件**：图片使用方式包含“AI generation”。如果不触发，直接跳到 Step 6（Step 6 的门禁仍必须满足）。

**读取角色定义**：
```
Read references/image-generator.md
```

**步骤**：
1. 从设计规范中提取所有状态为“pending generation”的图片
2. 生成提示词文档 → `<project_path>/images/image_prompts.md`
3. 生成图片（推荐使用 CLI 工具）：
   ```bash
   python3 ${SKILL_DIR}/scripts/image_gen.py "prompt" --aspect_ratio 16:9 --image_size 1K -o <project_path>/images
   ```

**✅ Checkpoint**: 确认所有图片准备就绪，进入 Step 6。输出格式：
```markdown
## ✅ Image_Generator Phase Complete
- [x] Prompt document created
- [x] All images saved to images/
```

---

### Step 6: 执行师阶段

**🚧 GATE**: Step 4（以及 Step 5 如果触发了）完成；所有前置交付物准备就绪。

**根据所选风格读取角色定义**：
```
Read references/executor-base.md          # REQUIRED: common guidelines
Read references/executor-general.md       # General flexible style
Read references/executor-consultant.md    # Consulting style
Read references/executor-consultant-top.md # Top consulting style (MBB level)
```
> 只需要读 executor-base + 一个风格文件。

**设计参数确认（强制）**：在生成第一个 SVG 之前，执行者必须审查并输出设计规范中的关键设计参数（画布尺寸、配色方案、字体计划、正文字号），以确保符合规范。详见 executor-base.md 第 2 节。

> ⚠️ **主代理专用规则**：Step 6 的 SVG 生成必须由当前主代理完成，因为页面设计依赖于完整的上游上下文（源内容、设计规范、模板映射、图片决策、跨页一致性）。不要将任何幻灯片 SVG 生成委托给子代理。
> ⚠️ **生成节奏规则**：确认全局设计参数后，执行者必须一页一页顺序生成，在同一连续主代理上下文中进行。不要将 Step 6 分成分组批量（如每批 5 页）。

**视觉构建阶段**：
- 顺序生成 SVG 页面，一页一页，一次连续完成 → `<project_path>/svg_output/`

**逻辑构建阶段**：
- 生成演讲备注 → `<project_path>/notes/total.md`

**✅ Checkpoint**: 确认所有 SVG 和备注已完全生成。直接进入 Step 7 后处理。输出格式：
```markdown
## ✅ Executor Phase Complete
- [x] All SVGs generated to svg_output/
- [x] Speaker notes generated at notes/total.md
```

---

### Step 7: 后处理与导出

**🚧 GATE**: Step 6 完成；所有 SVG 生成到 `svg_output/`；演讲备注 `notes/total.md` 生成。

> ⚠️ 以下三个子步骤必须**逐个单独执行**。每个命令必须完成并确认成功后才能运行下一个。
> ❌ **永远不要**将三个命令放在同一个代码块或同一个 shell 调用中。

**Step 7.1** — 分割演讲备注：
```bash
python3 ${SKILL_DIR}/scripts/total_md_split.py <project_path>
```

**Step 7.2** — SVG 后处理（图标嵌入/图片裁剪嵌入/文本扁平化/圆角矩形转路径）：
```bash
python3 ${SKILL_DIR}/scripts/finalize_svg.py <project_path>
```

**Step 7.3** — 导出 PPTX（默认嵌入演讲备注）：
```bash
python3 ${SKILL_DIR}/scripts/svg_to_pptx.py <project_path> -s final
# 默认生成两个文件：原生形状 (.pptx) + SVG 参考版 (_svg.pptx)
# 使用 --only native 跳过 SVG 参考版
# 使用 --only legacy 仅生成 SVG 图片版
```

> ❌ **永远不要**用 `cp` 代替 `finalize_svg.py` — 它执行多个关键处理步骤
> ❌ **永远不要**直接从 `svg_output/` 导出 — 必须使用 `-s final` 从 `svg_final/` 导出
> ❌ **永远不要**添加额外标志如 `--only`

---

### 角色切换协议

在切换角色前，AI 必须**首先读取**对应的参考文件 — 跳过是禁止的。输出标记：

```markdown
## [Role Switch: <Role Name>]
📖 Reading role definition: references/<filename>.md
📋 Current task: <brief description>
```

---

### 参考资源表

| 资源 | 路径 |
|------|------|
| 共享技术约束 | `references/shared-standards.md` |
| 画布格式规范 | `references/canvas-formats.md` |
| 图片布局规范 | `references/image-layout-spec.md` |
| SVG 图片嵌入 | `references/svg-image-embedding.md` |

---

### 备注

- 不要给后处理命令添加额外标志如 `--only` — 按原样运行
- 本地预览：`python3 -m http.server -d <project_path>/svg_final 8000`

---

## 你的理解确认

请确认：
1. 你是否理解了每个规则（特别是阻塞点、门禁、串行要求）？
2. 是否有任何术语或概念需要进一步解释？
3. 准备好后，我们可以继续分析下一个文件（例如 `references/strategist.md` 或 `references/executor-base.md`）。