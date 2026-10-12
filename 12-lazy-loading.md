## <span style="color:gold;font-weight:bold"> Lazy Loading </span>

lazy loading means **not loading something until it's actually needed**, so the app starts faster and uses less memory. in RN it shows up in a few places.

<hr />

### <span style="color:silver;font-weight:bold">1. Lists</span>

FlatList is already lazy (virtualization), but you can tune how lazy it is:
- **initialNumToRender**: how many items to render at first. just enough to fill the screen is ideal.
- **maxToRenderPerBatch**: how many items to render per batch while scrolling.
- **windowSize**: how many screens worth of items to keep rendered around the visible area (default 21, which is 10 above + the visible one + 10 below). smaller = less memory, but you might see blank areas when scrolling fast.
- **getItemLayout**: if all items have the same height, tell FlatList the height so it doesn't have to measure them. makes scrolling (and scrollToIndex) way faster.

<hr />

### <span style="color:silver;font-weight:bold">2. Images</span>

network images are only downloaded when the Image is rendered, so inside a FlatList they're lazy for free.
show a placeholder (*defaultSource*, or a gray box) while they load, and use <span style="color:chocolate">onLoad</span> to swap them. for caching images on disk, use a library, since the built-in Image cache is limited.

<hr />

### <span style="color:silver;font-weight:bold">3. Components and screens</span>

<span style="color:chocolate">React.lazy</span> + <span style="color:chocolate">Suspense</span> work in RN, but metro puts all your JS in **one bundle** anyway, so it doesn't save download time like on web.
the RN way is **inline requires**: turning on *inlineRequires: true* in metro.config.js makes modules run only when they're first used, not at startup.

> Also btw, navigators are lazy too: tab screens only mount the first time you open them, and stack screens only when you navigate to them.
