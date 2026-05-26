# Debian 13 QEMU / CI Notes

Debian 13 `loong64` is used for repeatable Loong64 CI builds and application
package validation. It is especially useful in GitHub Actions through
`docker/setup-qemu-action` plus a `linux/loong64` container image.

## Purpose

Use Debian 13 QEMU for:

- checking that release SDK archives can be unpacked and used in a clean
  environment;
- building Loong64 Linux desktop app bundles;
- building `.deb` packages in CI;
- catching missing cache/artifact bootstrap steps;
- avoiding dependency on a specific developer workstation.

Do not treat it as a replacement for native GPU/rendering or old-world runtime
tests.

## Flutter SDK Bootstrap Caveat

The Loong64 Flutter SDK is not published by Google's official Flutter infra.
That means a clean Flutter tool may try to download artifacts that do not exist
on official storage, such as:

- `dart-sdk-linux-loong64.zip`
- `linux-loong64` engine artifacts
- `engine_stamp.json` matching a Loong64 engine build

CI scripts must pre-populate or patch the Flutter cache so the tool uses the
published Loong64 SDK artifacts instead of requesting missing official files.

The `flutter-linglong-store` Loong64 CI helper handles this by:

- downloading the SDK from `flutter-loong64-releases`;
- setting `FLUTTER_ROOT`;
- pre-creating cache stamp files;
- writing a matching `engine_stamp.json`;
- copying or reusing `linux-loong64-release` engine artifacts for the build
  modes needed by the app package;
- marking the extracted SDK Git directory as safe in container builds.

## App Build Pattern

Typical CI flow:

```text
checkout source
setup QEMU for loong64
checkout current Loong64 helper scripts
download Flutter Loong64 SDK release
bootstrap Flutter cache
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter build linux --release --target-platform linux-loong64
package bundle / deb
upload or attach artifacts
```

## Known Generator Issue

Some current Dart 3.13 / analyzer / generator combinations expose invalid type
conversion failures under Loong64 QEMU. In `flutter-linglong-store`, the global
Riverpod app state provider was changed to handwritten `NotifierProvider` /
`Provider.autoDispose` declarations so the rest of code generation can run
normally.

This does not mean the SDK release is unusable. It means the app build needed a
source-level generator compatibility fix for this dependency stack.

