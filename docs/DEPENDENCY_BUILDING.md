# 依赖源码编译说明

本文记录 LoongArch Flutter/Dart 移植过程中常用依赖的源码编译方法。命令使用
`<workspace>`、`<install-prefix>`、`<release-version>` 这类占位符，不包含个人
机器路径、账号、密码、内网地址或代理订阅。

## 1. 通用本地环境

建议所有源码依赖都放在同一个工作目录内，避免污染系统全局环境。下面这种写法适合
CI 或一次性构建：

```bash
export WORKSPACE=<workspace>
export PREFIX=<install-prefix>

export CARGO_HOME="$WORKSPACE/.cargo"
export RUSTUP_HOME="$WORKSPACE/.rustup"
export CARGO_TARGET_DIR="$WORKSPACE/.cargo-target"
export PUB_CACHE="$WORKSPACE/.pub-cache"
export VCPKG_ROOT="$WORKSPACE/vcpkg"
export VCPKG_DEFAULT_BINARY_CACHE="$WORKSPACE/.vcpkg-cache"
export VCPKG_FORCE_SYSTEM_BINARIES=1

export PATH="$CARGO_HOME/bin:$PREFIX/bin:$PATH"
mkdir -p "$WORKSPACE" "$PREFIX" "$CARGO_HOME" "$RUSTUP_HOME" \
  "$CARGO_TARGET_DIR" "$PUB_CACHE" "$VCPKG_DEFAULT_BINARY_CACHE"
```

日常在真实 LoongArch 桌面机器上开发时，Rust 通常不需要重定向到 `WORKSPACE`，
按 rustup 默认值放在用户目录即可：

```text
$HOME/.cargo
$HOME/.rustup
$HOME/.cargo/bin
```

也就是不设置 `CARGO_HOME`、`RUSTUP_HOME` 时，rustup 默认会安装到 `~/` 下。

如果需要代理，只在当前 shell 里设置，不要提交到仓库：

```bash
export http_proxy=<proxy-url>
export https_proxy=<proxy-url>
export no_proxy=localhost,127.0.0.1
```

架构名约定：

| 场景 | 名称 |
| --- | --- |
| Debian/UOS 新世界包架构 | `loong64` |
| UOS 20 旧世界包架构 | `loongarch64` |
| Rust target arch | `loongarch64` |
| Rust GNU target triple | `loongarch64-unknown-linux-gnu` |
| Flutter target platform | `linux-loong64` |

## 2. 系统基础依赖

UOS 25/deepin 25/Debian 13 新世界：

```bash
sudo apt update
sudo apt install -y \
  git curl wget ca-certificates xz-utils unzip zip rsync file \
  python3 python3-pip python3-venv python3-httplib2 python3-six \
  build-essential gcc g++ clang lld cmake ninja-build pkg-config make \
  dpkg-dev fakeroot patchelf \
  libgtk-3-dev liblzma-dev libfontconfig1-dev \
  libgl1-mesa-dev libegl1-mesa-dev \
  libx11-dev libxcursor-dev libxinerama-dev libxi-dev \
  libxrandr-dev libxxf86vm-dev \
  nasm yasm \
  libasound2-dev libpulse-dev libpam0g-dev \
  libxcb-randr0-dev libxdo-dev libxfixes-dev \
  libxcb-shape0-dev libxcb-xfixes0-dev \
  libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
```

UOS 20 旧世界：

```bash
sudo apt update
sudo apt install -y \
  git curl wget ca-certificates xz-utils unzip zip rsync file \
  python3 python3-pip \
  gcc g++ clang cmake ninja-build pkg-config make \
  dpkg-dev fakeroot patchelf \
  libgtk-3-dev liblzma-dev libfontconfig1-dev \
  libgl1-mesa-dev libegl1-mesa-dev \
  libx11-dev libxcursor-dev libxinerama-dev libxi-dev \
  libxrandr-dev libxxf86vm-dev \
  nasm yasm \
  libasound2-dev libpulse-dev libpam0g-dev \
  libxcb-randr0-dev libxdo-dev libxfixes-dev \
  libxcb-shape0-dev libxcb-xfixes0-dev \
  libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
```

