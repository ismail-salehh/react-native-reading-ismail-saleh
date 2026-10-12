## <span style="color:gold;font-weight:bold"> Project Folder Structure </span>

### <span style="color:silver;font-weight:bold">1. What you get from the CLI</span>

when you create a project with *npx @react-native-community/cli init*, you get:

- **android/** and **ios/**: the real native projects (gradle for android, xcode for iOS). you touch them when installing native libraries, changing permissions, icons, app name, etc.
- **index.js**: the entry point, it registers the root component with <span style="color:chocolate">AppRegistry.registerComponent</span>.
- **App.tsx**: the root component, where your app actually starts.
- **app.json**: the app name used by the native side.
- **metro.config.js**: config of *metro*, the bundler that packs your JS for the app.
- **babel.config.js**: config of babel, where plugins like reanimated's go.
- **package.json**, **tsconfig.json**, **Gemfile** (ruby gems for cocoapods on iOS).

> Also btw, after changing anything inside android/ or ios/ you have to **rebuild** the app. a JS reload only picks up JS changes.

<hr />

### <span style="color:silver;font-weight:bold">2. Organizing your own code</span>

there is no official structure, but a common one is to put everything in a **src/** folder:

```
src/
  components/   reusable UI (Button, Card, ...)
  screens/      one folder or file per screen
  navigation/   navigators and their types
  hooks/        custom hooks
  services/     API calls
  context/      context providers (auth, theme, ...)
  utils/        helper functions
  constants/    colors, sizes, theme
  assets/       images, fonts
  types/        shared TypeScript types
```

for bigger apps, a **feature-based** structure is usually better: src/features/auth/, src/features/cart/, each with its own screens, components, and hooks, so everything about one feature lives together.
