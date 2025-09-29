# Playwright Tricks

A collection of helpful tricks for [Playwright](https://playwright.dev/) tests

## Import custom file types in tests

As of Sep 2025, Playwright doesn't support importing file types beyond JavaScript and TypeScript in tests:

- [[Feature] Support adding custom loaders to the test runner `#26822`](https://github.com/microsoft/playwright/issues/26822)
- [[Feature]: Support importing SCSS in ESM `#31689`](https://github.com/microsoft/playwright/issues/31689)

To import file types like MDX, image files, SCSS and others in Playwright tests, use Node.js ESM loaders via [Customization Hooks](https://nodejs.org/api/module.html#customization-hooks) APIs in `playwright.config.ts`.

For example, [`@nodejs-loaders/media`](https://www.npmjs.com/package/@nodejs-loaders/media) for image files and [`@mdx-js/node-loader`](https://www.npmjs.com/package/@mdx-js/node-loader) for MDX files:

`playwright.config.ts`

```ts
import { register } from 'node:module';
import type { PlaywrightTestConfig } from '@playwright/test';

// Node.js ESM loaders for image files and MDX imported by test files
// - https://github.com/microsoft/playwright/issues/26822#issuecomment-2692835230
register('@nodejs-loaders/media', import.meta.url);
register('@mdx-js/node-loader', import.meta.url);

const config: PlaywrightTestConfig = {
  // ...
};
 
export default config;
```

To import SCSS or other file types in Playwright tests, find a suitable Node.js ESM loader for the file type and use `register()` from `node:module` in `playwright.config.ts`.

## Interoperable text snapshots

Playwright does not add a newline at the end of files created with [non-image snapshots](https://playwright.dev/docs/test-snapshots#non-image-snapshots) - the text snapshots created with `expect().toMatchSnapshot()` - as discussed in [`microsoft/playwright#33416`](https://github.com/microsoft/playwright/issues/33416).

This means that the following code will create a file `snapshot.txt` with the content `abc`, without any newline at the end:

```ts
import { test, expect } from '@playwright/test';

test('example test', () => {
  expect('abc').toMatchSnapshot('snapshot.txt');
});
```

Snapshot files without newlines at the ends are problematic because commonly-used software like the GitHub "Edit in Place" feature and other common editor configurations will silently add a newline in the edge case of editing a snapshot file, which will cause the snapshot test to fail in a confusing way.

Also, [POSIX and *nix tools assume newlines at the end of files](https://stackoverflow.com/questions/729692/why-should-text-files-end-with-a-newline), so Playwright text snapshots will not play nice with those.

Unless the Playwright team reverses [their "working as intended" decision](https://github.com/microsoft/playwright/issues/33416#issuecomment-2455936144) and adds a fix to make text snapshots interoperable Create interoperable, this needs to be worked around.

[The current workaround](https://github.com/microsoft/playwright/issues/33416#issuecomment-2456363012) to create robust text snapshots with Playwright is to manually adding a newline at the end of the string passed to `expect()`:

```ts
import { test, expect } from '@playwright/test';

test('example test', () => {
  expect(
    'abc' +
      // Make Playwright snapshot file interoperable
      // - https://github.com/microsoft/playwright/issues/33416#issuecomment-2456363012
      '\n',
  ).toMatchSnapshot('snapshot.txt');
});
```

## Load all lazy images

Scroll to all visible lazy-loaded images and wait for [successful loading of image](#test-image-loading):

```ts
const lazyImages = await page.locator('img[loading="lazy"]:visible').all();

for (const lazyImage of lazyImages) {
  await lazyImage.scrollIntoViewIfNeeded();
  await expect(lazyImage).not.toHaveJSProperty('naturalWidth', 0);
}
```

Be aware, [using `.all()` can be problematic if new images are being added, removed, shown or hidden while the test code is running](https://github.com/microsoft/playwright/issues/31737).

One workaround for this is to assert the length of the `.all()` array (if you know it) to wait for it to stabilize:

```ts
const lazyImagesLocator = page.locator('img[loading="lazy"]:visible');

// Assert on length to wait for image visibility to stabilize
// after client-side JavaScript hides some images
// https://github.com/microsoft/playwright/issues/31737#issuecomment-2233775909
await expect(lazyImagesLocator).toHaveCount(13);

const lazyImages = await lazyImagesLocator.all();

for (const lazyImage of lazyImages) {
  await lazyImage.scrollIntoViewIfNeeded();
  await expect(lazyImage).not.toHaveJSProperty('naturalWidth', 0);
}
```

## Screenshot comparison tests of PDFs

Playwright does not (as of June 2024) have support for [visual comparison testing](https://playwright.dev/docs/test-snapshots) with PDFs.

There are [many issues asking for this feature](https://github.com/microsoft/playwright/issues?q=is%3Aissue+sort%3Aupdated-desc+pdfs+is%3Aclosed), but the current position of the Playwright team is that [PDF.js should be used instead](https://github.com/microsoft/playwright/issues/19253#issuecomment-1338955863), to render the PDF to a canvas.

It's not clear how the Playwright team suggests to do this, but one way is to navigate to `about:blank`, use [`page.setContent()`](https://playwright.dev/docs/api/class-page#page-set-content) to add a PDF.js viewer to the page, which accepts a URL, and then use [`expect(page).toHaveScreenshot()`](https://playwright.dev/docs/api/class-pageassertions#page-assertions-to-have-screenshot-2):

```ts
// HTML template string no-op for VS Code highlighting / formatting
function html(strings: TemplateStringsArray, ...values: unknown[]) {
  return strings.reduce((result, string, i) => {
    return result + string + (values[i] ?? '');
  }, '');
}

test('PDF has screenshot', async ({ page }) => {
  // Go to page without Content-Security-Policy header, to avoid CSP
  // prevention of script loading from https://mozilla.github.io
  await page.goto('about:blank');

  await page.setContent(html`
    <!doctype html>
    <html>
      <head>
        <meta charset="UTF-8" />
      </head>
      <body>
        <canvas></canvas>
        <script src="https://mozilla.github.io/pdf.js/build/pdf.mjs" type="module"></script>
        <script type="module">
          pdfjsLib.GlobalWorkerOptions.workerSrc =
            'https://mozilla.github.io/pdf.js/build/pdf.worker.mjs';

          try {
            const pdf = await pdfjsLib.getDocument(
               'https://raw.githubusercontent.com/mozilla/pdf.js/ba2edeae/examples/learning/helloworld.pdf',
            ).promise;

            const page = await pdf.getPage(1);
            const viewport = page.getViewport({ scale: 1.5 });

            const canvas = document.querySelector('canvas');
            canvas.height = viewport.height;
            canvas.width = viewport.width;

            await page.render({
              canvasContext: canvas.getContext('2d'),
              viewport,
            }).promise;
          } catch (error) {
            console.error('Error loading PDF:', error);
          }
        </script>
      </body>
    </html>
  `);

  await page.waitForTimeout(1000);

  await expect(page).toHaveScreenshot({ fullPage: true });
});
```

## Sync Playwright version from `package.json` to GitHub Actions container image

When using GitHub Actions, Playwright tests should be run [via containers using the official Microsoft Docker image](https://playwright.dev/docs/ci#via-containers) for performance - the slower alternative of [installing browsers with `playwright install --with-deps`](https://playwright.dev/docs/ci#on-pushpull_request) can take [5x as long](https://github.com/karlhorky/repro-dynamic-playwright-container-image#why) or longer versus the Docker image.

However, the Docker container approach hardcodes the Playwright version in a new place in the codebase - the GitHub Actions workflow files - requiring effort or automation to keep the Playwright versions in sync in both `package.json` and the GitHub Actions workflow files. There is a high chance of these versions getting out of sync as Playwright is upgraded.

To sync the Playwright version from `package.json` to the container image used in GitHub Actions, make sure you use an exact version of Playwright in your `package.json`:

`package.json`

```json
{
  "devDependencies": {
    "@playwright/test": "1.55.0"
  }
}
```

...and then read out the version with `yq` in a minimal job, set it as output and pass it along to the second job for usage to set the Docker image version in the [`jobs.<job_id>.container.image`](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container#defining-the-container-image) expression:

`.github/workflows/ci-container.yml`

```yaml
name: CI
on: [push]
jobs:
  resolve-playwright-version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps['resolve-playwright-version'].outputs.version }}
    steps:
      - uses: actions/checkout@v5
      - name: resolve-playwright-version
        id: resolve-playwright-version
        run: |
          version="$(yq -r '.devDependencies["@playwright/test"] // .dependencies["@playwright/test"] // ""' package.json)"
          test -n "$version" || { echo "No @playwright/test version found in package.json"; exit 1; }
          echo "version=$version" >> "$GITHUB_OUTPUT"

  ci:
    runs-on: ubuntu-latest
    needs: resolve-playwright-version
    timeout-minutes: 15
    container:
      image: mcr.microsoft.com/playwright:v${{ needs['resolve-playwright-version'].outputs.version }}
      options: --user 1001
    steps:
      - uses: actions/checkout@v5
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v5
        with:
          node-version: 'lts/*'
          cache: 'pnpm'
      - run: pnpm install
      - run: pnpm playwright test
```

Reproduction repo: https://github.com/karlhorky/repro-dynamic-playwright-container-image

## Test image loading

Test that `<img>` elements have a `src` attribute that is reachable and responds with image data:

```ts
const img = page.locator('img');
await expect(img).not.toHaveJSProperty('naturalWidth', 0);
```

Source: https://github.com/microsoft/playwright/issues/6046#issuecomment-1803609118
