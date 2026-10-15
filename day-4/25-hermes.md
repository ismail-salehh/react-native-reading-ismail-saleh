# Hermes

Hermes is a **JavaScript engine** built by Meta specifically for React Native. The engine is the program that runs your JS code. Hermes has been the **default engine since React Native 0.70**, so a new project already uses it.

Before Hermes, React Native used **JavaScriptCore (JSC)**, the engine from Safari, on both platforms.

## The main idea: compile ahead of time

Most JS engines (V8 in Chrome, JSC) receive **source code** at runtime, then parse it, compile it, and optimize it while the app runs (a JIT, "just-in-time" compiler). That's great for long-running websites, but it makes app **startup** slow: thousands of lines of JS must be parsed before the first screen appears.

Hermes moves that work to **build time**:

```
JSC / V8:
  build:   bundle JS source  ->  app.bundle (text)
  launch:  read text -> parse -> compile -> run       (on the user's phone, every launch)

Hermes:
  build:   bundle JS source  ->  compile to Hermes bytecode (.hbc)
  launch:  read bytecode -> run                        (parsing already done)
```

The phone loads ready-to-run **bytecode**, and can map it into memory lazily instead of reading it all at once.

## What you get

| Benefit | Why |
|---|---|
| **Faster startup** (Time To Interactive) | No parsing/compiling at launch |
| **Lower memory use** | No JIT, compact bytecode, a garbage collector (Hades) designed for mobile that does most work concurrently |
| **Smaller app** | Smaller engine than JSC on Android |
| **Consistent behavior** | Same engine on iOS and Android, instead of JSC versions that differ per OS version |

The trade-off: Hermes has **no JIT** (by design). Long-running, CPU-heavy JS loops can be slower than on a JIT engine. Mobile apps rarely do that, and startup and memory matter more on phones.

## Checking it's enabled

```ts
const isHermes = () => !!(global as any).HermesInternal;
```

React Native DevTools and the dev menu also show the engine.

It's controlled per platform:

- Android: `hermesEnabled=true` in `android/gradle.properties`.
- iOS: `:hermes_enabled => true` in the `Podfile` (the default).

## Debugging

- **React Native DevTools** (React Native 0.76+), opened from the dev menu or by pressing `j` in Metro, connects directly to Hermes: breakpoints, console, React components, profiler, memory heap snapshots.
- Hermes produces its own **sampling profiles** for CPU profiling.
- In release builds, crash stack traces point to bytecode offsets. Upload **source maps** to your crash reporter (Sentry, Crashlytics) so traces map back to your source files.

## Language support

Hermes supports modern JS (ES2015+ and many newer features), with a few gaps in features apps rarely use (for example, `with` statements are not supported). If a library fails only on Hermes, check its issue tracker for Hermes compatibility.

## Hermes and the New Architecture

Hermes is not the New Architecture, but they work together. The New Architecture's **JSI** ([Old vs new architecture](26-old-vs-new-architecture.md)) is an interface that lets C++ talk to the JS engine directly, and Hermes implements it. That's what allows JS to call native code without going through a bridge.

## Rule of thumb

Hermes is the default engine: it compiles JS to bytecode at build time so the app starts faster and uses less memory. Keep it on, debug with React Native DevTools, and upload source maps for release crash reports.
