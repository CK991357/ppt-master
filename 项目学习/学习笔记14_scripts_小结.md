## scripts/ 目录完整总结（编码专业角度）

`scripts/` 目录是 PPT Master 的**可执行实现层**，包含了从源文件转换到最终 PPTX 导出的所有工具脚本。它与 `references/`（规范层）和 `templates/`（资源层）共同构成三驾马车。以下从架构、设计模式、工程智慧等方面进行系统总结。

---

### 一、总体定位与分层

| 层次 | 目录/模块 | 职责 |
|------|-----------|------|
| **用户入口层** | `project_manager.py`, `image_gen.py`, `finalize_svg.py`, `svg_to_pptx.py` | 提供面向 AI 或用户的高级命令，封装复杂流程 |
| **核心转换层** | `pdf_to_md.py`, `doc_to_md.py`, `web_to_md.py` / `.cjs` | 将异构输入统一转换为 Markdown |
| **项目管理层** | `project_manager.py`, `project_utils.py`, `batch_validate.py`, `generate_examples_index.py` | 项目生命周期管理、结构验证、批量检查 |
| **图片处理层** | `analyze_images.py`, `image_gen.py`, `rotate_images.py`, `gemini_watermark_remover.py`, `image_backends/` | 图片分析、生成、方向修正、水印去除、多后端调度 |
| **SVG 后处理层** | `finalize_svg.py`, `svg_finalize/` 子模块（6 个） | 图标嵌入、图片裁剪/嵌入、文本扁平化、圆角转路径、宽高比修正 |
| **PPTX 导出层** | `svg_to_pptx.py`, `svg_to_pptx/` 子模块（14 个） | SVG 到 DrawingML 转换、PPTX 组装、兼容模式、备注嵌入 |
| **辅助工具层** | `svg_quality_checker.py`, `svg_position_calculator.py`, `pptx_animations.py`, `pptx_template_import.py`, `update_repo.py`, `error_helper.py` | 质量检查、坐标计算、动画生成、模板导入、仓库更新、错误帮助 |
| **文档层** | `docs/` | 各脚本的使用说明和最佳实践 |

---

### 二、核心脚本清单与职责速查

| 脚本 | 核心职责 | 输入 | 输出 |
|------|----------|------|------|
| `pdf_to_md.py` | PDF → Markdown（启发式标题/列表/表格） | PDF 文件 | `.md` + 媒体目录 |
| `doc_to_md.py` | Pandoc 包装，支持 10+ 文档格式 | DOCX, EPUB, HTML, LaTeX 等 | `.md` + 媒体目录 |
| `web_to_md.py` / `.cjs` | 网页抓取、正文提取、图片下载 | URL | `.md` + `_files/` |
| `project_manager.py` | 项目初始化、源文件导入、验证、信息查询 | 项目名 / 源文件 | 项目目录结构 |
| `project_utils.py` | 项目名解析、结构验证、viewBox 校验 | 项目路径 | 信息字典 / 验证结果 |
| `config.py` | 统一配置（画布、颜色、字体、SVG 约束） | 无 | 配置常量 |
| `error_helper.py` | 错误类型到修复建议的映射 | 错误类型 | 友好错误消息 |
| `analyze_images.py` | 图片尺寸、宽高比分析，生成布局建议 | `images/` 目录 | 报告 + CSV + Markdown 片段 |
| `image_gen.py` | 统一图片生成入口，支持 11 种后端 | 提示词 + 参数 | 图片文件 |
| `rotate_images.py` | EXIF 方向修正 + 可视化旋转工具 | 图片目录 / JSON 修复指令 | 修正后的图片 / HTML 工具 |
| `gemini_watermark_remover.py` | 去除 Gemini 水印（逆向混合算法） | 图片文件 | 去水印图片 |
| `total_md_split.py` | 分割演讲备注为每页独立文件 | `notes/total.md` | `notes/*.md` |
| `finalize_svg.py` | SVG 后处理统一入口 | `svg_output/` | `svg_final/` |
| `svg_finalize/` 子模块 | 图标嵌入、图片裁剪/嵌入、文本扁平化、圆角转路径、宽高比修正 | SVG 文件 | 修改后的 SVG |
| `svg_to_pptx.py` | SVG → PPTX 薄包装 | `svg_final/` | `.pptx` 文件 |
| `svg_to_pptx/` 子模块 | DrawingML 转换、XML 生成、PPTX 组装 | SVG + 备注 | 幻灯片 XML / 媒体 / 关系 |
| `svg_quality_checker.py` | 检查 SVG 技术规范（禁止元素、viewBox 等） | SVG 文件或目录 | 检查报告 |
| `svg_position_calculator.py` | 图表坐标预计算与验证 | 数据 / JSON | 坐标表格 / 验证结果 |
| `pptx_animations.py` | 生成过渡效果和进入动画 XML | 效果名、时长 | XML 片段 |
| `pptx_template_import.py` | 从 PPTX 提取模板资产（背景、颜色、字体） | `.pptx` 文件 | `manifest.json`, `analysis.md`, `assets/` |
| `batch_validate.py` | 批量验证多个项目结构 | 目录列表 | 控制台报告 + 导出文件 |
| `generate_examples_index.py` | 自动生成 `examples/README.md` | `examples/` 目录 | `README.md` |
| `update_repo.py` | 拉取最新代码，按需同步依赖 | 无 | 更新后的仓库 |

