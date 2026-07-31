# Export handoff reference

Hosted Agent cannot snapshot a rendered React deck: sandboxed shell cannot
reach the loopback dev server, and no current typed action returns the exact
bounded image artifacts. Report `execution_unavailable` for rendering and give
the user the handoff below. Do not create or run a browser automation script in
the sandbox.

## Prerequisites

| Tool | Requirement |
|---|---|
| Project-owned Playwright + Chromium | Already installed in the user's local project; never resolve it through a package runner |
| ImageMagick (for PDF) | Already installed in the user's local environment |
| img2pdf (PDF alternative) | Already installed in the user's local environment |

If any are missing, report the prerequisite and stop that export path. Never
run package managers or model-directed browser navigation from sandboxed shell.

## Step 1 — Prepare the user-run render handoff

Return the deck source path, slug, slide count, expected 1920×1080 viewport,
presenter-stripped route suffix (`?present=0`), and output directory
`~/Documents/slide-exports/<slug>-<date>/`. Ask the user to run their project's
approved local exporter. If they need to start the project first, provide this
as a user-run instruction:

```
Dev server isn't running. Start it with:
  cd ${SLIDE_DECK_REPO:-$HOME/code/your-slide-deck-site} && npm run dev

Then use your project's local Playwright/export command to save one PNG per
slide and attach or place those PNGs in the output directory.
```

Do not execute this command in Hosted Agent; it is long-running and its server
is not reachable from the sandbox network namespace.

## Step 2 — Count slides

Read `page.tsx` and count the top-level `id:` entries in `slides: Slide[]`.
Do not infer the count from a text grep for production decks.

## Step 3 — Combine supplied snapshots per output type

Continue only after the user or an advertised asset-safe host action supplies
the expected local PNG files. Verify the count and filenames before assembly.

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

For quick previews or chat thumbnails, tell the user's local exporter to use
1280×720 instead of 1920×1080. This halves file size at minimal visual loss.
