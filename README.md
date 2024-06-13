# Playwright Tricks

A collection of helpful tricks for [Playwright](https://playwright.dev/) tests

## Load All Lazy Images

Scroll to all visible lazy-loaded images and wait for [successful loading of image](#test-image-loading):

```ts
const lazyImages = await page.locator('img[loading="lazy"]:visible').all();

for (const lazyImage of lazyImages) {
  await lazyImage.scrollIntoViewIfNeeded();
  await expect(lazyImage).not.toHaveJSProperty('naturalWidth', 0);
}
```

## Screenshot Comparison Tests of PDFs

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
  // Go to page without Content-Security-Policy header, to avoid
  // CSP restrictions preventing loading of scripts on https://mozilla.github.io
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

## Test Image Loading

Test that `<img>` elements have a `src` attribute that is reachable and responds with image data:

```ts
const img = page.locator('img');
await expect(img).not.toHaveJSProperty('naturalWidth', 0);
```

Source: https://github.com/microsoft/playwright/issues/6046#issuecomment-1803609118
