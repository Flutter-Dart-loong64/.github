# 从零开始编译：UOS 25、UOS 20、Debian 13

本文记录三条从零开始的构建路线。最终已验证的发布路线、GN/Ninja 准备方式和构建期
临时修补以 [`VERIFIED_BUILD_ROUTES.md`](VERIFIED_BUILD_ROUTES.md) 为准：

- UOS 25 / deepin 25 新世界原生构建；
- UOS 20 旧世界原生构建；
- Debian 13 `loong64` QEMU/容器构建。

示例命令使用占位目录 `<workspace>`、`<install-prefix>`。不要把个人机器路径、
远程账号、密码或内网地址写进公开文档。

系统依赖、Rust/Cargo/vcpkg、mozjpeg、nix、RustDesk、应用依赖的源码编译步骤见
[`DEPENDENCY_BUILDING.md`](DEPENDENCY_BUILDING.md)。本文只展开 Flutter SDK、
Dart SDK 和 Engine 的主线构建流程。

## 0. 先判断系统类型

在目标机器上执行：

```bash
uname -m
dpkg --print-architecture 2>/dev/null || true
readelf -l /bin/ls | grep interpreter
```

常见结果：

| 系统 | `dpkg --print-architecture` | 动态链接器 | 使用仓库 |
| --- | --- | --- | --- |
| UOS 25 / deepin 25 新世界 | `loong64` | `/lib64/ld-linux-loongarch-lp64d.so.1` | `flutter-loong64-releases` |
| Debian 13 loong64 | `loong64` | `/lib64/ld-linux-loongarch-lp64d.so.1` | `flutter-loong64-releases` |
| UOS 20 旧世界 | `loongarch64` | `/lib64/ld.so.1` | `flutter-loongarch64-releases` |

新世界和旧世界二进制不兼容。不要把 UOS 25 / Debian 13 构建出的 SDK 或应用包
复制到 UOS 20 上运行。

## 0.1 依赖分层

从零构建时建议把依赖分成五层，排查问题也按这个顺序看：

| 层级 | 作用 | 常见内容 |
| --- | --- | --- |
| 基础构建工具 | 拉源码、解压、编译、打包 | `git`、`curl`、`xz-utils`、`zip`、`rsync`、`make`、`pkg-config`、`file` |
| Python/Chromium 工具 | 运行 Dart/Engine 构建脚本 | `python3`、`python3-pip`、`depot_tools`、`generate-ninja` 或 `ninja-build` |
| C/C++ 工具链 | 编译 Dart VM、Skia、Flutter Engine | `gcc`、`g++`、`clang`、`lld`、`cmake`、`ninja` |
| 桌面运行库开发包 | 构建 Linux GTK embedder | `libgtk-3-dev`、`libfontconfig1-dev`、`libgl1-mesa-dev`、`libegl1-mesa-dev`、X11 dev 包 |
| Flutter cache/artifacts | 让 Flutter tool 不去下载官方不存在的 Loong64 包 | `bin/cache/dart-sdk`、`bin/cache/artifacts/engine/linux-loong64-*`、stamp 文件 |

Loong64 官方 Flutter/Dart artifacts 目前不是 Google infra 原生提供，发布构建必须
使用本项目产出的 Dart SDK 和 Engine artifacts 预置 Flutter cache。

## 0.2 通用局部环境变量

下面变量用于统一三条构建路线。实际路径使用自己的工作目录即可，不要写入公开文档
或脚本默认值：

```bash
export WORKSPACE=<workspace>
export FLUTTER_RELEASE_VERSION=<release-version>

export DART_ROOT="$WORKSPACE/dart-sdk"
export FLUTTER_ROOT="$WORKSPACE/flutter"
export ENGINE_SRC="$FLUTTER_ROOT/engine/src"

export DEPOT_TOOLS="$WORKSPACE/tools/depot_tools"
export PUB_CACHE="$WORKSPACE/.pub-cache"

export PATH="$DEPOT_TOOLS:$FLUTTER_ROOT/bin:$FLUTTER_ROOT/bin/cache/dart-sdk/bin:$PATH"
export VPYTHON_BYPASS="manually managed python not supported by chrome operations"
```

