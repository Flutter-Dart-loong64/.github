# Old-World UOS 20 Notes

Old-world UOS 20 class systems need their own Flutter SDK release line. Do not
use new-world `loong64` binaries on old-world systems.

Use:

```text
https://github.com/Flutter-Dart-loong64/flutter-loongarch64-releases
```

## How To Identify Old-World

Typical old-world signs:

- system package architecture reports `loongarch64`;
- dynamic linker is usually `/lib64/ld.so.1`;
- new-world binaries built for `/lib64/ld-linux-loongarch-lp64d.so.1` do not
  start correctly;
- old-world linker/toolchain behavior differs for some relocation cases.

Check the interpreter on a binary:

```bash
readelf -l <binary> | grep interpreter
```

## Build Principles

The old-world SDK must be built on old-world-compatible LoongArch64 tooling.
The current old-world release repository records a validated path using an
old-world-capable GCC/binutils pair. The validated final-link toolchain was
GCC 13.4 with binutils 2.42; source build steps and environment variables are
documented in
[`DEPENDENCY_BUILDING.md`](DEPENDENCY_BUILDING.md#3-旧世界-gcc-134--binutils-242-工具链).

Important rules:

- Do not reuse new-world `libflutter_linux_gtk.so`, `gen_snapshot`, or `dart`.
- Keep `gen_snapshot`, Dart frontend, engine, and Flutter tool revision-aligned.
- Apply the old-world engine patch from `flutter-loongarch64-releases`.
- Validate startup with a real GTK Flutter application, not only command-line
  tools.

## Patch Repository

The old-world release repository stores:

- `patches/oldworld-loongarch64-engine.patch`
- `OLDWORLD_SUPPORT.md`
- `BUILDING.md`
- release history and rebuild notes

Apply the patch from an engine checkout root:

```bash
patch -p1 < patches/oldworld-loongarch64-engine.patch
```

## Application Packaging

Old-world applications must be built on old-world systems or with an
old-world-compatible sysroot/toolchain. New-world UOS 25 `.deb` packages are
not expected to run on UOS 20.
