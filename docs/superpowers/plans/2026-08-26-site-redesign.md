# Site Redesign (Signal + Blue) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restyle parameterfreak.com end-to-end into the "Signal + 블루" design (spec: `docs/superpowers/specs/2026-08-26-site-redesign-design.md`) without changing any URL.

**Architecture:** Keep the academicpages/Jekyll theme in place. Add one override partial `_sass/_pf.scss` (tokens, chrome, article layout) imported last, rewrite `_sass/_landing.scss` (landing components), replace three chrome/layout templates (`masthead.html`, `footer.html`, `single.html`), and rewrite two landing pages (`index.html`, `hmas.html`). Everything else inherits the new tokens.

**Tech Stack:** Jekyll 3.10 (github-pages gem 232), SCSS, Liquid, Google Fonts (Gothic A1 / Noto Sans KR / IBM Plex Mono), Playwright MCP for visual checks.

**Verification model:** There are no unit tests for a static site. Every task ends with `bundle exec jekyll build` succeeding plus a `grep` on `_site/` output, and visual tasks add a Playwright screenshot at 1280px and 390px. Do not claim a task done without the build output.

---

## File map

| File | Responsibility | Action |
|---|---|---|
| `_sass/_themes.scss` | theme font variables | modify 3 lines |
| `_sass/_pf.scss` | design tokens, theme overrides, nav, footer, article layout | **create** |
| `_sass/_landing.scss` | `.lp-*` landing components | **rewrite** |
| `assets/css/main.scss` | SCSS import list | add `pf` import, delete old custom rules |
| `_includes/head/custom.html` | Google Fonts link | add 3 lines |
| `_includes/masthead.html` | site nav | **rewrite** |
| `_includes/footer.html` | site footer | **rewrite** |
| `_includes/footer/custom.html` | remove Sitemap link (moved into footer) | modify |
| `_layouts/single.html` | article layout (posts, changelog, portfolio, legacy) | **rewrite** |
| `_pages/index.html` | home | **rewrite** |
| `_pages/hmas.html` | H-MAS landing | **rewrite** (same content, new markup/images) |
| `_pages/portfolio.html`, `_pages/changelog.html`, `_pages/year-archive.html` | listing pages | small edits |
| `_portfolio/H-MAS.md` | product doc | teaser + image paths |
| `_config.yml` | `og_image`, defaults cleanup | modify |

---

### Task 0: Branch and local build environment

**Files:** none in repo (environment only)

Homebrew is available but `/opt/homebrew` is owned by another user, so `brew install` cannot run. Instead use Homebrew's bundled portable Ruby 3.4.4 (read-only, has headers) with a user-local `GEM_HOME`. `source .rubyenv` (gitignored) sets this up; every later `bundle exec` in this plan assumes it. Where a later step says `export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"`, run `source .rubyenv` instead.

- [ ] **Step 1: Create the working branch**

```bash
cd /Users/research/ws/parameterfreak.github.io
git checkout -b redesign
```
Expected: `Switched to a new branch 'redesign'`

- [ ] **Step 2: Install Ruby 3.3 and bundler**

```bash
brew install ruby@3.3
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
gem install bundler -v 2.4.19 --no-document
```
Expected: last line `1 gem installed`.

- [ ] **Step 3: Install gems and build once**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle _2.4.19_ install
bundle exec jekyll build 2>&1 | tail -3
```
Expected: `done in N seconds.` with no `Error`. If `bundle install` fails on a native gem (`nokogiri`, `commonmarker`), run `bundle config set force_ruby_platform true` and retry.

- [ ] **Step 4: Confirm the current site builds identically to live**

```bash
ls _site/changelog/h-mas/2026-08-02/index.html _site/hmas/index.html _site/index.html
```
Expected: three paths printed, no error.

- [ ] **Step 5: Commit the spec and plan**

```bash
git add docs/superpowers
git commit -m "docs: site redesign spec and plan"
```

---

### Task 1: Design tokens and fonts

**Files:**
- Modify: `_sass/_themes.scss:17-19,42-44`
- Create: `_sass/_pf.scss`
- Modify: `assets/css/main.scss:43-59`
- Modify: `_includes/head/custom.html`

After this task every page (including legacy ones) renders in the new palette and typefaces, with the nav/footer still in the old markup.

- [ ] **Step 1: Swap the theme font variables**

In `_sass/_themes.scss` replace lines 17–19:

```scss
$serif                      : Georgia, Times, serif;
$sans-serif                 : "Noto Sans KR", -apple-system, "Segoe UI", "Helvetica Neue", Arial, sans-serif;
$monospace                  : "IBM Plex Mono", Menlo, Consolas, monospace;
```

and lines 42–44:

```scss
$global-font-family         : $sans-serif;
$header-font-family         : "Gothic A1", "Noto Sans KR", sans-serif;
$caption-font-family        : $sans-serif;
```

- [ ] **Step 2: Create `_sass/_pf.scss` with tokens and theme overrides**

```scss
/* ==========================================================================
   PARAMETERFREAK — design tokens, theme overrides, nav, footer, article.
   Imported last from assets/css/main.scss so it wins the cascade.
   ========================================================================== */

$pf-display: "Gothic A1", "Noto Sans KR", sans-serif;
$pf-body:    "Noto Sans KR", -apple-system, "Segoe UI", sans-serif;
$pf-mono:    "IBM Plex Mono", Menlo, Consolas, monospace;

:root {
  --pf-bg:        #ffffff;
  --pf-ink:       #16181d;
  --pf-grey:      #4b4f57;
  --pf-mute:      #6b6f78;
  --pf-line:      #e9eaec;
  --pf-soft:      #f7f7f8;
  --pf-blue:      #063ad7;
  --pf-blue-soft: #e8edfb;

  /* remap academicpages variables so untouched theme components follow the palette */
  --global-base-color:            var(--pf-grey);
  --global-bg-color:              var(--pf-bg);
  --global-footer-bg-color:       var(--pf-bg);
  --global-border-color:          var(--pf-line);
  --global-dark-border-color:     var(--pf-line);
  --global-code-background-color: var(--pf-soft);
  --global-code-text-color:       var(--pf-ink);
  --global-fig-caption-color:     var(--pf-mute);
  --global-link-color:            var(--pf-blue);
  --global-link-color-hover:      #042a9c;
  --global-link-color-visited:    var(--pf-blue);
  --global-masthead-link-color:   var(--pf-ink);
  --global-masthead-link-color-hover: var(--pf-blue);
  --global-text-color:            var(--pf-ink);
  --global-text-color-light:      var(--pf-mute);
  --global-thead-color:           var(--pf-soft);
}

/* ---- theme structure overrides -------------------------------------- */
html { position: static; }                 /* drop sticky-footer hack */
body {
  padding: 0;                              /* theme reserved space for a fixed masthead */
  font-size: 16px;
  line-height: 1.75;
  -webkit-font-smoothing: antialiased;
}
h1, h2, h3, h4, h5, h6 { letter-spacing: -0.02em; margin: 1.6em 0 0.5em; }
a { color: var(--pf-blue); }
#main { animation: none; }
.page__footer {
  position: static;
  float: none;
  width: auto;
  margin-top: 0;
  animation: none;
  border-top: 1px solid var(--pf-line);
  background: var(--pf-bg);
  color: var(--pf-grey);
  footer { max-width: none; margin: 0; padding: 0; }
}
```

- [ ] **Step 3: Import it and delete the old custom rules in `assets/css/main.scss`**

Replace everything from line 43 (`"landing"`) to the end of the file with:

```scss
    "pf",
    "landing"
;
```

`pf` comes before `landing` because `_landing.scss` (Task 4) uses the `$pf-*` Sass variables defined in `_pf.scss`; the two files style disjoint selectors, so cascade order between them does not matter.

- [ ] **Step 4: Add Google Fonts to `_includes/head/custom.html`**

Insert directly after `<!-- start custom head snippets -->`:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Gothic+A1:wght@700;900&family=Noto+Sans+KR:wght@400;500;700&family=IBM+Plex+Mono:wght@400;500&display=swap">
```

- [ ] **Step 5: Build and verify**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle exec jekyll build 2>&1 | tail -2
grep -c "fonts.googleapis.com/css2" _site/index.html
grep -o "\-\-pf-blue:#063ad7" _site/assets/css/main.css | head -1
grep -o 'font-family:"Gothic A1"' _site/assets/css/main.css | head -1
```
Expected: `done in …`, `1`, `--pf-blue:#063ad7`, `font-family:"Gothic A1"`.

- [ ] **Step 6: Commit**

```bash
git add _sass/_themes.scss _sass/_pf.scss assets/css/main.scss _includes/head/custom.html
git commit -m "style: design tokens and typefaces (Signal + blue)"
```

