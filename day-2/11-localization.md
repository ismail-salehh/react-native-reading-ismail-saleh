# Localization: i18next, language switching, RTL support

Localization in React Native happens in **three separate layers**, each run by a different engine:

| Layer | Question it answers | Engine |
|---|---|---|
| **Strings** | Which text to show for this key in this language? | **i18next** (pure JS, on the JS thread) |
| **Locale data** | How is `1234.5` or a date written in this locale? Which plural form applies? | **`Intl`** in **Hermes**, backed by the OS (ICU on Android, Foundation on iOS) |
| **Direction** | Does the layout run left-to-right or right-to-left? | **I18nManager** (native) + **Yoga** (C++ layout) + the platform **text engines** |

Most localization bugs come from assuming that changing one layer changes the others.

## 1. Strings: how i18next resolves `t('cart.items', { count: 3 })`

i18next is a JS object graph. Its main parts:

- **ResourceStore**: a nested object, `language -> namespace -> keys`. With bundled JSON files, every translation is loaded into the Hermes heap at startup.
- **Translator**: resolves a key.
- **LanguageUtils**: builds the **fallback chain**.
- **PluralResolver** and **Interpolator**: pick the plural form and fill in variables.

Resolving a key goes like this:

1. **Split the key.** `ns:` prefix → namespace (default `translation`). `.` → path inside the object: `cart` → `items`.
2. **Build the fallback chain** from the current language: `ar-EG` → `['ar-EG', 'ar', 'en']` (`fallbackLng` last). Each language is tried in order until the key is found.
3. **Plurals.** If `count` is passed, the PluralResolver calls **`new Intl.PluralRules(lng).select(count)`**. It returns a **CLDR plural category**: `zero`, `one`, `two`, `few`, `many` or `other`. i18next appends it to the key: `items` + `_few` → `items_few`.
   - English has 2 categories (`one`, `other`). Arabic has 6, and `select(3)` returns `few`, `select(11)` returns `many`. The rules come from the **Unicode CLDR** data built into the platform, not from i18next.
   - If the JS engine lacks `Intl.PluralRules`, i18next can't pick plural forms and logs a warning. Older Hermes builds had this problem, and the fix is the `intl-pluralrules` polyfill.
4. **Interpolation.** A regex replaces `{{name}}` with the value. `escapeValue: false` is safe in RN because `<Text>` renders a string, not HTML, so there's nothing to inject into.
5. Missing everywhere: it returns the key itself (or calls `missingKeyHandler`).

All of this is plain JS on the JS thread. A lookup is a few property reads and a regex, so it's cheap.

## 2. How a language change reaches the screen

i18next isn't React state. It is an **event emitter**. `react-i18next` connects it to React:

```
i18n.changeLanguage('ar')
  -> load resources for 'ar' (instant if bundled; async if using a backend)
  -> i18n.language = 'ar'
  -> emit('languageChanged', 'ar')
        -> every mounted useTranslation() has a listener (bound via i18n.on)
        -> listener calls its internal setState
        -> React re-renders those components; they get a NEW `t` bound to 'ar'
```

Consequences:

- **Only components that call `useTranslation`** (or `withTranslation` / `<Trans>`) update. Text produced by a direct `i18n.t(...)` call in module scope, a constant, or a navigation `options` object built once is **frozen in the old language**.
- `t` gets a new identity after a language change. Memoized children that receive `t` as a prop will re-render, which is what you want.
- If a namespace isn't loaded yet, `useTranslation` **suspends**: it throws a Promise, so React shows the nearest `<Suspense>` fallback until the namespace loads. With bundled JSON this never happens.

## 3. Finding the device language

`expo-localization`'s `getLocales()` (or `react-native-localize` in bare RN) calls a **native module** that reads the OS's ordered language preferences:

- **iOS**: `Locale.preferredLanguages` (Foundation).
- **Android**: the configuration's `LocaleList`.

It returns tags like `ar-EG`, plus region info. It's synchronous because the native side answers right away.

A choice the user makes **inside the app** has to be stored by you, usually in **AsyncStorage** (a file on iOS, SQLite on Android, see [09](09-redux-toolkit-or-zustand.md)). AsyncStorage reads are **asynchronous**, so on a cold start the saved language arrives *after* JS has started. If you render before it arrives, the first frame uses the device language and then visibly switches. That's why you keep the splash screen up until the saved language is read and `changeLanguage` has resolved.

## 4. Locale data: `Intl` inside Hermes

`Intl.NumberFormat`, `Intl.DateTimeFormat`, `Intl.PluralRules` and `Intl.Collator` are built into the JS engine. Hermes doesn't ship its own copy of the locale data. It **delegates to the platform**:

- **Android**: ICU, through `android.icu` / Java text APIs.
- **iOS**: Foundation formatters (`NumberFormatter`, `DateFormatter`, ...).