旧世界如果系统 GCC/binutils 链接出的 Engine 仍残留不该进入动态链接阶段的
`R_LARCH_B26` 重定位，需要改用 old-world-capable GCC/binutils wrapper：

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

## 3. 旧世界 GCC 13.4 + binutils 2.42 工具链

旧世界 UOS 20 的系统 linker 在最终链接 `libflutter_linux_gtk.so` 时曾留下
不该进入动态链接阶段的 `R_LARCH_B26` 分支重定位，运行时表现为 GTK engine
初始化卡住、窗口停在空白或 10x10 占位窗口。已验证的修复路线是在旧世界系统上
使用 GCC 13.4 driver + binutils 2.42 重新链接最终 Engine 共享库。

这套工具链必须在旧世界系统或旧世界 sysroot 上构建。不要在新世界 UOS 25 /
Debian 13 上构建后复制到 UOS 20 使用。

这条路线主要解决 Engine 最终链接问题。Dart SDK、Flutter tool 和应用仍然要在
同一套旧世界环境里构建，但不应该把新世界 sysroot 或新世界运行库混进来。

### 3.1 环境变量

```bash
export WORKSPACE=<workspace>
export TOOLCHAIN_SRC="$WORKSPACE/toolchain-src"
export TOOLCHAIN_BUILD="$WORKSPACE/toolchain-build"
export OLDWORLD_TOOLCHAIN="$WORKSPACE/toolchains/gcc-13.4-binutils-2.42-oldworld"
export GNU_TRIPLE=loongarch64-unknown-linux-gnu

mkdir -p "$TOOLCHAIN_SRC" "$TOOLCHAIN_BUILD" "$OLDWORLD_TOOLCHAIN"
```

确认当前机器是旧世界：

```bash
uname -m
readelf -l /bin/ls | grep interpreter
```

期望解释器：

```text
/lib64/ld.so.1
```

### 3.2 安装工具链编译依赖

```bash
sudo apt update
sudo apt install -y \
  git curl wget ca-certificates xz-utils bzip2 gzip tar \
  make gcc g++ file texinfo flex bison gawk \
  python3 perl patch diffutils \
  libgmp-dev libmpfr-dev libmpc-dev zlib1g-dev
```

如果发行版没有 `libgmp-dev`、`libmpfr-dev`、`libmpc-dev`，可让 GCC 源码自带脚本
下载匹配依赖，见后面的 `contrib/download_prerequisites`。

### 3.3 下载源码

```bash
cd "$TOOLCHAIN_SRC"

wget https://ftp.gnu.org/gnu/binutils/binutils-2.42.tar.xz
wget https://ftp.gnu.org/gnu/gcc/gcc-13.4.0/gcc-13.4.0.tar.xz

tar -xf binutils-2.42.tar.xz
tar -xf gcc-13.4.0.tar.xz
```

如果系统缺少 GMP/MPFR/MPC 开发包：

```bash
cd "$TOOLCHAIN_SRC/gcc-13.4.0"
./contrib/download_prerequisites
```

### 3.4 编译并安装 binutils 2.42

```bash
mkdir -p "$TOOLCHAIN_BUILD/binutils-2.42"
cd "$TOOLCHAIN_BUILD/binutils-2.42"

"$TOOLCHAIN_SRC/binutils-2.42/configure" \
  --prefix="$OLDWORLD_TOOLCHAIN" \
  --build="$GNU_TRIPLE" \
  --host="$GNU_TRIPLE" \
  --target="$GNU_TRIPLE" \
  --with-sysroot=/ \
  --disable-multilib \
  --disable-werror \
  --enable-plugins \
  --enable-ld=default \
  --enable-gold=no

make -j"$(nproc)"
make install
```

旧世界机器内存较小时，把并发数降到 2 或 4：

```bash
make -j2
make install
```

验证安装结果：