---

### Task 2: Masthead and footer

**Files:**
- Rewrite: `_includes/masthead.html`
- Rewrite: `_includes/footer.html`
- Modify: `_includes/footer/custom.html`
- Modify: `_sass/_pf.scss` (append)

- [ ] **Step 1: Replace `_includes/masthead.html`**

```html
{% include base_path %}

<header class="pf-nav">
  <div class="pf-nav__wrap">
    <a class="pf-nav__logo" href="{{ base_path }}/">parameterfreak</a>
    <nav class="pf-nav__links" aria-label="주 메뉴">
      {% for link in site.data.navigation.main %}
        <a href="{{ base_path }}{{ link.url }}"{% if page.url contains link.url %} aria-current="page"{% endif %}>{{ link.title }}</a>
      {% endfor %}
      <a class="pf-nav__cta" href="mailto:contact@parameterfreak.com">문의하기</a>
    </nav>
  </div>
</header>
```

- [ ] **Step 2: Replace `_includes/footer.html`**

```html
{% include base_path %}

<div class="pf-footer">
  <div class="pf-footer__wrap">
    <div class="pf-footer__brand">
      <a class="pf-nav__logo" href="{{ base_path }}/">parameterfreak</a>
      <p>AI 시스템을 설계하고 만듭니다</p>
    </div>
    <nav class="pf-footer__links" aria-label="푸터 메뉴">
      {% for link in site.data.navigation.main %}<a href="{{ base_path }}{{ link.url }}">{{ link.title }}</a>{% endfor %}
      <a href="{{ base_path }}/feed.xml">Feed</a>
      <a href="{{ base_path }}/sitemap/">Sitemap</a>
    </nav>
    <a class="pf-footer__mail" href="mailto:contact@parameterfreak.com">contact@parameterfreak.com</a>
  </div>
  <div class="pf-footer__wrap pf-footer__copy">&copy; {{ site.time | date: '%Y' }} parameterfreak</div>
</div>
```

- [ ] **Step 3: Remove the Sitemap link from `_includes/footer/custom.html`**

Delete the line `<a href="/sitemap/">Sitemap</a>` (keep the MathJax script tags).

- [ ] **Step 4: Append nav + footer styles to `_sass/_pf.scss`**

```scss
/* ---- nav ------------------------------------------------------------- */
.pf-nav {
  background: var(--pf-bg);
  border-bottom: 1px solid var(--pf-line);
}
.pf-nav__wrap {
  max-width: 1080px;
  margin: 0 auto;
  padding: 0 1.5rem;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}
.pf-nav__logo {
  font-family: $pf-display;
  font-weight: 900;
  font-size: 1rem;
  letter-spacing: -0.01em;
  color: var(--pf-ink);
  text-decoration: none;
  white-space: nowrap;
}
.pf-nav__links {
  display: flex;
  align-items: center;
  gap: 1.6rem;
  font-size: 0.9rem;
  font-weight: 500;
  a { color: var(--pf-grey); text-decoration: none; }
  a:hover, a[aria-current="page"] { color: var(--pf-ink); }
}
.pf-nav__cta {
  background: var(--pf-blue);
  color: #fff !important;
  padding: 0.45rem 0.95rem;
  border-radius: 999px;
  font-size: 0.85rem;
  &:hover { background: #042a9c; }
}
@media (max-width: 600px) {
  .pf-nav__wrap { height: 56px; padding: 0 1rem; }
  .pf-nav__links { gap: 1rem; font-size: 0.82rem; }
  .pf-nav__cta { display: none; }
}

/* ---- footer ---------------------------------------------------------- */
.pf-footer { font-size: 0.85rem; }
.pf-footer__wrap {
  max-width: 1080px;
  margin: 0 auto;
  padding: 2.5rem 1.5rem;
  display: flex;
  flex-wrap: wrap;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1.5rem 2rem;
}
.pf-footer__brand p { margin: 0.4rem 0 0; color: var(--pf-mute); }
.pf-footer__links {
  display: flex;
  flex-wrap: wrap;
  gap: 1.2rem;
  a { color: var(--pf-grey); text-decoration: none; &:hover { color: var(--pf-ink); } }
}
.pf-footer__mail { color: var(--pf-ink); text-decoration: none; font-weight: 500; }
.pf-footer__copy {
  padding-top: 0;
  padding-bottom: 2rem;
  color: var(--pf-mute);
  font-size: 0.78rem;
}
```

- [ ] **Step 5: Build and verify**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle exec jekyll build 2>&1 | tail -1
grep -o 'class="pf-nav__logo"[^>]*>[^<]*' _site/index.html | head -1
grep -o 'aria-current="page"' _site/changelog/index.html | wc -l
grep -c "AcademicPages" _site/index.html
grep -o 'pf-footer__copy">[^<]*' _site/index.html
```
Expected: `done in …`, `class="pf-nav__logo" href="/">parameterfreak`, `1`, `0`, `pf-footer__copy">© 2026 parameterfreak`.

- [ ] **Step 6: Visual check**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle exec jekyll serve --skip-initial-build --port 4000 > /dev/null 2>&1 &
```
With Playwright: navigate to `http://localhost:4000/changelog/`, resize 1280×900 → screenshot; resize 390×800 → screenshot. Confirm: wordmark fully visible (no clipped "P"), Changelog link darker than the other two, CTA pill hidden at 390px, footer three columns with © line.

- [ ] **Step 7: Commit**

```bash
git add _includes/masthead.html _includes/footer.html _includes/footer/custom.html _sass/_pf.scss
git commit -m "feat: new masthead and footer"
```

---

### Task 3: Article layout (posts, changelog, portfolio, legacy pages)

**Files:**
- Rewrite: `_layouts/single.html`
- Modify: `_sass/_pf.scss` (append)
- Modify: `_config.yml` defaults (remove unused keys)

- [ ] **Step 1: Replace `_layouts/single.html`**

```html
---
layout: default
---

{% include base_path %}

<div id="main" role="main" class="art">
  <aside class="art__toc" hidden>
    <p class="art__toc-label">On this page</p>
    <nav id="art-toc"></nav>
  </aside>

  <article class="art__body" itemscope itemtype="http://schema.org/CreativeWork">
    {% if page.collection == "changelog" %}
      <p class="art__crumb"><a href="{{ base_path }}/changelog/">Changelog</a> · {{ page.categories[0] }}</p>
    {% elsif page.collection == "posts" %}
      <p class="art__crumb"><a href="{{ base_path }}/posts/">Blog</a></p>
    {% elsif page.collection == "portfolio" %}
      <p class="art__crumb"><a href="{{ base_path }}/solution/">Products</a></p>
    {% endif %}

    {% if page.title %}<h1 class="art__title" itemprop="headline">{{ page.title | markdownify | remove: "<p>" | remove: "</p>" }}</h1>{% endif %}

    {% if page.date or page.read_time %}
      <p class="art__meta">
        {% if page.date %}<time datetime="{{ page.date | date_to_xmlschema }}" itemprop="datePublished">{{ page.date | date: "%Y.%m.%d" }}</time>{% endif %}
        {% if page.read_time %}<span>{% include read-time.html %}</span>{% endif %}
      </p>
    {% endif %}

    <div class="art__content" itemprop="text">
      {{ content }}
    </div>

    {% if page.tags[0] %}
      <p class="art__tags">
        {% for tag in page.tags %}<a href="{{ base_path }}/tags/#{{ tag | slugify }}" rel="tag">{{ tag }}</a>{% endfor %}
      </p>
    {% endif %}

    {% if page.previous or page.next %}
      <nav class="art__pager">
        {% if page.previous %}
          <a href="{{ base_path }}{{ page.previous.url }}"><small>이전</small>{{ page.previous.title | markdownify | strip_html }}</a>
        {% else %}<span></span>{% endif %}
        {% if page.next %}
          <a class="art__pager-next" href="{{ base_path }}{{ page.next.url }}"><small>다음</small>{{ page.next.title | markdownify | strip_html }}</a>
        {% endif %}
      </nav>
    {% endif %}
  </article>
</div>

<script>
  (function () {
    var heads = document.querySelectorAll('.art__content h2[id]');
    if (heads.length < 2) return;
    var nav = document.getElementById('art-toc');
    heads.forEach(function (h) {
      var a = document.createElement('a');
      a.href = '#' + h.id;
      a.textContent = h.textContent;
      nav.appendChild(a);
    });
    nav.parentNode.hidden = false;
  })();
</script>
```

- [ ] **Step 2: Append article styles to `_sass/_pf.scss`**

