# AGENTS.md — Agentic Infrastructure

Publishing rules for this site. Read this before changing anything in `src/` or pushing.

## 1. Project facts

| Item | Value |
|---|---|
| Repository | https://github.com/munawarabadullah/agentic-infrastructure.git |
| Local folder | `D:\WEBSITEs\Github-MunawarAbadullah\agentic-infrastructure` |
| Published source | `src/` (this is the only folder that gets published) |
| Live site | https://munawarabadullah.github.io/agentic-infrastructure/ |
| Hosting | GitHub Pages, source = GitHub Actions |
| Deploy workflow | `.github/workflows/pages.yml`, artifact path `./src` |
| Branch | `main` only |

Nothing outside `src/` is published. `.MDs/`, `.Todo/`, `.Archieve/` are local working folders and are ignored by git, with one exception: source images under `.MDs/Article-*/` are tracked so the compression gate can be verified against the originals.

Page map:

| Page | File | Role |
|---|---|---|
| Home | `src/index.html` | Root hub, the thesis |
| Journal | `src/journal/index.html` | Hub listing all essays |
| Article | `src/from-ai-assisted-coding-to-agentic-infrastructure/index.html` | Spoke |
| Site | `src/site/index.html` | Human-readable index of every page |

The structure is hub and spoke: home and journal are hubs, articles are spokes, and every spoke links back to a hub.

## 2. Hard rules (non-negotiable)

1. **Upload to `main` only.** No feature branches, no pull requests, no other long-lived branches. Every publish is `git push origin main`.
2. **Never force-push.** History on `main` is append-only. If a mistake is committed, fix it with a new commit.
3. **`sitemap.xml` is updated before every upload.** No page goes live unlisted.
4. **`sitemap.xml` is validated before every upload.** A publish with a broken or stale sitemap is a failed publish.
5. **JSON-LD is mandatory on every page.** All page data is expressed as schema.org JSON-LD first; HTML and CSS exist to present it.
6. **Verify production after the push.** A deploy is not done until the live URLs return 200.
7. **Every page carries the full social and SEO metadata set.** No page ships without `og:image`, `og:image:width`, `og:image:height`, `og:image:alt`, `twitter:card` and `twitter:image`. See section 7.
8. **No source PNG or JPG ships in `src/`.** Every published image is AVIF (primary) plus WebP (fallback), and the published bytes must be at least 70 percent smaller than the source file. See section 8.
9. **Every `<img>` has `alt`, explicit `width` and `height`, and a `srcset`.** Broken or unlabeled images fail the publish.

## 3. Publish procedure (follow in this order)

1. **Edit** the pages in `src/`.
2. **Convert** every new image through the pipeline in section 8 before it is referenced. AVIF plus WebP, 70 percent minimum reduction, explicit dimensions.
3. **Complete** the SEO and social head block from section 7 on every page touched. A page without `og:image` is not ready.
4. **Update** `src/sitemap.xml` for every page added, renamed, or materially changed (see section 4).
5. **Update** the page's JSON-LD `@graph` so it matches the new content (see section 5).
6. **Validate** the sitemap, the JSON-LD, the meta set, the image weight and the compression gate (see section 6).
7. **Preview locally** with `python -m http.server` from `src/`, and get approval for visual changes.
8. **Stage** with `git add -A`, then review `git status` and `git diff --staged`.
9. **Commit** on `main` with a descriptive message in the repo's existing style.
10. **Push** with `git push origin main`.
11. **Wait** for the `Deploy GitHub Pages` workflow to report success.
12. **Verify** the live URLs, including `/sitemap.xml`, `/robots.txt`, every `og:image` and the AVIF files.

## 4. Sitemap rules

`src/sitemap.xml` is the single source of truth. It must be valid XML and follow this shape:

```xml
<url>
  <loc>https://munawarabadullah.github.io/agentic-infrastructure/</loc>
  <lastmod>YYYY-MM-DD</lastmod>
  <changefreq>monthly</changefreq>
  <priority>1.0</priority>
</url>
```

- Every published page appears exactly once. No duplicates, no missing pages, no unlisted pages.
- `loc` is the canonical `github.io` URL, with a trailing slash, matching the page's `<link rel="canonical">`.
- `lastmod` is the date of the substantive change, in `YYYY-MM-DD`.
- Priority convention: home `1.0`, journal articles `0.9`, journal hub `0.8`, site index `0.5`.
- Bump the home entry's `lastmod` whenever the home page changes.
- `robots.txt` must keep pointing at the sitemap: `Sitemap: https://munawarabadullah.github.io/agentic-infrastructure/sitemap.xml`.

## 5. Structured data (JSON-LD)

