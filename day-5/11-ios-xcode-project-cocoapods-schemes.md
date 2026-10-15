# Xcode project, .xcodeproj vs .xcworkspace, Podfile, CocoaPods, schemes (reading only)

The `ios/` folder is a normal native iOS project that Xcode builds. You don't need to write Swift to work with it, but you need to know what each part is so you can follow library install docs and fix build problems.

## The ios/ folder

```
ios/
 ├─ MyApp.xcodeproj/        the app's Xcode project
 ├─ MyApp.xcworkspace/      the project + the Pods project together  ← open this
 ├─ Podfile                 native dependencies (like package.json)
 ├─ Podfile.lock            exact installed versions (like package-lock.json)
 ├─ Pods/                   downloaded dependencies (like node_modules)
 └─ MyApp/
     ├─ AppDelegate.swift   app startup (AppDelegate.mm in older templates)
     ├─ Info.plist          app settings and permission strings (Day 4: Info.plist keys)
     ├─ Images.xcassets     app icon and images
     └─ LaunchScreen.storyboard   the splash shown while the app starts
```

## .xcodeproj vs .xcworkspace

| | `.xcodeproj` | `.xcworkspace` |
|---|---|---|
| What | **One project**: your app's files, targets, build settings | A **container** of several projects |
| Contains | `MyApp` | `MyApp.xcodeproj` + `Pods.xcodeproj` |
| Created by | The React Native template | `pod install` |

**Always open the `.xcworkspace`.** If you open the `.xcodeproj`, Xcode doesn't see the Pods, and the build fails with errors like *"'React/RCTBridgeModule.h' file not found"* or missing modules.

```bash
xed ios        # opens the workspace
```

## Inside the project: targets and build configurations

- **Target**: one thing that gets built. The main one is the app (`MyApp`). There may be others: tests, a Notification Service Extension, widgets.
- **Build configurations**: named sets of build settings. By default `Debug` and `Release`; you can add `Debug-Stage`, `Release-Stage` for environments ([Different bundle IDs](05-bundle-ids-per-environment.md)).
- **Build settings**: bundle ID, version, signing, deployment target (minimum iOS version), and many more, each with a value per configuration.

## CocoaPods and the Podfile

**CocoaPods** is the iOS dependency manager React Native uses. React Native itself and every native library are installed as **pods**.

```ruby
# ios/Podfile (simplified)
platform :ios, min_ios_version_supported

target 'MyApp' do
  config = use_native_modules!          # autolinking: adds every native npm package
  use_react_native!(:path => config[:reactNativePath])
end
```

| Command | Does |
|---|---|
| `pod install` | Installs what the `Podfile` asks for, using the versions in `Podfile.lock`. Run after `npm install` of a native package |
| `pod update` | Updates pods to newer versions and rewrites `Podfile.lock`. **Don't** run it casually |
| `bundle exec pod install` | Runs with the CocoaPods version pinned in the project's `Gemfile`, so everyone uses the same one |

- **Commit** `Podfile` and `Podfile.lock`. `Pods/` is usually ignored and recreated by `pod install`.
- After `npm install` of a native package: `cd ios && pod install`, then rebuild (Day 4: [Installing a native package](../day-4/06-native-packages-autolinking.md)).
- If pods are in a bad state: delete `ios/Pods` and `ios/Podfile.lock`, run `pod install` again. Also try `pod repo update` when a pod version "can't be found".

## Schemes

A **scheme** tells Xcode **what** to build and **which configuration** to use for each action:

| Action | Typical configuration |
|---|---|
| Run (⌘R) | `Debug` |
| Test | `Debug` |
| Profile | `Release` |
| **Archive** (for TestFlight / App Store) | `Release` |

The template has one scheme, `MyApp`. For environments you add one scheme per environment:

```
MyApp-Dev     Run: Debug-Dev     Archive: Release-Dev
MyApp-Stage   Run: Debug-Stage   Archive: Release-Stage
MyApp         Run: Debug         Archive: Release
```

- Mark schemes as **Shared** (Manage Schemes -> Shared), so they're saved in `xcshareddata` and committed. Otherwise only you have them, and CI can't find them.
- Schemes can run **pre-actions**, e.g. copying the right `.env` file ([.env and different API URLs](04-env-api-urls.md)).
- From the command line: `npx react-native run-ios --scheme MyApp-Stage --mode Debug-Stage`.

## Useful when builds break

- Clean build folder: Product -> Clean Build Folder (⇧⌘K).
- Delete derived data (Xcode's build cache): `rm -rf ~/Library/Developer/Xcode/DerivedData`.
- Reinstall pods as above, then rebuild.

## Rule of thumb

`.xcodeproj` is your app's project; `.xcworkspace` combines it with the Pods, so always open the workspace. CocoaPods installs native dependencies from the `Podfile` (commit it and `Podfile.lock`). Build configurations hold the settings per environment, and schemes choose which configuration each action uses.