Note: code blocks stay light (`--pf-soft`) rather than the ink-dark block in the spec mockup — the bundled Rouge syntax theme is tuned for a light background and would lose contrast on dark.

```scss
/* ---- article --------------------------------------------------------- */
#main.art {
  max-width: 1080px;
  margin: 0 auto;
  padding: 3rem 1.5rem 5rem;
  display: grid;
  grid-template-columns: minmax(0, 680px);
  justify-content: center;
  gap: 3.5rem;
  @media (min-width: 1024px) { grid-template-columns: 200px minmax(0, 680px); }
}
.art__toc {
  display: none;
  @media (min-width: 1024px) {
    display: block;
    position: sticky;
    top: 1.5rem;
    align-self: start;
    font-size: 0.85rem;
  }
  &[hidden] { display: none; }
}
.art__toc-label {
  font-size: 0.7rem;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  color: var(--pf-mute);
  font-weight: 500;
  margin: 0 0 0.6rem;
}
#art-toc a {
  display: block;
  padding: 0.3rem 0 0.3rem 0.8rem;
  border-left: 2px solid var(--pf-line);
  color: var(--pf-mute);
  text-decoration: none;
  line-height: 1.4;
  &:hover { color: var(--pf-ink); border-color: var(--pf-blue); }
}
.art__crumb {
  font-size: 0.8rem;
  font-weight: 500;
  color: var(--pf-blue);
  letter-spacing: 0.04em;
  margin: 0 0 0.9rem;
  a { color: inherit; text-decoration: none; }
}
.art__title {
  font-size: 2.375rem;
  line-height: 1.2;
  letter-spacing: -0.03em;
  margin: 0 0 1rem;
  text-wrap: balance;
  @media (max-width: 600px) { font-size: 1.75rem; }
}
.art__meta {
  display: flex;
  gap: 1.2rem;
  font-size: 0.82rem;
  color: var(--pf-mute);
  padding-bottom: 1.25rem;
  border-bottom: 1px solid var(--pf-line);
  margin: 0 0 1.75rem;
  time { font-family: $pf-mono; color: var(--pf-ink); }
}
.art__content {
  font-size: 0.97rem;
  line-height: 1.85;
  color: #2a2f3a;
  h2 { font-size: 1.375rem; margin: 2.2rem 0 0.75rem; padding: 0; border: 0; }
  h3 { font-size: 1.1rem; margin: 1.6rem 0 0.5rem; }
  h4 { font-size: 1rem; margin: 1.2rem 0 0.4rem; }
  p { margin: 0 0 1.1rem; }
  ul, ol { padding-left: 1.3rem; margin: 0 0 1.1rem; }
  li { margin: 0.2rem 0; }
  a { color: var(--pf-blue); text-decoration: underline; text-underline-offset: 2px; }
  code { font-family: $pf-mono; font-size: 0.85em; background: var(--pf-soft); padding: 0.1em 0.4em; border-radius: 4px; }
  pre {
    font-family: $pf-mono;
    font-size: 0.8rem;
    line-height: 1.7;
    background: var(--pf-soft);
    border: 1px solid var(--pf-line);
    padding: 1rem 1.1rem;
    border-radius: 10px;
    overflow-x: auto;
    margin: 0 0 1.2rem;
    code { background: none; padding: 0; font-size: inherit; }
  }
  div.highlighter-rouge, figure.highlight { border-radius: 10px; margin: 0 0 1.2rem; }
  img { max-width: 100%; height: auto; border-radius: 8px; border: 1px solid var(--pf-line); }
  blockquote {
    margin: 0 0 1.2rem;
    padding: 0.2rem 0 0.2rem 1.1rem;
    border-left: 3px solid var(--pf-blue);
    color: var(--pf-grey);
    font-style: normal;
  }
  table { width: 100%; border-collapse: collapse; font-size: 0.88rem; margin: 0 0 1.2rem; display: block; overflow-x: auto; }
  th, td { padding: 0.55rem 0.75rem; border-bottom: 1px solid var(--pf-line); text-align: left; vertical-align: top; }
  th { font-weight: 600; border-bottom-color: var(--pf-ink); background: none; }
  hr { border: 0; border-top: 1px solid var(--pf-line); margin: 2rem 0; }
}
.art__tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  margin: 2rem 0 0;
  a {
    font-size: 0.75rem;
    border: 1px solid var(--pf-line);
    padding: 0.25rem 0.7rem;
    border-radius: 999px;
    color: var(--pf-grey);
    text-decoration: none;
    &:hover { border-color: var(--pf-ink); color: var(--pf-ink); }
  }
}
.art__pager {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
  margin-top: 2.5rem;
  padding-top: 1.5rem;
  border-top: 1px solid var(--pf-line);
  a {
    text-decoration: none;
    color: var(--pf-ink);
    font-weight: 500;
    font-size: 0.92rem;
    line-height: 1.4;
    small {
      display: block;
      font-size: 0.72rem;
      color: var(--pf-mute);
      letter-spacing: 0.1em;
      text-transform: uppercase;
      margin-bottom: 0.25rem;
    }
  }
  .art__pager-next { text-align: right; }
}
```

- [ ] **Step 3: Drop unused defaults in `_config.yml`**

Under `defaults:`, in the `posts`, `changelog`, `portfolio`, `teaching`, `publications`, and `talks` scopes delete the lines `comments: true`, `comment: true`, `share: true`, `related: true` (keep `layout`, `author_profile`, `read_time`). Use:

```bash
sed -i '' -E '/^      (comments|comment|share|related): true$/d' _config.yml
grep -cE "^      (comments|comment|share|related): true$" _config.yml
```
Expected: `0`.

- [ ] **Step 4: Build and verify**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle exec jekyll build 2>&1 | tail -1
grep -o 'art__crumb"><a href="/changelog/">Changelog</a> · H-MAS' _site/changelog/h-mas/2026-08-02/index.html
POST=$(ls -d _site/posts/2026/*/*/ | head -1); echo "$POST"
grep -o 'art__crumb"><a href="/posts/">Blog' "$POST/index.html" | head -1
grep -c 'art__pager' _site/changelog/h-mas/2026-08-02/index.html
grep -c 'social-share\|page__related\|sidebar__right' _site/changelog/h-mas/2026-08-02/index.html
grep -c 'class="art"' _site/cv/index.html
```
Expected: crumb line, a post path, `art__crumb"><a href="/posts/">Blog`, `1` or more, `0`, `1`.

- [ ] **Step 5: Visual check**

Playwright on `http://localhost:4000/changelog/h-mas/2026-08-02/` at 1280 and 390: TOC visible at 1280 with the h2 list, hidden at 390; body column ≈680px; tags as pills; prev/next row at bottom. Also open `http://localhost:4000/solution/H-MAS/` and scroll — images bordered, tables readable.

- [ ] **Step 6: Commit**

```bash
git add _layouts/single.html _sass/_pf.scss _config.yml
git commit -m "feat: article layout with side TOC"
```

---

### Task 4: Rewrite landing components (`_landing.scss`)

**Files:**
- Rewrite: `_sass/_landing.scss`

Class names are preserved so `hmas.html`, `portfolio.html`, `changelog.html`, `year-archive.html` keep working before they are touched. Three classes are new: `.lp-status`, `.lp-shot`, `.lp-pair`, `.lp-hero--center`.

- [ ] **Step 1: Replace `_sass/_landing.scss` entirely**

