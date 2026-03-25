# HTML Template for Daily Digest

**Date:** 2026-03-25
**Status:** Draft
**Scope:** 新增 `template/` 目录 + SKILL.md 集成 HTML 生成步骤

## Problem

每天生成的 digest 只有 `.md` 格式。想在浏览器中以更好的排版和交互体验阅读，需要每次手写 HTML 或没有可视化方案。

## Solution

在仓库中提供两个 HTML 模板文件，SKILL.md 流程末尾自动将 markdown digest 转为 HTML 并套入模板。同时生成索引页支持日期导航。

## File Structure

```
no-more-fomo/               # 项目仓库
├── template/
│   ├── digest.html         # Digest 页面模板
│   └── index.html          # 索引页模板
├── SKILL.md
└── ...

~/no-more-fomo/             # 用户输出目录
├── index.html              # 索引页 (每次更新)
├── 2026-03-22.html
├── 2026-03-22.md
├── 2026-03-25.html
├── 2026-03-25.md
└── ...
```

## Design: digest.html Template

### Three-View Layout

单份 HTML 结构，通过 CSS class 切换三种布局视图：

1. **Newspaper** — 经典单栏报纸风格，内容线性展开，最大化阅读宽度
2. **Sidebar** — 左侧 section 导航（sticky），右侧内容区，适合快速跳转
3. **Grid** — 每个 section 一张卡片，网格排列，紧凑概览

视图切换器在页面右上角，三个按钮（pill 形式），纯 CSS + 极少 JS：

```js
function setView(view) {
  document.getElementById('mainContent').className = 'content view-' + view;
}
```

所有视图共享同一份 HTML DOM，通过 `.view-newspaper` / `.view-sidebar` / `.view-grid` class 控制显示。

### Dual Theme

使用 my-design-system 的双主题 token：

- Light（`data-theme="light"`）：暖学术风 `--bg: #faf8f4`
- Dark（`data-theme="dark"`）：深灰板岩 `--bg: #16191e`
- 默认跟随系统（`prefers-color-scheme`），带手动切换按钮

```js
// 初始化
const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
document.documentElement.setAttribute('data-theme', prefersDark ? 'dark' : 'light');
```

### UI Language Toggle

页面顶部有 `中/EN` 切换按钮。影响范围仅限 UI 外壳（按钮标签、section 标题），不翻译 digest 正文。

实现方式：所有可切换元素带 `data-zh` / `data-en` 属性：

```html
<button class="view-btn" data-zh="报纸" data-en="Newspaper">报纸</button>
<div class="section-header" data-zh="模型与发布" data-en="Models & Releases">模型与发布</div>
```

JS 切换：

```js
function setLang(lang) {
  document.documentElement.setAttribute('data-lang', lang);
  document.querySelectorAll('[data-' + lang + ']').forEach(el => {
    el.textContent = el.getAttribute('data-' + lang);
  });
}
```

### Page Structure

```html
<div class="header">
  <!-- 标题: No More FOMO -->
  <!-- 日期 + items 统计 -->
  <!-- 控件: view switcher + theme toggle + lang toggle -->
</div>

<div class="highlights">
  <!-- Today's Highlights: top 3 -->
  <!-- DIGEST_HIGHLIGHTS 占位符 -->
</div>

<div class="content" id="mainContent">
  <div class="content-wrap">
    <nav class="sidebar-nav">
      <!-- section 导航链接 (sidebar 视图用) -->
    </nav>
    <div class="main-content">
      <div class="sections-container">
        <!-- DIGEST_SECTIONS 占位符 -->
        <!-- 每个 section: .section div with .section-header + .item list -->
      </div>
    </div>
  </div>
</div>

<div class="footer">
  <!-- DIGEST_FOOTER 占位符 -->
</div>
```

### Template Placeholders

| Placeholder | Content |
|-------------|---------|
| `<!-- DIGEST_DATE -->` | `2026-03-25` |
| `<!-- DIGEST_META -->` | `53 items \| Tier1-KOLs(12) ...` |
| `<!-- DIGEST_HIGHLIGHTS -->` | Highlights HTML 片段 |
| `<!-- DIGEST_SIDEBAR_NAV -->` | Sidebar 导航链接 |
| `<!-- DIGEST_SECTIONS -->` | 所有 section 的 HTML |
| `<!-- DIGEST_FOOTER -->` | Sources 行 |

### CSS Component Reference (from my-design-system)

模板复用以下 design token 和组件模式：

- CSS custom properties：全套 light/dark tokens
- `.card` 组件：hover border-color 变化
- `.badge` 组件：source badge（accent 色）、green badge（HN pts）
- Font stack：system sans-serif + monospace for data
- Font sizes：xs=10px, sm=11px, base=13px, md=14px, lg=18px

新增组件（digest 专用）：

