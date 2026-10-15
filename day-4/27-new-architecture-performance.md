# Why the new architecture improves performance

The New Architecture ([Old vs new architecture](26-old-vs-new-architecture.md)) doesn't make your JS run faster by itself. It removes **overhead between JS and native**, and it gives React tools to **schedule** work better. Each improvement comes from one of the pieces.

## 1. No serialization (JSI)

**Old**: every call to native converted the arguments to JSON, copied the string across, and parsed it on the other side. The result came back the same way.

**New**: JS calls a C++ function directly and passes values by reference.

```
old: JS object -> JSON.stringify -> copy -> parse -> native object      (every call)
new: JS -> C++ function call                                            (no copy)
```

This matters most for **frequent** or **large** data: scroll and gesture events, camera frames (VisionCamera frame processors only exist because of JSI), large lists, storage libraries (MMKV is fast because it reads synchronously through JSI).

## 2. Synchronous when needed (JSI + Fabric)

**Old**: everything was async. To measure a view and position a tooltip next to it, JS asked native, waited for the reply (at least one frame later), then rendered again. The user could see the tooltip **jump**.

**New**: layout can be read synchronously in `useLayoutEffect`, before the frame is shown:

```tsx
useLayoutEffect(() => {
  // with Fabric, this callback runs synchronously, before the frame is drawn
  ref.current?.measureInWindow((x, y, width, height) => {
    setTooltipPosition({ x, y: y + height });
  });
}, []);
```

Fewer round-trips means fewer extra frames and less visual flicker.

## 3. No single bottleneck

**Old**: all traffic shared one async queue. A burst of messages (fast scrolling, a big list update) delayed everything else, including the response to a tap.

**New**: each call goes directly to its target. One busy module doesn't hold up the others.

## 4. Faster startup (TurboModules)

**Old**: every native module was created at launch, whether the user needed it or not. An app with 40 native libraries initialized all 40 before showing the first screen.

**New**: TurboModules are **lazy**. The camera module is created the first time the camera is used. Less work before the first screen means a faster start, especially in large apps.

Hermes adds to this with precompiled bytecode ([Hermes](25-hermes.md)).

## 5. Better scheduling (Fabric + concurrent React)

**Old**: React rendered every update with the same priority. A big update (filtering a long list) blocked typing in the search box until it finished.

**New**: Fabric supports React's concurrent rendering. You can mark updates as less urgent:

```tsx
const [text, setText] = useState('');
const [query, setQuery] = useState('');

function onChangeText(value: string) {
  setText(value);                          // urgent: the input updates now
  startTransition(() => setQuery(value));  // can wait: the list filters after
}
```

React keeps the input responsive and renders the expensive list update when it can, throwing away outdated work. The total work is the same; the **perceived** speed is much better.

## 6. Shared C++ core

Fabric's rendering logic is in **one C++ implementation** for both platforms. Less duplicated code, fewer platform-specific bugs, and performance work done once benefits iOS and Android.

## What it doesn't fix

The New Architecture removes overhead in the **connection**. It doesn't fix slow code on either side:

- Unnecessary re-renders still waste the JS thread ([Re-renders and large lists](21-re-renders-and-large-lists.md)).
- Huge images still use too much memory ([Image optimization](22-image-optimization.md)).
- Heavy loops in JS still block the JS thread ([JS thread vs UI thread](20-js-thread-vs-ui-thread.md)).
- Leaks still leak ([Memory leaks](24-memory-leaks.md)).

An app with these problems is still slow on the New Architecture.

## Summary

| Old problem | New solution | Result |
|---|---|---|
| JSON serialization on every call | JSI direct calls | Less CPU per JS <-> native call |
| Everything async | Sync calls, sync layout | Fewer round-trips, no layout jumps |
| One shared queue | Direct calls | No bottleneck under heavy traffic |
| All modules loaded at startup | Lazy TurboModules | Faster startup |
| No priorities in rendering | Fabric + concurrent React | Input stays responsive during big updates |
| Separate renderer per platform | Shared C++ Fabric | Consistent, optimized once |

## Rule of thumb

The New Architecture is faster because JS and native call each other directly instead of exchanging JSON through one async queue, native modules load only when needed, and React can prioritize urgent updates. It removes overhead between the two sides; it doesn't fix slow code inside them.