```scss
/* ==========================================================================
   LANDING COMPONENTS (.lp-*) — Signal + blue
   Used by splash-layout pages: /, /hmas/, /solution/, /changelog/, /posts/
   ========================================================================== */

.lp {
  color: var(--pf-ink);
  font-size: 16px;
  line-height: 1.7;
  a { text-decoration: none; }
}

/* ---- section scaffolding --------------------------------------------- */
.lp-section {
  padding: 4.5rem 0;
  &--soft { background: var(--pf-soft); }
  @media (max-width: 820px) { padding: 3.25rem 0; }
}
.lp-wrap {
  max-width: 1080px;
  margin: 0 auto;
  padding: 0 1.5rem;
}
.lp-eyebrow {
  text-transform: uppercase;
  letter-spacing: 0.14em;
  font-size: 0.75rem;
  font-weight: 500;
  color: var(--pf-blue);
  margin: 0 0 0.7rem;
}
.lp-h2 {
  font-family: $pf-display;
  font-size: 2rem;
  line-height: 1.2;
  font-weight: 900;
  letter-spacing: -0.03em;
  margin: 0 0 0.8rem;
  text-wrap: balance;
  @media (max-width: 820px) { font-size: 1.625rem; }
}
.lp-sub {
  font-size: 1.05rem;
  color: var(--pf-grey);
  max-width: 680px;
  margin: 0 0 2.4rem;
}

/* ---- hero -------------------------------------------------------------- */
.lp-hero {
  padding: 5.5rem 0 4rem;
  @media (max-width: 820px) { padding: 3.5rem 0 3rem; }
}
.lp-hero__grid {
  display: grid;
  grid-template-columns: 1.05fr 0.95fr;
  gap: 3rem;
  align-items: center;
  @media (max-width: 820px) { grid-template-columns: 1fr; gap: 2rem; }
}
.lp-hero__eyebrow {
  text-transform: uppercase;
  letter-spacing: 0.14em;
  font-size: 0.75rem;
  font-weight: 500;
  color: var(--pf-blue);
  margin: 0 0 1rem;
}
.lp-hero__title {
  font-family: $pf-display;
  font-size: 3.375rem;
  line-height: 1.1;
  font-weight: 900;
  letter-spacing: -0.035em;
  margin: 0 0 1.2rem;
  text-wrap: balance;
  u { text-decoration: none; border: 0; color: var(--pf-blue); }
  @media (max-width: 820px) { font-size: 2.125rem; }
}
.lp-hero__title .lp-line {
  display: inline-block;
  @media (max-width: 560px) { display: inline; }
}
.lp-hero__lead {
  font-size: 1.0625rem;
  line-height: 1.7;
  color: var(--pf-grey);
  max-width: 520px;
  margin: 0 0 1.75rem;
}
.lp-hero--center {
  text-align: center;
  .lp-hero__title, .lp-hero__lead, .lp-sub { margin-left: auto; margin-right: auto; }
  .lp-hero__title { max-width: 820px; }
  .lp-hero__lead { max-width: 620px; }
  .lp-cta, .lp-badges { justify-content: center; }
}
.lp-status {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.82rem;
  color: var(--pf-grey);
  margin: 0 0 1.4rem;
  &::before {
    content: "";
    width: 8px; height: 8px;
    border-radius: 50%;
    background: var(--pf-blue);
    box-shadow: 0 0 0 4px var(--pf-blue-soft);
  }
}
.lp-shot {
  background: var(--pf-bg);
  border: 1px solid var(--pf-line);
  border-radius: 14px;
  padding: 8px;
  box-shadow: 0 30px 60px -32px rgba(22, 24, 29, 0.35);
  img { display: block; width: 100%; height: auto; border-radius: 8px; }
}
.lp-hero__img { @extend .lp-shot; }
.lp-badges {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 1.4rem;
}
.lp-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  background: var(--pf-soft);
  border: 1px solid var(--pf-line);
  color: var(--pf-grey);
  padding: 0.35rem 0.8rem;
  border-radius: 999px;
  font-size: 0.78rem;
  font-weight: 500;
}

/* ---- buttons ----------------------------------------------------------- */
.lp-cta {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6rem;
}
.lp-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.8rem 1.4rem;
  border-radius: 999px;
  font-weight: 500;
  font-size: 0.9rem;
  line-height: 1;
  text-decoration: none !important;
  transition: background 0.12s ease, border-color 0.12s ease;
}
.lp-btn--primary, .lp-btn--solid {
  background: var(--pf-blue);
  color: #fff !important;
  &:hover { background: #042a9c; }
}
.lp-btn--ghost, .lp-btn--line {
  background: var(--pf-bg);
  color: var(--pf-ink) !important;
  border: 1px solid #cfd1d6;
  &:hover { border-color: var(--pf-ink); }
}

/* ---- cards ------------------------------------------------------------- */
.lp-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1.75rem;
  @media (max-width: 860px) { grid-template-columns: repeat(2, 1fr); }
  @media (max-width: 560px) { grid-template-columns: 1fr; }
}
.lp-cards--2 {
  grid-template-columns: repeat(2, 1fr);
  @media (max-width: 720px) { grid-template-columns: 1fr; }
}
.lp-cards--4 {
  grid-template-columns: repeat(4, 1fr);
  @media (max-width: 860px) { grid-template-columns: repeat(2, 1fr); }
  @media (max-width: 480px) { grid-template-columns: 1fr; }
}
.lp-card {
  display: block;
  padding: 1.25rem 0 0;
  border-top: 2px solid var(--pf-ink);
  color: inherit;
}
.lp-card__icon { display: none; }
.lp-card__title {
  font-family: $pf-display;
  font-size: 1.2rem;
  font-weight: 700;
  letter-spacing: -0.02em;
  margin: 0 0 0.6rem;
}
.lp-card__body {
  font-size: 0.9rem;
  line-height: 1.7;
  color: var(--pf-grey);
  margin: 0;
}
.lp-card__more {
  display: inline-block;
  margin-top: 0.9rem;
  font-size: 0.85rem;
  font-weight: 500;
  color: var(--pf-blue);
}

/* product card (thumbnail + body) */
.lp-card--product {
  padding: 0;
  border: 1px solid var(--pf-line);
  border-radius: 14px;
  overflow: hidden;
  background: var(--pf-bg);
  transition: border-color 0.12s ease;
  &:hover { border-color: var(--pf-ink); }
}
.lp-card__thumb {
  aspect-ratio: 16 / 9;
  overflow: hidden;
  background: var(--pf-soft);
  border-bottom: 1px solid var(--pf-line);
  img { width: 100%; height: 100%; object-fit: cover; object-position: top; display: block; }
}
.lp-card__pad { padding: 1.4rem 1.5rem 1.6rem; }

/* ---- stats ------------------------------------------------------------- */
.lp-stats {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1.75rem;
  @media (max-width: 720px) { grid-template-columns: repeat(2, 1fr); }
}
.lp-stat {
  padding: 1.25rem 0 0;
  border-top: 2px solid var(--pf-ink);
}
.lp-stat__num {
  font-family: $pf-display;
  font-size: 2.5rem;
  line-height: 1;
  font-weight: 900;
  color: var(--pf-blue);
  letter-spacing: -0.035em;
  margin-bottom: 0.6rem;
}
.lp-stat__label {
  font-size: 0.9rem;
  color: var(--pf-grey);
}

/* ---- figure / screenshots --------------------------------------------- */
.lp-figure { margin: 0; text-align: center; }
.lp-figure img {
  max-width: 100%;
  height: auto;
  border-radius: 10px;
  border: 1px solid var(--pf-line);
}
.lp-figure figcaption {
  font-size: 0.82rem;
  color: var(--pf-mute);
  margin-top: 0.7rem;
}
.lp-shots {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1.5rem;
  align-items: start;
  @media (max-width: 820px) { grid-template-columns: 1fr; }
  .lp-figure img {
    width: 100%;
    aspect-ratio: 16 / 9;
    object-fit: cover;
    object-position: top left;
    display: block;
  }
}
.lp-pair {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem;
  @media (max-width: 720px) { grid-template-columns: 1fr; }
}

/* ---- split / lists ------------------------------------------------------ */
.lp-split {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2.5rem;
  align-items: center;
  @media (max-width: 820px) { grid-template-columns: 1fr; gap: 1.5rem; }
}
.lp-vlist { display: flex; flex-direction: column; }
.lp-vitem {
  padding: 0.9rem 0;
  border-bottom: 1px solid var(--pf-line);
  &:first-child { border-top: 1px solid var(--pf-ink); }
  b { display: block; font-size: 0.97rem; }
  span { font-size: 0.88rem; color: var(--pf-grey); }
  &--accent b { color: var(--pf-blue); }
}
.lp-tag {
  display: inline-block;
  font-size: 0.68rem;
  font-weight: 500;
  color: var(--pf-blue);
  background: var(--pf-blue-soft);
  padding: 0.1rem 0.5rem;
  border-radius: 999px;
  vertical-align: middle;
}
.lp-check { list-style: none; padding: 0; margin: 0; }
.lp-check li {
  position: relative;
  padding-left: 1.6rem;
  margin-bottom: 0.7rem;
  font-size: 0.95rem;
  color: var(--pf-grey);
  b { color: var(--pf-ink); }
  &::before {
    content: "✓";
    position: absolute; left: 0; top: 0;
    color: var(--pf-blue);
    font-weight: 700;
  }
}
.lp-note { font-size: 0.9rem; color: var(--pf-mute); margin-top: 1.2rem; }
.lp-quote {
  margin-top: 2.4rem;
  padding: 0 0 0 1.25rem;
  border-left: 3px solid var(--pf-blue);
  font-size: 1.05rem;
  font-weight: 500;
  color: var(--pf-ink);
}
.lp-quote__src {
  display: block;
  margin-top: 0.5rem;
  font-size: 0.85rem;
  font-weight: 400;
  color: var(--pf-mute);
}

/* ---- steps ------------------------------------------------------------- */
.lp-steps {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1rem;
  margin-bottom: 2rem;
  @media (max-width: 700px) { grid-template-columns: repeat(2, 1fr); }
}
.lp-step {
  display: flex;
  align-items: center;
  gap: 0.7rem;
  padding: 0.9rem 0;
  border-top: 2px solid var(--pf-line);
  font-weight: 500;
  font-size: 0.92rem;
  &--accent { border-top-color: var(--pf-blue); }
}
.lp-step__n {
  flex: none;
  width: 24px; height: 24px;
  display: flex; align-items: center; justify-content: center;
  border-radius: 50%;
  background: var(--pf-ink);
  color: #fff;
  font-size: 0.75rem;
  font-family: $pf-mono;
  .lp-step--accent & { background: var(--pf-blue); }
}

/* ---- listing rows (blog / changelog) ------------------------------------ */
.lp-year {
  font-family: $pf-display;
  font-size: 1.25rem;
  font-weight: 700;
  letter-spacing: -0.02em;
  margin: 2.5rem 0 0;
  padding: 0 0 0.6rem;
  border-bottom: 1px solid var(--pf-ink);
  &:first-child { margin-top: 0; }
}
.lp-post {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 1.5rem;
  padding: 1rem 0;
  border-bottom: 1px solid var(--pf-line);
  color: inherit;
  &:first-of-type { border-top: 1px solid var(--pf-ink); }   /* list without a .lp-year header (home) */
  &:hover .lp-post__title { color: var(--pf-blue); }
  @media (max-width: 600px) { flex-direction: column; gap: 0.35rem; }
}
.lp-year + .lp-post { border-top: 0; }                         /* header already draws the rule */
.lp-post__title {
  font-family: $pf-body;
  font-size: 1rem;
  font-weight: 500;
  letter-spacing: 0;
  margin: 0;
  color: var(--pf-ink);
  transition: color 0.12s ease;
}
.lp-post__excerpt {
  font-size: 0.86rem;
  color: var(--pf-mute);
  margin: 0.3rem 0 0;
}
.lp-post__meta {
  flex: none;
  display: flex;
  gap: 0.8rem;
  font-size: 0.8rem;
  color: var(--pf-mute);
  white-space: nowrap;
}
.lp-post__date { font-family: $pf-mono; }
.lp-post__tag { color: var(--pf-blue); font-weight: 500; }

/* ---- misc ----------------------------------------------------------------- */
.lp-nowrap { white-space: nowrap; }
.lp-final {
  text-align: center;
  border-top: 1px solid var(--pf-line);
  .lp-sub { margin-left: auto; margin-right: auto; }
  .lp-cta { justify-content: center; }
}
```