**Data first, presentation second.** Every page carries one `<script type="application/ld+json">` block containing an `@graph`. The JSON-LD is the page's data layer; the HTML is a readable rendering of the same facts, and CSS decides how it looks. Never the other way round.

Rules:

1. **Every page has JSON-LD.** No page ships without it.
2. **JSON-LD and visible content must agree.** Every name, title, date, description, URL and list entry in JSON-LD is true of and visible on that page. Never describe a page, person, price, rating or date the page does not show.
3. **Stable `@id`s, reused across pages.** Use the canonical identifiers:
   - `https://munawarabadullah.github.io/agentic-infrastructure/#person`
   - `https://munawarabadullah.github.io/agentic-infrastructure/#website`
   - `https://munawarabadullah.github.io/agentic-infrastructure/#webpage`
   - `<page-url>#article` for an article
4. **Required types per page:**

   | Page | Required `@graph` nodes |
   |---|---|
   | All pages | `Person`, `WebSite`, a page node (`WebPage`, `CollectionPage` or `BlogPosting`), `BreadcrumbList` |
   | Home | `WebPage` |
   | Journal | `CollectionPage` + `ItemList` listing every article |
   | Article | `BlogPosting` with `headline`, `datePublished`, `dateModified`, `author`, `image`, `wordCount`, `timeRequired`, `keywords`, `articleSection` |
   | Site | `CollectionPage` + `ItemList` listing every page |

5. **Presentation stays in HTML and CSS.** Do not populate the visible page from JSON-LD with client-side JavaScript. The HTML must stand on its own with JSON-LD disabled; the JSON-LD is for machines, the markup is for people.
6. **Visual quality is a CSS job.** Data never dictates layout. Style the rendered HTML with the shared design system in `src/assets/theme.css` plus page-scoped CSS.
7. **A new page or entry updates three places:** the HTML page, its JSON-LD graph, and `sitemap.xml`. All three must agree.

## 6. Validation checks (run before commit)

```powershell
# 1. Sitemap is well-formed XML
[xml](Get-Content -LiteralPath "src\sitemap.xml" -Raw) | Out-Null; "sitemap XML OK"

# 2. Every sitemap <loc> has a matching file
#    (home -> src\index.html, /slug/ -> src\slug\index.html)

# 3. JSON-LD exists and parses on every page
Get-ChildItem "src" -Recurse -Filter index.html | ForEach-Object {
  $html = Get-Content -LiteralPath $_.FullName -Raw
  $m = [regex]::Match($html, '(?s)<script type="application/ld\+json">(.*?)</script>')
  if (-not $m.Success) { "MISSING JSON-LD: $($_.FullName)" }
  else {
    try { $null = $m.Groups[1].Value | ConvertFrom-Json; "JSON-LD OK: $($_.FullName)" }
    catch { "INVALID JSON-LD: $($_.FullName) - $($_.Exception.Message)" }
  }
}

# 4. Page hygiene
#    exactly one <h1>, no em dash characters, no dead #anchors,
#    no horizontal overflow at 390px, no console errors

# 5. After deploy
Invoke-WebRequest "https://munawarabadullah.github.io/agentic-infrastructure/sitemap.xml" -UseBasicParsing | Select-Object StatusCode

# 6. Meta completeness: required tags on every page
$required = 'og:title','og:description','og:type','og:url','og:image','og:image:type',
            'og:image:width','og:image:height','og:image:alt','twitter:card','twitter:image'
Get-ChildItem "src" -Recurse -Filter index.html | ForEach-Object {
  $html = Get-Content -LiteralPath $_.FullName -Raw
  foreach ($tag in $required) {
    if ($html -notmatch [regex]::Escape('"' + $tag + '"')) { "MISSING $tag -> $($_.FullName)" }
  }
}

# 7. Every og:image and twitter:image URL returns 200 image/*
Get-ChildItem "src" -Recurse -Filter index.html | ForEach-Object {
  $html = Get-Content -LiteralPath $_.FullName -Raw
  [regex]::Matches($html, 'property="og:image" content="([^"]+)"') | ForEach-Object {
    try {
      $r = Invoke-WebRequest $_.Groups[1].Value -UseBasicParsing -TimeoutSec 30
      if ($r.StatusCode -ne 200 -or $r.Headers['Content-Type'] -notmatch '^image/') { "BAD OG IMAGE -> $($_.Groups[1].Value)" }
      else { "og:image OK -> $($_.Groups[1].Value)" }
    } catch { "UNREACHABLE OG IMAGE -> $($_.Groups[1].Value)" }
  }
}

# 8. Image weight gate: no shipped image over 250 KB, total per page assets under 600 KB
$imgs = Get-ChildItem "src\assets\images" -File
$imgs | Where-Object { $_.Length -gt 250KB } | ForEach-Object { "OVER 250KB -> $($_.Name) ($([math]::Round($_.Length/1KB)) KB)" }
"Total published image weight: $([math]::Round((($imgs | Measure-Object Length -Sum).Sum)/1KB)) KB"

# 9. Compression gate: published output at least 70 percent smaller than its source
#    Compare each source in .MDs against the matching slug in src\assets\images.
Get-ChildItem ".MDs" -Recurse -File -Include *.png,*.jpg | ForEach-Object {
  $source = $_
  $slug = ($source.BaseName -replace '^\d{8}\s+','' -replace '[^A-Za-z0-9]+','-').ToLower().Trim('-')
  $out = Get-ChildItem "src\assets\images" -File | Where-Object { $_.Name -like "*$slug*" }
  if (-not $out) { "NO PUBLISHED IMAGE FOR SOURCE -> $($source.Name)" ; return }
  $largest = $out | Sort-Object Length -Descending | Select-Object -First 1
  $ratio = 1 - ($largest.Length / $source.Length)
  $pct = [math]::Round($ratio * 100)
  if ($ratio -lt 0.70) { "INSUFFICIENT COMPRESSION ($pct percent, need 70) -> $($largest.Name) vs $($source.Name)" }
  else { "Compression OK ($pct percent) -> $($largest.Name)" }
}
```

