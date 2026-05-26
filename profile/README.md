# Flutter-Dart-loong64

`Flutter-Dart-loong64` 是面向 LoongArch64/Loong64 的 Flutter 与 Dart
社区移植项目，目标是在龙芯 Linux 桌面系统上提供可用的原生 Flutter SDK、
Dart SDK、Flutter Engine 和相关 native tooling。

当前项目覆盖三类环境：

- 新世界 LoongArch64：UOS 25、deepin 25、Debian 13 等，Debian 架构名通常
  是 `loong64`；
- 旧世界 LoongArch64：UOS 20 类系统，常见架构名是 `loongarch64`，运行时
  ABI 与新世界不兼容；
- Debian 13 `loong64` QEMU/容器：用于 GitHub Actions、本地模拟、应用打包
  和可重复 CI 验证。

## 主要仓库

| 仓库 | 作用 |
| --- | --- |
| [`flutter`](https://github.com/Flutter-Dart-loong64/flutter) | Flutter framework/tool fork，维护 Linux `loong64` 目标、主机识别、cache/artifact 命名、engine artifact 选择和 native assets 集成。 |
| [`sdk`](https://github.com/Flutter-Dart-loong64/sdk) | Dart SDK 与 Dart VM fork，维护 Loong64 backend、assembler/disassembler、JIT/AOT codegen、runtime entry、snapshot、`dart` 和 `gen_snapshot`。 |
| [`engine`](https://github.com/Flutter-Dart-loong64/engine) | 独立 Flutter Engine fork，用于 Linux GTK Loong64 engine 补丁开发和审查。实际 SDK 发布通常优先使用 `flutter` fork 内的 `engine/src` 以保证 `DEPS` 对齐。 |
| [`native`](https://github.com/Flutter-Dart-loong64/native) | Dart native assets / FFI tooling fork，维护 Loong64 target metadata 和 native asset bundling 支持。 |
| [`flutter-loong64-releases`](https://github.com/Flutter-Dart-loong64/flutter-loong64-releases) | 新世界 `loong64` Flutter SDK、Dart SDK、Engine 发布仓库，包含安装文档、构建文档、历史记录和自动化。 |
| [`flutter-loongarch64-releases`](https://github.com/Flutter-Dart-loong64/flutter-loongarch64-releases) | 旧世界 `loongarch64` Flutter SDK 发布仓库，面向 UOS 20 类系统，包含旧世界补丁和重建记录。 |

`flutter-linglong-store`、`rustdesk`、`mozjpeg`、`mozjpeg-sys`、`nix`、
`mozjpeg-rust` 等仓库用于真实应用和依赖兼容验证。除非 release 仓库明确说明，
它们不属于 Flutter SDK 发布包本体。

## 快速使用

新世界 LoongArch64 系统请从
[`flutter-loong64-releases`](https://github.com/Flutter-Dart-loong64/flutter-loong64-releases/releases)
下载 SDK，加入 `PATH` 后执行：

```bash
flutter config --enable-linux-desktop
flutter config --enable-loong64
flutter build linux --release --target-platform linux-loong64
```

旧世界 UOS 20 类系统请使用
[`flutter-loongarch64-releases`](https://github.com/Flutter-Dart-loong64/flutter-loongarch64-releases/releases)
的旧世界 SDK。新世界和旧世界二进制不能混用。

## 文档

- [中文总览](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/CN_PROJECT_GUIDE.md)
- [项目目标与总体架构](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/PROJECT_OVERVIEW.md)
- [仓库关系说明](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/REPOSITORIES.md)
- [平台和构建矩阵](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/PLATFORM_MATRIX.md)
- [构建路线：UOS 25、UOS 20、Debian 13](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/BUILD_ROUTES.md)
- [依赖源码编译说明](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/DEPENDENCY_BUILDING.md)
- [构建与发布脚本说明](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/BUILD_AND_RELEASE_SCRIPTS.md)
- [应用验证说明](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/APPLICATION_VALIDATION.md)
- [常见问题排查](https://github.com/Flutter-Dart-loong64/.github/blob/main/docs/TROUBLESHOOTING.md)

## 状态

这是实验性的社区移植项目。发布产物用于 LoongArch Linux 桌面开发和验证，
不是 Flutter/Dart 上游官方发布。