- [ ] **Step 2: Build and verify no undefined variables**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle exec jekyll build 2>&1 | tail -2
grep -o "\.lp-status::before" _site/assets/css/main.css | head -1
grep -c "linear-gradient(140deg" _site/assets/css/main.css
```
Expected: `done in …` (no `Undefined variable`), `.lp-status::before`, `0`.

- [ ] **Step 3: Visual check**

Playwright `http://localhost:4000/changelog/` and `/solution/` at 1280: rows with hairlines, section headers with ink underline, product cards white with border. `/hmas/` will look half-migrated (light hero on old markup) — that is expected until Task 6.

- [ ] **Step 4: Commit**

```bash
git add _sass/_landing.scss
git commit -m "style: rewrite landing components for Signal + blue"
```

---

### Task 5: Home page

**Files:**
- Rewrite: `_pages/index.html`

- [ ] **Step 1: Replace `_pages/index.html`**

```html
---
layout: splash
permalink: /
title: "parameterfreak — AI 시스템을 설계하고 만듭니다"
description: "온프레미스 AI 서빙 플랫폼 H-MAS, 임베딩·이상탐지 연구, 그리고 이를 제품으로 만드는 AI 소프트웨어. parameterfreak은 AI 시스템을 직접 설계하고 만듭니다."
author_profile: false
redirect_from:
  - /about/
  - /about.html
---

{% include base_path %}

<style>
#main { max-width: none !important; padding: 0 !important; margin-top: 0 !important; }
.page__content { margin: 0 !important; }
</style>

<div class="lp">

  <!-- ===================== HERO ===================== -->
  <section class="lp-section lp-hero">
    <div class="lp-wrap">
      <div class="lp-hero__grid">
        <div>
          <p class="lp-status">H-MAS v0.7 Feature Preview 출시</p>
          <h1 class="lp-hero__title">AI 시스템을<br/>설계하고, <u>만듭니다.</u></h1>
          <p class="lp-hero__lead">
            GPU를 묶어 AI를 서빙하는 플랫폼, 임베딩·이상탐지 연구,
            그리고 그 기술을 담은 AI 소프트웨어까지. 탐구에 그치지 않고 실제 동작하는 제품으로 만듭니다.
          </p>
          <div class="lp-cta">
            <a class="lp-btn lp-btn--primary" href="{{ base_path }}/hmas/">H-MAS 알아보기</a>
            <a class="lp-btn lp-btn--ghost" href="{{ base_path }}/solution/">제품 전체</a>
          </div>
        </div>
        <div class="lp-shot">
          <img src="{{ base_path }}/images/portfolio/hmas/v0.7/02-dashboard.png" alt="H-MAS 대시보드" width="1600" height="900"/>
        </div>
      </div>
    </div>
  </section>

  <!-- ===================== WHAT WE DO ===================== -->
  <section class="lp-section">
    <div class="lp-wrap">
      <p class="lp-eyebrow">What we do</p>
      <h2 class="lp-h2">세 가지 축</h2>
      <p class="lp-sub">플랫폼을 만들고, 모델을 연구하고, 소프트웨어로 묶습니다.</p>
      <div class="lp-cards">
        <a class="lp-card" href="{{ base_path }}/hmas/">
          <h3 class="lp-card__title">AI 서빙 플랫폼</h3>
          <p class="lp-card__body">흩어진 GPU 서버를 하나로 묶어 모델을 최적의 하드웨어에 자동 배치하는 온프레미스 서빙 플랫폼 H-MAS. 멀티 클러스터 스케줄링, 통합 추론 API, GPU 활용률 최적화.</p>
          <span class="lp-card__more">H-MAS →</span>
        </a>
        <a class="lp-card" href="{{ base_path }}/posts/">
          <h3 class="lp-card__title">임베딩 · 이상탐지 연구</h3>
          <p class="lp-card__body">도메인 특화 임베딩 모델(법령 LexEM 등)과 검색·RAG 아키텍처, 그리고 시계열·로그 기반 이상탐지. 연구 노트는 블로그에 공개합니다.</p>
          <span class="lp-card__more">연구 노트 →</span>
        </a>
        <a class="lp-card" href="{{ base_path }}/solution/">
          <h3 class="lp-card__title">AI 소프트웨어</h3>
          <p class="lp-card__body">연구와 플랫폼 위에 올리는 응용 소프트웨어. AI 문서 생성 PaperOps, RAG 데이터셋 버전 관리 DataProc, MCP 도구 등.</p>
          <span class="lp-card__more">제품 전체 →</span>
        </a>
      </div>
    </div>
  </section>

  <!-- ===================== CHANGELOG ===================== -->
  <section class="lp-section lp-section--soft">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Changelog</p>
      <h2 class="lp-h2">최근 릴리스</h2>
      {% assign notes = site.changelog | sort: "date" | reverse %}
      {% for post in notes limit: 4 %}
        <a class="lp-post" href="{{ base_path }}{{ post.url }}">
          <h3 class="lp-post__title">{{ post.title }}</h3>
          <div class="lp-post__meta">
            <span class="lp-post__tag">{{ post.categories[0] }}</span>
            <span class="lp-post__date">{{ post.date | date: "%Y.%m.%d" }}</span>
          </div>
        </a>
      {% endfor %}
      <p style="margin:1.5rem 0 0;"><a class="lp-card__more" href="{{ base_path }}/changelog/">전체 Changelog →</a></p>
    </div>
  </section>

  <!-- ===================== BLOG ===================== -->
  <section class="lp-section">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Blog</p>
      <h2 class="lp-h2">최근 글</h2>
      {% for post in site.posts limit: 3 %}
        <a class="lp-post" href="{{ base_path }}{{ post.url }}">
          <div>
            <h3 class="lp-post__title">{{ post.title | markdownify | remove: "<p>" | remove: "</p>" }}</h3>
            {% if post.excerpt %}
              <p class="lp-post__excerpt">{{ post.excerpt | markdownify | strip_html | strip_newlines | truncate: 120 }}</p>
            {% endif %}
          </div>
          <div class="lp-post__meta"><span class="lp-post__date">{{ post.date | date: "%Y.%m.%d" }}</span></div>
        </a>
      {% endfor %}
      <p style="margin:1.5rem 0 0;"><a class="lp-card__more" href="{{ base_path }}/posts/">블로그 →</a></p>
    </div>
  </section>

  <!-- ===================== CONTACT ===================== -->
  <section class="lp-section lp-final">
    <div class="lp-wrap">
      <h2 class="lp-h2">함께 이야기해요</h2>
      <p class="lp-sub">제품 도입, 연구 협업, 무엇이든 편하게 연락 주세요.</p>
      <div class="lp-cta">
        <a class="lp-btn lp-btn--line" href="{{ base_path }}/solution/">제품 보기</a>
        <a class="lp-btn lp-btn--solid" href="mailto:contact@parameterfreak.com">contact@parameterfreak.com</a>
      </div>
    </div>
  </section>

</div>
```

