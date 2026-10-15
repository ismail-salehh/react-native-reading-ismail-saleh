# Reanimated (only if the app has animation)

`react-native-reanimated` is the standard library for smooth animations in React Native. Its main idea: **animation code runs on the UI thread**, so it stays at 60/120 FPS even when the JS thread is busy ([JS thread vs UI thread](20-js-thread-vs-ui-thread.md)).

Use it when the app has real animation: interactive gestures (swipe, drag, pinch), animations tied to scroll, or complex sequences. For a simple fade, React Native's built-in `Animated` with `useNativeDriver: true` or Reanimated's layout animations are enough.

## Installation

```bash
npm install react-native-reanimated react-native-worklets
```

Add the Babel plugin (it must be **last** in the list) in `babel.config.js`:

```js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: ['react-native-worklets/plugin'],
};
```

Then rebuild the app and restart Metro with `--reset-cache`. Version notes: Reanimated 4 requires the New Architecture and uses `react-native-worklets`; Reanimated 3 used `react-native-reanimated/plugin` instead. Follow the docs for the version you install.

## Worklets

A **worklet** is a JS function that can run on the UI thread. Reanimated's Babel plugin finds them and makes a copy that runs on a separate JS runtime on the UI thread.

Functions passed to `useAnimatedStyle`, gesture callbacks, and scroll handlers are worklets automatically. You can mark your own with the `'worklet'` directive:

```ts
function clamp(value: number, min: number, max: number) {
  'worklet';
  return Math.min(Math.max(value, min), max);
}
```

Because worklets run on another thread, they **can't** directly use React state, call `setState`, or call normal JS functions. To call back into the JS thread, use `scheduleOnRN` (Reanimated 4, from `react-native-worklets`) or `runOnJS` (Reanimated 3).

## The three building blocks

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
} from 'react-native-reanimated';

function Box() {
  // 1. a value that lives on the UI thread
  const offset = useSharedValue(0);

  // 2. a style computed from it, on the UI thread, every frame
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: offset.value }],
  }));

  return (
    <>
      {/* 3. an Animated component that accepts animated styles */}
      <Animated.View style={[styles.box, animatedStyle]} />
      <Button title="Move" onPress={() => { offset.value = withSpring(offset.value + 100); }} />
    </>
  );
}
```

| Piece | What it is |
|---|---|
| `useSharedValue(initial)` | A value readable and writable from both threads. Changing `.value` **doesn't re-render** the component |
| `useAnimatedStyle(fn)` | Recomputes the style on the UI thread whenever a shared value it reads changes |
| `Animated.View` / `Text` / `Image`, `Animated.createAnimatedComponent` | Components that accept animated styles and props |

Don't read `sharedValue.value` during render in your component body. It's meant for worklets and event handlers.

## Animation functions

Assigning an animation function to `.value` animates it instead of jumping:

| Function | Does |
|---|---|
| `withTiming(to, { duration, easing })` | Time-based animation |
| `withSpring(to, config)` | Physics-based spring (natural for gestures) |
| `withDecay({ velocity })` | Slows down from a velocity (flinging) |
| `withDelay(ms, anim)` | Starts later |
| `withSequence(a, b, ...)` | One after another |
| `withRepeat(anim, count, reverse)` | Loops (`-1` = forever) |

```ts
// shake
offset.value = withSequence(
  withTiming(-10, { duration: 50 }),
  withRepeat(withTiming(10, { duration: 100 }), 3, true),
  withTiming(0, { duration: 50 }),
);
```

## Layout animations

Animate components when they **mount, unmount, or move**, without shared values:

```tsx
import Animated, { FadeIn, FadeOut, LinearTransition } from 'react-native-reanimated';

<Animated.View entering={FadeIn} exiting={FadeOut} layout={LinearTransition}>
  <TodoRow todo={todo} />
</Animated.View>
```

Good for list items being added or removed, toasts, and expanding cards.

## With gestures and scroll

Reanimated's real strength is **following the finger** with [Gesture Handler](31-gesture-handler.md), and scroll-driven effects:

```tsx
const scrollY = useSharedValue(0);
const onScroll = useAnimatedScrollHandler((e) => {
  scrollY.value = e.contentOffset.y;
});

const headerStyle = useAnimatedStyle(() => ({
  height: interpolate(scrollY.value, [0, 150], [200, 60], Extrapolation.CLAMP),
}));

<Animated.ScrollView onScroll={onScroll} scrollEventThrottle={16}>...</Animated.ScrollView>
```

`interpolate` maps one range to another: scroll 0-150 -> height 200-60.

## Things to keep in mind

- Animate **`transform` and `opacity`** when possible. Animating `width`, `height`, or `top` triggers layout every frame and is more expensive.
- Respect **reduced motion**: `useReducedMotion()` tells you if the user turned on "Reduce Motion". Skip or shorten non-essential animations.
- Animations are a feature, not decoration. If the app has none that need it, don't add the dependency.

## Rule of thumb

Shared values hold the state, `useAnimatedStyle` turns them into styles on the UI thread, and `withTiming` / `withSpring` animate them. Use layout animations for mount/unmount, and combine with Gesture Handler for anything that follows the finger.