```bash
"$OLDWORLD_TOOLCHAIN/bin/ld" -v
"$OLDWORLD_TOOLCHAIN/bin/as" --version | head -n 1
"$OLDWORLD_TOOLCHAIN/bin/objdump" --version | head -n 1
```

### 3.5 编译并安装 GCC 13.4

GCC 可以完整 bootstrap，也可以为了节省旧世界机器时间使用 `--disable-bootstrap`。
发布用工具链建议尽量做完整 bootstrap；如果只是为了修复 Engine 最终链接，
`--disable-bootstrap` 的 native compiler 也能完成验证。

```bash
mkdir -p "$TOOLCHAIN_BUILD/gcc-13.4.0"
cd "$TOOLCHAIN_BUILD/gcc-13.4.0"

export PATH="$OLDWORLD_TOOLCHAIN/bin:$PATH"

"$TOOLCHAIN_SRC/gcc-13.4.0/configure" \
  --prefix="$OLDWORLD_TOOLCHAIN" \
  --build="$GNU_TRIPLE" \
  --host="$GNU_TRIPLE" \
  --target="$GNU_TRIPLE" \
  --with-sysroot=/ \
  --with-native-system-header-dir=/usr/include \
  --with-arch=loongarch64 \
  --with-abi=lp64d \
  --disable-multilib \
  --enable-languages=c,c++ \
  --enable-shared \
  --enable-threads=posix \
  --enable-__cxa_atexit \
  --enable-libstdcxx-time=yes \
  --disable-libsanitizer \
  --disable-werror

make -j"$(nproc)"
make install
```

旧世界机器内存较小时：

```bash
make -j2
make install
```

如果机器性能不足，可以在 `configure` 参数里加：

```bash
--disable-bootstrap
```

但要在发布记录里写清楚该工具链是否经过 bootstrap。

验证 GCC：

```bash
"$OLDWORLD_TOOLCHAIN/bin/gcc" -v
"$OLDWORLD_TOOLCHAIN/bin/g++" -v
"$OLDWORLD_TOOLCHAIN/bin/gcc" -print-sysroot
"$OLDWORLD_TOOLCHAIN/bin/gcc" -print-prog-name=ld
```

### 3.6 smoke test

```bash
cat > "$WORKSPACE/oldworld-toolchain-smoke.c" <<'EOF'
#include <stdio.h>

int main(void) {
  puts("oldworld toolchain ok");
  return 0;
}
EOF

"$OLDWORLD_TOOLCHAIN/bin/gcc" \
  "$WORKSPACE/oldworld-toolchain-smoke.c" \
  -o "$WORKSPACE/oldworld-toolchain-smoke"

file "$WORKSPACE/oldworld-toolchain-smoke"
readelf -l "$WORKSPACE/oldworld-toolchain-smoke" | grep interpreter
"$WORKSPACE/oldworld-toolchain-smoke"
```

解释器必须仍然是：

```text
/lib64/ld.so.1
```

如果这里出现新世界解释器，说明用了错误 sysroot 或在新世界系统上构建了工具链。

### 3.7 给 Engine 构建使用的变量

```bash
export OLDWORLD_TOOLCHAIN=<workspace>/toolchains/gcc-13.4-binutils-2.42-oldworld
export PATH="$OLDWORLD_TOOLCHAIN/bin:$PATH"

export CC="$OLDWORLD_TOOLCHAIN/bin/gcc"
export CXX="$OLDWORLD_TOOLCHAIN/bin/g++"
export AR="$OLDWORLD_TOOLCHAIN/bin/ar"
export NM="$OLDWORLD_TOOLCHAIN/bin/nm"
export RANLIB="$OLDWORLD_TOOLCHAIN/bin/ranlib"
export STRIP="$OLDWORLD_TOOLCHAIN/bin/strip"
export OBJCOPY="$OLDWORLD_TOOLCHAIN/bin/objcopy"
export OBJDUMP="$OLDWORLD_TOOLCHAIN/bin/objdump"
unset LD_LIBRARY_PATH
```

如果此前已经用系统工具链跑过 GN，切换工具链后要重新生成输出目录：