变量含义：

- `WORKSPACE`：本次构建的根目录，所有源码、工具和临时产物放在它下面；
- `FLUTTER_RELEASE_VERSION`：发布版本号，例如与上游 Flutter/Dart revision 对齐的
  预发布号；
- `DART_ROOT`：`Flutter-Dart-loong64/sdk` 源码目录；
- `FLUTTER_ROOT`：`Flutter-Dart-loong64/flutter` 源码目录；
- `ENGINE_SRC`：Flutter engine 源码目录，当前 fork 放在 `flutter/engine/src`；
- `DEPOT_TOOLS`：Chromium/Flutter engine 构建工具；
- `PUB_CACHE`：本次构建独立的 Dart pub cache，避免污染用户全局 cache；
- `VPYTHON_BYPASS`：绕开部分 Chromium vpython 环境假设，使用系统 Python。

如果机器需要代理，只在当前 shell 设置，不要提交到仓库：

```bash
export http_proxy=<proxy-url>
export https_proxy=<proxy-url>
export no_proxy=localhost,127.0.0.1
```

如果需要固定工具链，可以额外设置：

```bash
export CC=<cc>
export CXX=<cxx>
export AR=<ar>
export NM=<nm>
export STRIP=<strip>
```

普通新世界构建通常不需要手动设置 `LD_LIBRARY_PATH`。如果必须临时设置，构建结束后
要 `unset LD_LIBRARY_PATH`，避免把错误运行库带进 Flutter tool 或应用打包过程。

## 0.3 Flutter cache 和目标平台约定

Flutter tool 对 LoongArch 的关键目录约定如下：

```text
$FLUTTER_ROOT/bin/cache/dart-sdk
$FLUTTER_ROOT/bin/cache/artifacts/engine/linux-loong64
$FLUTTER_ROOT/bin/cache/artifacts/engine/linux-loong64-debug
$FLUTTER_ROOT/bin/cache/artifacts/engine/linux-loong64-profile
$FLUTTER_ROOT/bin/cache/artifacts/engine/linux-loong64-release
```

说明：

- Flutter tool 的目标平台名统一使用 `linux-loong64`；
- 新世界 Debian/UOS 包名使用 `loong64`；
- 旧世界系统包名常见为 `loongarch64`，但 Flutter tool 仍使用 `linux-loong64`
  target platform；
- `gen_snapshot` 必须来自同一套 Dart SDK/Engine revision；
- `libflutter_linux_gtk.so` 必须链接 `fontconfig`，因此 Engine GN 参数需要
  `--enable-fontconfig`；
- `engine.stamp`、`engine_stamp.json`、`linux-sdk.stamp` 用于阻止 Flutter tool
  到 Google storage 下载不存在的官方 Loong64 artifacts。

## 1. UOS 25 / deepin 25 新世界原生构建

这是新世界 Flutter SDK 的首选构建环境。它适合构建 Dart SDK、Flutter Engine
和完整 Flutter SDK 发布包。

### 1.1 安装依赖

```bash
sudo apt update
sudo apt install -y \
  git curl ca-certificates xz-utils unzip zip rsync \
  python3 python3-pip \
  clang cmake ninja-build pkg-config make \
  gcc g++ dpkg-dev fakeroot file \
  libgtk-3-dev liblzma-dev libstdc++-dev libfontconfig1-dev \
  libgl1-mesa-dev libegl1-mesa-dev \
  libx11-dev libxcursor-dev libxinerama-dev libxi-dev \
  libxrandr-dev libxxf86vm-dev
```