Site-wide conventions that validation depends on:

- One `<h1>` per page; section headings start at `<h2>`.
- No em dash character in page copy.
- Canonical URL uses the `github.io` domain, never a custom domain.
- Relative asset paths only (`assets/...`, `../assets/...`) so local preview and the deploy both work.
- Shared styling lives in `src/assets/theme.css`; page-specific CSS stays inline in that page.
- New pages are directories with an `index.html`, so URLs stay clean.

## 7. SEO and social metadata (mandatory on every page)

Every page ships this complete head block. A page missing any line below is not publishable.

```html
<title>Page title, 45 to 60 characters (about 60 is where Google truncates)</title>
<meta name="description" content="Unique summary, 120 to 158 characters.">
<link rel="canonical" href="https://munawarabadullah.github.io/agentic-infrastructure/<slug>/">
<meta name="robots" content="index,follow,max-image-preview:large">
<meta property="og:type" content="article">
<meta property="og:site_name" content="Agentic Infrastructure">
<meta property="og:title" content="Same as title">
<meta property="og:description" content="Same as description">
<meta property="og:url" content="Same as canonical">
<meta property="og:image" content="https://munawarabadullah.github.io/agentic-infrastructure/assets/images/<slug>-1672.webp">
<meta property="og:image:type" content="image/webp">
<meta property="og:image:width" content="1672">
<meta property="og:image:height" content="941">
<meta property="og:image:alt" content="What the image actually shows, one sentence">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Same as title">
<meta name="twitter:description" content="Same as description">
<meta name="twitter:image" content="Same as og:image">
<meta name="twitter:image:alt" content="Same as og:image:alt">
```

Rules:

1. `og:url` equals `<link rel="canonical">` equals the `sitemap.xml` `loc`. All three agree.
2. Every URL in the head is absolute `https://munawarabadullah.github.io/...`. Never a relative path in `og:*` or `twitter:*`.
3. `og:image` is **WebP or PNG, never AVIF**. Facebook, LinkedIn, Slack, X and WhatsApp do not render AVIF previews.
4. `og:image` width and height are declared and are the true pixel dimensions of the referenced file. No guessing.
5. `og:image`, `og:image:alt`, `twitter:image` and `twitter:image:alt` are mandatory on every page, not only on articles. Home, journal hub and site index included.
6. `og:title`, `og:description`, `twitter:title` and `twitter:description` match the `<title>` and `<meta name="description">` exactly.
7. `<meta name="description">` and `og:description` are identical, unique per page, and 120 to 158 characters.
8. `og:type` is `article` for articles, `website` for home, journal hub and site index.
9. The JSON-LD `@graph` repeats the same title, description, URL and image. Two sources of truth are a bug.
10. After the push, fetch each page and confirm `og:image` returns `200` with an `image/*` content type.
11. Default social image for a page with no bespoke illustration: `assets/images/agentic-infrastructure-intelligence-everywhere-1672.webp`.

## 8. Image pipeline (AVIF first, WebP fallback, 70 percent minimum reduction)

The installed converter is `C:\Users\Munawar\Scripts\Convert-ToAVIF.ps1`, also wired to the Explorer right-click menu as **→ AVIF** for every image extension. It calls `ffmpeg` with `libsvtav1`, `-preset 6`, `-still-picture 1`, and `-crf (100 - Quality)`, default `Quality 70` so `crf 30`. It flattens transparency onto white and deletes the source unless `-KeepOriginal` is passed.

