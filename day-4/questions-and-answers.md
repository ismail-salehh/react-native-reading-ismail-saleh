# Day 4: Questions and answers

Short answers to the points that matter. Each section links to the full topic files in [README.md](README.md).

---

## Permissions

**1. What are the permission states, and which ones let you show the system prompt?**
`granted`, `denied`, `blocked`, `limited`, `unavailable`. You can prompt only on `denied` (and `limited`, to ask for more).

**2. What is the difference between `denied` and `blocked`?**
Both mean "can't use it now". With `denied` you can still ask; with `blocked` the OS won't show the prompt again, so only the Settings app can fix it.

**3. How do iOS and Android reach `blocked` differently?**
iOS: after the first "Don't Allow". Android: after the user refuses twice (Android 11+).

**4. Why is the first iOS prompt "precious"?**
It's shown only once. If refused, the app can only send the user to Settings.

**5. What does `limited` mean and how should the UI treat it?**
Partial access (e.g. only selected photos). Treat it as working, with less data. Don't show an error; offer a way to add more.

**6. What's the correct check/request flow?**
`check()` first, `request()` only if `denied`, `openSettings()` if `blocked`.

**7. What happens if you call `request()` on a `blocked` permission?**
Nothing is shown; it returns `blocked` again.

**8. What is a pre-prompt and why use it?**
Your own screen explaining why, shown before the system prompt. If the user says "not now" on it, you haven't spent the real prompt.

**9. When should you ask for a permission?**
In context, at the moment the feature is used (tap "Scan" → ask camera), not on first launch.

**10. Why re-check permissions when the app returns to the foreground?**
The user may have changed it in Settings. Listen to `AppState` and `check()` again.

**11. What should happen when a permission is refused?**
The feature degrades gracefully: type a city instead of location, type a code instead of scanning.

**12. What two things does Android need for a dangerous permission?**
Declared in `AndroidManifest.xml` and requested at runtime.

**13. What happens if a permission is requested but missing from the manifest?**
The request fails immediately with no dialog — a classic "nothing happens" bug.

**14. Normal vs dangerous vs special permissions on Android?**
Normal (`INTERNET`): granted at install. Dangerous (`CAMERA`, `POST_NOTIFICATIONS`): runtime prompt. Special (overlay, exact alarms): user toggles it on a Settings page.

**15. What does `never_ask_again` from `PermissionsAndroid` mean?**
The `blocked` state.

**16. Why check the merged manifest before release?**
Libraries can add permissions through manifest merging. Remove unused ones with `tools:node="remove"`.

**17. What does `maxSdkVersion` on a permission do?**
Limits it to older Android versions (e.g. `READ_EXTERNAL_STORAGE` up to API 32).

**18. Name permissions that changed by Android version.**
`POST_NOTIFICATIONS` (13), `READ_MEDIA_IMAGES` replacing `READ_EXTERNAL_STORAGE` (13), `BLUETOOTH_SCAN/CONNECT` (12), approximate location (12), background location as a separate request (11).

**19. How do you check the Android version in JS?**
`Platform.Version` — on Android it's the API level number.

**20. Why does `targetSdkVersion` matter for permissions?**
New rules usually apply only when you target the new API level. Play forces a recent target, so re-check permission changes each time you raise it.

**21. How does iOS handle permissions instead of a manifest?**
Each protected feature needs an `NS...UsageDescription` string in `Info.plist`, shown inside the prompt.

**22. What happens if the `Info.plist` key is missing?**
iOS terminates the app immediately. No prompt, no catchable error.

**23. Which permission has no `Info.plist` key?**
Notifications — the prompt comes from calling the notifications API.

**24. What makes a good usage description?**
What is used, for which feature, in plain words. Vague strings ("Camera access required") can get the app rejected.

**25. Where do capabilities like Push Notifications or Associated Domains go?**
In the `.entitlements` file via Xcode's Signing & Capabilities, not in `Info.plist`.