```bash
cd "$WORKSPACE/flutter/engine/src"
rm -rf out/linux_release_loong64_gtk_oldworld

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
```

推荐完整重建最终产物：

```bash
ninja -C out/linux_release_loong64_gtk_oldworld \
  gen_snapshot \
  libflutter_linux_gtk.so \
  libflutter_engine.so \
  zip_archives/linux-loong64-release/linux-loong64-flutter-gtk.zip \
  zip_archives/linux-loong64-release/artifacts.zip \
  zip_archives/linux-loong64/font-subset.zip
```

如果前面已经完成过大部分编译，只想验证 GCC 13.4 + binutils 2.42 的最终链接，
也可以在重新跑 GN 后清掉共享库目标再构建：

```bash
rm -f out/linux_release_loong64_gtk_oldworld/libflutter_linux_gtk.so
rm -f out/linux_release_loong64_gtk_oldworld/libflutter_engine.so
ninja -C out/linux_release_loong64_gtk_oldworld \
  libflutter_linux_gtk.so \
  libflutter_engine.so
```

发布包构建仍建议完整跑一次 `gen_snapshot`、Engine `.so` 和 zip artifacts，
不要只拿局部 relink 结果直接发布。

### 3.8 验证 Engine 链接结果

```bash
out="$WORKSPACE/flutter/engine/src/out/linux_release_loong64_gtk_oldworld"

file "$out/libflutter_linux_gtk.so" "$out/gen_snapshot"
ldd "$out/libflutter_linux_gtk.so"
readelf -l "$out/gen_snapshot" | grep interpreter
readelf -rW "$out/libflutter_linux_gtk.so" | grep R_LARCH_B26 || true
readelf -rW "$out/libflutter_engine.so" | grep R_LARCH_B26 || true
```

期望结果：

- `gen_snapshot` 解释器是 `/lib64/ld.so.1`；
- `libflutter_linux_gtk.so` 能解析 GTK、fontconfig、OpenGL/X11 依赖；
- `readelf -rW` 不再输出动态 `R_LARCH_B26`；
- 真实 Flutter GTK 应用可以显示主窗口，而不是卡在初始化。

如果仍有 `R_LARCH_B26`，先确认：

- `which gcc` 指向 `$OLDWORLD_TOOLCHAIN/bin/gcc`；
- `"$OLDWORLD_TOOLCHAIN/bin/gcc" -print-prog-name=ld` 指向同一套工具链；
- GN 输出目录是在切换工具链后重新生成的；
- 没有设置会指向新世界库的 `LD_LIBRARY_PATH`、`LIBRARY_PATH` 或 `C_INCLUDE_PATH`。

## 4. Chromium depot_tools / Engine 三方依赖

Flutter Engine 构建需要 Chromium 工具链脚本。`depot_tools` 不需要编译，只需要
拉取并加入 `PATH`：

```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git \
  "$WORKSPACE/tools/depot_tools"
export PATH="$WORKSPACE/tools/depot_tools:$PATH"
export VPYTHON_BYPASS="manually managed python not supported by chrome operations"
```

本项目发布构建优先使用 `flutter` fork 中已经带着的 `engine/src`，这种路线不需要
单独执行 `gclient sync`。只有在独立调试 `engine` fork、需要拉取 Chromium/Skia
三方依赖时，才创建单独 Engine checkout：

```bash
mkdir -p "$WORKSPACE/engine-checkout"
cd "$WORKSPACE/engine-checkout"
gclient config https://github.com/Flutter-Dart-loong64/engine.git
gclient sync --no-history --shallow
gclient runhooks
```

不要把 `gclient` 拉下来的临时 cache、`.cipd` 用户目录或本地代理信息提交到仓库。

## 5. Dart SDK 作为构建依赖

Dart SDK 是 Flutter tool 和 Engine 的核心依赖。Loong64 发布构建必须先构建
本项目 fork 的 Dart SDK：

