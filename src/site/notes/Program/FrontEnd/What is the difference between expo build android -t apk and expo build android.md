---
{"dg-publish":true,"permalink":"/Program/FrontEnd/What is the difference between expo build android -t apk and expo build android/","noteIcon":"","created":"2025-03-06T21:28:25.973+08:00"}
---

If you type the following you will get some more details:

```bash
$ expo build:android --help

  Usage: build:android|ba [options] [project-dir]

  Build a standalone APK or App Bundle for your project, signed and ready for submission to the Google Play Store.

  Options:

    -c, --clear-credentials           Clear stored credentials.
    --release-channel <channel-name>  Pull from specified release channel. (default: default)
    --no-publish                      Disable automatic publishing before building.
    --no-wait                         Exit immediately after triggering build.
    --keystore-path <app.jks>         Path to your Keystore.
    --keystore-alias <alias>          Keystore Alias
    --generate-keystore               Generate Keystore if one does not exist
    --public-url <url>                The URL of an externally hosted manifest (for self-hosted apps)
    -t --type <build>                 Type of build: [app-bundle|apk]. (default: apk)
    --config [file]                   Specify a path to app.json
    -h, --help                        output usage information
```

The `-t` option allows you to build either an APK or an App Bundle. App bundles are the new way of doing things. App bundles allow the Google Play Store to provide the phone/tablet with a version of the app customized for the device’s processor. This means the package that the device downloads is smaller so takes less data to download and less storage space.

APKs, on the other hand, have to bundle e.g. 32 bit code and 64 bit code for both ARM and x86 in order to be installable on all devices.

According to the help output above, the default is to build an APK, so there is no difference between `expo build:android` and `expo build:android -t apk`. So it’s a bit strange for the docs to imply there is a difference !

Ideally you should build an app bundle, but an APK will also work. The app bundle will be as large as or larger than the APK, but when the device downloads it a customized version will be generated on the fly by the Google Play Store. 
 [https://forums.expo.dev/t/what-is-the-difference-between-expo-build-android-t-apk-and-expo-build-android/30607](https://forums.expo.dev/t/what-is-the-difference-between-expo-build-android-t-apk-and-expo-build-android/30607)
