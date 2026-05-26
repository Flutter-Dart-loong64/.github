# Repository Map

## Core Source Forks

| Repository | Upstream | Role |
| --- | --- | --- |
| [`flutter`](https://github.com/Flutter-Dart-loong64/flutter) | `flutter/flutter` | Flutter framework/tool fork. Adds Linux `loong64` target plumbing, host detection, artifact naming, cache layout, and native asset integration. The SDK release archive is assembled from this tree. |
| [`sdk`](https://github.com/Flutter-Dart-loong64/sdk) | `dart-lang/sdk` | Dart SDK and Dart VM fork. Carries Loong64 backend work, assembler/disassembler, JIT/AOT code generation, runtime entries, snapshot tooling, and architecture selection. |
| [`engine`](https://github.com/Flutter-Dart-loong64/engine) | `flutter/engine` | Standalone Flutter Engine fork. Used for engine-only patch review and development. Release builds may use `flutter/engine/src` from the Flutter fork to keep `DEPS` aligned. |
| [`native`](https://github.com/Flutter-Dart-loong64/native) | `dart-lang/native` | Native assets and FFI tooling fork. Tracks Loong64 target names and bundling behavior needed by Dart and Flutter tools. |

## Release Repositories

| Repository | Target | Role |
| --- | --- | --- |
| [`flutter-loong64-releases`](https://github.com/Flutter-Dart-loong64/flutter-loong64-releases) | New-world `loong64` | Publishes Flutter SDK, Dart SDK, and Flutter Engine archives for UOS 25/deepin 25/Debian 13 class systems. Contains build docs, release history, and automation. |
| [`flutter-loongarch64-releases`](https://github.com/Flutter-Dart-loong64/flutter-loongarch64-releases) | Old-world `loongarch64` | Publishes old-world SDK archives for UOS 20 class systems and stores the old-world engine patch and rebuild notes. |
| [`.github`](https://github.com/Flutter-Dart-loong64/.github) | Documentation | Organization profile, repository map, build matrix, and public project explanation. |

## Validation and Dependency Forks

| Repository | Role |
| --- | --- |
| [`flutter-linglong-store`](https://github.com/Flutter-Dart-loong64/flutter-linglong-store) | Real Flutter Linux desktop application used to validate SDK packaging, `linux-loong64` builds, Debian packaging, new-world/old-world architecture mapping, and CI package builds. |
| [`rustdesk`](https://github.com/Flutter-Dart-loong64/rustdesk) | Large Rust/Flutter-adjacent desktop application used to validate Rust, C/C++, multimedia, packaging, and LoongArch dependency issues. |
| [`mozjpeg`](https://github.com/Flutter-Dart-loong64/mozjpeg) | JPEG codec fork used by dependent desktop applications. |
| [`mozjpeg-sys`](https://github.com/Flutter-Dart-loong64/mozjpeg-sys) | Rust FFI bindings for mozjpeg, including LoongArch build/runtime compatibility work. |
| [`mozjpeg-rust`](https://github.com/Flutter-Dart-loong64/mozjpeg-rust) | Rust wrapper around mozjpeg. |
| [`nix`](https://github.com/Flutter-Dart-loong64/nix) | Rust `nix` fork for LoongArch syscall/ioctl constant support gaps. |

## Relationship Between `engine` and `flutter/engine/src`

There are two engine locations in normal development:

- `engine`: an independent fork of `flutter/engine`.
- `flutter/engine/src`: the engine checkout nested inside the Flutter SDK fork.

The standalone `engine` repository is useful for reviewing and maintaining
engine patches. For SDK releases, the nested `flutter/engine/src` checkout is
often preferred because Flutter pins a specific engine revision through `DEPS`.
Building from the nested checkout reduces the risk of mixing framework, engine,
Dart SDK, and `gen_snapshot` revisions.

Do not mix artifacts from unrelated commits. `dart`, `frontend_server`,
`gen_snapshot`, `libapp.so`, and `libflutter_linux_gtk.so` must come from a
compatible Dart/engine revision set.