如果发行版包名不同，安装等价的 GTK 3、fontconfig、OpenGL/EGL、X11、Clang、
CMake、Ninja、Python 3 开发工具即可。

### 1.1.1 设置局部环境

```bash
export WORKSPACE=<workspace>
export FLUTTER_RELEASE_VERSION=<release-version>

export DART_ROOT="$WORKSPACE/dart-sdk"
export FLUTTER_ROOT="$WORKSPACE/flutter"
export ENGINE_SRC="$FLUTTER_ROOT/engine/src"
export ENGINE_OUT="$ENGINE_SRC/out/linux_release_loong64_gtk"
export ENGINE_CACHE="$FLUTTER_ROOT/bin/cache/artifacts/engine/linux-loong64-release"

export DEPOT_TOOLS="$WORKSPACE/tools/depot_tools"
export PUB_CACHE="$WORKSPACE/.pub-cache"
export PATH="$DEPOT_TOOLS:$FLUTTER_ROOT/bin:$FLUTTER_ROOT/bin/cache/dart-sdk/bin:$PATH"
export VPYTHON_BYPASS="manually managed python not supported by chrome operations"
```

新世界原生构建默认使用系统 `clang`/`gcc` 即可。只有在系统编译器版本过旧时，
才需要显式设置 `CC`、`CXX`、`AR`、`NM`、`STRIP`。

### 1.2 拉取源码

```bash
mkdir -p "$WORKSPACE"
cd "$WORKSPACE"

git clone https://github.com/Flutter-Dart-loong64/sdk.git "$DART_ROOT"
git clone https://github.com/Flutter-Dart-loong64/flutter.git "$FLUTTER_ROOT"
git clone https://github.com/Flutter-Dart-loong64/flutter-loong64-releases.git flutter-loong64-releases
```

如果需要 Chromium/Flutter engine 依赖工具：

```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git "$DEPOT_TOOLS"
```

