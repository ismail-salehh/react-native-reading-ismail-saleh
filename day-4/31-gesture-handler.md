# Gesture Handler basics

`react-native-gesture-handler` recognizes touches (tap, pan, pinch, long press, swipe, ...) **natively**, on the UI thread, instead of in JS. It's already installed in most apps, because React Navigation uses it (Day 1: [Installation and setup](../day-1/17-installation-setup.md)).

## Why not the built-in touch system?

React Native's built-in `onPress` / `PanResponder` send every touch event to the **JS thread**, which decides what the gesture is. If JS is busy, gestures lag. Two gestures competing (a swipeable row inside a scrolling list) are also hard to coordinate in JS.

Gesture Handler:

- recognizes gestures on the **UI thread**, with the platform's native recognizers,
- lets you declare how gestures **interact** (simultaneous, one wins, ...),
- together with Reanimated, runs the **whole** interaction (finger -> animation) on the UI thread.

## Setup

```bash
npm install react-native-gesture-handler
```

Wrap the app root once:

```tsx
import { GestureHandlerRootView } from 'react-native-gesture-handler';

export default function App() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <NavigationContainer>{/* ... */}</NavigationContainer>
    </GestureHandlerRootView>
  );
}
```

Gestures outside `GestureHandlerRootView` don't work. Modals rendered in a separate native window (React Native's `Modal` on Android) need their own root view inside.

## The API: Gesture + GestureDetector

You **create** a gesture with `Gesture.X()`, **configure** it with chained methods, and **attach** it to a view with `GestureDetector`:

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const doubleTap = Gesture.Tap()
  .numberOfTaps(2)
  .onEnd(() => {
    console.log('double tap');
  });

<GestureDetector gesture={doubleTap}>
  <View style={styles.photo} />
</GestureDetector>
```

Common gestures:

| Gesture | Recognizes | Useful data |
|---|---|---|
| `Gesture.Tap()` | Taps (`.numberOfTaps(n)`) | position |
| `Gesture.LongPress()` | Press and hold (`.minDuration(ms)`) | position |
| `Gesture.Pan()` | Dragging | `translationX/Y`, `velocityX/Y` |
| `Gesture.Pinch()` | Two-finger zoom | `scale`, `focalX/Y` |
| `Gesture.Rotation()` | Two-finger rotate | `rotation` |
| `Gesture.Fling()` | A quick swipe in a direction | direction |

Lifecycle callbacks: `onBegin`, `onStart`, `onUpdate` (every movement), `onEnd`, `onFinalize`.

## Pan + Reanimated: a draggable box

```tsx
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';

function DraggableBox() {
  const x = useSharedValue(0);
  const y = useSharedValue(0);
  const startX = useSharedValue(0);
  const startY = useSharedValue(0);

  const pan = Gesture.Pan()
    .onStart(() => {
      startX.value = x.value;
      startY.value = y.value;
    })
    .onUpdate((e) => {
      x.value = startX.value + e.translationX;
      y.value = startY.value + e.translationY;
    })
    .onEnd(() => {
      x.value = withSpring(0); // snap back
      y.value = withSpring(0);
    });

  const style = useAnimatedStyle(() => ({
    transform: [{ translateX: x.value }, { translateY: y.value }],
  }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[styles.box, style]} />
    </GestureDetector>
  );
}
```

When Reanimated is installed, gesture callbacks run as **worklets on the UI thread**. The box follows the finger without touching the JS thread. To update React state from a callback, call back to JS (`scheduleOnRN` / `runOnJS`, see [Reanimated](30-reanimated.md)), or use `.runOnJS(true)` on the gesture to run all its callbacks on JS (simpler, slower).

## Combining gestures

| Composition | Meaning | Example |
|---|---|---|
| `Gesture.Simultaneous(a, b)` | Both can be active at the same time | Pinch + rotate a photo |
| `Gesture.Exclusive(a, b)` | Only one; `a` has priority, `b` waits until `a` fails | Double tap (like) vs single tap (open) |
| `Gesture.Race(a, b)` | The first one to activate wins, the other is cancelled | Pan vs long press |

```ts
const singleTap = Gesture.Tap().onEnd(openPhoto);
const doubleTap = Gesture.Tap().numberOfTaps(2).onEnd(likePhoto);

const taps = Gesture.Exclusive(doubleTap, singleTap);
```

For gestures on **different** views that must cooperate with a scroll view, use `.simultaneousWithExternalGesture()` or `.requireExternalGestureToFail()`.

## Ready-made components

The library also provides components built on these gestures:

- **`ReanimatedSwipeable`**: list rows that swipe to reveal actions (delete, archive).
- **`Pressable`**, `ScrollView`, `FlatList`, `Switch` versions that cooperate with other gestures.
- `DrawerLayout` (React Navigation's drawer and stack swipe-back use Gesture Handler under the hood).

## Version note

The `Gesture` / `GestureDetector` API above is Gesture Handler 2. Newer major versions change the API style; the concepts (gesture types, callbacks, composition, worklets) stay the same. Check the docs for the version in your project.

## Rule of thumb

Wrap the app in `GestureHandlerRootView`, build gestures with `Gesture.X()` and attach them with `GestureDetector`, and combine them with `Simultaneous`, `Exclusive`, or `Race`. With Reanimated, update shared values in the callbacks so the whole interaction runs on the UI thread.
