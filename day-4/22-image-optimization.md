# Image optimization and lazy loading

Images are usually the heaviest thing an app shows. They cost **download time**, **memory**, and **decoding time**, and a few large images can make scrolling stutter or crash the app with an out-of-memory error.

## Memory is about pixels, not file size

A compressed JPEG on disk is small. To display it, the OS decodes it into a bitmap with **4 bytes per pixel**:

```
photo from a phone camera: 4000 x 3000 pixels
in memory: 4000 x 3000 x 4 bytes = 48 MB          (the file may be 3 MB)

the same photo as a 100 x 100 avatar on screen:
needed at 3x density: 300 x 300 x 4 = 0.36 MB
```

Showing that photo in a 100-point thumbnail still decodes 48 MB unless something resizes it. A list of 20 such thumbnails is close to 1 GB.

## 1. Request the right size

The best fix is on the server: **don't download pixels you won't show**.

- Use an image CDN or resizing service: `https://cdn.example.com/p/42.jpg?w=300&h=300`.
- Ask for `width x PixelRatio.get()` so it's sharp on high-density screens:

```ts
import { PixelRatio } from 'react-native';

const size = PixelRatio.getPixelSizeForLayoutSize(100); // 100pt -> 300px on a 3x screen
const uri = `${CDN}/products/${id}.jpg?w=${size}&h=${size}`;
```

- Use modern formats: **WebP** (supported on both platforms) or AVIF where supported, often 25-50% smaller than JPEG at the same quality.

## 2. Use a cached image component

React Native's `<Image>` has basic caching that differs between platforms and is hard to control. Most apps use a dedicated component:

- **`@d11/react-native-fast-image`**: a maintained fork of `react-native-fast-image` with the same API. Memory + disk caching, priorities, preloading.
- The original `react-native-fast-image` was the common choice for years, but it is no longer actively maintained.

```tsx
import FastImage from '@d11/react-native-fast-image';

<FastImage
  source={{ uri, priority: FastImage.priority.normal, cache: FastImage.cacheControl.immutable }}
  style={{ width: 100, height: 100 }}
  resizeMode={FastImage.resizeMode.cover}
/>
```

## 3. Lazy loading

"Lazy loading" means loading images only when they're about to be visible.

- **In lists, virtualization does it for you.** `FlatList` / `FlashList` only render rows near the screen, so only their images load ([Unnecessary re-renders and large lists](21-re-renders-and-large-lists.md)).
- **Placeholders** keep the layout stable while loading: a solid color, a skeleton, or a **blurhash** / thumbhash (a tiny string the server computes, decoded into a blurry preview).
- **Always give images a size** (width/height or aspect ratio). An image without dimensions makes the layout jump when it loads.
- **Prefetch** images the user will see next, e.g. the detail image when they tap a product:

```ts
Image.prefetch(detailImageUrl);                // from 'react-native'
FastImage.preload([{ uri: detailImageUrl }]); // with FastImage
```

Lazy-loading **screens** and **code** is a related idea (Day 1: [Lazy Loading](../day-1/12-lazy-loading.md)).

## 4. Local images

Images bundled with the app (`require('./logo.png')`):

- Provide `@2x` and `@3x` versions; React Native picks the right one for the screen.
- Compress them before adding them to the project.
- Use **SVG** (`react-native-svg`) or an icon font for icons, instead of many PNGs.
- Large bundled images increase the app download size for every user.

## Checklist

- [ ] Server sends images at display size, in WebP.
- [ ] A caching image component is used for remote images.
- [ ] Every image has a fixed size or aspect ratio.
- [ ] Lists use FlatList/FlashList, with small thumbnails.
- [ ] Placeholders while loading, and an error fallback.
- [ ] Full-size images only on the screen that shows them full size.

## Rule of thumb

Memory cost is width x height x 4 bytes, so never show a big image in a small box. Get the right size from the server, cache it, give it fixed dimensions, and let virtualized lists do the lazy loading.
