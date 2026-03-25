# HTML Template Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create reusable HTML templates so each daily digest also outputs a styled, interactive `.html` file with three-view layout, dual theme, and language toggle.

**Architecture:** Two template files (`template/digest.html`, `template/index.html`) with `{{PLACEHOLDER}}` markers. SKILL.md gets a new Step D that reads template, converts markdown sections to HTML fragments, replaces placeholders, and writes output. No build tools, no frameworks — vanilla HTML/CSS/JS, self-contained per file.

**Tech Stack:** Vanilla HTML + CSS custom properties + JS. my-design-system tokens. `file:///` compatible.

**Spec:** `docs/superpowers/specs/2026-03-25-html-template-design.md`

**Important — locating SKILL.md edits:** Use anchor text, NOT line numbers, since previous tasks may shift line positions.

---

### Task 1: Create `template/digest.html`

**Files:**
- Create: `template/digest.html`

This is the main template — a complete self-contained HTML file with all CSS and JS inline, using `{{PLACEHOLDER}}` markers where Claude inserts content.

- [ ] **Step 1: Create the template directory**

```bash
mkdir -p template
```

- [ ] **Step 2: Write the complete digest.html template**

Create `template/digest.html` with the following structure. This is the full file — CSS (~300 lines), HTML skeleton with placeholders, and JS (~40 lines):

**CSS sections to include (all using my-design-system tokens):**
- `:root` light mode tokens + `[data-theme="dark"]` dark mode tokens
- Base: `body`, `a`, `*` reset
- Header: `.header`, `.title`, `.meta`, `.controls`
- View switcher: `.view-switcher`, `.view-btn`, `.view-btn.active`
- Theme/lang toggles: `.theme-btn`, `.lang-btn`
- Highlights: `.highlights`, `.highlights-title`
- Content layout: `.content`, `.content-wrap`, `.main-content`, `.sections-container`
- Section: `.section`, `.section-header`, `.section-count`
- Item: `.item`, `.item-title`, `.item-desc`, `.item-meta`, `.item-links`
- Item empty: `.item-empty` (muted, italic)
- Badge: `.badge`, `.badge-source`, `.badge-green`
- Podcast: `.podcast-summary`, `.podcast-summary .tldr`, `.chapters`, `.quote`, `.podcast-summary.loading`
- Community: `.community-discussion`
- Footer: `.footer`
- **View: Newspaper** — `.view-newspaper .sidebar-nav { display: none; }`, single column
- **View: Sidebar** — `.view-sidebar .content-wrap { display: flex; }`, `.sidebar-nav` sticky, 160px wide
- **View: Grid** — `.view-grid .sections-container { display: grid; grid-template-columns: 1fr 1fr; }`, `.section` as card, `.section.wide { grid-column: 1 / -1; }`

**HTML skeleton:**
```html
<!DOCTYPE html>
<html data-theme="light" data-lang="{{DIGEST_LANG}}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>No More FOMO — {{DIGEST_DATE}}</title>
  <style>/* ALL CSS HERE */</style>
</head>
<body>
  <div class="header">
    <div class="header-top">
      <div>
        <div class="title">No More <span>FOMO</span></div>
        <div class="meta">{{DIGEST_DATE}} | {{DIGEST_META}}</div>
      </div>
      <div class="controls">
        <div class="view-switcher">
          <button class="view-btn active" data-zh="报纸" data-en="Newspaper" onclick="setView('newspaper')">报纸</button>
          <button class="view-btn" data-zh="侧栏" data-en="Sidebar" onclick="setView('sidebar')">侧栏</button>
          <button class="view-btn" data-zh="网格" data-en="Grid" onclick="setView('grid')">网格</button>
        </div>
        <button class="theme-btn" onclick="toggleTheme()">🌓</button>
        <button class="lang-btn" onclick="toggleLang()">中/EN</button>
      </div>
    </div>
  </div>

  <div class="highlights">
    {{DIGEST_HIGHLIGHTS}}
  </div>

  <div class="content view-newspaper" id="mainContent">
    <div class="content-wrap">
      <nav class="sidebar-nav">
        {{DIGEST_SIDEBAR_NAV}}
      </nav>
      <div class="main-content">
        <div class="sections-container">
          {{DIGEST_SECTIONS}}
        </div>
      </div>
    </div>
  </div>

  <div class="footer">{{DIGEST_FOOTER}}</div>

  <script>/* ALL JS HERE */</script>
</body>
</html>
```

**JS functions:**
- `setView(view)` — set class on `#mainContent`, update active button, persist to `localStorage('fomo-view')`
- `toggleTheme()` — toggle `data-theme` between light/dark, persist to `localStorage('fomo-theme')`
- `toggleLang()` — toggle `data-lang` between zh/en, call `setLang()`, persist to `localStorage('fomo-lang')`
- `setLang(lang)` — update all elements with `data-zh`/`data-en` attributes
- Init block: read localStorage for saved view/theme/lang, apply; fallback to newspaper/system-theme/`{{DIGEST_LANG}}`

