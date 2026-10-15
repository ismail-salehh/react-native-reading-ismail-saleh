# Linking config in React Navigation

Once the OS delivers a URL to the app ([URL schemes and App Links](14-url-schemes-app-links.md), [Universal Links](15-universal-links.md)), something has to turn `myapp://orders/812` into "open the `OrderDetails` screen with `{ orderId: '812' }`". React Navigation does this with a `linking` config: a map from **URL paths to screens**.

## The basic config

```tsx
import { LinkingOptions, NavigationContainer } from '@react-navigation/native';

const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://', 'https://example.com'],
  config: {
    screens: {
      Home: '',
      Products: 'products',
      ProductDetails: 'products/:id',
      Settings: 'settings',
      NotFound: '*',
    },
  },
};

export default function App() {
  return (
    <NavigationContainer linking={linking} fallback={<SplashScreen />}>
      <RootStack />
    </NavigationContainer>
  );
}
```

- **`prefixes`**: the parts of the URL to strip. `myapp://products/42` and `https://example.com/products/42` both become `products/42`.
- **`screens`**: each screen name maps to a path pattern.
- **`:id`**: a path parameter. It becomes `route.params.id`.
- **Query strings** become params too: `products?sort=price` gives `{ sort: 'price' }`.
- **`'*'`**: catch-all for unknown paths, to show a "not found" screen instead of doing nothing.
- **`fallback`**: rendered while the initial URL is being resolved.

## Params are strings

Everything from a URL is a string. `products/42` gives `{ id: '42' }`, not `42`. Convert with `parse`, and convert back with `stringify` (used when React Navigation builds URLs, e.g. on the web):

```ts
ProductDetails: {
  path: 'products/:id',
  parse: { id: Number },
  stringify: { id: (id: number) => String(id) },
},
```

Typing the config with `LinkingOptions<RootStackParamList>` lets TypeScript check the screen names (Day 1: [Typing navigation](../day-1/26-typing-navigation-typescript.md)).

## Nested navigators

The config must mirror the navigator tree. For tabs inside a stack:

```
RootStack
 ├─ Tabs
 │   ├─ Feed
 │   └─ Profile
 └─ OrderDetails
```

```ts
config: {
  screens: {
    Tabs: {
      screens: {
        Feed: 'feed',
        Profile: 'profile/:username',
      },
    },
    OrderDetails: 'orders/:orderId',
  },
},
```

`myapp://profile/sara` now opens `RootStack > Tabs > Profile` with `{ username: 'sara' }`. React Navigation builds the whole state for you, including the parent navigators (Day 1: [Nested navigation](../day-1/24-nested-navigation.md)).

### Back button after a deep link

When a link opens `OrderDetails` directly, the stack has only that screen, so there's no "back". To put a screen underneath, set `initialRouteName` on the navigator in the config:

```ts
config: {
  initialRouteName: 'Tabs',
  screens: { /* ... */ },
},
```

Now back from `OrderDetails` goes to `Tabs`.

## Where URLs come from

By default, React Navigation listens to React Native's `Linking` module:

- **cold start**: `Linking.getInitialURL()`,
- **app already running**: `Linking.addEventListener('url', ...)`.

You can replace these with `getInitialURL` and `subscribe` in the config, for example to also handle notification taps as links ([Deep linking from a notification](19-deep-link-from-notification.md)). The different app states are explained in [Cold start, background app, and already-open app](17-deep-link-app-states.md).

## Testing

```bash
npx uri-scheme open "myapp://products/42" --ios
npx uri-scheme open "myapp://profile/sara" --android
```

Test each path in the config, unknown paths, and wrong params (`products/abc` when `id` should be a number): the screen should handle invalid params, because anyone can craft a URL.

## Links inside the app

The same config lets you navigate by URL from JS, which is useful for server-driven content (a banner with a link):

```tsx
import { useLinkTo } from '@react-navigation/native';

const linkTo = useLinkTo();
linkTo('/products/42');
```

## Rule of thumb

`prefixes` say which URLs are yours, `screens` maps paths to screens with the same nesting as your navigators, params arrive as strings, and a `'*'` route catches the rest. Treat every param as untrusted input.