```bash
git clone https://github.com/Flutter-Dart-loong64/sdk.git "$WORKSPACE/dart-sdk"
cd "$WORKSPACE/dart-sdk"

./tools/build.py \
  -m release \
  -a loong64 \
  --gn-args="use_sysroot=false" \
  create_sdk dartaotruntime gen_snapshot analyze_snapshot
```

验证：

```bash
"$WORKSPACE/dart-sdk/out/ReleaseLOONG64/dart-sdk/bin/dart" --version
"$WORKSPACE/dart-sdk/out/ReleaseLOONG64/gen_snapshot" --version
file "$WORKSPACE/dart-sdk/out/ReleaseLOONG64/dart-sdk/bin/dart"
```

复制到 Flutter cache：

```bash
rm -rf "$WORKSPACE/flutter/bin/cache/dart-sdk"
mkdir -p "$WORKSPACE/flutter/bin/cache"
cp -a "$WORKSPACE/dart-sdk/out/ReleaseLOONG64/dart-sdk" \
  "$WORKSPACE/flutter/bin/cache/dart-sdk"
```

## 6. Flutter Engine 作为构建依赖

Engine 产物是 Flutter Linux 应用构建和运行的核心依赖。新世界构建：

```bash
cd "$WORKSPACE/flutter/engine/src"
dart_commit="$(git -C "$WORKSPACE/dart-sdk" rev-parse HEAD)"

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

旧世界构建在应用旧世界补丁后使用 `--no-clang` 和旧世界输出目录：

```bash
cd "$WORKSPACE/flutter/engine/src"
patch -p1 < "$WORKSPACE/flutter-loongarch64-releases/patches/oldworld-loongarch64-engine.patch"

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

验证：

```bash
out="$WORKSPACE/flutter/engine/src/out/linux_release_loong64_gtk"
file "$out/libflutter_linux_gtk.so" "$out/gen_snapshot"
ldd "$out/libflutter_linux_gtk.so" | grep fontconfig
"$out/gen_snapshot" --version
```

旧世界还要确认解释器：

```bash
out="$WORKSPACE/flutter/engine/src/out/linux_release_loong64_gtk_oldworld"
readelf -l "$out/gen_snapshot" | grep interpreter
readelf -r "$out/libflutter_linux_gtk.so" | grep R_LARCH_B26 || true
```

## 7. Rust 工具链

新世界 Debian/UOS 通常可使用标准 Rustup：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
  --default-toolchain stable \
  --profile minimal

source "$HOME/.cargo/env"
rustup target add loongarch64-unknown-linux-gnu
rustc -vV
cargo -V
```

上面是正常安装方式，安装目录为：

```text
$HOME/.cargo/bin/rustc
$HOME/.cargo/bin/cargo
$HOME/.rustup/toolchains/
```

如果 shell 没有自动加载 Cargo 环境，在 `~/.bashrc`、`~/.zshrc` 或当前 shell
加入：

```bash
source "$HOME/.cargo/env"
```

验证：

```bash
which rustc
which cargo
rustup show
rustc -vV
cargo -V
```

旧世界优先使用龙芯/Loongnix 提供的 Rust 工具链或 rustup 安装方式，并确认生成的
可执行文件使用旧世界动态链接器：

```bash
rustc -vV
cargo -V

cat > "$WORKSPACE/rust-smoke.rs" <<'EOF'
fn main() {
    println!("loongarch rust ok");
}
EOF

rustc "$WORKSPACE/rust-smoke.rs" -o "$WORKSPACE/rust-smoke"
readelf -l "$WORKSPACE/rust-smoke" | grep interpreter
```

如果是在 CI、容器或一次性构建目录里，才建议固定局部 cache：

```bash
export CARGO_HOME="$WORKSPACE/.cargo"
export RUSTUP_HOME="$WORKSPACE/.rustup"
export CARGO_TARGET_DIR="$WORKSPACE/.cargo-target"
export CARGO_NET_GIT_FETCH_WITH_CLI=true

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
  --default-toolchain stable \
  --profile minimal
