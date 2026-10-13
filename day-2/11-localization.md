# Localization: i18next, language switching, RTL support

Localization (i18n) means adapting the app to a user's language and region: translated text, formatted dates/numbers/currency, and right-to-left (RTL) layout for languages like Arabic and Hebrew.

## Install

```bash
npm install i18next react-i18next
npx expo install expo-localization @react-native-async-storage/async-storage
```

(For bare React Native without Expo, use `react-native-localize` instead of `expo-localization`.)

## 1. Translation files

```js
// locales/en.json
{
  "home": {
    "title": "Welcome",
    "greeting": "Hello, {{name}}!"
  },
  "cart": {
    "items_one": "{{count}} item",
    "items_other": "{{count}} items"
  }
}

// locales/ar.json
{
  "home": {
    "title": "مرحبا",
    "greeting": "أهلاً، {{name}}!"
  },
  "cart": {
    "items_zero": "لا عناصر",
    "items_one": "عنصر واحد",
    "items_two": "عنصران",
    "items_few": "{{count}} عناصر",
    "items_many": "{{count}} عنصراً",
    "items_other": "{{count}} عنصر"
  }
}
```

Keys like `items_one` / `items_other` are i18next plural suffixes. Arabic has 6 plural forms, English only 2.

## 2. Initialize i18next

```js
// i18n.js
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import { getLocales } from 'expo-localization';
import en from './locales/en.json';
import ar from './locales/ar.json';

const deviceLanguage = getLocales()[0]?.languageCode ?? 'en';

i18n.use(initReactI18next).init({
  resources: {
    en: { translation: en },
    ar: { translation: ar },
  },
  lng: deviceLanguage,
  fallbackLng: 'en',
  interpolation: { escapeValue: false }, // React already escapes
});

export default i18n;
```

Import it once at the app entry, before rendering: `import './i18n';`

## 3. Use translations in components

```jsx
import { useTranslation } from 'react-i18next';

function Home() {
  const { t } = useTranslation();
  return (
    <>
      <Text>{t('home.title')}</Text>
      <Text>{t('home.greeting', { name: 'Ismail' })}</Text>
      <Text>{t('cart.items', { count: 3 })}</Text>
    </>
  );
}
```

`useTranslation` re-renders the component when the language changes.

## 4. Language switching

```jsx
import AsyncStorage from '@react-native-async-storage/async-storage';
import i18n from '../i18n';

async function changeLanguage(lang) {
  await i18n.changeLanguage(lang);
  await AsyncStorage.setItem('language', lang);
}

<Button title="English" onPress={() => changeLanguage('en')} />
<Button title="العربية" onPress={() => changeLanguage('ar')} />
```

On app start, read the saved language from AsyncStorage and use it instead of the device language, so the user's choice survives restarts. Wait for it before first render (keep the splash screen visible), otherwise the UI flashes in the wrong language.

## 5. RTL support

Switching to Arabic or Hebrew must also flip the layout. React Native handles this through `I18nManager`.

```js
import { I18nManager } from 'react-native';
import * as Updates from 'expo-updates';

const RTL_LANGUAGES = ['ar', 'he', 'fa', 'ur'];

async function changeLanguage(lang) {
  await i18n.changeLanguage(lang);
  await AsyncStorage.setItem('language', lang);

  const isRTL = RTL_LANGUAGES.includes(lang);
  if (I18nManager.isRTL !== isRTL) {
    I18nManager.allowRTL(isRTL);
    I18nManager.forceRTL(isRTL);
    await Updates.reloadAsync(); // layout direction applies only after a reload
  }
}
```

Key points:

- `I18nManager.forceRTL` takes effect **only after the app reloads**. In Expo use `Updates.reloadAsync()`; in bare RN use `react-native-restart`. Note `reloadAsync` doesn't work in Expo Go the same way in every case, test on a dev build.
- Changing only the text language without reloading leaves the layout in the old direction.
- Check `I18nManager.isRTL` to read the current direction.

### Writing RTL-safe styles

React Native flips most of the layout automatically in RTL (`flexDirection: 'row'` reverses, text aligns to the right). Avoid physical directions, use logical ones:

| Avoid | Use |
|---|---|
| `marginLeft`, `marginRight` | `marginStart`, `marginEnd` |
| `paddingLeft`, `paddingRight` | `paddingStart`, `paddingEnd` |
| `left`, `right` (absolute) | `start`, `end` |
| `textAlign: 'left'` | `textAlign: 'auto'` (or omit it) |
| `borderTopLeftRadius` | `borderTopStartRadius` |

Other things to handle:

- **Icons with direction** (back arrows, chevrons) must be flipped manually: `transform: [{ scaleX: I18nManager.isRTL ? -1 : 1 }]`. Icons without direction (a heart, a search glass) should not be flipped.
- **Images** are not flipped automatically.
- **Numbers, phone numbers, and Latin text** inside Arabic text stay left-to-right, this is normal bidi behavior.
- **Navigation**: React Navigation follows `I18nManager.isRTL` for headers and gestures.
- **Animations / swipes / sliders** that move horizontally may need their direction reversed.

## Formatting dates, numbers, currency

Don't build these strings by hand. Use `Intl`:

```js
new Intl.NumberFormat(i18n.language, { style: 'currency', currency: 'USD' }).format(1234.5);
new Intl.DateTimeFormat(i18n.language, { dateStyle: 'long' }).format(new Date());
```

Hermes supports `Intl` on modern React Native versions.

## Tips and pitfalls

- Never concatenate translated strings (`t('hello') + name`). Word order differs between languages, use interpolation (`{{name}}`).
- Never hard-code user-facing text in components. Put it in the JSON files from day one.
- Text length changes a lot between languages (German is long, Chinese is short). Don't use fixed widths on text containers.
- Use namespaced keys (`home.title`) so files stay organized.
- Test both an LTR and an RTL language on every screen before shipping.
- Use TypeScript key typing for `t()` if you can, to catch missing keys at build time.

## Summary

1. `i18next` + `react-i18next` for translations and `useTranslation` in components.
2. Detect the device language with `expo-localization`, save the user's choice in AsyncStorage.
3. Switching language = `i18n.changeLanguage`.
4. RTL = `I18nManager.forceRTL` plus a reload, and logical (`start`/`end`) styles.