Loong64 构建还需要目标系统 native `gn` 和 `ninja`。新世界可由
`generate-ninja` / `ninja-build` 提供；UOS 缺包时在目标系统源码编译 GN。最终要把
`gn` 和 `ninja` 链接到 Dart SDK 的 `buildtools/gn`、`buildtools/ninja/ninja`，
以及 Engine 的 `flutter/third_party/gn/gn`、`third_party/ninja/ninja`。完整命令见
[`VERIFIED_BUILD_ROUTES.md`](VERIFIED_BUILD_ROUTES.md#2-gnninja-准备)。

当前发布构建优先使用 `flutter` fork 中的 `engine/src`，因为它跟 Flutter
framework 的 `DEPS` revision 对齐。把 engine 内的 Dart SDK 指向本地构建的
Dart 源码：

```bash
cd "$ENGINE_SRC/flutter"
rm -rf third_party/dart
ln -s "$DART_ROOT" third_party/dart
```

### 1.3 构建 Dart SDK

```bash
cd "$DART_ROOT"
./tools/build.py \
  -m release \
  -a loong64 \
  --gn-args="use_sysroot=false" \
  create_sdk dartaotruntime gen_snapshot analyze_snapshot
```

构建完成后应存在：

```text
$WORKSPACE/dart-sdk/out/ReleaseLOONG64/dart-sdk
```

复制到 Flutter cache：

```bash
rm -rf "$WORKSPACE/flutter/bin/cache/dart-sdk"
mkdir -p "$WORKSPACE/flutter/bin/cache"
cp -a "$DART_ROOT/out/ReleaseLOONG64/dart-sdk" \
  "$WORKSPACE/flutter/bin/cache/dart-sdk"

"$WORKSPACE/flutter/bin/cache/dart-sdk/bin/dart" --version
```

### 1.4 构建 Flutter Engine Linux GTK

必须保留 `--enable-fontconfig`，否则 UOS/deepin 桌面上中文字体 fallback 容易
异常。

```bash
cd "$ENGINE_SRC"
dart_commit="$(git -C "$DART_ROOT" rev-parse HEAD)"

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
  --gn-args="content_hash=\"$dart_commit\" system_libdir=\"lib/loongarch64-linux-gnu\" skia_use_vulkan=false shell_enable_vulkan=false impeller_enable_vulkan=false test_enable_vulkan=false"

ninja -C out/linux_release_loong64_gtk \
  libflutter_linux_gtk.so \
  gen_snapshot \
  font-subset \
  flutter_patched_sdk \
  sky_engine \
  const_finder
```

检查版本和 fontconfig：

```bash
out="$ENGINE_OUT"
"$out/gen_snapshot" --version
ldd "$out/libflutter_linux_gtk.so" | grep fontconfig
file "$out/libflutter_linux_gtk.so" "$out/gen_snapshot"
```

### 1.5 组装 Flutter SDK cache

```bash
engine_out="$ENGINE_OUT"
engine_cache="$ENGINE_CACHE"

mkdir -p "$engine_cache"
cp -a "$engine_out/libflutter_linux_gtk.so" "$engine_cache/"
cp -a "$engine_out/gen_snapshot" "$engine_cache/"
cp -a "$engine_out/icudtl.dat" "$engine_cache/" 2>/dev/null || true
cp -a "$engine_out/impellerc" "$engine_cache/" 2>/dev/null || true
cp -a "$engine_out/font-subset" "$engine_cache/" 2>/dev/null || true
cp -a "$engine_out/shader_lib" "$engine_cache/" 2>/dev/null || true

if [ ! -d "$engine_cache/flutter_linux" ]; then
  cp -a "$WORKSPACE/flutter/engine/src/flutter/shell/platform/linux/public/flutter_linux" \
    "$engine_cache/flutter_linux"
fi
```

如果需要 debug/profile 目录，可先复制 release artifact 作为工具 cache 的兜底：

```bash
cache_root="$WORKSPACE/flutter/bin/cache/artifacts/engine"
for mode in linux-loong64 linux-loong64-debug linux-loong64-profile; do
  rm -rf "$cache_root/$mode"
  cp -a "$engine_cache" "$cache_root/$mode"
done
```

### 1.6 验证 Flutter tool

```bash
export PATH="$FLUTTER_ROOT/bin:$PATH"

flutter --version --suppress-analytics
dart --version
flutter config --enable-linux-desktop
flutter config --enable-loong64
flutter doctor -v
```

### 1.7 构建测试应用

```bash
cd "$WORKSPACE"
flutter create smoke_app
cd smoke_app
flutter build linux --release --target-platform linux-loong64
file build/linux/loong64/release/bundle/smoke_app
```

真实应用验证建议再构建 `flutter-linglong-store`：

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-linglong-store.git
cd flutter-linglong-store
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter build linux --release --target-platform linux-loong64
```

### 1.8 打包发布 archive

```bash
cd "$WORKSPACE/flutter-loong64-releases"

FLUTTER_ROOT="$WORKSPACE/flutter" \
DART_ROOT="$WORKSPACE/dart-sdk" \
ENGINE_SRC="$WORKSPACE/flutter/engine/src" \
ENGINE_OUT="$WORKSPACE/flutter/bin/cache/artifacts/engine/linux-loong64-release" \
./scripts/package-loong64-release.sh <release-version> "$WORKSPACE/dist/<release-version>"

cd "$WORKSPACE/dist/<release-version>"
sha256sum -c SHA256SUMS
```

发布前检查：

```bash
tar -tJf flutter-sdk-linux-loong64-*.tar.xz | grep 'linux-loong64-release/libflutter_linux_gtk.so'
tar -tJf flutter-sdk-linux-loong64-*.tar.xz | grep 'bin/cache/dart-sdk/bin/dart'
```

不要把临时备份目录、旧 `gen_snapshot` 备份或个人路径打进 SDK 包。

## 2. Debian 13 loong64 QEMU / 容器构建

Debian 13 QEMU 适合 CI 和应用包构建。完整 SDK/Engine 在 QEMU 下能跑，但会比
原生 LoongArch64 硬件慢很多。

### 2.1 amd64 主机准备 QEMU

```bash
docker run --privileged --rm tonistiigi/binfmt --install loong64
docker run --rm --platform linux/loong64 ghcr.io/loong64/debian:trixie uname -m
```

期望输出包含：

```text
loongarch64
```

### 2.2 进入 Debian 13 loong64 容器

```bash
export WORKSPACE=<workspace>
mkdir -p "$WORKSPACE"

docker run --rm -it \
  --platform linux/loong64 \
  -v "$WORKSPACE:$WORKSPACE" \
  -w "$WORKSPACE" \
  ghcr.io/loong64/debian:trixie \
  bash
```

容器内安装依赖：

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y --no-install-recommends \
  ca-certificates curl git python3 python3-pip xz-utils unzip zip rsync \
  python3-httplib2 python3-six \
  build-essential clang cmake generate-ninja ninja-build pkg-config \
  libgtk-3-dev liblzma-dev libfontconfig1-dev \
  libgl1-mesa-dev libegl1-mesa-dev \
  libx11-dev libxcursor-dev libxinerama-dev libxi-dev \
  libxrandr-dev libxxf86vm-dev
```

如果系统提供 Clang 19，优先安装：

```bash
apt-get install -y --no-install-recommends clang-19 llvm-19 lld-19 || true
```

### 2.2.1 容器内环境变量

```bash
export WORKSPACE=<container-workspace>
export FLUTTER_RELEASE_VERSION=<release-version>

export DART_TAG=<upstream-dart-tag>
export RELEASE_ID="$FLUTTER_RELEASE_VERSION"
export BOOTSTRAP_DART_SDK_URL=<existing-loong64-dart-sdk-archive-url>

export DART_ROOT="$WORKSPACE/dart-sdk"
export FLUTTER_ROOT="$WORKSPACE/flutter"
export ENGINE_SRC="$FLUTTER_ROOT/engine/src"

export PUB_CACHE="$WORKSPACE/.pub-cache"
export PATH="$FLUTTER_ROOT/bin:$FLUTTER_ROOT/bin/cache/dart-sdk/bin:$PATH"
```

变量含义：

- `DART_TAG`：用于同步的上游 Dart SDK tag；
- `RELEASE_ID`：本项目发布版本号，通常与 `FLUTTER_RELEASE_VERSION` 相同；
- `BOOTSTRAP_DART_SDK_URL`：已有 Loong64 Dart SDK 压缩包地址，供下一次
  QEMU 构建 bootstrap 使用；
- `PUB_CACHE`：容器内独立 pub cache，CI 可缓存这个目录减少重复下载。

### 2.3 用 release 仓库脚本构建 SDK

`flutter-loong64-releases` 提供 QEMU release 脚本。该脚本用于上游 Dart tag
触发的 Loong64 SDK 构建，需要已有 Loong64 Dart SDK 作为 bootstrap。

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-loong64-releases.git
cd flutter-loong64-releases

export WORKSPACE="$PWD/.work"

./scripts/ci-qemu-loong64-release.sh
```

输出目录通常为：

```text
flutter-loong64-releases/dist/<release-version>/
```

### 2.4 Debian 13 上构建应用包

如果只是验证应用，不需要从零构建 SDK。直接使用
`flutter-loong64-releases` 已发布的 Debian 13 或新世界 SDK。

应用仓库可以使用类似流程：

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-linglong-store.git
cd flutter-linglong-store

export LOONG64_QEMU_IMAGE=ghcr.io/loong64/debian:trixie
export LOONG64_FLUTTER_RELEASE_REPO=Flutter-Dart-loong64/flutter-loong64-releases
export LOONG64_FLUTTER_RELEASE_TAG=<sdk-release-tag>
export LOONG64_FLUTTER_SDK_ARCHIVE=<flutter-sdk-archive-name>
export LINGLONG_RELEASE_SKIP_BUILD_RUNNER=0
export LINGLONG_RELEASE_ALLOW_RIVERPOD_GENERATOR_FAILURE=0

bash build/scripts/build-loong64-in-container.sh \
  --version 3.3.6 \
  --channel nightly
```

该脚本会：

- 进入 `linux/loong64` 容器；
- 下载指定的 Flutter Loong64 SDK release；
- 预置 Flutter cache 和 engine stamp；
- 安装 GTK/CMake/Ninja 等依赖；
- 执行 `flutter pub get`；
- 执行完整 `build_runner`；
- 执行 `flutter build linux --release --target-platform linux-loong64`；
- 生成 bundle 和 `.deb` 包。

应用构建变量含义：

- `LOONG64_QEMU_IMAGE`：`linux/loong64` 容器镜像；
- `LOONG64_FLUTTER_RELEASE_REPO`：下载 Flutter Loong64 SDK 的 release 仓库；
- `LOONG64_FLUTTER_RELEASE_TAG`：SDK release tag；
- `LOONG64_FLUTTER_SDK_ARCHIVE`：release asset 中的 SDK 压缩包文件名；
- `LINGLONG_RELEASE_SKIP_BUILD_RUNNER`：是否跳过 `build_runner`；
- `LINGLONG_RELEASE_ALLOW_RIVERPOD_GENERATOR_FAILURE`：是否允许 Riverpod 生成失败。

### 2.5 Debian 13 注意事项

Debian 13 QEMU 环境里，Flutter tool 不能依赖 Google 官方 infra 下载
Loong64 包，因为官方没有这些 artifacts。脚本必须准备：

- `bin/cache/dart-sdk`
- `bin/cache/artifacts/engine/linux-loong64-release`
- `engine.stamp`
- `engine_stamp.json`
- `linux-sdk.stamp`
- 必要的 `linux-loong64` debug/profile/release cache 目录

如果 Flutter 尝试下载 `dart-sdk-linux-loong64.zip` 或官方
`engine_stamp.json`，说明 cache bootstrap 不完整。

## 3. UOS 20 旧世界原生构建

UOS 20 旧世界需要单独构建，不能复用 UOS 25 / Debian 13 新世界产物。

### 3.1 确认旧世界环境

```bash
uname -m
dpkg --print-architecture 2>/dev/null || true
readelf -l /bin/ls | grep interpreter
```

期望动态链接器：

```text
/lib64/ld.so.1
```

### 3.2 安装依赖

```bash
sudo apt update
sudo apt install -y \
  git curl ca-certificates xz-utils unzip zip rsync \
  python3 python3-pip \
  gcc g++ clang cmake ninja-build pkg-config make \
  libgtk-3-dev liblzma-dev libfontconfig1-dev \
  libgl1-mesa-dev libegl1-mesa-dev \
  libx11-dev libxcursor-dev libxinerama-dev libxi-dev \
  libxrandr-dev libxxf86vm-dev
```

旧世界最终链接可能需要 old-world-capable GCC/binutils。已验证路线使用过
GCC 13.4 + binutils 2.42 来避免 `R_LARCH_B26` 残留重定位导致 GTK engine
初始化卡住。工具链本身的源码下载、编译、安装、wrapper 环境变量和验证步骤见
[`DEPENDENCY_BUILDING.md`](DEPENDENCY_BUILDING.md#3-旧世界-gcc-134--binutils-242-工具链)。

### 3.2.1 设置旧世界局部环境

```bash
export WORKSPACE=<workspace>
export FLUTTER_RELEASE_VERSION=<release-version>

export DART_ROOT="$WORKSPACE/dart-sdk"
export FLUTTER_ROOT="$WORKSPACE/flutter"
export ENGINE_SRC="$FLUTTER_ROOT/engine/src"
export ENGINE_OUT="$ENGINE_SRC/out/linux_release_loong64_gtk_oldworld"
export ENGINE_CACHE_ROOT="$FLUTTER_ROOT/bin/cache/artifacts/engine"
export OLDWORLD_RELEASE_REPO="$WORKSPACE/flutter-loongarch64-releases"

export DEPOT_TOOLS="$WORKSPACE/tools/depot_tools"
export PUB_CACHE="$WORKSPACE/.pub-cache"
export PATH="$DEPOT_TOOLS:$FLUTTER_ROOT/bin:$FLUTTER_ROOT/bin/cache/dart-sdk/bin:$PATH"
export VPYTHON_BYPASS="manually managed python not supported by chrome operations"
export PKG_CONFIG_PATH="/usr/lib/loongarch64-linux-gnu/pkgconfig:/usr/share/pkgconfig"
```

如果系统工具链不能正确链接旧世界 Engine，再把旧世界 GCC/binutils wrapper 放到
`PATH` 最前面：

```bash
export OLDWORLD_TOOLCHAIN=<oldworld-gcc-binutils-wrapper>
export PATH="$OLDWORLD_TOOLCHAIN:$PATH"
export CC=<oldworld-gcc>
export CXX=<oldworld-g++>
export AR=<oldworld-ar>
export NM=<oldworld-nm>
export STRIP=<oldworld-strip>
unset LD_LIBRARY_PATH
```

已验证路线对应：

```bash
export OLDWORLD_TOOLCHAIN=<workspace>/toolchains/gcc-13.4-binutils-2.42-oldworld
export PATH="$OLDWORLD_TOOLCHAIN/bin:$PATH"
export CC="$OLDWORLD_TOOLCHAIN/bin/gcc"
export CXX="$OLDWORLD_TOOLCHAIN/bin/g++"
export AR="$OLDWORLD_TOOLCHAIN/bin/ar"
export NM="$OLDWORLD_TOOLCHAIN/bin/nm"
export RANLIB="$OLDWORLD_TOOLCHAIN/bin/ranlib"
export STRIP="$OLDWORLD_TOOLCHAIN/bin/strip"
unset LD_LIBRARY_PATH
```

旧世界调试时优先检查两件事：

- `readelf -l <binary> | grep interpreter` 是否仍是 `/lib64/ld.so.1`；
- `readelf -r <binary> | grep R_LARCH_B26` 是否还有不该留到动态链接阶段的重定位。

### 3.3 拉取源码和旧世界 release 仓库

```bash
mkdir -p "$WORKSPACE"
cd "$WORKSPACE"

git clone https://github.com/Flutter-Dart-loong64/sdk.git "$DART_ROOT"
git clone https://github.com/Flutter-Dart-loong64/flutter.git "$FLUTTER_ROOT"
git clone https://github.com/Flutter-Dart-loong64/flutter-loongarch64-releases.git "$OLDWORLD_RELEASE_REPO"
```

### 3.4 构建旧世界 Dart SDK

```bash
cd "$DART_ROOT"
./tools/build.py \
  -m release \
  -a loong64 \
  --gn-args="use_sysroot=false" \
  create_sdk dartaotruntime gen_snapshot analyze_snapshot
```

复制到 Flutter cache：

```bash
rm -rf "$WORKSPACE/flutter/bin/cache/dart-sdk"
mkdir -p "$WORKSPACE/flutter/bin/cache"
cp -a "$DART_ROOT/out/ReleaseLOONG64/dart-sdk" \
  "$WORKSPACE/flutter/bin/cache/dart-sdk"
```

### 3.5 应用旧世界 Engine 补丁

```bash
cd "$ENGINE_SRC"
patch -p1 < "$OLDWORLD_RELEASE_REPO/patches/oldworld-loongarch64-engine.patch"
```

该补丁覆盖：

- old-world GCC 构建兼容；
- 静态 GCC runtime 链接；
- 避免依赖 LSX/LASX-only 源码路径；
- Linux GTK `linux-loong64` artifact plumbing；
- 当前 Flutter framework 需要的部分 `dart:ui` ABI 兼容。

### 3.6 配置并构建旧世界 Engine

```bash
cd "$ENGINE_SRC"

python3 ./flutter/tools/gn \
  --linux \
  --linux-cpu loong64 \
  --runtime-mode release \
  --enable-fontconfig \
  --no-enable-unittests \
  --no-goma \
  --no-clang \
  --target-sysroot / \
  --prebuilt-dart-sdk \
  --target-dir linux_release_loong64_gtk_oldworld \
  --gn-args='is_qnx=false system_libdir="lib/loongarch64-linux-gnu" skia_use_vulkan=false shell_enable_vulkan=false impeller_enable_vulkan=false test_enable_vulkan=false glfw_vulkan_library=""'

ninja -C out/linux_release_loong64_gtk_oldworld \
  gen_snapshot \
  libflutter_linux_gtk.so \
  libflutter_engine.so \
  zip_archives/linux-loong64-release/linux-loong64-flutter-gtk.zip \
  zip_archives/linux-loong64-release/artifacts.zip \
  zip_archives/linux-loong64/font-subset.zip
```

如果系统 linker 留下 `R_LARCH_B26` 动态重定位，需要切换到旧世界可用的
GCC/binutils wrapper 后重新链接：

```bash
export PATH="$OLDWORLD_TOOLCHAIN:$PATH"
ninja -C out/linux_release_loong64_gtk_oldworld libflutter_linux_gtk.so
```

检查旧世界解释器和依赖：

```bash
out="$ENGINE_OUT"
file "$out/gen_snapshot" "$out/libflutter_linux_gtk.so"
readelf -l "$out/gen_snapshot" | grep interpreter
ldd "$out/libflutter_linux_gtk.so"
```

解释器应是：

```text
/lib64/ld.so.1
```

### 3.7 组装旧世界 Flutter SDK

```bash
out="$ENGINE_OUT"
cache="$ENGINE_CACHE_ROOT"

for mode in linux-loong64 linux-loong64-profile linux-loong64-release; do
  mkdir -p "$cache/$mode"
  cp "$out/libflutter_linux_gtk.so" "$cache/$mode/libflutter_linux_gtk.so"
  cp "$out/gen_snapshot" "$cache/$mode/gen_snapshot"
  cp "$out/icudtl.dat" "$cache/$mode/icudtl.dat"
done
```

验证 Flutter：

```bash
export PATH="$FLUTTER_ROOT/bin:$PATH"

flutter --no-version-check --version
flutter --no-version-check doctor -v
```

### 3.8 构建旧世界应用

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-linglong-store.git
cd flutter-linglong-store
flutter --no-version-check pub get
dart run build_runner build --delete-conflicting-outputs
flutter --no-version-check build linux --release --target-platform linux-loong64
```

安装/启动测试时要确认 `.deb` 是在旧世界环境构建的，不要使用新世界 UOS 25
或 Debian 13 构建出的包。

### 3.9 打包旧世界 SDK

```bash
cd "$WORKSPACE"
tar -czf flutter-<version>-loongarch64-oldworld-uos20.tar.gz flutter
sha256sum flutter-<version>-loongarch64-oldworld-uos20.tar.gz \
  > flutter-<version>-loongarch64-oldworld-uos20.tar.gz.sha256
```

发布到：

```text
https://github.com/Flutter-Dart-loong64/flutter-loongarch64-releases/releases
```

并同步更新：

- `README.md`
- `BUILDING.md`
- `OLDWORLD_SUPPORT.md`
- `HISTORY.md`
- `patches/oldworld-loongarch64-engine.patch`
