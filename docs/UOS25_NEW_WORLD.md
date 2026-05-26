# New-World UOS 25 / deepin 25 Notes

UOS 25 and deepin 25 class LoongArch64 systems are the primary native target for
the new-world `loong64` release line.

## Install SDK

Download the SDK from:

```text
https://github.com/Flutter-Dart-loong64/flutter-loong64-releases/releases
```

Then install it under a user-chosen prefix:

```bash
tar -xf flutter-sdk-linux-loong64-<version>-<revision>.tar.xz -C <install-prefix>
export FLUTTER_ROOT="<install-prefix>/flutter"
export PATH="$FLUTTER_ROOT/bin:$PATH"
```

Enable desktop and Loong64 support:

```bash
flutter config --enable-linux-desktop
flutter config --enable-loong64
flutter --version --suppress-analytics
dart --version
```

Build an application:

```bash
flutter pub get
flutter build linux --release --target-platform linux-loong64
```

The bundle is normally written to:

```text
build/linux/loong64/release/bundle/
```

## Native Source Build Outline

The native source build order is:

1. Build Dart SDK / VM for Loong64.
2. Build Flutter Engine Linux GTK artifacts with the matching Dart revision.
3. Populate Flutter SDK cache artifacts.
4. Run Flutter tool smoke tests.
5. Build real applications and package them.

Engine builds should keep `--enable-fontconfig`. Without fontconfig support,
Chinese font fallback and desktop text rendering may fail or display boxes.

Typical engine build options include:

```bash
./flutter/tools/gn \
  --linux \
  --linux-cpu loong64 \
  --runtime-mode release \
  --enable-fontconfig \
  --no-enable-unittests \
  --no-goma \
  --target-sysroot / \
  --prebuilt-dart-sdk \
  --target-dir linux_release_loong64_gtk \
  --gn-args='system_libdir="lib/loongarch64-linux-gnu" skia_use_vulkan=false shell_enable_vulkan=false impeller_enable_vulkan=false test_enable_vulkan=false'
```

Build the core artifacts:

```bash
ninja -C out/linux_release_loong64_gtk libflutter_linux_gtk.so gen_snapshot
```

## Validation

Validate with more than `flutter --version`.

Recommended checks:

- `flutter doctor -v`
- `flutter create` and `flutter build linux --target-platform linux-loong64`
- a real app bundle run on the desktop session
- `ldd libflutter_linux_gtk.so | grep fontconfig`
- `file` and `readelf -l` on `dart`, `gen_snapshot`, and app executables

