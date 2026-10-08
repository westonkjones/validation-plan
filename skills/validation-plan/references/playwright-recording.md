# Recording scenario videos with Playwright

Each scenario (the happy path, then each chosen edge case) gets its own short clip. Viewers
watch these without the code in front of them, so slow the actions down enough to follow and
put a caption on screen saying what the clip shows.

## Setup

Make a recording directory in the scratchpad, outside the repo:

```bash
REC_DIR="$SCRATCHPAD/validation-videos"   # session scratchpad directory
mkdir -p "$REC_DIR"
```

Prefer the repo's own Playwright if it has one (`@playwright/test` or `playwright` in any
`package.json`, including tooling packages under `tools/`), since it already matches the
repo's browsers. Install that package's dependencies and point the scripts at it:

```bash
(cd "$REPO/<package dir>" && npm ci && npx playwright install chromium)
export PLAYWRIGHT_PKG="$REPO/<package dir>/package.json"
```

Keep scripts in the scratchpad either way. Node resolves a bare `import 'playwright'` from
the script's own directory, so the template loads it through `createRequire` from
`PLAYWRIGHT_PKG`. If the repo has no Playwright, install a pinned copy in `$REC_DIR` instead
and leave `PLAYWRIGHT_PKG` unset:

```bash
cd "$REC_DIR"
npm init -y >/dev/null
npm install --silent playwright@1.48.2
npx playwright install chromium
```

Make sure the app is running from the plan's setup section before recording, and export
`BASE_URL` as the UI URL that setup serves. Sign in the
way a reader would locally (dev bypass, mock sign-in, seeded user). Never record against
shared staging or production.

## Script template

Write one script per scenario in `$REC_DIR` and run it from there with `node <script>.mjs`, so
the relative video paths land in `$REC_DIR`. Fill in the
steps from the flow you mapped in the code; use real selectors from the components (roles,
labels, test ids) rather than guessing.

```js
// happy-path.mjs
import { createRequire } from 'node:module';

// Load Playwright from the repo package in PLAYWRIGHT_PKG, or from next to this script.
const require = createRequire(process.env.PLAYWRIGHT_PKG || import.meta.url);
const { chromium } = require('playwright');

const SCENARIO = 'happy-path';             // file name slug
const CAPTION = 'Happy path: create a shop and publish it';
const BASE_URL = process.env.BASE_URL;   // the UI URL from the plan's setup
if (!BASE_URL) throw new Error('Set BASE_URL to the local UI URL from the plan setup');

// Pins a caption bar to the top of the page so the clip explains itself.
async function caption(page, text) {
  await page.evaluate((t) => {
    let el = document.getElementById('__vp_caption');
    if (!el) {
      el = document.createElement('div');
      el.id = '__vp_caption';
      el.style.cssText = 'position:fixed;top:0;left:0;right:0;z-index:2147483647;' +
        'padding:8px 14px;font:600 15px system-ui;color:#fff;background:rgba(17,17,17,.85);' +
        'pointer-events:none';
      document.body.appendChild(el);
    }
    el.textContent = t;
  }, text);
}

const browser = await chromium.launch({ slowMo: 300 });
const context = await browser.newContext({
  viewport: { width: 1280, height: 720 },
  recordVideo: { dir: './raw', size: { width: 1280, height: 720 } },
});
const page = await context.newPage();

await page.goto(`${BASE_URL}/...`);
await caption(page, CAPTION);
// Re-apply the caption after each navigation, and update it to narrate steps:
// await page.getByRole('button', { name: 'Create shop' }).click();
// await caption(page, `${CAPTION} · 2. Fill in shop details`);

await page.waitForTimeout(1500);           // hold the final state on screen
const video = page.video();
await context.close();                     // flushes the video to disk
await video.saveAs(`./${SCENARIO}.webm`);
await browser.close();
```

Name files `happy-path.webm` and `edge-<slug>.webm` (e.g. `edge-empty-catalog.webm`).

## Check and convert

Watch for a script that "passes" without showing the behavior. Have each script wait for or
read the element that proves the scenario (and log its text), then look at a screenshot of
the final state before the clip goes in the plan. Check that the caption bar and the
viewport edge don't hide the thing the clip is meant to show; scroll it into the middle of
the frame.

Playwright writes VP8 WebM. If `ffmpeg` is available, also produce an MP4 for broader
playback and embed that instead:

```bash
ffmpeg -loglevel error -i happy-path.webm -c:v libx264 -pix_fmt yuv420p -movflags +faststart happy-path.mp4
```

Without ffmpeg, embed the WebM as is.

## Embedding

Publish the clips as supporting files of the page (the Artifact `files` map, e.g.
`"videos/happy-path.webm": "<local path>"`) and reference them relatively:

```html
<video src="videos/happy-path.webm" controls muted playsinline preload="metadata"></video>
```