source "$CARGO_HOME/env"
```

本地日常构建不要同时混用 `$HOME/.cargo` 和 `$WORKSPACE/.cargo`。如果切换过，
先执行：

```bash
unset CARGO_HOME
unset RUSTUP_HOME
source "$HOME/.cargo/env"
```

## 8. vcpkg 与多媒体依赖

RustDesk 这类大型应用会用到 `libvpx`、`libyuv`、`opus`、`aom`、`ffmpeg` 等
C/C++ 依赖。先构建本地 vcpkg：

```bash
git clone https://github.com/microsoft/vcpkg.git "$VCPKG_ROOT"
git -C "$VCPKG_ROOT" checkout 2023.04.15
"$VCPKG_ROOT/bootstrap-vcpkg.sh" -disableMetrics
export VCPKG_ROOT
export VCPKG_FORCE_SYSTEM_BINARIES=1
export VCPKG_DEFAULT_BINARY_CACHE="$WORKSPACE/.vcpkg-cache"
```

在 RustDesk fork 中使用 overlay ports：

```bash
git clone https://github.com/Flutter-Dart-loong64/rustdesk.git "$WORKSPACE/rustdesk"
cd "$WORKSPACE/rustdesk"

"$VCPKG_ROOT/vcpkg" install \
  --x-install-root="$VCPKG_ROOT/installed" \
  --overlay-ports="$WORKSPACE/rustdesk/res/vcpkg" \
  libvpx libyuv opus aom
```

如果开启硬编解码或 ffmpeg 相关功能，再构建对应依赖：

```bash
"$VCPKG_ROOT/vcpkg" install \
  --x-install-root="$VCPKG_ROOT/installed" \
  --overlay-ports="$WORKSPACE/rustdesk/res/vcpkg" \
  ffmpeg
```

vcpkg 在 LoongArch 上遇到不认识的 triplet 时，需要在 fork 中补 triplet 或用
项目脚本里的 overlay/patch。不要把 x86_64 vcpkg 产物复制到 LoongArch 使用。

## 9. mozjpeg C 库

`mozjpeg` 是 `mozjpeg-sys` 的底层 C 库。源码构建方式：

```bash
git clone https://github.com/Flutter-Dart-loong64/mozjpeg.git "$WORKSPACE/mozjpeg"
cd "$WORKSPACE/mozjpeg"

cmake -S . -B build-loong64 -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$PREFIX/mozjpeg" \
  -DENABLE_SHARED=OFF \
  -DWITH_SIMD=1 \
  -DWITH_JPEG8=1 \
  -DWITH_TURBOJPEG=1

cmake --build build-loong64 --parallel
ctest --test-dir build-loong64 --output-on-failure
cmake --install build-loong64
```

确认 LSX/LASX 编译器支持：

```bash
printf 'int main(void){return 0;}\n' | cc -x c - -mlsx -c -o "$WORKSPACE/lsx.o"
printf 'int main(void){return 0;}\n' | cc -x c - -mlasx -c -o "$WORKSPACE/lasx.o" || true
```

`-mlasx` 不支持时可以只构建 LSX 路径；不能因为没有 LASX 就回退到完全无 SIMD。

## 10. mozjpeg-sys

`mozjpeg-sys` 是 Rust FFI crate，会在 Cargo build script 中编译 vendored
MozJPEG。LoongArch 支持重点是 `vendor/simd/loongarch` 下的 LSX/LASX 后端和
`jsimd_*` 符号。

```bash
git clone https://github.com/Flutter-Dart-loong64/mozjpeg-sys.git "$WORKSPACE/mozjpeg-sys"
cd "$WORKSPACE/mozjpeg-sys"

export CARGO_TARGET_DIR="$WORKSPACE/.cargo-target/mozjpeg-sys"
export TARGET_CPU=loongarch64

cargo build --release --features with_simd,turbojpeg_api,jpegtran
cargo test --release --features with_simd,turbojpeg_api,jpegtran
```

检查是否使用 LoongArch SIMD：

```bash
find "$CARGO_TARGET_DIR/release/build" -name 'libmozjpeg*.a' -print
nm -g "$CARGO_TARGET_DIR"/release/build/mozjpeg-sys-*/out/libmozjpeg*.a \
  | grep ' jsimd_'