Consequences:

- The app bundle stays small, because the locale data comes from the OS.
- Output can differ a little **between OS versions and between iOS and Android** (spacing characters, Arabic-Indic vs Latin digits, currency placement). Test formatted strings on both platforms instead of hard-coding expected values.
- Pass `i18n.language` as the locale so the formatting follows the in-app language, not only the device's.

## 5. RTL: what actually flips the layout

Text language and layout direction are **separate systems**. i18next knows nothing about direction.

### Who decides "is this app RTL?"

**I18nManager** is a native module. At startup, native code computes:

```
isRTL = allowRTL && (forceRTL || device/app preferred language is RTL)
```

- `I18nManager.forceRTL(true)` / `allowRTL(...)` **write a flag to persistent native storage**: `NSUserDefaults` on iOS, `SharedPreferences` on Android. The flag is read when the app process starts.
- `I18nManager.isRTL` is a **constant** that the native module exported to JS when it was initialized. It doesn't change during the app's life.
- On iOS, the "app preferred language" is only Arabic if the app **declares an Arabic localization**. Without that, an Arabic device can still launch your app LTR. On Android, the manifest needs `android:supportsRtl="true"`, which RN templates set. Expo exposes both through app config (`supportsRTL`).

### How the direction reaches layout: Yoga

**Yoga** is the C++ flexbox engine that turns styles into frames for the **Fabric shadow tree**. Each Yoga node has a **direction** (`inherit`, `ltr`, `rtl`), and the root node gets it from `isRTL` when the React Native surface is created. During layout, Yoga then:

- flips the main axis of `flexDirection: 'row'`, so children are placed right to left,
- resolves **logical edges**: `marginStart`/`End`, `paddingStart`/`End`, `start`/`end`, `borderTopStartRadius`, to the physical right or left,
- positions absolutely placed children by `start`/`end`.

RN by default also **swaps `left`/`right` style props in RTL** (`I18nManager.swapLeftAndRightInRTL`, on by default). Logical props still say what you mean explicitly and don't depend on that flag, so use them.

### Why switching direction needs a reload

The direction is fixed when the process starts and the root surface is laid out. The `isRTL` constant has already been handed to JS, the root Yoga node already has its direction, and native containers (navigation controllers, Android views with `layoutDirection`) were already created with it. `forceRTL` only changes the **stored flag for the next start**. That's why you reload after calling it: `Updates.reloadAsync()` in Expo, or `react-native-restart` in bare RN. A JS-only reload doesn't always re-read native constants, so check on a real build, not just Expo Go.

### What Yoga doesn't flip

Yoga computes **boxes** (position and size). It doesn't know what's drawn inside them:

- **Text**: inside a `<Text>`, character order and alignment come from the platform text engine (CoreText/TextKit on iOS, Android's text layout with ICU BiDi). They apply the **Unicode Bidirectional Algorithm**, which is why numbers, phone numbers and Latin words in Arabic text stay left-to-right. `textAlign: 'auto'` maps to the platform's "natural" alignment, which follows the paragraph direction. `textAlign: 'left'` forces a physical side.
- **Images and icons**: they are bitmaps or glyphs. Yoga moves the box but never mirrors the pixels. Icons that point in a direction (back arrow, chevron) need `transform: [{ scaleX: I18nManager.isRTL ? -1 : 1 }]`. Icons without a direction (heart, search) must not be flipped.
- **Transforms and animations**: `translateX` is a physical value. A slide-in from the "start" side needs its sign inverted when `isRTL` is true.
- **Navigation**: native-stack uses `UINavigationController` (iOS) and Fragments (Android), which follow the native layout direction. Header placement and the back-swipe gesture flip on their own.

## 6. Full picture of one language switch to Arabic

```
user taps "العربية"
 -> i18n.changeLanguage('ar')           JS: emitter -> useTranslation listeners -> re-render with Arabic strings
 -> AsyncStorage.setItem('language')   native: write to file / SQLite
 -> I18nManager.forceRTL(true)         native: write flag to NSUserDefaults / SharedPreferences
 -> reload
      native start: read flag -> isRTL = true -> export constant
      Yoga root direction = RTL -> rows reversed, start/end resolved to the right
      JS start: read saved language -> init i18next with 'ar' -> hide splash
```

## Summary

1. **Strings**: i18next resolves keys through a fallback chain and picks plural forms with `Intl.PluralRules` (CLDR). React updates through a `languageChanged` event that `useTranslation` listens to.
2. **Locale data**: `Intl` in Hermes, backed by OS ICU/Foundation. Pass the app's language explicitly.
3. **Direction**: a native flag read at process start, applied by Yoga to boxes and by platform text engines to text. Changing it needs a restart. Use logical (`start`/`end`) styles, and mirror icons and animations yourself.
