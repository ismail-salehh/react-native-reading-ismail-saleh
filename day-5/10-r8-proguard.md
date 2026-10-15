# R8 / ProGuard

**R8** is the tool that shrinks and obfuscates the **native (Java/Kotlin) code** of an Android release build. It replaced **ProGuard**, but it reads the same rule files, which is why both names are still used (`proguard-rules.pro`, "ProGuard rules").

It doesn't touch your JavaScript. JS is minified by Metro and compiled to bytecode by Hermes.

## What it does

| Step | Effect |
|---|---|
| **Shrinking** | Removes classes and methods that nothing calls (including unused parts of libraries) |
| **Obfuscation** | Renames classes and methods to short names (`com.lib.UserManager` → `a.b.c`) |
| **Optimization** | Inlines and simplifies code |
| **Resource shrinking** (`shrinkResources true`) | Removes images and layouts no code references |

Result: a smaller app that is harder to reverse engineer.

## Turning it on

In `android/app/build.gradle`, the template has it **off** by default:

```gradle
def enableProguardInReleaseBuilds = true   // template default is false

android {
    buildTypes {
        release {
            minifyEnabled enableProguardInReleaseBuilds
            shrinkResources enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
        }
    }
}
```

It runs only for release builds. Debug builds are never minified, which is why R8 problems appear **only in release**.

## The problem: code R8 can't see being used

R8 decides what is used by reading the code. It can't see code used **by name at runtime**:

- **Reflection**: a library looks up a class from a string.
- **JSON parsing into classes** (Gson, Moshi without codegen): field names must stay the same.
- **JNI**: C++ code calls Java methods by name.
- Classes named only in XML or config.

R8 removes or renames them, and the app crashes in release:

```
java.lang.ClassNotFoundException: com.somelib.Something
java.lang.NoSuchMethodError: ...
```

## Keep rules

You tell R8 what to leave alone in `android/app/proguard-rules.pro`:

```proguard
# keep these model classes and all their members (used by Gson)
-keep class com.app.models.** { *; }

# keep classes that have this annotation
-keep @com.somelib.Keep class * { *; }

# needed by some libraries for annotations and generics
-keepattributes *Annotation*, Signature

# a library references an optional class that isn't there; it's safe to ignore
-dontwarn com.somelib.optional.**
```

- Most libraries ship **consumer rules** that Gradle applies automatically. React Native's own rules are included too.
- When a library's docs list ProGuard rules, copy them into `proguard-rules.pro`.
- Only use `-dontwarn` when the library docs say the missing class is optional. It hides the warning, it doesn't fix anything.
- Don't "fix" problems with `-keep class ** { *; }`. That disables most of what R8 does.

## Stack traces after obfuscation

Crash reports from release builds show the renamed names (`a.b.c.d`). R8 writes a **mapping file** that reverses them:

```
android/app/build/outputs/mapping/release/mapping.txt
```

- Google Play: upload it with the release (Play Console can take it with the AAB) to see readable crashes in Android vitals.
- Crashlytics / Sentry: their Gradle plugins upload it automatically when configured.
- Keep it for every released version. It's different each build.

This is the native equivalent of JS **source maps** (Day 4: [Hermes](../day-4/25-hermes.md)).

## Finding R8 problems

1. Build and run the release variant (`run-android --mode release`) and use every main screen.
2. On a crash, read `adb logcat` for `ClassNotFoundException` / `NoSuchMethodError` and the class name.
3. Check that library's docs for ProGuard rules, add them, rebuild.
4. To see what R8 removed: `app/build/outputs/mapping/release/usage.txt`.

## Rule of thumb

R8 shrinks and renames native code in release builds; ProGuard rules tell it what to keep. Turn it on, add the keep rules libraries document, test the release build, and keep each version's `mapping.txt` so crash reports are readable.