```

如果 RustDesk 报：

```text
undefined symbol: jsimd_idct_ifast
```

说明链接到的 mozjpeg/mozjpeg-sys 产物没有完整提供 `jsimd_*` 后端，应该重新
构建 `mozjpeg-sys` 并确认 RustDesk 的 Cargo 依赖指向本项目 fork。

## 11. mozjpeg-rust

高层 Rust JPEG 封装验证：

```bash
git clone https://github.com/Flutter-Dart-loong64/mozjpeg-rust.git "$WORKSPACE/mozjpeg-rust"
cd "$WORKSPACE/mozjpeg-rust"

export CARGO_TARGET_DIR="$WORKSPACE/.cargo-target/mozjpeg-rust"
cargo test --release
```

如果需要强制使用本项目 `mozjpeg-sys`，在依赖项目中加入：

```toml
[patch.crates-io]
mozjpeg-sys = { git = "https://github.com/Flutter-Dart-loong64/mozjpeg-sys.git" }
```

实际应用仓库建议固定到已验证 commit，而不是浮动分支。

## 12. nix crate

`nix` crate 用于补齐 LoongArch ioctl/系统常量。构建和测试：

```bash
git clone https://github.com/Flutter-Dart-loong64/nix.git "$WORKSPACE/nix"
cd "$WORKSPACE/nix"

export CARGO_TARGET_DIR="$WORKSPACE/.cargo-target/nix"
cargo test --features ioctl
cargo test --target loongarch64-unknown-linux-gnu --features ioctl
```

在应用中临时使用 fork：

```toml
[patch.crates-io]
nix = { git = "https://github.com/Flutter-Dart-loong64/nix.git" }
```

如果应用锁定了 `nix = 0.23` 或 `0.25`，补丁版本要和应用的 `Cargo.lock`
匹配，避免因为 semver 变化引入额外行为差异。

## 13. RustDesk 依赖和打包

RustDesk Flutter 版本依赖 Rust、Flutter、vcpkg 和多个 C/C++ 多媒体库。
新世界构建：

```bash
git clone --recurse-submodules https://github.com/Flutter-Dart-loong64/rustdesk.git \
  "$WORKSPACE/rustdesk"
cd "$WORKSPACE/rustdesk"

export FLUTTER_ROOT=<loong64-flutter-sdk>
export PATH="$FLUTTER_ROOT/bin:${CARGO_HOME:-$HOME/.cargo}/bin:$PATH"
export VCPKG_ROOT="$WORKSPACE/vcpkg"
export DEB_ARCH=loong64
export CARGO_TARGET_DIR="$WORKSPACE/.cargo-target/rustdesk"

flutter --version --suppress-analytics
flutter config --enable-linux-desktop
flutter config --enable-loong64

"$VCPKG_ROOT/vcpkg" install \
  --x-install-root="$VCPKG_ROOT/installed" \
  --overlay-ports="$WORKSPACE/rustdesk/res/vcpkg" \
  libvpx libyuv opus aom

cargo fetch --locked
cargo build --release --lib --features flutter
python3 build.py --flutter
```

如果启用硬编解码：

```bash
"$VCPKG_ROOT/vcpkg" install \
  --x-install-root="$VCPKG_ROOT/installed" \
  --overlay-ports="$WORKSPACE/rustdesk/res/vcpkg" \
  ffmpeg

