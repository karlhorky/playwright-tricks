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

## Test Image Loading

Test that `<img>` elements have a `src` attribute that is reachable and responds with image data:

```ts
const img = page.locator('img');
await expect(img).not.toHaveJSProperty('naturalWidth', 0);
```

Source: https://github.com/microsoft/playwright/issues/6046#issuecomment-1803609118
