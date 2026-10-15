# Old vs new architecture: JSI, Fabric, TurboModules, Codegen, bridgeless

React Native has to connect two worlds: **JavaScript** (your React code) and **native** (iOS and Android views and APIs). The "architecture" is how those two talk to each other. The old way was **the bridge**. The New Architecture replaces it with direct calls through **JSI**.

Timeline: the New Architecture was announced in 2018, became the **default in React Native 0.76** (2024), and since **0.82** the old architecture can no longer be turned on.

## The old architecture: the bridge

```
   JS thread                         Native side
 ┌────────────┐   JSON messages   ┌──────────────────┐
 │  React     │ ────────────────► │  UIManager       │
 │  your code │ ◄──────────────── │  Native Modules  │
 └────────────┘    (async,        └──────────────────┘
                    batched,
                    serialized)
```

Every interaction became a **message**:

1. JS wants to call a native method (or update a view).
2. The arguments are **serialized to JSON**.
3. The message waits in a queue and is sent in a **batch**.
4. Native **deserializes** the JSON and runs the method.
5. The result comes back the same way, **asynchronously**.

Problems:

- **Always asynchronous.** JS could never get an answer immediately, e.g. measure a view's size and position something in the same frame.
- **Serialization cost.** Big data (images, large arrays, scroll events) was copied and converted to JSON and back.
- **A single bottleneck.** Everything shared one queue. Heavy traffic (fast scrolling sending many events) delayed everything else.
- **All native modules loaded at startup**, even the ones a user never touched.
- **No type safety.** JS and native agreed on method names and argument types only by convention. A mismatch was a runtime crash.

## The new architecture: the pieces

### JSI (JavaScript Interface)

The foundation. JSI is a **C++ API** that lets native code and the JS engine reference each other **directly**.

- A C++ object can be exposed to JS as a "host object". Calling a method on it from JS calls C++ **directly**, with no JSON and no queue.
- Calls can be **synchronous** when needed (or asynchronous when that's better).
- It is engine-independent: Hermes implements it ([Hermes](25-hermes.md)).

Analogy: the bridge was sending letters through a post office; JSI is a phone call.

### Fabric (the new renderer)

Fabric is the new **rendering system**, the part that turns your React components into native views.

- Written in **C++** and shared by iOS and Android (more consistent behavior across platforms).
- Uses JSI, so React can talk to the view tree directly.
- Supports **synchronous layout** reads (e.g. measuring a view in `useLayoutEffect` before the frame is drawn, without flicker).
- Supports **React 18+ concurrent features**: `startTransition`, `useDeferredValue`, Suspense. React can prepare a big update with low priority while keeping urgent updates (typing) responsive.

The old renderer is called **Paper**.

### TurboModules (the new native modules)

The new way to write **native modules** (camera, keychain, storage, ...).

- Called **through JSI**: direct, typed, optionally synchronous.
- **Lazy loaded**: a module is initialized the first time JS uses it, not at app start. Faster startup when the app has many native libraries.

### Codegen

A build-time tool that generates the **glue code** between JS and native from a **typed spec** written in TypeScript (or Flow):

```ts
// NativeCalculator.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  add(a: number, b: number): number;
}

export default TurboModuleRegistry.getEnforcing<Spec>('Calculator');
```

From this, Codegen generates C++/Objective-C/Java interfaces that the native code must implement. If the native side doesn't match the spec, the **build fails** instead of the app crashing at runtime. The same is done for native components (Fabric).

### Bridgeless mode

The final step: the bridge is **removed entirely**. All communication goes through JSI. Things that still relied on bridge-only infrastructure (some error handling, timers, global event emitters) were rebuilt on top of JSI. Bridgeless became the default together with the New Architecture.

Old libraries written for the bridge keep working through an **interop layer**, but the goal is for libraries to move to TurboModules and Fabric components.

## Side by side

| | Old architecture | New architecture |
|---|---|---|
| JS <-> native communication | Bridge: async JSON messages | JSI: direct C++ calls |
| Sync calls | Not possible | Possible |
| Rendering | Paper (per platform) | Fabric (shared C++) |
| React concurrent features | No | Yes |
| Native modules | Native Modules, all loaded at startup | TurboModules, lazy loaded |
| JS/native types | By convention | Generated and checked by Codegen |
| The bridge | Central | Removed (bridgeless) |

## In your own words

The old architecture made JS and native talk by **sending serialized messages through a single async queue**. The New Architecture lets them **hold references to each other and call each other directly** through JSI. Fabric uses that for rendering, TurboModules use it for native APIs, Codegen makes it type-safe, and bridgeless removes the old queue completely.

Why that makes apps faster is in [Why the new architecture improves performance](27-new-architecture-performance.md).

## Rule of thumb

Bridge = async JSON messages through one queue. JSI = direct C++ calls. Fabric renders through JSI, TurboModules expose native APIs through JSI, Codegen generates the typed glue, and bridgeless means the bridge is gone.