- [ ] **Step 2: Build and verify**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle exec jekyll build 2>&1 | tail -1
grep -o 'v0.7/02-dashboard.png' _site/index.html | wc -l
grep -o 'href="/changelog/[a-z-]*/[0-9-]*/"' _site/index.html | wc -l
grep -o 'href="/posts/20[^"]*"' _site/index.html | wc -l
grep -c 'FLAGSHIP\|lp-card__icon' _site/index.html
```
Expected: `done in …`, `1`, `4`, `3`, `0`.

- [ ] **Step 3: Visual check**

Playwright `http://localhost:4000/` at 1280 and 390. Compare with the v3 mockup: status dot line, "만듭니다." in blue, screenshot framed on the right (below text at 390), three cards with ink top border, four changelog rows with blue product tag and mono date, three blog rows.

- [ ] **Step 4: Commit**

```bash
git add _pages/index.html
git commit -m "feat: redesign home page"
```

---

### Task 6: H-MAS landing

**Files:**
- Rewrite: `_pages/hmas.html`

Content, section order and links are unchanged. Markup changes: centered hero with `.lp-hero--center`, one large screenshot below the hero, emoji icons removed, v0.7 screenshots, a light/dark pair in the RELIABILITY section, `.lp-final` now light.

- [ ] **Step 1: Replace `_pages/hmas.html`**