python3 build.py --flutter --hwcodec
```

旧世界构建原则：

```bash
export FLUTTER_ROOT=<oldworld-flutter-sdk>
export DEB_ARCH=loongarch64
export PATH="$FLUTTER_ROOT/bin:${CARGO_HOME:-$HOME/.cargo}/bin:$OLDWORLD_TOOLCHAIN/bin:$PATH"
unset LD_LIBRARY_PATH
```

旧世界必须在旧世界系统或旧世界 sysroot/toolchain 上重新编译 Rust、vcpkg 依赖、
Flutter bundle 和 `.deb`。不能复用 UOS 25/Debian 13 的新世界库。

验证包内容：

```bash
dpkg-deb -I rustdesk-*.deb
dpkg-deb -c rustdesk-*.deb | grep librustdesk.so
file rustdesk-*.deb
```

安装后验证：

```bash
ldd /usr/share/rustdesk/lib/librustdesk.so
rustdesk --version || true
```

## 14. flutter-linglong-store 依赖和打包

该仓库用于验证 Flutter SDK、Dart codegen、包架构识别和 Debian 打包。

原生构建：

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-linglong-store.git \
  "$WORKSPACE/flutter-linglong-store"
cd "$WORKSPACE/flutter-linglong-store"

export FLUTTER_ROOT=<loong64-or-oldworld-flutter-sdk>
export PATH="$FLUTTER_ROOT/bin:$FLUTTER_ROOT/bin/cache/dart-sdk/bin:$PATH"
export PUB_CACHE="$WORKSPACE/.pub-cache"

flutter config --enable-linux-desktop
flutter config --enable-loong64
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter build linux --release --target-platform linux-loong64
```

新世界 Debian 13 QEMU/容器打包：

```bash
export LOONG64_QEMU_IMAGE=ghcr.io/loong64/debian:trixie
export LOONG64_FLUTTER_RELEASE_REPO=Flutter-Dart-loong64/flutter-loong64-releases
export LOONG64_FLUTTER_RELEASE_TAG=<sdk-release-tag>
export LOONG64_FLUTTER_SDK_ARCHIVE=<flutter-sdk-archive-name>
export LINGLONG_RELEASE_SKIP_BUILD_RUNNER=0
export LINGLONG_RELEASE_ALLOW_RIVERPOD_GENERATOR_FAILURE=0

bash build/scripts/build-loong64-in-container.sh \
  --version <app-version> \
  --channel <channel>
```

如果 generator 报错，应先修应用源码或依赖版本，不应通过跳过 codegen 掩盖问题。

## 15. LocalSend 类 Flutter 应用

普通 Flutter Linux 桌面应用验证流程：

```bash
git clone <application-repo> "$WORKSPACE/app"
cd "$WORKSPACE/app"

export FLUTTER_ROOT=<loong64-or-oldworld-flutter-sdk>
export PATH="$FLUTTER_ROOT/bin:$FLUTTER_ROOT/bin/cache/dart-sdk/bin:$PATH"
export PUB_CACHE="$WORKSPACE/.pub-cache"

flutter pub get
dart run build_runner build --delete-conflicting-outputs || true
flutter build linux --release --target-platform linux-loong64
file build/linux/loong64/release/bundle/*
```

如果应用没有 `build_runner`，该步骤可以跳过；如果有生成代码，必须跑完整生成。

## 16. Debian 13 QEMU 依赖缓存

在 CI 或本地 amd64 模拟时，建议缓存：

```text
$WORKSPACE/.pub-cache
$WORKSPACE/.cargo
$WORKSPACE/.cargo-target
$WORKSPACE/.vcpkg-cache
$WORKSPACE/vcpkg/downloads
$WORKSPACE/vcpkg/buildtrees
```

但发布包里不能包含这些 cache。发布包只应包含 Flutter SDK、Dart SDK、Engine
artifacts、license、version/stamp 和校验文件。

## 17. 最终检查清单

每次依赖升级后至少检查：

```bash
dart --version
flutter --version --suppress-analytics
flutter doctor -v
file <binary-or-shared-library>
ldd <shared-library>
readelf -l <binary> | grep interpreter
dpkg-deb -I <package>.deb
```

旧世界额外检查：

```bash
readelf -l <binary> | grep '/lib64/ld.so.1'
readelf -r <shared-library> | grep R_LARCH_B26 || true
```

Rust/Cargo 依赖额外检查：

```bash
cargo tree -i nix || true
cargo tree -i mozjpeg-sys || true
cargo build -vv
```

Flutter 应用额外检查：

```bash
flutter build linux --release --target-platform linux-loong64 -v
find build/linux/loong64/release/bundle -maxdepth 2 -type f -print
```
