# Play Store internal testing (concept)

Before releasing to everyone, you share builds with a small group through Google Play **testing tracks**. Testers install the app from the real Play Store, through the same download and update process as real users, which catches problems a sideloaded APK doesn't.

## The tracks

| Track | Who | Limit | Review | Typical use |
|---|---|---|---|---|
| **Internal testing** | People you add by email | Up to 100 testers | Available in minutes, usually without full review | Team and QA, every build |
| **Closed testing** | Email lists or Google Groups you manage | Larger groups | Reviewed | Beta users, clients, stakeholders |
| **Open testing** | Anyone can join from the Play listing | Optional limit | Reviewed | Public beta |
| **Production** | Everyone | none | Reviewed | Real release, optionally staged (e.g. 10% of users first) |

A build usually **moves up** the tracks: internal -> closed -> production. You can **promote** the same release to the next track without uploading again.

## How internal testing works

```
1. Play Console: create the app (name, default language, free/paid)
2. Testing -> Internal testing -> Testers: add tester emails (Google accounts)
3. Create release -> upload the signed AAB (see 28-signed-aab-release-build.md)
4. Add release notes -> Review -> Start rollout to internal testing
5. Copy the opt-in link and send it to testers
6. Tester opens the link -> "Become a tester" -> Install from Play Store
```

- Testers must use the **Google account** that's on the list, and be signed in to the Play Store with it on the device.
- New builds arrive as normal **updates**, so you can also test the update path from the previous version.
- The first upload also sets things up permanently: the **package name** can never change, and Play App Signing is enrolled.

## Why test through Play and not just send an APK

- The user gets **Play-generated, Play-signed APKs**. App Links, Google sign-in, and Firebase that check the signing certificate behave as in production.
- **Updates** are tested: data migrations, a stored session surviving an update.
- **Pre-launch report**: Play automatically runs the app on real devices and reports crashes, accessibility issues, and screenshots.
- **Android vitals and crash reports** in Play Console start collecting data.

## Requirements before production

Google Play has requirements that are easier to meet early:

- **Store listing**: description, icon, screenshots, feature graphic.
- **App content** forms: privacy policy URL, data safety form (what data the app collects, matching what your code and SDKs really do), content rating, target audience, ads declaration.
- **Permissions** declarations for sensitive ones (background location, SMS, call logs, ...).
- **Target API level** must be recent ([Android runtime permissions](02-android-runtime-permissions.md)).
- **New personal developer accounts** must run a **closed test with at least 12 testers for 14 days** before they can apply for production access. Plan for this, it can't be skipped.

## Internal app sharing (alternative)

**Internal app sharing** gives you a link for a single build, without a track, version rules, or review. Anyone with the link (and enabled in their Play Store settings) can install it. Useful for a quick check of one branch build; internal testing is better for a team's regular builds.

## The iOS equivalent

On iOS the same role is played by **TestFlight**: internal testers (your App Store Connect team, available right away) and external testers (up to 10,000, after a short beta review).

## Rule of thumb

Upload every release candidate to internal testing first and install it from the Play Store on real devices. Promote the same build to closed testing and then production, and fill in the Play Console forms early, especially if your account needs the 14-day closed test.