```html
---
layout: splash
title: "H-MAS — 온프레미스 AI 서빙 플랫폼"
permalink: /hmas/
excerpt: "흩어진 GPU 서버를 하나로 묶어, AI 모델을 최적의 하드웨어에 자동 배치하는 온프레미스 AI 서빙 플랫폼"
description: "H-MAS는 흩어진 GPU 서버를 하나의 컨트롤 플레인으로 묶어 AI 모델을 최적의 하드웨어에 자동 배치하는 온프레미스 AI 서빙 플랫폼입니다. 모델 서빙, 통합 추론(인퍼런스) API, GPU 클러스터 관리, 멀티 클러스터 스케줄링, 실시간 모니터링, 자동 장애 복구를 제공합니다."
keywords: "H-MAS, AI 서빙 플랫폼, 모델 서빙, model serving, AI 추론, 인퍼런스, inference, LLM 서빙, GPU 관리, GPU 클러스터, GPU 스케줄링, GPU 활용률, 온프레미스 AI, on-premise LLM, 멀티 클러스터 AI 인프라"
header:
  og_image: portfolio/hmas/v0.7/02-dashboard.png
---

{% include base_path %}

<style>
#main { max-width: none !important; padding: 0 !important; margin-top: 0 !important; }
.page__content { margin: 0 !important; }
</style>

<div class="lp">

  <!-- ===================== HERO ===================== -->
  <section class="lp-section lp-hero lp-hero--center">
    <div class="lp-wrap">
      <p class="lp-hero__eyebrow">H-MAS · Hardware-aware Multi-Cluster AI Serving Platform</p>
      <h1 class="lp-hero__title"><span class="lp-line">흩어진 GPU 서버를 하나로 묶어,</span> <span class="lp-line">AI 모델을 <u>최적의 하드웨어에</u> 자동 배치</span></h1>
      <p class="lp-hero__lead">
        클라우드 없이도 AI를 가장 효율적으로 서빙하는 온프레미스 플랫폼.
        복잡한 설정 없이, 웹 콘솔에서 모델을 선택하면 끝납니다.
      </p>
      <div class="lp-cta">
        <a class="lp-btn lp-btn--primary" href="{{ base_path }}/solution/H-MAS/">자세한 제품 문서 →</a>
        <a class="lp-btn lp-btn--ghost" href="https://docs.google.com/forms/d/e/1FAIpQLSexMhx10zbFTWJF8daSFhOPLjQ7vEryVbIT21unpvphVVuAaQ/viewform" target="_blank" rel="noopener">PoC·데모 문의</a>
      </div>
      <div class="lp-badges">
        <span class="lp-badge">v0.7 Feature Preview 출시</span>
        <span class="lp-badge">저작권 등록 C-2026-020641</span>
        <span class="lp-badge">토폴로지 인식 스케줄링</span>
      </div>
      <div class="lp-shot" style="max-width:960px;margin:3rem auto 0;">
        <img src="{{ base_path }}/images/portfolio/hmas/v0.7/02-dashboard.png" alt="H-MAS 통합 대시보드" width="1600" height="900"/>
      </div>
    </div>
  </section>

  <!-- ===================== 문제 ===================== -->
  <section class="lp-section">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Problem</p>
      <h2 class="lp-h2">"배보다 배꼽이 큰" 서빙 비용</h2>
      <p class="lp-sub">AI가 '구축'에서 '서비스'로 넘어가며, 서빙 비용이 새로운 병목이 됐습니다.</p>
      <div class="lp-stats" style="grid-template-columns:repeat(3,1fr);">
        <div class="lp-stat">
          <div class="lp-stat__num">80~90%</div>
          <p class="lp-card__body">AI 시스템 생애주기 비용이 학습이 아니라 <b>서빙</b>에서 발생 — 학습은 한 번, 서빙은 365일 24시간 돌아갑니다. <span style="opacity:.7;">(Stanford AI Index 2025)</span></p>
        </div>
        <div class="lp-stat">
          <div class="lp-stat__num">4만 원↑</div>
          <p class="lp-card__body">시간당 클라우드 GPU 단가. 서비스를 키울수록 서빙 인프라 비용이 서비스 자체보다 커집니다. 금융·공공은 보안 규정상 클라우드도 불가 → <b>자체 GPU 강제</b>.</p>
        </div>
        <div class="lp-stat">
          <div class="lp-stat__num">25%</div>
          <p class="lp-card__body">GPU 활용률. 자기 GPU를 사도 제대로 쓸 SW가 없습니다. 기본 K8s는 GPU 물리 구조(NVLink/PCIe/NUMA)를 몰라 병목이 생기고, <b>비싼 GPU의 3/4이 놀고 있습니다</b>.</p>
        </div>
      </div>
      <div class="lp-quote">
        "웹 화면에서 모델을 띄우고, GPU 점유율을 보고, 미사용 모델은 자동 해제하고, 장애 나면 알림 받고 싶다."
        <span class="lp-quote__src">— GPU 10~20대 보유 기업들의 반복 문의. 수요는 이미 현장에서 들어오고 있습니다.</span>
      </div>
    </div>
  </section>

  <!-- ===================== 솔루션 ===================== -->
  <section class="lp-section lp-section--soft">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Solution</p>
      <h2 class="lp-h2">GPU를 묶고, 자동 배치하는 서빙 인프라</h2>
      <p class="lp-sub">흩어진 GPU 서버를 하나의 컨트롤 플레인으로 묶고, 모델을 최적의 하드웨어에 자동 배치합니다.</p>
      <div class="lp-split">
        <figure class="lp-figure">
          <img src="{{ base_path }}/images/portfolio/hmas/pitch/architecture.png" alt="H-MAS 시스템 아키텍처"/>
          <figcaption>Management Cluster가 HP/GP/Standard 클러스터로 자동 분배</figcaption>
        </figure>
        <div class="lp-vlist">
          <div class="lp-vitem"><b>통합 관리</b><span>여러 GPU 서버를 하나의 컨트롤 플레인으로 통합</span></div>
          <div class="lp-vitem"><b>자동 배치</b><span>모델 크기별 고성능(HP)/가성비(GP) 클러스터로 트래픽 자동 분배</span></div>
          <div class="lp-vitem"><b>하드웨어 인지 <span class="lp-tag">v0.9 로드맵</span></b><span>NVLink/PCIe/NUMA 토폴로지 인식, 통신 병목 최소화</span></div>
          <div class="lp-vitem"><b>고가용성</b><span>장애 시 다른 클러스터로 자동 Failover</span></div>
          <div class="lp-vitem lp-vitem--accent"><b>활용률 극대화</b><span>GPU 활용률 25% → 90%+ 목표</span></div>
        </div>
      </div>
    </div>
  </section>

  <!-- ===================== 동작 ===================== -->
  <section class="lp-section">
    <div class="lp-wrap">
      <p class="lp-eyebrow">How it works</p>
      <h2 class="lp-h2">웹 콘솔에서 모델만 고르면 끝</h2>
      <p class="lp-sub">관리자는 선택만, 나머지는 H-MAS가 자동으로 처리합니다.</p>
      <div class="lp-steps">
        <div class="lp-step"><span class="lp-step__n">1</span><span class="lp-step__t">모델 선택</span></div>
        <div class="lp-step"><span class="lp-step__n">2</span><span class="lp-step__t">정책 자동 추천</span></div>
        <div class="lp-step"><span class="lp-step__n">3</span><span class="lp-step__t">배포</span></div>
        <div class="lp-step lp-step--accent"><span class="lp-step__n">4</span><span class="lp-step__t">실시간 대시보드</span></div>
      </div>
      <div class="lp-shots">
        <figure class="lp-figure"><img src="{{ base_path }}/images/portfolio/hmas/v0.7/04-models.png" alt="모델 저장소"/><figcaption>① 모델 저장소 — 모델·런타임 선택</figcaption></figure>
        <figure class="lp-figure"><img src="{{ base_path }}/images/portfolio/hmas/v0.7/05-deploy.png" alt="배포 마법사"/><figcaption>② 배포 마법사 — 4단계 배포</figcaption></figure>
        <figure class="lp-figure"><img src="{{ base_path }}/images/portfolio/hmas/v0.7/08-monitoring.png" alt="실시간 모니터링"/><figcaption>③ 실시간 모니터링 — 메트릭 + Pod 로그</figcaption></figure>
      </div>
    </div>
  </section>

  <!-- ===================== 차별점 ===================== -->
  <section class="lp-section lp-section--soft">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Why us</p>
      <h2 class="lp-h2">서빙 전용 + 온프레미스 + 토폴로지 인지</h2>
      <p class="lp-sub">서빙 인프라가 갖춰야 할 네 가지를 H-MAS가 모두 충족합니다.</p>
      <div class="lp-cards lp-cards--4">
        <div class="lp-card">
          <h3 class="lp-card__title">서빙 전용</h3>
          <p class="lp-card__body">학습이 아닌 서빙에 최적화된 설계.</p>
        </div>
        <div class="lp-card">
          <h3 class="lp-card__title">온프레미스</h3>
          <p class="lp-card__body">클라우드를 못 쓰는 금융·공공·제조 환경에 그대로 설치.</p>
        </div>
        <div class="lp-card">
          <h3 class="lp-card__title">토폴로지 인지 <span class="lp-tag">v0.9 로드맵</span></h3>
          <p class="lp-card__body">GPU 연결 구조를 인식해 통신 병목을 최소화하고 성능을 끌어올립니다.</p>
        </div>
        <div class="lp-card">
          <h3 class="lp-card__title">멀티 클러스터</h3>
          <p class="lp-card__body">흩어진 GPU 서버를 하나의 컨트롤 플레인으로 통합.</p>
        </div>
      </div>
      <p class="lp-note" style="text-align:center;">
        GPU 3~20대를 보유한 중소·스타트업·연구소를 위한 가볍고 빠른 도입.
        기존 추론 엔진(vLLM 등) 위에서 동작하는 <b>인프라 관리 레이어</b>입니다.
      </p>
    </div>
  </section>

  <!-- ===================== 트랙션 ===================== -->
  <section class="lp-section">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Reliability</p>
      <h2 class="lp-h2">바로 도입할 수 있는 검증된 제품</h2>
      <p class="lp-sub">컨셉이 아니라, 모델 등록부터 서빙·모니터링까지 실제로 동작하는 제품입니다.</p>
      <div class="lp-split">
        <div>
          <ul class="lp-check">
            <li><b>검증된 배포 파이프라인</b> — 모델 등록 → 배포 → API 호출까지 멀티 클러스터에서 안정 동작</li>
            <li><b>통합 추론 API</b> — 모든 모델을 하나의 OpenAI 호환 엔드포인트로 호출</li>
            <li><b>실시간 모니터링</b> — Prometheus 기반 TPS·지연시간·GPU 메트릭 제공</li>
            <li><b>운영 안정성</b> — 이미지 사전 캐싱으로 재배포 시간 단축, 장애 자동 감지</li>
            <li><b>라이트·다크 모드</b> — 운영 환경에 맞는 웹 콘솔 테마</li>
            <li><b>지식재산권 확보</b> — 컴퓨터프로그램 저작권 등록 (한국저작권위원회 C-2026-020641)</li>
          </ul>
          <div class="lp-stats" style="grid-template-columns:repeat(2,1fr);margin-top:1.75rem;">
            <div class="lp-stat"><div class="lp-stat__num">5종+</div><div class="lp-stat__label">지원 서빙 런타임</div></div>
            <div class="lp-stat"><div class="lp-stat__num">OpenAI</div><div class="lp-stat__label">호환 통합 추론 API</div></div>
          </div>
        </div>
        <figure class="lp-figure">
          <img src="{{ base_path }}/images/portfolio/hmas/copyright-2604.png" alt="컴퓨터프로그램 저작권 등록증"/>
          <figcaption>컴퓨터프로그램 저작권 등록증 — 제 C-2026-020641 호</figcaption>
        </figure>
      </div>
      <div class="lp-pair" style="margin-top:2.5rem;">
        <figure class="lp-figure"><img src="{{ base_path }}/images/portfolio/hmas/v0.7/02-dashboard.png" alt="대시보드 라이트 모드"/><figcaption>라이트 모드</figcaption></figure>
        <figure class="lp-figure"><img src="{{ base_path }}/images/portfolio/hmas/v0.7/02-dashboard-dark.png" alt="대시보드 다크 모드"/><figcaption>다크 모드</figcaption></figure>
      </div>
      <div class="lp-quote">보유하신 GPU 환경에 맞춰 PoC·데모를 도와드립니다. 실제 동작을 직접 확인해 보세요.</div>
    </div>
  </section>

  <!-- ===================== 시장 ===================== -->
  <section class="lp-section lp-section--soft">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Who it's for</p>
      <h2 class="lp-h2">이런 분들께 H-MAS가 필요합니다</h2>
      <p class="lp-sub">AI 비용의 대부분은 학습이 아니라 서빙에서 발생합니다. 자체 GPU를 두고도 제대로 활용하지 못하고 있다면.</p>
      <div class="lp-cards">
        <div class="lp-card">
          <h3 class="lp-card__title">자체 GPU를 보유한 기업</h3>
          <p class="lp-card__body">GPU 서버 3~20대를 두고 있지만 개별 운영되어 활용률이 낮은 AI 스타트업·중소기업.</p>
        </div>
        <div class="lp-card">
          <h3 class="lp-card__title">클라우드를 못 쓰는 환경</h3>
          <p class="lp-card__body">보안 규정상 클라우드 사용이 불가해 자체 GPU + 온프레미스 서빙이 필요한 금융·공공·제조.</p>
        </div>
        <div class="lp-card">
          <h3 class="lp-card__title">이기종 GPU 연구 조직</h3>
          <p class="lp-card__body">A100·RTX 등 여러 종류의 GPU를 묶어 모델별로 효율적으로 서빙하고 싶은 연구소·대학.</p>
        </div>
      </div>
    </div>
  </section>

  <!-- ===================== 도입 방식 ===================== -->
  <section class="lp-section">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Deployment</p>
      <h2 class="lp-h2">보유하신 환경에 직접 설치합니다</h2>
      <p class="lp-sub">기존에 운영 중인 Kubernetes 환경 위에 온프레미스로 설치합니다. 데이터와 모델은 모두 고객 인프라 안에 머뭅니다.</p>
      <div class="lp-cards">
        <div class="lp-card">
          <h3 class="lp-card__title">온프레미스 구축형</h3>
          <p class="lp-card__body">고객사 GPU 인프라에 직접 설치·운영. 외부로 데이터가 나가지 않아 보안 규정이 엄격한 환경에도 적합합니다.</p>
        </div>
        <div class="lp-card">
          <h3 class="lp-card__title">합리적인 운영 비용</h3>
          <p class="lp-card__body">이미 보유한 자체 GPU를 효율적으로 활용해, 시간당 과금되는 클라우드 GPU 대비 운영 비용 부담을 낮춥니다.</p>
        </div>
        <div class="lp-card">
          <h3 class="lp-card__title">PoC부터 시작</h3>
          <p class="lp-card__body">보유하신 GPU 환경에 맞춰 PoC·데모를 먼저 진행합니다. 도입 규모와 조건은 환경에 맞춰 협의해 드립니다.</p>
        </div>
      </div>
    </div>
  </section>

  <!-- ===================== 비전 / CTA ===================== -->
  <section class="lp-section lp-final">
    <div class="lp-wrap">
      <p class="lp-eyebrow">Vision</p>
      <h2 class="lp-h2">클라우드 없이도, AI를 가장 효율적으로</h2>
      <p class="lp-sub">자체 GPU를 가진 모든 조직이 가장 효율적으로 AI를 서빙하는 표준을 만듭니다.<br/>보유하신 GPU 환경에 맞춰 <span class="lp-nowrap">PoC·데모를 도와드립니다.</span></p>
      <div class="lp-cta">
        <a class="lp-btn lp-btn--line" href="{{ base_path }}/solution/H-MAS/">자세한 제품 문서 보기</a>
        <a class="lp-btn lp-btn--solid" href="https://docs.google.com/forms/d/e/1FAIpQLSexMhx10zbFTWJF8daSFhOPLjQ7vEryVbIT21unpvphVVuAaQ/viewform" target="_blank" rel="noopener">PoC·데모 문의하기</a>
      </div>
      <p class="lp-note">또는 이메일로 문의: <a href="mailto:contact@parameterfreak.com">contact@parameterfreak.com</a></p>
    </div>
  </section>

</div>
```

