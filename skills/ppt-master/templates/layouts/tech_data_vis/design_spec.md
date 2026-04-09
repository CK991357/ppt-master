# Tech Data Visualization Template - Design Specification

> 适用于技术文章配套 PPT、数据报告、产品发布、技术分享等场景，与公众号可视化风格完全一致。

---

## I. Template Overview

| Property          | Description                                           |
| ----------------- | ----------------------------------------------------- |
| **Template Name** | tech_data_vis (Tech Data Visualization Style)         |
| **Use Cases**     | Technical articles, data reports, product launches, tech sharing |
| **Design Tone**   | Professional, clear, data-driven, structured, restrained |
| **Theme Mode**    | Light theme (white/light gray background)             |

---

## II. Canvas Specification

| Property        | Value                         |
| --------------- | ----------------------------- |
| **Format**      | Standard 16:9                 |
| **Dimensions**  | 1280 × 720 px                |
| **viewBox**     | `0 0 1280 720`                |
| **Page Margins**| Left/Right 60px, Top 50px, Bottom 50px |
| **Safe Area**   | x: 60-1220, y: 50-670        |

---

## III. Color Scheme

### Core Colors (for distinguishing categories/entities)

| Role               | HEX       | Usage                                                       |
| ------------------ | --------- | ----------------------------------------------------------- |
| **Tech Deep Blue** | `#1F4E7A` | AWS, cloud services, specialized integration, reliable tech |
| **Eco Forest Green**| `#2E7D32` | NVIDIA, mature ecosystems, industry standards, stable growth |
| **Breakthrough Purple** | `#6A1B9A` | Core innovation, paradigm shift, key breakthroughs (e.g., RLVR) |
| **Data Cyan**      | `#00838F` | Data flow, information transfer, connections, processes     |

### Functional Colors

| Role               | HEX       | Usage                                                       |
| ------------------ | --------- | ----------------------------------------------------------- |
| **Warning Orange** | `#EF6C00` | Highlight differences, risks, cost items, performance bottlenecks |

### Neutral Grays

| Role               | HEX       | Usage                                                       |
| ------------------ | --------- | ----------------------------------------------------------- |
| **Mid Gray**       | `#90A4AE` | Auxiliary lines, secondary borders, background grids        |
| **Dark Gray**      | `#546E7A` | Primary text, axes, important icons                         |

### Background & Text

| Role               | HEX       | Usage                                                       |
| ------------------ | --------- | ----------------------------------------------------------- |
| **Main Background**| `#FFFFFF` | Page main background                                        |
| **Light Gray Background** | `#F8F9FA` | Card inner background, subtle areas                         |
| **Primary Text**   | `#546E7A` | Titles, important text (Dark Gray)                          |
| **Secondary Text** | `#90A4AE` | Annotations, page numbers, tips (Mid Gray)                  |
| **White Text**     | `#FFFFFF` | Text on dark/colored backgrounds                            |

---

## IV. Typography System

### Font Stack

**Font Stack**: `system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", "Noto Sans CJK SC", Arial, sans-serif`

> Uses system UI font stack with Chinese support for cross-platform consistency.

### Font Size Hierarchy

| Level | Usage                | Size   | Weight      |
| ----- | -------------------- | ------ | ----------- |
| H1    | Cover main title     | 56px   | 700 (Bold)  |
| H2    | Page main title      | 40px   | 700 (Bold)  |
| H3    | Module/section title | 28px   | 600         |
| H4    | Card title/subtitle  | 24px   | 600         |
| P     | Body content         | 20px   | 400         |
| Data  | Large data numbers   | 52px   | 700 (Bold)  |
| Label | Data labels/descriptions | 16px | 500        |
| Sub   | Auxiliary text/page number | 14px | 400       |

---

## V. Page Structure

### General Layout

| Area               | Position/Height | Description                                |
| ------------------ | --------------- | ------------------------------------------ |
| **Top Decorative Bar** | y=0, h=4px   | Single color or subtle gradient (optional) |
| **Title Area**     | y=50, h=60px    | Page title + optional underline            |
| **Content Area**   | y=130, h=500px  | Main content area                          |
| **Footer**         | y=660, h=60px   | Page number, source, optional logo         |

### Signature Design Elements

#### 1. Clean Flat Design
- No heavy gradients or shadows
- Minimal use of soft shadows (only for layer separation)
- Rounded corners: 12px for cards

#### 2. Color-Coded Accents
- Use core colors for key data points, icons, and section dividers
- Warning orange for risk/highlight elements

#### 3. Structured Layouts
- Ample white space
- Clear visual hierarchy
- Consistent spacing

---

## VI. Page Types

### 1. Cover Page (01_cover.svg)
- White or light gray background
- Centered main title + subtitle
- Optional: subtle geometric decoration (circle or hexagon) in Tech Deep Blue or Breakthrough Purple
- Date, author, and WeChat public account name at bottom
- Clean, minimal

### 2. Table of Contents Page (02_toc.svg)
- White background
- Page title with Data Cyan underline
- Chapter list with color-coded dots (using core colors in rotation)
- Page numbers aligned right

