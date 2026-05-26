# Troubleshooting

## Flutter Tries To Download Official Loong64 Artifacts

Symptom:

```text
404 from storage.googleapis.com/flutter_infra_release
dart-sdk-linux-loong64.zip is missing
engine_stamp.json is missing
```

Cause:

Google's official Flutter infra does not publish Loong64 artifacts for this
community port.

Fix:

- Use an SDK archive from `flutter-loong64-releases`.
- Ensure `bin/cache/dart-sdk` exists.
- Ensure `bin/cache/artifacts/engine/linux-loong64-release` exists.
- Ensure Flutter cache stamp files match the packaged engine revision.

## Chinese Text Renders As Boxes

Common causes:

- engine was built without `--enable-fontconfig`;
- target system lacks usable CJK fonts;
- runtime cannot load fontconfig libraries.

Checks:

```bash
ldd libflutter_linux_gtk.so | grep fontconfig
fc-match sans
```

Engine release builds should keep `--enable-fontconfig`.

## `gen_snapshot` Or AOT Build Fails

Common cause:

`gen_snapshot`, Dart frontend, and Flutter/engine artifacts are from mismatched
revisions.

Fix:

- rebuild Dart SDK and engine from matching revisions;
- do not mix `gen_snapshot` from one SDK with `libflutter_linux_gtk.so` from
  another engine revision;
- rebuild the Flutter SDK cache after replacing engine artifacts.

## New-World Binary Fails On Old-World System

Common cause:

new-world and old-world LoongArch runtimes are ABI-incompatible.

Check:

```bash
readelf -l <binary> | grep interpreter
```

Use `flutter-loongarch64-releases` for old-world systems.

## `build_runner` Fails In Loong64 QEMU

Some generator/analyzer combinations may fail under Loong64 QEMU even when the
SDK is otherwise usable.

Recommended approach:

- run the failing generator locally in the same `linux/loong64` container;
- identify the source file/provider/model that produces an invalid analyzer
  type;
- prefer a small source compatibility fix over checking in generated files;
- keep full `build_runner` enabled so other generated files are not skipped.

## OpenGL Or GPU Startup Is Black

First determine whether the issue is Flutter tool, engine, driver, or desktop
session:

```bash
glxinfo -B
ldd libflutter_linux_gtk.so
FLUTTER_LINUX_RENDERER=software <app>
FLUTTER_LINUX_RENDERER=opengl <app>
```

QEMU/container builds cannot validate LoongGPU acceleration. Test GPU behavior
on native hardware with a real desktop session and working display stack.

## Debian Package Installs But App Does Not Launch

Check:

- package architecture (`dpkg --print-architecture`, `dpkg -I package.deb`);
- runtime linker with `readelf -l`;
- missing shared libraries with `ldd`;
- desktop session logs;
- application coredumps;
- whether the package was built for new-world or old-world.