**26. Why use `react-native-permissions` over `PermissionsAndroid`?**
One API for both platforms, unified states, and it hides many Android version checks.

**27. Why include only the iOS permission modules you use in `react-native-permissions`?**
Apple rejects apps that link permission APIs they never use.

**28. What can happen to your app when the user changes a permission in Android Settings?**
The process may be restarted, so in-memory state is lost.

---

## Camera, location, photos, microphone, Bluetooth, background

**29. Why turn the camera off (`isActive={false}`) when not focused?**
It drains battery and keeps the camera indicator on.

**30. When don't you need a custom camera screen?**
When you only need to take a photo: use a picker's `launchCamera`.

**31. Foreground vs background location — which should you ask for?**
Foreground ("While using") for almost everything. Background is strictly reviewed and needs a real reason (navigation, fitness).

**32. What is approximate location and how do you design for it?**
Since iOS 14 / Android 12 the user can give only a rough position. A store finder works with it; turn-by-turn doesn't.

**33. What must you do after `watchPosition`?**
`clearWatch` when done, or the GPS keeps running.

**34. Besides the permission, what else can block location or Bluetooth?**
The service being turned off for the whole device.

**35. What is the best way to let users pick photos?**
The system photo picker. On Android 13+ and with iOS `PHPickerViewController`, it needs no library permission at all.

**36. When do you need full photo-library access?**
Only if the app browses the library itself (gallery app, custom grid).

**37. When is the microphone permission needed besides voice recording?**
Any video recording with sound.

**38. What does Bluetooth need on Android 12+ vs 11 and lower?**
12+: `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT`. 11 and lower: `ACCESS_FINE_LOCATION`.

**39. Local vs push notifications?**
Local: created by the app on the device (timers, reminders). Push: sent by a server through APNs/FCM.

**40. What is an Android notification channel?**
Since Android 8, each notification belongs to a channel ("Orders", "Promotions"). Users can disable channels individually; importance is per channel.

**41. What happens to JS when an app goes to the background?**
The OS suspends it soon after; JS stops running.

**42. How do you run continuous background work (music, navigation) on Android?**
A foreground service with a visible notification (and a declared type since Android 14).

**43. Can you rely on iOS `BGTaskScheduler` timing?**
No — the OS decides when, maybe never. Force-quit apps get no background time.

**44. What should you do if something must happen at an exact time?**
Let the server do it and send a push.

---

## Native packages

**45. How do you tell a package has native code?**
It has `ios/`, `android/`, a `.podspec`, or docs that say "pod install" / "rebuild".

**46. What are the steps after installing a native package?**
`npm install`, `cd ios && pod install`, then rebuild the app (`run-ios` / `run-android`), plus any library-specific setup.

**47. What is autolinking?**
At build time, the CLI scans `node_modules` for native packages and adds them to the Podfile and Gradle build automatically (RN 0.60+).

**48. Why isn't a Metro reload enough after installing a native package?**
Metro only sends JS. Native code is compiled into the binary, so a rebuild is required.

**49. What does "TurboModuleRegistry... could not be found" / "Native module cannot be null" usually mean?**
You didn't run `pod install` and rebuild.

**50. What's the troubleshooting order when it still fails?**
Metro `--reset-cache` → `pod install` (delete `Pods`/`Podfile.lock` if needed) → `./gradlew clean` → check library supports your RN version and New Architecture → re-read install docs.

---

## Firebase and FCM

**51. Why do most RN apps use FCM for iOS too?**
One setup for both; FCM forwards to APNs on iOS.

**52. What must match when registering apps in Firebase?**
Android package name = `applicationId`; iOS bundle ID = Xcode's bundle ID.