- [ ] **Step 3: Verify the template**

Open `template/digest.html` directly in a browser to verify the CSS renders correctly (placeholders will show as literal `{{...}}` text — that's expected).

- [ ] **Step 4: Commit**

```bash
git add template/digest.html
git commit -m "feat: add digest.html template with three-view layout, dual theme, lang toggle"
```

---

### Task 2: Create `template/index.html`

**Files:**
- Create: `template/index.html`

The index/archive page. Simpler than digest — header + date input + date card grid.

- [ ] **Step 1: Write the complete index.html template**

Same CSS token foundation as digest.html (inline, self-contained), but simpler layout — no view switcher, no sections. Include:

**CSS additions beyond digest shared tokens:**
- `.quick-nav`: input + button row
- `.date-grid`: grid layout for date cards
- `.date-card`: card with hover, link style
- `.date-card.latest`: accent border highlight
- `.date-card-date`, `.date-card-meta`, `.date-card-highlight`

**HTML skeleton:**
```html
<!DOCTYPE html>
<html data-theme="light" data-lang="zh">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>No More FOMO — Archive</title>
  <style>/* CSS: shared tokens + index-specific styles */</style>
</head>
<body>
  <div class="header">
    <div class="header-top">
      <div>
        <div class="title">No More <span>FOMO</span></div>
        <div class="meta" data-zh="历史存档" data-en="Archive">历史存档</div>
      </div>
      <div class="controls">
        <button class="theme-btn" onclick="toggleTheme()">🌓</button>
        <button class="lang-btn" onclick="toggleLang()">中/EN</button>
      </div>
    </div>
  </div>

  <div class="content">
    <div class="quick-nav">
      <input type="date" id="dateInput">
      <button onclick="goToDate()" data-zh="跳转" data-en="Go">跳转</button>
    </div>
    <div class="date-grid">
      {{INDEX_ENTRIES}}
    </div>
  </div>

  <script>
    function goToDate() {
      const d = document.getElementById('dateInput').value;
      if (d) window.location.href = './' + d + '.html';
    }
    function toggleTheme() { /* same as digest */ }
    function toggleLang() { /* same as digest */ }
    function setLang(lang) { /* same as digest */ }
    // Init: localStorage restore
  </script>
</body>
</html>
```

- [ ] **Step 2: Verify the template**

Open `template/index.html` directly in browser to verify CSS and layout.

- [ ] **Step 3: Commit**

```bash
git add template/index.html
git commit -m "feat: add index.html template with date grid and quick navigation"
```

---

### Task 3: Add `--no-html` flag to SKILL.md

**Files:**
- Modify: `SKILL.md` (Arguments section)

- [ ] **Step 1: Add `--no-html` to the Arguments table**

Find the `## Arguments` section in SKILL.md. In the arguments table, after the `--no-save` row, add:

```markdown
| `--no-html` | Only generate .md, skip HTML output |
```

- [ ] **Step 2: Add to flag combinations table**

In the "Flag combinations" table, add:

```markdown
| `--no-html` | .md only, no HTML generation |
| `--quick --no-html` | Phase 1 .md only, no Phase 2, no HTML |
```

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "feat: add --no-html flag to skip HTML generation"
```

---

### Task 4: Add Phase 2 Step D to SKILL.md

**Files:**
- Modify: `SKILL.md` (Phase 2 section)

- [ ] **Step 1: Insert Step D after Step C**

Find `### Phase 2 Step C: Update Digest File` section in SKILL.md. After the last line of Step C content (the line about `社区补充` and `发现`), insert:

```markdown

### Phase 2 Step D: Generate HTML

Skip this step if `--no-save` or `--no-html` flag is set.

1. **Read template:** Read `template/digest.html` from the skill's directory (the repo where SKILL.md lives).

2. **Convert digest to HTML fragments:** Using the current digest content (already in memory from Steps A-C), generate HTML for each component. Do NOT parse the .md file — use the structured data you already have.

   **Conversion rules:**
   - Section headers → `<div class="section" id="SECTION_ID"><div class="section-header" data-zh="中文标题" data-en="English Title">中文标题 <span class="section-count">N</span></div>`
   - Items → `<div class="item"><div class="item-title">TITLE</div><div class="item-desc">DESC</div><div class="item-meta"><span class="item-links"><a href="URL" target="_blank" rel="noopener">label</a></span> <span class="badge badge-source">@handle</span> NL</div></div>`
   - Podcast items → same as item but with `<div class="podcast-summary"><div class="tldr">...</div><div class="chapters">...</div><div class="quote">...</div></div>` appended
   - Podcast loading state (--quick) → `<div class="podcast-summary loading"><div class="tldr" data-zh="⏳ 深度摘要生成中..." data-en="⏳ Generating deep summary...">⏳ 深度摘要生成中...</div></div>`
   - Community discussion → `<div class="community-discussion">社区热议: ...</div>` inside the `.item`
   - Empty sections → `<div class="item-empty" data-zh="本周无新内容" data-en="No new content this week">本周无新内容</div>`
   - Highlights → `<div class="highlights-title" data-zh="今日要点" data-en="Today's Highlights">今日要点</div><ol><li>...</li></ol>`
   - Sidebar nav → `<a href="#SECTION_ID" data-zh="中文(N)" data-en="English(N)">中文(N)</a>` for each section
   - **HTML escape** all digest text content: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;` (in attribute values)
   - All external links: add `target="_blank" rel="noopener"`

3. **Replace placeholders:** Replace `{{DIGEST_DATE}}`, `{{DIGEST_LANG}}`, `{{DIGEST_META}}`, `{{DIGEST_HIGHLIGHTS}}`, `{{DIGEST_SIDEBAR_NAV}}`, `{{DIGEST_SECTIONS}}`, `{{DIGEST_FOOTER}}` in the template.

4. **Write HTML file:** Write to `~/no-more-fomo/YYYY-MM-DD.html`

5. **Update index:** Scan `~/no-more-fomo/` for files matching `YYYY-MM-DD.html` pattern (ignore `index.html` and other files). For each digest date, extract meta info from the corresponding `.md` file (or from current session memory if it's today's digest):
   - **Items count:** from the `Sources:` line (e.g., "Total: 53 items")
   - **Top highlight:** first item from the `## 今日要点` / `## Top Highlights` section
   Generate `{{INDEX_ENTRIES}}` with one date card per file (newest first, newest gets `.latest` class). Read `template/index.html`, replace placeholder, write to `~/no-more-fomo/index.html`.

6. **`{{DIGEST_FOOTER}}`** is the `Sources: Tier1-KOLs(N) ... Total: N items` line from the digest, wrapped in `<span class="footer-text">`.

**For `--quick` mode:** Run this step at the end of Phase 1 instead of Phase 2 (podcast summaries will show loading state). If the user later runs without `--quick`, Phase 2 Step D **fully regenerates** the HTML (re-reads template, re-replaces all placeholders) and overwrites the file. This is NOT an incremental edit — it's a complete rebuild to ensure template consistency.
```

- [ ] **Step 2: Verify the new step reads correctly**

Read the Phase 2 section to confirm Step D is coherent and follows Step C.

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "feat: add Phase 2 Step D — HTML generation from template"
```

---

### Task 5: Update README.md and docs/zh.md

**Files:**
- Modify: `README.md`
- Modify: `docs/zh.md`

- [ ] **Step 1: Update README.md**

1. In "What makes this different" section, add a new bullet after the existing ones:
```markdown
- Every digest also outputs a **styled HTML version** with three-view layout, dark mode, and date navigation
```

2. In Usage section, add:
```bash
/no-more-fomo --no-html          # Markdown only, skip HTML output
```

3. In "How it works" section, after step 6 (Output), add:
```markdown
7. **Render** — generates styled HTML from template with three-view layout (newspaper/sidebar/grid)
```

- [ ] **Step 2: Update docs/zh.md**

1. In "和其他 AI 新闻工具的区别" section, add:
```markdown
- 每份 digest 同时输出 **HTML 可视化版本**，支持三种布局、暗色主题、日期导航
```

2. In usage section, add:
```bash
/no-more-fomo --no-html          # 只生成 markdown，跳过 HTML
```

- [ ] **Step 3: Commit**

```bash
git add README.md docs/zh.md
git commit -m "docs: document HTML template output and --no-html flag"
```

---

### Task 6: Final Verification

**Files:**
- Read: `template/digest.html`, `template/index.html`, `SKILL.md`

- [ ] **Step 1: Verify template files are valid HTML**

Open both template files in a browser and confirm:
- CSS renders correctly (light theme default)
- Theme toggle works
- View switcher works (digest.html)
- Lang toggle works
- Placeholder text (`{{...}}`) is visible in the correct locations

- [ ] **Step 2: Cross-reference with spec**

Read the spec and verify:
- [ ] Three-view layout (newspaper/sidebar/grid) in digest.html
- [ ] Dual theme with system detection + manual toggle
- [ ] UI language toggle (zh/en) with localStorage persistence
- [ ] All 7 placeholders present in digest.html
- [ ] {{INDEX_ENTRIES}} placeholder in index.html
- [ ] Date input quick-nav in index.html
- [ ] Phase 2 Step D in SKILL.md with full conversion rules
- [ ] `--no-html` flag in Arguments table
- [ ] HTML escaping rules documented
- [ ] External links with target="_blank"
- [ ] Empty section, community discussion, podcast loading states

- [ ] **Step 3: Fix any issues found**

- [ ] **Step 4: Final commit (if fixes needed)**

```bash
git add template/ SKILL.md README.md docs/zh.md
git commit -m "fix: address issues from final verification"
```
