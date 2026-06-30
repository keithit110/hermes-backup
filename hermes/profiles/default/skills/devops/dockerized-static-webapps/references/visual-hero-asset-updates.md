# Visual hero/background asset updates

Use this reference when changing a website hero/background image, especially for Keith's booking/marketplace-style landing pages.

## Workflow

1. Preserve the selected composition as a local static asset instead of hotlinking remote generator/CDN URLs.
   - Example Django/static path: `app/static/img/<descriptive-name>.jpg`
   - Reference from CSS with a relative URL such as `url('../img/<descriptive-name>.jpg')`.
2. Verify the original generated image dimensions before calling it high quality.
   - A prompt that says "4K" or "8K" does not prove the output is 4K.
   - Use PIL/ImageMagick/file metadata to confirm dimensions and size.
3. If upscaling is used, label it honestly as an upscale, not a true native 4K render.
   - Prefer working from the original PNG, not a previously compressed JPEG.
   - Export a production asset at 3840×2160 for desktop hero use when appropriate.
4. Cache-bust static assets after replacing the hero image/CSS.
   - Example: bump `site.css?v=N` in the base template.
5. Rebuild/restart the container/site and verify real delivery.
   - `home:200`, `health:200`, hero image URL returns `200`, and CSS version is the expected one.
   - Confirm the downloaded image dimensions from the served URL, not just the source file.
6. Browser-verify desktop and mobile screenshots.
   - Check that the focal object is visible after `background-size: cover` cropping.
   - Check headline readability against overlays/gradients.
   - Check browser console for errors.

## Pitfalls

- Upscaling a 1024×576 generated image to 1920×1080 or 3840×2160 improves delivery dimensions but cannot invent true native detail. If the user complains about pixelation, acknowledge this directly.
- Image generators may ignore "4K/8K" prompt language and still return low-resolution output. Always inspect dimensions.
- Generated images may include fake/gibberish text. Reject those for website backgrounds unless text will be fully cropped/hidden.
- A background that looks good as a full image may crop badly on mobile. Verify the live page, not only the source image.