# 中文总览：LoongArch Flutter/Dart 移植说明

本文是 `Flutter-Dart-loong64` 的中文总览，适合第一次了解这个组织的人阅读。
从零编译、局部环境和变量设置见 [`FROM_ZERO_BUILD.md`](FROM_ZERO_BUILD.md)，
依赖源码编译步骤见 [`DEPENDENCY_BUILDING.md`](DEPENDENCY_BUILDING.md)。

## 项目目的

本项目的目的不是单独维护某一个 Flutter 应用，而是补齐 LoongArch64 平台上
Flutter/Dart 原生 Linux 桌面开发所需的整条链路：

- Dart VM 能在 Loong64 上运行；
- Dart SDK 能提供 `dart`、`frontend_server`、`gen_snapshot` 等工具；
- Flutter Engine 能产出 Linux GTK `loong64` engine artifacts；
- Flutter tool 能识别 LoongArch 主机和 `linux-loong64` target platform；
- native assets、FFI、依赖包和应用打包能识别 Loong64 架构；
- 最终能构建、安装、启动真实 Flutter Linux 桌面应用。

## 仓库分工

核心源码仓库：

- `sdk`：Dart SDK / Dart VM。这里是 Loong64 后端的核心，包括汇编器、
  反汇编器、JIT/AOT codegen、runtime entry、snapshot、`dart` 和
  `gen_snapshot`。
- `engine`：独立 Flutter Engine fork，方便维护 engine-only 补丁。
- `flutter`：Flutter framework/tool fork。当前 SDK 发布通常从这个仓库内的
  `engine/src` 构建 engine，避免 Flutter `DEPS` 和 engine revision 错位。
- `native`：Dart native assets / FFI 工具链，维护 Loong64 target metadata 和
  native asset bundling。

发布仓库：

- `flutter-loong64-releases`：新世界 `loong64` 发布仓库，用于 UOS 25、
  deepin 25、Debian 13 等系统。
- `flutter-loongarch64-releases`：旧世界 `loongarch64` 发布仓库，用于 UOS 20
  类系统。
- `.github`：组织首页和总文档仓库，不放 SDK 包。

验证和依赖仓库：

- `flutter-linglong-store`：真实 Flutter 桌面应用，用来验证 SDK、codegen、
  架构识别、`.deb` 打包和新旧世界行为。
- `rustdesk`：大型桌面应用，用来验证 Rust/C/C++/图像编解码/系统依赖。
- `mozjpeg`、`mozjpeg-sys`、`mozjpeg-rust`：图像编解码依赖兼容。
- `nix`：Rust `nix` crate 的 LoongArch syscall/ioctl 常量兼容。

这些应用和依赖仓库是验证工作，不是 Flutter SDK 发布包本体。

## 新世界：UOS 25 / deepin 25 / Debian 13

新世界系统通常特征：

- Debian 架构名：`loong64`
- GNU triplet：`loongarch64-linux-gnu`
- ABI：LP64D
- 动态链接器：`/lib64/ld-linux-loongarch-lp64d.so.1`
- Flutter target platform：`linux-loong64`

新世界用户应使用：

```text
https://github.com/Flutter-Dart-loong64/flutter-loong64-releases
```

基础使用流程：

```bash
tar -xf flutter-sdk-linux-loong64-<version>-<revision>.tar.xz -C <install-prefix>
export FLUTTER_ROOT="<install-prefix>/flutter"
export PATH="$FLUTTER_ROOT/bin:$PATH"

flutter config --enable-linux-desktop
flutter config --enable-loong64
flutter --version --suppress-analytics
dart --version
```

构建应用：

```bash
flutter pub get
flutter build linux --release --target-platform linux-loong64
```

产物通常在：

```text
build/linux/loong64/release/bundle/
```

## 旧世界：UOS 20

旧世界系统通常特征：

- 架构名常见为 `loongarch64`
- 动态链接器通常为 `/lib64/ld.so.1`
- ABI 和新世界不兼容
- 不能直接运行新世界 `loong64` SDK 或新世界应用包

旧世界用户应使用：

```text
https://github.com/Flutter-Dart-loong64/flutter-loongarch64-releases
```

旧世界发布仓库记录：

- old-world engine patch；
- UOS 20 构建过程；
- old-world SDK 包；
- SHA256 校验；
- 已验证的重建说明。

旧世界构建原则：

- 不复用 UOS 25 / Debian 13 新世界二进制；
- `dart`、`gen_snapshot`、`libflutter_linux_gtk.so`、Flutter tool 要保持 revision
  匹配；
- engine 需要旧世界兼容补丁；
- 必须用真实 GTK Flutter 应用验证启动和界面，不只看 `flutter --version`。

## Debian 13 QEMU / CI

Debian 13 `loong64` 容器/QEMU 用于：

- GitHub Actions 上模拟 Loong64；
- 本地 amd64 机器上模拟跑 Loong64 app 构建；
- 验证 SDK release 是否能在干净环境中使用；
- 构建应用 `.deb` 包；
- 避免 CI 依赖某台本地机器。

它不适合替代：

- LoongGPU / OpenGL 实机验证；
- UOS 25 桌面集成验证；
- UOS 20 旧世界 ABI 验证；
- 性能压测。

需要注意：Google 官方 Flutter infra 不提供 Loong64 artifacts，所以 CI 不能
直接按官方 Flutter x64/arm64 的方式下载：

- `dart-sdk-linux-loong64.zip`
- `linux-loong64` engine artifacts
- Loong64 对应的 `engine_stamp.json`

因此 Loong64 CI 脚本需要从 `flutter-loong64-releases` 下载 SDK，并预先填充
Flutter cache、engine stamp 和 engine artifacts。

## 构建脚本原则

源码同步和发布要分开：

- 平时同步 upstream：只 rebase `sdk`、`engine`、`flutter`、`native`；
- 遇到冲突：跳过对应仓库，不强行发布；
- 只有明确的 release 事件或人工确认时才构建发布；
- 上游普通更新不应该自动发布新 SDK 包。

新世界 release 产物通常包括：

- Dart SDK archive；
- Flutter Engine Linux GTK Loong64 archive；
- Flutter SDK archive；
- `SHA256SUMS`；
- release notes / history。

应用仓库里的 `build-loong64-in-container.sh`、`build-linux-bundle.sh`、
`package-deb.sh` 等脚本是应用验证脚本，不是 Flutter SDK 本体构建脚本。

## 常见问题

### Flutter 尝试去 Google 下载 Loong64 包

说明 SDK cache 没有准备完整。需要使用本项目 release 包，并确认：

- `bin/cache/dart-sdk` 存在；
- `bin/cache/artifacts/engine/linux-loong64-release` 存在；
- `engine.stamp` / `engine_stamp.json` 与包内 engine revision 匹配。

### 中文显示方框

优先检查 engine 是否启用了 fontconfig：

```bash
ldd libflutter_linux_gtk.so | grep fontconfig
fc-match sans
```

构建 engine 时应保留 `--enable-fontconfig`。

### `gen_snapshot` 或 AOT 失败

通常是 Dart frontend、`gen_snapshot`、engine artifact revision 不一致。需要
重新构建并确保这些文件来自同一套 Dart/engine revision。

### 新世界包在旧世界不能运行

这是预期情况。新世界和旧世界 ABI 不兼容。请使用
`flutter-loongarch64-releases` 的旧世界 SDK 和旧世界应用包。