### 3. Chapter Page (02_chapter.svg)
- Light gray background (#F8F9FA)
- Large chapter number (gradient from Tech Deep Blue to Breakthrough Purple) as foreground visual anchor
- Chapter title in Dark Gray below the number
- Optional English subtitle
- Data Cyan underline at bottom

### 4. Content Page (03_content.svg)
- White background
- Page title + Data Cyan underline (4px)
- Content area: flexible, supports text, charts, images
- Footer: left side source, right side page number
- Optional: logo in top-right corner

### 5. Ending Page (04_ending.svg)
- Light gray background
- Centered "Thank You" in Breakthrough Purple
- Subtitle and closing message
- WeChat public account name in place of contact info
- Copyright in footer

---

## VII. Layout Patterns

| Pattern                | Use Cases                                      |
| ---------------------- | ---------------------------------------------- |
| **Centered Card**      | Cover, ending, key quotes                      |
| **Left Text Right Image** | Text description + chart/diagram            |
| **KPI Grid (2×2/2×3)** | Data overview, key metrics                    |
| **Three-Column Cards** | Feature lists, project highlights              |
| **Four Quadrants**     | Category display, SWOT analysis                |
| **Top-Bottom Split**   | Two related topics side by side                |
| **Timeline**           | Roadmap, development history                   |
| **Flowchart Style**    | Process, architecture diagrams                 |

---

## VIII. Spacing Guidelines

| Element              | Value    |
| -------------------- | -------- |
| Page margins         | 60px     |
| Title-to-content gap | 30-40px  |
| Module gap           | 60-80px  |
| Card gap             | 20-24px  |
| Card padding         | 20px     |
| Card border radius   | 12px     |
| Icon-to-text gap     | 15px     |

---

## IX. SVG Technical Constraints

### Mandatory Rules
1. viewBox: `0 0 1280 720`
2. Use `<rect>` elements for backgrounds
3. Use `<tspan>` for text wrapping (no `<foreignObject>`)
4. Use `fill-opacity` / `stroke-opacity` for transparency
5. Define gradients using `<linearGradient>` within `<defs>` (if needed)

### Prohibited Elements
- `clipPath`, `mask`
- `<style>` tag, `class` attribute
- `foreignObject`
- `textPath`
- `animate*` animation elements
- `script`
- `marker`, `marker-end`
- `rgba()` color format (use HEX + opacity instead)

### Shadow Implementation
- Use subtle border color variations to simulate shadows
- Or accept that `filter` may be ignored in older PPT versions; for modern PPT, `<filter>` with `feGaussianBlur` is acceptable but keep minimal

---

## X. Placeholder Specification

| Placeholder            | Description                          |
| ---------------------- | ------------------------------------ |
| `{{TITLE}}`            | Main title                           |
| `{{SUBTITLE}}`         | Subtitle                             |
| `{{DATE}}`             | Date                                 |
| `{{AUTHOR}}`           | Author name                          |
| `{{WECHAT_PUBLIC_NAME}}` | WeChat public account name         |
| `{{PAGE_TITLE}}`       | Page title                           |
| `{{CHAPTER_NUM}}`      | Chapter number (e.g., 01)            |
| `{{CHAPTER_TITLE}}`    | Chapter title                        |
| `{{CHAPTER_TITLE_EN}}` | English subtitle (optional)          |
| `{{PAGE_NUM}}`         | Page number                          |
| `{{SOURCE}}`           | Data source footnote                 |
| `{{CONTENT_AREA}}`     | Content area (for reference)         |
| `{{TOC_ITEM_N_TITLE}}` | TOC item title (N=1..6)              |
| `{{TOC_ITEM_N_PAGE}}`  | TOC item page number (N=1..6)        |
| `{{THANK_YOU}}`        | Thank you message                    |
| `{{ENDING_SUBTITLE}}`  | Ending subtitle                      |
| `{{CLOSING_MESSAGE}}`  | Closing message                      |
| `{{COPYRIGHT}}`        | Copyright notice                      |

---

## XI. Color Usage Examples

### Architecture / System Diagram
- AWS-related modules → Tech Deep Blue `#1F4E7A`
- NVIDIA-related modules → Eco Forest Green `#2E7D32`
- Generic components → Data Cyan `#00838F`
- Innovation layer → Breakthrough Purple `#6A1B9A`

### Comparison Analysis
- Left side (Trainium) → Tech Deep Blue
- Right side (NVIDIA) → Eco Forest Green
- Highlight differences → Warning Orange `#EF6C00`

### Flowchart
- Process steps → Data Cyan rectangles
- Decision/judgment nodes → Breakthrough Purple diamonds
- Start/End → rounded rectangles in Dark Gray
- Arrows → Dark Gray `#546E7A`

### Data Trend Charts
- Line colors follow core color sequence
- Background grid → Mid Gray `#90A4AE` (very light)
- Key data points labeled directly

---

## XII. Usage Instructions

1. Copy template files to `templates/` in your PPT project
2. Place your logo as `logo.png` in the same directory (optional)
3. Select appropriate page type based on content
4. Replace placeholders with actual content
5. Use the core colors consistently as defined above
6. Maintain ample white space and clear hierarchy
7. For complex diagrams, refer to the color mapping rules

---

_This specification is based on the "Tech Article Visualization Style V3" and adapted for PPT Master project._