# Project Overview

`Flutter-Dart-loong64` is a community engineering workspace for enabling native
Flutter and Dart support on LoongArch64 Linux.

The work is split because Flutter desktop support depends on several layers:

1. Dart VM support for the target architecture.
2. Dart SDK tools such as `dart`, `frontend_server`, and `gen_snapshot`.
3. Flutter Engine Linux GTK artifacts for the same Dart revision.
4. Flutter framework/tool changes for target detection, cache paths, artifact
   names, and Linux desktop build orchestration.
5. Native assets and package tooling so Flutter applications can bundle native
   dependencies correctly.
6. Real application builds that expose gaps in the SDK, engine, system ABI, and
   third-party dependencies.

The high-level dependency flow is:

```text
native target metadata
  -> Dart SDK / VM Loong64 support
  -> Flutter Engine Loong64 Linux GTK artifacts
  -> Flutter tool cache and target platform support
  -> Flutter SDK release archives
  -> real application package validation
```

## Target Families

LoongArch Linux currently needs two release tracks:

| Track | Common systems | Debian architecture | Runtime ABI | Release repository |
| --- | --- | --- | --- | --- |
| New-world | UOS 25, deepin 25, Debian 13 | `loong64` | LP64D new-world | `flutter-loong64-releases` |
| Old-world | UOS 20 class systems | `loongarch64` | ABI1.0 old-world | `flutter-loongarch64-releases` |

The two tracks are binary-incompatible. A new-world SDK should not be copied to
an old-world system, and an old-world SDK should not be used as a new-world
runtime.

## What Is Considered Core

Core Flutter/Dart enablement lives in:

- `sdk`
- `engine`
- `flutter`
- `native`
- `flutter-loong64-releases`
- `flutter-loongarch64-releases`

Application repositories and dependency forks are validation/support work. They
are useful because they find real-world problems, but they are not part of the
Flutter SDK release payload.

## Current Practical Capability

The current new-world SDK line can build native Linux desktop Flutter
applications for `linux-loong64` when the SDK cache contains matching Dart SDK,
`gen_snapshot`, and Linux GTK engine artifacts.

The old-world SDK line has a separate patch and release flow because old-world
toolchains and runtime ABI details differ. The old-world release repository
records the UOS 20 bring-up process and patch file.

Debian 13 `loong64` container/QEMU builds are useful for CI and package checks,
but they do not replace native UOS/deepin hardware validation for engine and
desktop runtime behavior.

The final verified UOS 25, Debian 13, and UOS 20 build routes are maintained in
[`VERIFIED_BUILD_ROUTES.md`](VERIFIED_BUILD_ROUTES.md).
