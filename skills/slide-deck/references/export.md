# Export reference

Snapshot a rendered React deck to portable HTML or PDF via Playwright. Hosted
publication remains unavailable until a reviewed asset/deployment action can
bind the exact files.

## Prerequisites

| Tool | Install | Check |
|---|---|---|
| Playwright + Chromium | Must already be present in the image/project | `npx playwright --version` |
| ImageMagick (for PDF) | Must already be installed by the user/image build | `magick --version` |
| img2pdf (PDF alternative, simpler) | Must already be installed by the user/image build | `img2pdf --version` |

If any are missing, report the prerequisite and stop that export path. Do not
run package managers from sandboxed shell.

## Step 1 — Verify dev server

Sandboxed shell cannot reach loopback. Check the existing dev process through
the host's advertised process/browser tooling when available. If no such tool
is advertised, report `execution_unavailable` for live rendering and ask the
user to start and verify the deck in their own terminal/browser.

If not running:

```
Dev server isn't running. Start it with:
  cd ${SLIDE_DECK_REPO:-$HOME/code/your-slide-deck-site} && npm run dev

Then re-run /slide-deck export <slug> <html|pdf|vercel>.
```

Don't auto-start — `npm run dev` is long-running.

## Step 2 — Count slides

```bash
# Counts top-level objects in the `slides: Slide[]` array
grep -E "^\s+\{$" ${SLIDE_DECK_REPO:-$HOME/code/your-slide-deck-site}/src/app/slides/<slug>/page.tsx | wc -l
```

Better: open the file and count `id:` entries in the slides array. Don't trust grep for production decks — verify by reading the file.

## Step 3 — Snapshot script

Save as `~/Documents/slide-exports/_scripts/snapshot.mjs` (create on first run):

```javascript
import { chromium } from 'playwright';
import { mkdir } from 'node:fs/promises';
import path from 'node:path';

const [, , slug, slideCountStr, outDir] = process.argv;
const slideCount = parseInt(slideCountStr, 10);
const host = process.env.SLIDE_DECK_DEV_HOST || 'localhost:3000';
const baseUrl = `http://${host}/slides/${slug}`;

await mkdir(outDir, { recursive: true });

const browser = await chromium.launch();
const ctx = await browser.newContext({ viewport: { width: 1920, height: 1080 } });
const page = await ctx.newPage();

// Reset persisted slide index so we start at slide 0
await page.addInitScript(({ slug }) => {
  localStorage.setItem(`slides:/slides/${slug}`, '0');
}, { slug });

// Load presenter-stripped view
await page.goto(`${baseUrl}?present=0`, { waitUntil: 'networkidle' });
await page.waitForTimeout(500);

for (let i = 0; i < slideCount; i++) {
  const filename = path.join(outDir, `slide-${String(i).padStart(3, '0')}.png`);
  await page.screenshot({ path: filename, fullPage: false });
  console.log(`Saved ${filename}`);
  if (i < slideCount - 1) {
    await page.keyboard.press('ArrowRight');
    await page.waitForTimeout(400); // let any animation settle
  }
}

await browser.close();
console.log(`Done: ${slideCount} slides`);
```

Run:

```bash
mkdir -p ~/Documents/slide-exports/<slug>-$(date +%Y-%m-%d)
node ~/Documents/slide-exports/_scripts/snapshot.mjs <slug> <count> ~/Documents/slide-exports/<slug>-$(date +%Y-%m-%d)
```

## Step 4 — Combine per output type

### html (standalone, portable, with arrow-key nav)

Generate a single `index.html` next to the PNGs:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title><DECK_TITLE></title>
  <style>
    html, body { margin: 0; padding: 0; height: 100%; background: #fff; overflow: hidden; }
    .stage { display: flex; align-items: center; justify-content: center; height: 100vh; }
    .stage img { max-width: 100%; max-height: 100%; object-fit: contain; display: none; }
    .stage img.active { display: block; }
    .nav { position: fixed; bottom: 1rem; right: 1rem; font: 14px/1 system-ui; color: #4d4d4d; opacity: 0.5; }
  </style>
</head>
<body>
  <div class="stage" id="stage">
    <!-- Inject one <img src="slide-000.png" class="active"> per slide -->
  </div>
  <div class="nav"><span id="n">1</span> / <span id="t">N</span></div>
  <script>
    const slides = document.querySelectorAll('.stage img');
    let i = 0;
    document.getElementById('t').textContent = slides.length;
    function go(n) {
      i = Math.max(0, Math.min(slides.length - 1, n));
      slides.forEach((s, j) => s.classList.toggle('active', j === i));
      document.getElementById('n').textContent = i + 1;
    }
    document.addEventListener('keydown', (e) => {
      if (e.key === 'ArrowRight' || e.key === ' ' || e.key === 'PageDown') go(i + 1);
      if (e.key === 'ArrowLeft' || e.key === 'PageUp') go(i - 1);
      if (e.key === 'Home') go(0);
      if (e.key === 'End') go(slides.length - 1);
    });
    go(0);
  </script>
</body>
</html>
```

Inject the `<img>` tags by iterating the PNG filenames in the directory.

### pdf

Simplest: `img2pdf` (preserves aspect, smaller output than ImageMagick):

```bash
img2pdf ~/Documents/slide-exports/<slug>-<date>/slide-*.png \
  -o ~/Documents/slide-exports/<slug>-<date>/<slug>.pdf
```

Alternative: ImageMagick (more flexible, larger output):

```bash
magick ~/Documents/slide-exports/<slug>-<date>/slide-*.png \
  ~/Documents/slide-exports/<slug>-<date>/<slug>.pdf
```

Open after generation:

```bash
open ~/Documents/slide-exports/<slug>-<date>/<slug>.pdf
```

### vercel (shareable URL)

Ensure the standalone `html` has been generated, then return
`execution_unavailable` for hosted publication and report the portable folder
path. Do not invoke a hosting CLI or upload the folder through the generic JSON
integration action; this path needs the separately designed typed asset
transport. The user may publish or remove it through their approved hosting
workflow.

## Caveats to surface to the user

- **Animations are not preserved.** Exports are static slide snapshots — any micro-interactions, presenter view, or animations live only in the React version.
- **For interactive demos, present the React version live.** Exports are for sharing, sending, printing, archiving.
- **File sizes**: a 20-slide deck at 1920×1080 PNG is ~30–50 MB before PDF compression. PDF output is usually 5–15 MB depending on content.
- **Snapshot quality requires the dev server to render fonts and gradients correctly.** If a slide looks wrong in the PNG, it'll look wrong in the export. First-time export of a brand-new deck: open the dev server and eyeball every slide first.

## Optional: low-resolution preview

For quick previews / Slack thumbnails, snapshot at 1280×720 instead of 1920×1080. Halves the file size at minimal visual loss:

```javascript
// In snapshot.mjs, change:
viewport: { width: 1280, height: 720 }
```