**53. Where do the config files go?**
`google-services.json` in `android/app/`. `GoogleService-Info.plist` added through Xcode into the app target (copying into the folder isn't enough).

**54. Is `google-services.json` a secret?**
No, it ships in every APK. The backend's **service account key** is the secret and must never be in the app.

**55. What causes "No matching client found for package name"?**
The `package_name` in `google-services.json` doesn't match `applicationId`.

**56. How do you handle dev and prod Firebase projects?**
Android flavors (`src/<flavor>/google-services.json`), or iOS schemes with a build-phase script that copies the right plist.

**57. What is the FCM registration token?**
An address for this app installation on this device. The backend uses it to send pushes.

**58. Does the app send pushes itself?**
No. It gets the token and sends it to the backend; the backend sends via FCM.

**59. When does the token change?**
Reinstall, data cleared, restore on a new device, FCM rotation, `deleteToken()`.

**60. How do you keep the backend's token current?**
Listen to `onTokenRefresh`, and call `getToken()` on every app start after login, sending it if it changed.

**61. Does the token identify the user?**
No, the installation. The backend links token → user at login.

**62. What do you do with the token on logout?**
Tell the backend to unlink it, then `deleteToken()`, so the next user doesn't get the previous user's pushes.

**63. Can a user have several tokens?**
Yes (phone, tablet). The backend sends to all.

**64. What should the backend do when FCM returns `UNREGISTERED`?**
Delete that token — the app was uninstalled.

**65. What emulator is needed to test FCM on Android?**
One with Google Play services.

---

## Notifications: permission, app states, taps

**66. What changed in Android 13 for notifications?**
`POST_NOTIFICATIONS` runtime permission. Without it notifications are silently not shown.

**67. Does an app upgraded from an older version lose notifications on Android 13?**
No, the permission is pre-granted on upgrade. New installs start without it.

**68. Does `messaging().requestPermission()` show the Android 13 prompt?**
No, it's for iOS. Use `PermissionsAndroid` or `requestNotifications()` from react-native-permissions.

**69. Even with permission, what can still hide notifications?**
A disabled channel, the app turned off in Settings, Do Not Disturb, battery savers.

**70. What are the three app states?**
Foreground (on screen), background (process alive, maybe suspended), killed (no process).

**71. Notification message vs data-only message?**
Notification message has `notification` and is shown by the OS when not in foreground. Data-only has only `data` and is shown by nobody — your code decides.

**72. What happens to a notification message in the foreground?**
Not shown automatically. `onMessage` fires and you display it yourself (e.g. Notifee) or show a toast / refetch.

**73. Where must `setBackgroundMessageHandler` be registered and why?**
In `index.js`, outside React, before `registerComponent`. When killed, the OS starts JS only to run this handler; there's no component tree.

**74. Rules for the background handler?**
No UI, no React state, no navigation, keep it short (a few seconds).

**75. What iOS limits apply to data-only messages?**
Need `content-available: 1`, are throttled, and aren't delivered at all if the user force-quit the app.

**76. Which message type should you send for things the user must see?**
A notification message (with `data` for where to go). Data-only is for silent updates that may be dropped.

**77. What type must `data` values be?**
Strings.

**78. How do you detect a notification tap in each state?**
Killed: `getInitialNotification()`. Background: `onNotificationOpenedApp()`. Foreground (Notifee-displayed): `notifee.onForegroundEvent` with `EventType.PRESS`.

**79. What should a notification's `data` carry?**
An id or a link, not the actual data. Fetch fresh data on the screen.

**80. How do you navigate from outside a screen?**
`navigationRef` from `createNavigationContainerRef()`, passed to `NavigationContainer`.

**81. Common tap-handling pitfalls?**
Navigating before the navigator is ready, user not logged in yet, handling the initial notification twice, trusting `data` without validation.

---

## APNs

**82. Does every iOS push go through APNs?**
Yes. FCM is just a middleman that forwards to APNs.

**83. Android pushes work but iOS ones never arrive. First thing to check?**
APNs setup: `.p8` key uploaded to Firebase, Push Notifications and Background Modes → Remote notifications capabilities.

**84. `.p8` key vs `.p12` certificate?**
`.p8`: one key for all apps and both environments, never expires, downloadable once. `.p12`: per app, expires yearly.

**85. What are the two tokens on iOS?**
APNs device token (Apple's address) and the FCM token (what your backend uses). With FCM you only use the FCM token.

**86. Sandbox vs production APNs?**
Sandbox for Xcode development builds, production for TestFlight/App Store. Tokens don't cross environments — a common "works in dev, not TestFlight" cause when sending directly.

**87. What do `content-available` and `mutable-content` do?**
`content-available: 1` wakes the app silently (throttled). `mutable-content: 1` lets a Notification Service Extension modify it (images, decryption).

**88. Is a paid Apple Developer account needed for push?**
Yes.

---

## Deep links

**89. Two kinds of deep links?**
Custom URL scheme (`myapp://...`) and verified https links (App Links on Android, Universal Links on iOS).

**90. Problems with custom schemes?**
Any app can claim them, no fallback if the app isn't installed, often not clickable in email/chat.

**91. When are custom schemes still fine?**
OAuth redirects, links between your own apps, testing.

**92. What does `launchMode="singleTask"` do?**
A link opens the existing app instance instead of a second copy.

**93. How does Android verify an App Link?**
`autoVerify="true"` intent filter + `/.well-known/assetlinks.json` with the package name and signing certificate SHA-256.

**94. Which fingerprint goes in `assetlinks.json` with Play App Signing?**
Google's app signing key (from Play Console), not your upload key. Add the debug key for debug builds.

**95. What happens if App Link verification fails?**
Android shows an "Open with" chooser or opens the browser.

**96. What two pieces do Universal Links need?**
The Associated Domains capability (`applinks:example.com`) and an AASA file at `/.well-known/apple-app-site-association`.

**97. AASA file rules?**
No `.json` extension, HTTPS with a valid cert, `application/json`, no redirects. `appIDs` = `TEAMID.bundle.id`.

**98. Why doesn't typing a Universal Link in Safari open the app?**
Universal Links only trigger on taps, and usually not for same-domain links.

**99. Why might AASA changes not show up?**
Apple caches it via its CDN; iOS fetches at install. Use `?mode=developer` and reinstall.

**100. How do you test deep links?**
`adb shell am start -a android.intent.action.VIEW -d "<url>"`, `xcrun simctl openurl booted "<url>"`, or `npx uri-scheme open`.

---

## React Navigation linking

**101. What do `prefixes` and `screens` do?**
`prefixes` strip the scheme/domain; `screens` maps path patterns to screen names.

**102. What type are URL params?**
Always strings. Use `parse` (e.g. `{ id: Number }`) and `stringify`.

**103. How do you handle unknown paths?**
A `'*'` catch-all route (e.g. `NotFound`).

**104. How must the config be structured with nested navigators?**
It mirrors the navigator tree with nested `screens` objects.

**105. How do you get a back button after a deep link opens a detail screen directly?**
Set `initialRouteName` in the config so a screen sits underneath.

**106. How do you navigate by URL from inside the app?**
`useLinkTo()` → `linkTo('/products/42')`.

**107. Why treat every URL param as untrusted?**
Anyone can craft a URL. Screens must handle invalid params without crashing.

---

## Deep link app states

**108. How does JS get the URL on cold start vs when running?**
Cold start: `Linking.getInitialURL()` (ask once). Running: `Linking.addEventListener('url')` (event).

**109. What's the classic deep link bug?**
Handling only one of the two, so links work cold or warm but not both.

**110. Does React Navigation's `linking` handle both?**
Yes: initial URL builds the initial state; the listener navigates while running.

**111. What's different about each state?**
Cold start: auth state may not be known yet. Background: preserve the user's place, push on top. Already open: user may be on the target screen — `navigate` updates it, `push` duplicates.

**112. Why call `getInitialURL()` only once?**
On Android it can return the old launch URL again after the app is resumed.

**113. How many cases should you test per link?**
Six: killed / background / foreground × logged in / logged out.

---

## Auth links

**114. Why is a link to a protected screen lost when logged out?**
With conditional auth stacks, the protected screen doesn't exist in the navigator.

**115. How do you fix it?**
Save the pending URL (in `getInitialURL`/`subscribe`), and after login call `Linking.openURL(pendingUrl)`.

**116. Why keep the splash screen until auth state is known?**
Otherwise on cold start every link looks like "logged out" while the token is still being read from the Keychain.

**117. Rules for links carrying tokens (reset password, magic link)?**
Use verified https links, not custom schemes. Tokens are short-lived and single-use. Exchange immediately, then navigate with `reset` so the token isn't kept in state. Don't log the URL. Handle "expired" clearly. The website must handle the same link.

**118. How should OAuth login be done?**
System browser (`ASWebAuthenticationSession` / Custom Tabs), not a WebView, with PKCE. Exclude the redirect from your linking config.

**119. What does PKCE protect against?**
A stolen redirect `code` is useless without the `code_verifier`.

---

## Deep linking from notifications

**120. Why route notification taps through the linking config?**
One navigation path for links, emails and notifications: same parsing, auth handling, nested navigators, not-found handling.

**121. How do you wire it up?**
Payload has `data.link`. In `getInitialURL`: fall back to `getInitialNotification()`. In `subscribe`: add `onNotificationOpenedApp`. Foreground Notifee taps: `Linking.openURL(link)`.

---

## JS thread vs UI thread

**122. What runs on each thread?**
UI thread: native views, drawing, scrolling, touches. JS thread: React renders, state, handlers, logic.

**123. What does the user see when each is blocked?**
UI blocked: whole app freezes, ANR. JS blocked: taps don't respond, JS animations stop, but native scrolling continues.

**124. What's the frame budget?**
~16.7 ms at 60 FPS (8.3 ms at 120 Hz). 200 ms of JS work drops ~12 frames.

**125. How do you tell which thread is the problem?**
Perf Monitor: JS FPS dropping = slow JS; UI FPS dropping = heavy native work (big images, too many views).

**126. Why measure in a release build?**
Dev mode is much slower (checks, unoptimized bundle).

**127. How do you keep animations smooth when JS is busy?**
`useNativeDriver: true` (transform/opacity only), Reanimated worklets, Gesture Handler.

**128. Does the New Architecture let you block the JS thread?**
No. The thread model is the same.

---

## Re-renders and lists

**129. When is a re-render actually a problem?**
When many components re-render often (every list row on every keystroke).

**130. Common causes of unnecessary re-renders?**
State too high up, new inline objects/functions to memoized children, one big context, selecting the whole store.

**131. How do you find them?**
React DevTools "Highlight updates" and the Profiler. Measure before adding `memo`.

**132. Why not `ScrollView` + `.map()` for long lists?**
It renders every item at once. `FlatList` renders only items near the screen.

**133. Key FlatList props?**
`keyExtractor`, `getItemLayout` (fixed heights), `initialNumToRender`, `maxToRenderPerBatch`, `windowSize`, `removeClippedSubviews`.

**134. How do you keep list rows cheap?**
`React.memo` rows, memoized `renderItem` and handlers, no formatting in render, small images.

**135. What does FlashList do differently, and what's the catch?**
Recycles row views. Local row state can appear in another item — key it to the item or lift it up.

---

## Images

**136. How much memory does a decoded image use?**
width × height × 4 bytes. A 4000×3000 photo = 48 MB regardless of a 3 MB file size.

**137. Best fix for heavy images?**
Get the right size from the server (CDN `?w=`), using `PixelRatio.getPixelSizeForLayoutSize`, in WebP/AVIF.

**138. Which image component should you use?**
A caching one like `@d11/react-native-fast-image` (maintained fork). The original `react-native-fast-image` is unmaintained.

**139. How is lazy loading done in lists?**
Virtualization handles it. Add placeholders and always give images a fixed size or aspect ratio.

**140. What is `Image.prefetch` for?**
Loading images the user will see next (e.g. the detail image on tap).

---

## Debounce and throttle

**141. Debounce vs throttle?**
Debounce: run once after events stop for N ms ("wait until done"). Throttle: at most once per N ms ("not more often than this").

**142. Which for search-as-you-type? For scroll?**
Search: debounce. Scroll/drag: throttle.

**143. In a search screen, what is debounced — the input or the query?**
Only the query. The input updates on every key so typing feels instant.

**144. Why does `debounce()` inside a component body not work?**
A new debounced function every render. Wrap in `useMemo`/`useRef` and `.cancel()` on unmount.

**145. Should you throttle scroll values that drive an animation?**
No. Use `Animated.event` with native driver or Reanimated's `useAnimatedScrollHandler`.

**146. Best way to prevent a double "Pay" tap?**
Disable the button while the request is pending.

**147. Typical delays?**
Search 250–500 ms, autosave 1–2 s, scroll logic 100–200 ms.

---

## Memory leaks

**148. What is a memory leak in JS?**
A long-lived thing (global listener, timer, module variable) holding a reference that should have been released.

**149. Most common RN leaks?**
Listeners not removed, timers not cleared, sockets/subscriptions not closed, caches that only grow, deep stacks of mounted screens.

**150. What should every subscribing `useEffect` do?**
Return a cleanup that undoes it (`sub.remove()`, `clearInterval`, `unsubscribe()`).

**151. Do screens below the current one in a stack stay mounted?**
Yes. Use `navigate` instead of `push` when a new copy isn't needed.

**152. Is "setState on unmounted component" a real leak?**
Usually not; React 18 removed the warning. Still cancel requests to avoid wasted work.

**153. How do you find leaks?**
Repeat a flow 10–20 times and compare heap snapshots. Memory that keeps stepping up = leak; growing then dropping = normal GC.

---

## Hermes

**154. What is Hermes?**
Meta's JS engine for RN, default since 0.70.

**155. Main idea of Hermes?**
Compiles JS to bytecode at build time, so the phone skips parsing/compiling at launch.

**156. Benefits?**
Faster startup, lower memory, smaller app, same engine on both platforms.

**157. Trade-off?**
No JIT — long CPU-heavy loops can be slower. Rarely matters on mobile.

**158. Why upload source maps for release builds?**
Hermes crash traces point to bytecode offsets; source maps translate them back.

**159. How does Hermes relate to the New Architecture?**
Separate things, but Hermes implements JSI, which the New Architecture is built on.

---

## Old vs New Architecture

**160. How did the old architecture work?**
The bridge: async, batched, JSON-serialized messages through one queue.

**161. Problems with the bridge?**
Always async, serialization cost, single bottleneck, all native modules loaded at startup, no type safety.

**162. What is JSI?**
A C++ API letting JS and native hold references and call each other directly, synchronously if needed. (Bridge = letters; JSI = phone call.)

**163. What is Fabric?**
The new renderer in shared C++. Uses JSI, supports synchronous layout and React concurrent features. The old one was Paper.

**164. What are TurboModules?**
New native modules called through JSI: typed, optionally sync, and lazy-loaded.

**165. What is Codegen?**
Generates the JS↔native glue from a typed TS spec. A mismatch fails the build instead of crashing at runtime.

**166. What is bridgeless mode?**
The bridge is fully removed; everything goes through JSI. Old libraries work via an interop layer.

**167. When did the New Architecture become default / mandatory?**
Default in 0.76; the old one can't be enabled since 0.82.

**168. Explain old vs new in one sentence.**
Old: JS and native sent serialized messages through one async queue. New: they call each other directly through JSI — Fabric for rendering, TurboModules for native APIs, Codegen for types, and no bridge.

**169. Why is the New Architecture faster?**
No serialization, sync calls when needed (no layout jumps), no single bottleneck, lazy modules (faster startup), concurrent rendering (`startTransition`) keeping input responsive.

**170. What doesn't it fix?**
Re-renders, huge images, heavy JS loops, leaks. Slow code is still slow.

---

## Release build and Play Store

**171. APK vs AAB?**
APK installs directly. AAB is the Play publishing format; Play generates optimized per-device APKs. AAB is required for new Play apps.

**172. Upload key vs app signing key?**
You sign with the upload key; Google re-signs with the app signing key it keeps. A lost upload key can be reset.

**173. Where should keystore passwords live?**
`~/.gradle/gradle.properties` or CI secrets — never in git. Back up the keystore.

**174. Most common signing mistake in the default template?**
Release builds are signed with the debug key. Point `release` to `signingConfigs.release`.

**175. What must increase on every upload?**
`versionCode`.

**176. How do you build the AAB?**
`cd android && ./gradlew bundleRelease`.

**177. How do you test a release build?**
`npx react-native run-android --mode release`, or `bundletool` to install the exact AAB.

**178. Things that break only in release?**
`http://` blocked, R8/ProGuard stripping classes, wrong env/API URL, `__DEV__` code skipped, missing assets, SHA-checked services (Google sign-in, App Links, Firebase) without the right fingerprints.

**179. What are the Play testing tracks?**
Internal (≤100, minutes, no full review) → closed → open → production (optionally staged).

**180. Why test through Play instead of sending an APK?**
Play-signed APKs (so SHA-checked features behave like production), real update path, pre-launch report, vitals.

**181. What can never change after the first upload?**
The package name.

**182. What requirement do new personal developer accounts have?**
A closed test with at least 12 testers for 14 days before production.

**183. iOS equivalent of internal testing?**
TestFlight.

---

## Reanimated and Gesture Handler

**184. Main idea of Reanimated?**
Animation code runs on the UI thread, so it stays smooth while JS is busy.

**185. When do you need Reanimated vs plain `Animated`?**
Reanimated for gestures, scroll-driven or complex animations. Simple fades: `Animated` with native driver or layout animations.

**186. What is a worklet?**
A JS function that runs on the UI thread. Can't use React state directly; call back with `scheduleOnRN` (v4) / `runOnJS` (v3).

**187. The three building blocks?**
`useSharedValue` (doesn't re-render), `useAnimatedStyle`, `Animated.View`.

**188. Common animation functions?**
`withTiming`, `withSpring`, `withDecay`, `withDelay`, `withSequence`, `withRepeat`.

**189. What are layout animations?**
`entering`/`exiting`/`layout` props for mount, unmount and move — no shared values needed.

**190. Which properties should you animate?**
`transform` and `opacity`. `width`/`height`/`top` trigger layout every frame.

**191. Installation gotcha?**
The Babel plugin must be last; rebuild and `--reset-cache`. Reanimated 4 requires the New Architecture.

**192. Why Gesture Handler over `onPress`/`PanResponder`?**
Gestures are recognized natively on the UI thread, can be composed, and with Reanimated the whole interaction skips JS.

**193. What setup is required?**
Wrap the root in `GestureHandlerRootView`. Android `Modal` needs its own.

**194. How do you build and attach a gesture?**
`Gesture.Pan()...` with callbacks (`onStart`, `onUpdate`, `onEnd`), wrapped in `<GestureDetector gesture={...}>`.

**195. `Simultaneous` vs `Exclusive` vs `Race`?**
Simultaneous: both active (pinch + rotate). Exclusive: first has priority, second waits for it to fail (double vs single tap). Race: first to activate wins (pan vs long press).