---

### 三、核心设计模式与工程智慧

#### 1. **统一入口 + 薄包装模式**
- `finalize_svg.py` 聚合 6 个子模块，提供单一命令；`svg_to_pptx.py` 是薄包装，实际逻辑在 `svg_to_pptx/` 包中。
- **好处**：降低用户学习成本，保持向后兼容，便于重构。

#### 2. **策略模式（图片后端）**
- `image_gen.py` 通过 `IMAGE_BACKEND` 环境变量选择后端，每个后端独立实现 `generate()` 函数。
- `BACKEND_REGISTRY` 定义了核心/扩展/实验三级，支持别名映射。
- **好处**：新增后端只需注册并实现接口，不修改主入口。

#### 3. **流水线串行执行**
- `total_md_split.py` → `finalize_svg.py` → `svg_to_pptx.py` 必须按顺序执行，禁止批量化。
- 每个步骤有明确的输入输出契约，中间产物可人工检查。
- **好处**：错误隔离，易于调试，符合 UNIX 哲学。

#### 4. **配置集中化与容错**
- `config.py` 集中管理画布、颜色、字体、SVG 约束等常量。
- 其他脚本通过 `try/except ImportError` 提供后备配置，避免因模块缺失而崩溃。
- **好处**：单点维护，健壮性强。

#### 5. **错误处理与用户友好**
- `error_helper.py` 将技术错误（如“缺少 README.md”）映射为可操作的修复步骤。
- 验证工具（`project_manager.py validate`, `batch_validate.py`, `svg_quality_checker.py`）输出结构化问题列表。
- **好处**：降低用户挫败感，提升可维护性。

#### 6. **跨平台兼容**
- `rotate_images.py` 使用 `sys.stdout.reconfigure(encoding='utf-8')` 解决 Windows 中文乱码。
- 路径处理使用 `pathlib` 和 `os.path` 混合，支持 Windows 反斜杠。
- 字体映射表（`drawingml_utils.FONT_FALLBACK_WIN`）将 macOS/Linux 字体映射到 Windows 等效字体。
- **好处**：项目可在 Windows/macOS/Linux 上无缝运行。

#### 7. **渐进式智能（PDF 转换）**
- `pdf_to_md.py` 通过字体大小统计、页眉页脚去噪、列表检测、代码字体识别等启发式规则，从无结构的 PDF 中提取结构化 Markdown。
- **好处**：无需外部 OCR，快速且隐私安全。

#### 8. **占位符驱动的模板系统**
- `embed_icons.py` 使用 `<use data-icon="...">` 占位符，后处理时替换为实际路径。
- 设计规范中的图片资源清单也使用占位符，由策略师填写。
- **好处**：生成阶段保持简洁，后处理阶段完成具体化。

#### 9. **两阶段处理（旋转图片）**
- `rotate_images.py` 先 `auto` 自动修正 EXIF，再 `gen` 生成 HTML 交互工具让用户手动调整，最后 `fix` 应用修复。
- **好处**：自动化处理常见问题，人工介入处理边缘情况。