- `.section` + `.section-header`：带 accent 色下划线的 section 标题
- `.item`：单条 digest 条目（标题 + 描述 + 链接 + badge）
- `.podcast-summary`：播客摘要卡片（accent 色左边框 + TLDR/章节/引用）
- `.highlights`：顶部 highlights 卡片（surface-warm 背景）
- `.view-switcher`：pill 形式的视图切换按钮组

### Podcast Summary Format

播客条目在 HTML 中有特殊样式：

```html
<div class="item">
  <div class="item-title">[Dwarkesh] Terence Tao — Kepler...</div>
  <div class="item-meta">Mar 20 | <a href="...">link</a></div>
  <div class="podcast-summary">
    <div class="tldr">TLDR: Tao 认为 AI 已将假设生成成本...</div>
    <div class="chapters">
      章节: <em>开普勒——高温采样的 LLM</em> | <em>验证是新瓶颈</em> | <em>数学发现的未来</em>
    </div>
    <div class="quote">
      "开普勒本质上在运行一个高温采样过程..." — Terence Tao [00:15:32]
    </div>
  </div>
</div>
```

## Design: index.html Template

### 功能

1. **静态日期列表** — SKILL.md 每次生成 digest 时同步更新，硬编码所有已知日期
2. **快速日期输入** — 输入框 + 跳转按钮，构造 `./YYYY-MM-DD.html` 链接
3. **最新 digest 高亮** — 列表顶部，视觉突出

### 结构

```html
<div class="header">
  <!-- No More FOMO Archive -->
  <!-- theme + lang toggle -->
</div>

<div class="content">
  <div class="quick-nav">
    <input type="date" id="dateInput">
    <button onclick="goToDate()">Go</button>
  </div>

  <div class="date-grid">
    <!-- INDEX_ENTRIES 占位符 -->
    <!-- 每条: date + items count + top highlight, 链接到 .html -->
  </div>
</div>
```

### Placeholder

| Placeholder | Content |
|-------------|---------|
| `<!-- INDEX_ENTRIES -->` | 日期卡片列表 HTML |

每张日期卡片：

```html
<a href="./2026-03-25.html" class="date-card latest">
  <div class="date-card-date">2026-03-25</div>
  <div class="date-card-meta">53 items</div>
  <div class="date-card-highlight">Thariq's technical writing (4.9K likes)</div>
</a>
```

索引页复用同套 design system（tokens + theme toggle + lang toggle），样式自包含在模板内。

## SKILL.md Integration

### 新增步骤：Phase 2 Step D (Generate HTML)

在 Phase 2 Step C 之后（或 Phase 1 末尾如果 `--quick`），新增：

```
### Phase 2 Step D: Generate HTML

1. Read template/digest.html from the skill's directory
2. Convert current digest sections to HTML fragments:
   - Section headers → <div class="section"><div class="section-header" data-zh="..." data-en="...">
   - Items → <div class="item"> with .item-title, .item-desc, .item-meta, .item-links
   - Podcast summaries → <div class="podcast-summary"> with .tldr, .chapters, .quote
   - Badges → <span class="badge badge-source">
   - Links → <a href="...">
3. Replace placeholders in template → Write ~/no-more-fomo/YYYY-MM-DD.html
4. Scan ~/no-more-fomo/*.html (excluding index.html):
   - Extract date from filename
   - Extract meta (items count) and first highlight from each file
   - Read template/index.html, replace INDEX_ENTRIES → Write ~/no-more-fomo/index.html
```

### Markdown → HTML 转换规则

不需要通用 markdown 解析器。digest 格式是固定的，Claude 在生成时已知每个 section 的结构：

| Markdown Pattern | HTML Output |
|-----------------|-------------|
| `**text**` | `<b>text</b>` |
| `[text](url)` | `<a href="url">text</a>` |
| `> blockquote` | podcast-summary div |
| `- **Title** — desc \| links \| @source \| NL` | `.item` div with sub-elements |
| `## Section Title` | `.section` div with `.section-header` |
| `@handle` | `<span class="badge badge-source">` |

### Flag Behavior

| Flag | HTML 生成 |
|------|----------|
| (default) | Phase 2 Step D 生成 HTML + 更新 index |
| `--quick` | Phase 1 末尾生成基础 HTML（播客带占位符），不等 Phase 2 |
| `--no-save` | 不生成 HTML |
| `--no-html` | 只生成 .md，跳过 HTML |

### `--quick` 下的 HTML 生成

Phase 1 完成后立即生成 HTML（播客部分有 `⏳ 深度摘要生成中...` 占位符）。如果后续用户不带 `--quick` 重跑，Phase 2 完成后会覆盖同一天的 `.html` 文件。

## Non-Goals

- 服务端渲染或构建工具
- JS 框架（React/Vue）
- 在线托管（GitHub Pages 等）
- digest 正文翻译（只有 UI 外壳语言切换）
- 搜索/过滤功能（YAGNI，可后续加）