- [ ] **Step 2: Build and verify**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle exec jekyll build 2>&1 | tail -1
grep -o 'pitch/[a-z-]*\.png' _site/hmas/index.html | sort -u
grep -o 'v0.7/[0-9a-z-]*\.png' _site/hmas/index.html | sort -u
grep -c 'lp-card__icon' _site/hmas/index.html
grep -c 'lp-hero--center' _site/hmas/index.html
```
Expected: `done in …`; pitch list = `pitch/architecture.png` only; v0.7 list = `02-dashboard-dark.png 02-dashboard.png 04-models.png 05-deploy.png 08-monitoring.png`; `0`; `1`.

- [ ] **Step 3: Visual check**

Playwright `http://localhost:4000/hmas/` at 1280 and 390, full-page screenshot. Confirm: centered hero, "최적의 하드웨어에" blue, big dashboard shot below badges, three stat numbers in blue with ink top rule, three-shot grid becomes one column at 390, light/dark pair, light final CTA section.

- [ ] **Step 4: Commit**

```bash
git add _pages/hmas.html
git commit -m "feat: restyle H-MAS landing with v0.7 screenshots"
```

---

### Task 7: Listing pages, product doc, og_image

**Files:**
- Modify: `_pages/portfolio.html`
- Modify: `_pages/changelog.html`
- Modify: `_pages/year-archive.html`
- Modify: `_portfolio/H-MAS.md`
- Modify: `_config.yml:og_image`

- [ ] **Step 1: Changelog rows — show read time after date, product label already in section header**

In `_pages/changelog.html` replace the `<div class="lp-post__meta">…</div>` block (3 lines) with:

```html
              <div class="lp-post__meta">
                <span class="lp-post__date">{{ post.date | date: "%Y.%m.%d" }}</span>
              </div>
```

Leave the `<div class="lp-post__main">` wrapper as is. Change the eyebrow text `<p class="lp-hero__eyebrow">CHANGELOG</p>` → `<p class="lp-hero__eyebrow">Changelog</p>`.

- [ ] **Step 2: Blog listing — same meta simplification**

In `_pages/year-archive.html` replace the `<div class="lp-post__meta">…</div>` block with:

```html
          <div class="lp-post__meta">
            <span class="lp-post__date">{{ post.date | date: "%Y.%m.%d" }}</span>
          </div>
```

and `<p class="lp-hero__eyebrow">BLOG</p>` → `<p class="lp-hero__eyebrow">Blog</p>`.

- [ ] **Step 3: Products page — eyebrow case**

In `_pages/portfolio.html`: `<p class="lp-hero__eyebrow">PRODUCTS</p>` → `<p class="lp-hero__eyebrow">Products</p>`.

- [ ] **Step 4: H-MAS product doc — teaser and screenshot paths**

In `_portfolio/H-MAS.md`:
- front matter `teaser: portfolio/hmas/pitch/dashboard.png` → `teaser: portfolio/hmas/v0.7/02-dashboard.png`
- body image paths:

```bash
sed -i '' \
  -e 's|pitch/dashboard.png|v0.7/02-dashboard.png|g' \
  -e 's|pitch/deploy-form.png|v0.7/05-deploy.png|g' \
  -e 's|pitch/monitoring.png|v0.7/08-monitoring.png|g' \
  -e 's|pitch/instances.png|v0.7/06-instances.png|g' \
  -e 's|pitch/instance-detail.png|v0.7/07-instance-detail.png|g' \
  -e 's|pitch/model-registry.png|v0.7/04-models.png|g' \
  _portfolio/H-MAS.md
grep -o 'pitch/[a-z-]*\.png' _portfolio/H-MAS.md | sort -u
```
Expected remaining pitch images: `architecture.png cluster-classification.png deploy-workflow.png gpu-topology.png runtime-compatibility.png` (diagrams, not UI — keep).

- [ ] **Step 5: `_config.yml` og_image**

`og_image : "portfolio/hmas/pitch/dashboard.png"` → `og_image : "portfolio/hmas/v0.7/02-dashboard.png"`.

- [ ] **Step 6: Build and verify**

```bash
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"
bundle exec jekyll build 2>&1 | tail -1
grep -o 'v0.7/02-dashboard.png' _site/solution/index.html | wc -l
grep -o 'og:image" content="[^"]*' _site/changelog/index.html
grep -o 'class="lp-post"' _site/changelog/index.html | wc -l
grep -o 'class="lp-post"' _site/posts/index.html | wc -l
```
Expected: `done in …`, `1`, `og:image" content="https://parameterfreak.com/images/portfolio/hmas/v0.7/02-dashboard.png`, `45`, `25`.

- [ ] **Step 7: Commit**

```bash
git add _pages/portfolio.html _pages/changelog.html _pages/year-archive.html _portfolio/H-MAS.md _config.yml
git commit -m "feat: listing pages and product doc on new design"
```

---

### Task 8: Full verification, merge, deploy

**Files:** none new

- [ ] **Step 1: Full local screenshot pass**

With `jekyll serve` running, use Playwright to capture full-page screenshots at 1280×900 and 390×800 for:
`/`, `/hmas/`, `/solution/`, `/solution/H-MAS/`, `/changelog/`, `/changelog/h-mas/2026-08-02/`, `/posts/`, one blog post (path from `ls -d _site/posts/2026/*/*/ | tail -1`), `/cv/`, `/404.html`.

Checklist per page: no horizontal scrollbar at 390; nav wordmark not clipped; fonts are Gothic A1 / Noto Sans KR (inspect `document.fonts.check('900 16px "Gothic A1"')` → `true`); no leftover navy gradient; footer renders once.

- [ ] **Step 2: Stop the server and confirm a clean tree**

```bash
pkill -f "jekyll serve"; git status --short
```
Expected: empty (or only `_site/`, which is ignored).

- [ ] **Step 3: Merge and push**

```bash
git checkout master
git merge --ff-only redesign
git push
gh run list --limit 1
```
Expected: fast-forward merge, push to `master`, a `pages build and deployment` run appears. Wait for it:

```bash
gh run watch "$(gh run list --limit 1 --json databaseId --jq '.[0].databaseId')" --exit-status > /dev/null && echo BUILD_OK
```
Expected: `BUILD_OK`.

- [ ] **Step 4: Live check**

```bash
B=https://parameterfreak.com
for u in / /hmas/ /solution/ /changelog/ /posts/ /changelog/h-mas/2026-08-02/ /about/ /posts/2026/07/h-mas-weekly-release-notes-0726-0801; do printf "%-52s %s\n" "$u" "$(curl -s -o /dev/null -w '%{http_code}' "$B$u")"; done
curl -s $B/ | grep -o 'class="pf-nav__logo"[^>]*>[^<]*'
curl -s $B/ | grep -o 'fonts.googleapis.com/css2[^"]*' | head -1
```
Expected: all `200`; `class="pf-nav__logo" href="/">parameterfreak`; the fonts URL.

Playwright on `https://parameterfreak.com/` and `/hmas/` at 1280 — compare with the local screenshots.

- [ ] **Step 5: Delete the branch**

```bash
git branch -d redesign
```