Measured baseline on this repo's article illustration: 2,386,146 byte PNG becomes a 185,949 byte AVIF, a **92 percent reduction**.

Rules:

1. Source images live outside `src/`, in `.MDs/` or equivalent. Only converted output ships.
2. Every published image exists twice: `.avif` as the primary source in `<picture>`, `.webp` as the `<img>` fallback. No PNG or JPG in `src/assets/images/`.
3. Publish two widths per image, `800` and `1672` for full-bleed hero art, named `<slug>-<width>.avif` and `<slug>-<width>.webp`. Narrower column art uses its own two widths.
4. The total published bytes for an image must be at least **70 percent smaller** than the original source file. If the conversion does not reach 70 percent, lower `-Quality` (higher CRF) and convert again. 70 percent is the floor, not the target.
5. Hard ceiling: no single shipped image over 250 KB, no page with more than 600 KB of images.
6. `og:image` and `twitter:image` point at the WebP variant, because social crawlers do not decode AVIF.
7. Every `<picture>` carries explicit `width`, `height`, real `alt`, `loading="lazy"` and `decoding="async"`. The article hero is `loading="eager"` with `fetchpriority="high"`.
8. `srcset` lists every width and `sizes` describes the real layout width, so mobile never downloads the 1672 pixel file.
9. Originals are kept, never deleted from `.MDs/`. The `-KeepOriginal` switch exists for a reason.
10. No unreferenced file sits in `src/assets/images/`. Delete orphaned variants in the same commit that removes their usage.

Conversion commands:

```powershell
# Primary conversion (deletes the source unless -KeepOriginal)
& "C:\Users\Munawar\Scripts\Convert-ToAVIF.ps1" ".MDs\Article-20261025\hero.png" -KeepOriginal -Quality 70

# Second width, then WebP fallbacks, from the same source
$src = ".MDs\Article-20261025\hero.png"
$slug = "agentic-infrastructure-hero"
$dir = "src\assets\images"
foreach ($w in 1672, 800) {
  ffmpeg -y -hide_banner -loglevel error -i $src -frames:v 1 -vf "scale=${w}:-2" -c:v libsvtav1 -crf 30 -preset 6 -b:v 0 -still-picture 1 "$dir\$slug-$w.avif"
  ffmpeg -y -hide_banner -loglevel error -i $src -frames:v 1 -vf "scale=${w}:-2" -c:v libwebp -quality 80 -compression_level 6 "$dir\$slug-$w.webp"
}

# Report the reduction, must be 70 percent or better
Get-ChildItem $dir -Filter "$slug-*" | ForEach-Object {
  "{0,-52} {1,8} KB  ({2} percent smaller than source)" -f $_.Name, [math]::Round($_.Length/1KB),
    [math]::Round((1 - $_.Length / (Get-Item $src).Length) * 100)
}
```

Alternative tools installed on this machine, in order of preference: `ffmpeg` for scripted batch work above, `XnConvert` at `C:\Program Files\XnConvert\xnconvert.exe` for large one-off GUI batches with a saved preset, and Pillow 12.3 through `python` for resizing, cropping or format checks. Use the right-click **→ AVIF** entry for single files.

Markup pattern:

```html
<picture>
  <source type="image/avif"
          srcset="../assets/images/slug-800.avif 800w, ../assets/images/slug-1672.avif 1672w"
          sizes="(max-width: 760px) 100vw, 900px">
  <img src="../assets/images/slug-1672.webp"
       srcset="../assets/images/slug-800.webp 800w, ../assets/images/slug-1672.webp 1672w"
       sizes="(max-width: 760px) 100vw, 900px"
       width="1672" height="941" alt="..." loading="lazy" decoding="async">
</picture>
```

## 9. Never commit

- `.playwright-mcp/`, preview screenshots, or other tool artifacts.
- Markdown drafts under `.MDs/`, and everything in `.Todo/` and `.Archieve/`.
- Secrets, tokens, or local environment files.

Exception: source image files under `.MDs/Article-*/` are tracked. They are the originals the 70 percent compression gate in section 8 measures against, so they must stay in the repository.

## 10. Deployment notes

- Pages is published by `.github/workflows/pages.yml` on every push to `main`; the artifact is `src/`, so `src/index.html` becomes the site root.
- The Pages source must remain **GitHub Actions** (`build_type: workflow`). If it reverts to "Deploy from a branch", the site serves `README.md` instead of the site.
- `src/google5834b9afc4776dac.html` is a Google Search Console verification file. Do not delete it.