#### 10. **模块化转换器（DrawingML）**
- `svg_to_pptx/drawingml_elements.py` 为每种 SVG 元素提供独立转换器，`drawingml_converter.py` 负责调度。
- 样式、路径、上下文等拆分为独立模块。
- **好处**：易于测试、扩展和维护。

---

### 四、数据流与依赖关系

```
源文件 (PDF/DOCX/URL)
    │
    ├── pdf_to_md.py / doc_to_md.py / web_to_md.py
    ▼
Markdown (sources/*.md)
    │
    ├── project_manager.py import-sources (移动 + 转换)
    ▼
项目目录 (sources/, images/, svg_output/ 待生成)
    │
    ├── 策略师 + 执行师 (AI) → 生成 SVG 到 svg_output/
    ▼
svg_output/*.svg
    │
    ├── total_md_split.py → notes/*.md
    ├── finalize_svg.py → svg_final/*.svg
    │       ├── embed_icons.py
    │       ├── crop_images.py
    │       ├── fix_image_aspect.py
    │       ├── embed_images.py
    │       ├── flatten_tspan.py
    │       └── svg_rect_to_path.py
    ▼
svg_final/*.svg + notes/*.md
    │
    └── svg_to_pptx.py → .pptx (原生形状 + SVG 参考)
            ├── drawingml_converter.py
            ├── pptx_builder.py
            └── pptx_* 模块
```

**关键依赖**：
- `project_utils.py` 被 `project_manager.py`, `batch_validate.py`, `generate_examples_index.py` 等共享。
- `config.py` 被 `project_utils.py` 和 `pptx_dimensions.py` 引用。
- `error_helper.py` 被 `project_utils.py` 和 `svg_quality_checker.py` 可选使用。

---

### 五、可复用模式（供其他 AI 项目借鉴）

1. **环境变量驱动的多后端架构**：`image_gen.py` 通过 `IMAGE_BACKEND` + 后端特定 key 实现灵活切换，避免全局冲突。
2. **占位符 + 后处理替换**：在生成阶段使用轻量级占位符，后处理阶段完成具体化，降低生成时的复杂度。
3. **启发式结构化提取**：`pdf_to_md.py` 展示如何从非结构化文档（PDF）中提取标题、列表、表格。
4. **批量验证 + 友好错误**：`batch_validate.py` 和 `error_helper.py` 提供可操作的反馈，适合 CI 集成。
5. **模板导入简化**：`pptx_template_import.py` 只提取关键资产和元数据，而不是完整逆向工程，降低实现难度。
6. **坐标预计算工具**：`svg_position_calculator.py` 为图表生成提供精确坐标，避免 AI 估算错误。

---

### 六、潜在改进点（个人观察）

1. **脚本入口不一致**：有些脚本使用 `argparse`，有些直接 `sys.argv`，建议统一为 `argparse`。
2. **日志系统缺失**：当前使用 `print` 输出，可引入 `logging` 模块支持级别控制。
3. **单元测试覆盖**：核心转换逻辑（如 `pdf_to_md.py` 的启发式规则）缺少自动化测试。
4. **配置热加载**：`config.py` 修改后需要重启脚本，可考虑支持动态重载。
5. **异步支持**：`image_gen.py` 目前同步生成多张图片，可支持并发以提升效率。
6. **`svg_to_pptx` 包依赖 `pptx_animations` 可选导入**：若动画模块缺失，静默降级，但缺少明确警告。

---

### 七、总结

`scripts/` 目录是 PPT Master 的**工程化核心**，体现了以下设计哲学：

- **管道化**：每个脚本做好一件事，通过串行组合完成复杂任务。
- **容错性**：多处后备逻辑（配置、字体映射、PIL 缺失），确保基本功能可用。
- **用户友好**：详细的进度输出、统计信息、错误修复建议。
- **可扩展性**：多后端、多格式支持、插件式子模块。
- **跨平台**：路径、编码、字体映射均考虑了 Windows/macOS/Linux 差异。

与 `references/`（规范层）和 `templates/`（资源层）形成清晰的分层，共同支撑了 PPT Master 的自动化演示文稿生成能力。