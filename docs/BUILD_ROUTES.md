# LoongArch Flutter 构建路线

本文只记录已经走通并用于发布或应用打包的最终路线。中途排错、失败尝试、个人机器
路径、远程账号、密码、内网地址和代理订阅都不写入公开文档。

示例命令统一使用占位符：

```bash
export WORKSPACE=<workspace>
export RELEASE_ID=<release-version>
```

`<workspace>` 是构建工作目录。真实构建时通常把源码、工具、cache、产物都放在这个
目录下面，但不要把个人目录写死到仓库脚本或文档里。

## 1. 三条最终路线

| 路线 | 目标系统 | 架构名 | 发布仓库 | 主要用途 |
| --- | --- | --- | --- | --- |
| UOS 25 / deepin 25 原生 | 新世界 LoongArch64 | `loong64` | `flutter-loong64-releases` | 新世界 SDK、Engine、真实桌面应用验证 |
| Debian 13 QEMU/容器 | 新世界 LoongArch64 | `loong64` | `flutter-loong64-releases` | CI、可重复构建、应用 `.deb` 打包 |
| UOS 20 原生 | 旧世界 LoongArch64 | `loongarch64` | `flutter-loongarch64-releases` | 旧世界 SDK、旧世界应用包 |

先判断系统：

```bash
uname -m
dpkg --print-architecture 2>/dev/null || true
readelf -l /bin/ls | grep interpreter
```

新世界常见动态链接器：

```text
/lib64/ld-linux-loongarch-lp64d.so.1
```

旧世界常见动态链接器：

```text
/lib64/ld.so.1
```

新世界和旧世界二进制不兼容，SDK、Engine、应用 `.deb` 都不能混用。

## 2. GN/Ninja 准备

Loong64 发布构建不使用 Google CIPD 预编译 GN/Ninja，因为官方 Flutter/Dart infra
没有 LoongArch64 对应 artifacts。最终成功路线统一使用目标系统上的 native `gn`
和 `ninja`，再把它们链接到 Dart SDK 与 Flutter Engine 期望的位置。

UOS 25 / deepin 25 和 Debian 13 新世界可优先安装发行版提供的 GN 包：

```bash
sudo apt update
sudo apt install -y generate-ninja ninja-build

export SYSTEM_GN="$(command -v gn)"
export SYSTEM_NINJA="$(command -v ninja)"
gn --version
ninja --version
```

已验证的新世界环境里，`generate-ninja` 提供 `/usr/bin/gn`。如果 UOS 系统没有可用
包，GN 按源码方式在目标系统上编译，最终仍然只暴露一个 native `gn` 可执行文件：

```bash
mkdir -p "$WORKSPACE/tools"
git clone https://gn.googlesource.com/gn "$WORKSPACE/tools/gn-src"
cd "$WORKSPACE/tools/gn-src"

python3 build/gen.py
ninja -C out

mkdir -p "$WORKSPACE/tools/gn-bin"
install -m 0755 out/gn "$WORKSPACE/tools/gn-bin/gn"

export PATH="$WORKSPACE/tools/gn-bin:$PATH"
export SYSTEM_GN="$(command -v gn)"
export SYSTEM_NINJA="$(command -v ninja)"
gn --version
```

Dart SDK 源码树需要：

```bash
cd "$DART_ROOT"
mkdir -p buildtools/ninja
ln -sfn "$SYSTEM_GN" buildtools/gn
ln -sfn "$SYSTEM_NINJA" buildtools/ninja/ninja
```

Flutter Engine 源码树需要：

```bash
cd "$ENGINE_SRC"
mkdir -p flutter/third_party/gn "$FLUTTER_ROOT/third_party/ninja"
ln -sfn "$SYSTEM_GN" flutter/third_party/gn/gn
ln -sfn "$SYSTEM_NINJA" "$FLUTTER_ROOT/third_party/ninja/ninja"
```

旧世界 UOS 20 也按同样原则处理：要么系统里已有 old-world native GN，要么在旧世界
系统上源码编译 GN，再把 Engine 的 `third_party/gn/gn` 链到该 native GN。不要从
新世界复制 GN/Ninja 到旧世界使用。

## 3. 通用源码和环境

核心源码：

```bash
mkdir -p "$WORKSPACE"
cd "$WORKSPACE"

git clone https://github.com/Flutter-Dart-loong64/sdk.git dart-sdk
git clone https://github.com/Flutter-Dart-loong64/flutter.git flutter
git clone https://github.com/Flutter-Dart-loong64/native.git native
```

新世界 release 仓库：

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-loong64-releases.git \
  flutter-loong64-releases
```

旧世界 release 仓库：

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-loongarch64-releases.git \
  flutter-loongarch64-releases
```

通用变量：

```bash
export DART_ROOT="$WORKSPACE/dart-sdk"
export FLUTTER_ROOT="$WORKSPACE/flutter"
export ENGINE_SRC="$FLUTTER_ROOT/engine/src"
export PUB_CACHE="$WORKSPACE/.pub-cache"
export DEPOT_TOOLS="$WORKSPACE/tools/depot_tools"

export PATH="$DEPOT_TOOLS:$FLUTTER_ROOT/bin:$FLUTTER_ROOT/bin/cache/dart-sdk/bin:$PATH"
export VPYTHON_BYPASS="manually managed python not supported by chrome operations"
```

如果需要 Flutter Engine 的 gclient 工具：

```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git \
  "$DEPOT_TOOLS"
```

## 4. UOS 25 / deepin 25 新世界原生路线

### 4.1 系统依赖

```bash
sudo apt update
sudo apt install -y \
  git curl ca-certificates xz-utils unzip zip rsync \
  python3 python3-pip python3-httplib2 python3-six \
  build-essential gcc g++ clang cmake generate-ninja ninja-build pkg-config make \
  dpkg-dev fakeroot file \
  libgtk-3-dev liblzma-dev libstdc++-dev libfontconfig1-dev \
  libgl1-mesa-dev libegl1-mesa-dev \
  libx11-dev libxcursor-dev libxinerama-dev libxi-dev \
  libxrandr-dev libxxf86vm-dev
```

准备 `SYSTEM_GN`、`SYSTEM_NINJA` 后，按第 2 节链接到 Dart 和 Engine 源码树。

### 4.2 Dart SDK

```bash
cd "$DART_ROOT"

mkdir -p buildtools/ninja
ln -sfn "$SYSTEM_GN" buildtools/gn
ln -sfn "$SYSTEM_NINJA" buildtools/ninja/ninja

./tools/build.py \
  -m release \
  -a loong64 \
  --gn-args="use_sysroot=false" \
  create_sdk dartaotruntime gen_snapshot analyze_snapshot
```

安装到 Flutter cache：

```bash
rm -rf "$FLUTTER_ROOT/bin/cache/dart-sdk"
mkdir -p "$FLUTTER_ROOT/bin/cache"
cp -a "$DART_ROOT/out/ReleaseLOONG64/dart-sdk" \
  "$FLUTTER_ROOT/bin/cache/dart-sdk"

"$FLUTTER_ROOT/bin/cache/dart-sdk/bin/dart" --version
```

### 4.3 Engine 源码树整理

当前发布优先使用 `flutter` fork 内的 `engine/src`，保证 Flutter framework、
Engine `DEPS` 和 Dart SDK revision 对齐：

```bash
cd "$ENGINE_SRC"

rm -rf flutter/third_party/dart flutter/prebuilts/linux-loong64
mkdir -p flutter/prebuilts/linux-loong64 flutter/third_party/gn "$FLUTTER_ROOT/third_party/ninja"

ln -s "$DART_ROOT" flutter/third_party/dart
ln -s "$DART_ROOT/out/ReleaseLOONG64/dart-sdk" \
  flutter/prebuilts/linux-loong64/dart-sdk
ln -sfn "$SYSTEM_GN" flutter/third_party/gn/gn
ln -sfn "$SYSTEM_NINJA" "$FLUTTER_ROOT/third_party/ninja/ninja"
```

如果是干净 checkout 并需要同步 Engine 三方依赖，使用 gclient 且跳过官方不存在的
Loong64 CIPD artifacts：

```bash
cd "$FLUTTER_ROOT"
python3 - <<'PY'
from pathlib import Path

Path(".gclient").write_text("""solutions = [
  {
    "name": ".",
    "url": "https://github.com/Flutter-Dart-loong64/flutter.git",
    "deps_file": "DEPS",
    "managed": False,
    "custom_vars": {
      "download_dart_sdk": False,
    },
    "custom_deps": {
      "engine/src/flutter/third_party/dart/tools/sdks/dart-sdk:dart/dart-sdk/${platform}": None,
      "engine/src/flutter/third_party/gn:gn/gn/${platform}": None,
      "engine/src/flutter/third_party/java/openjdk:flutter/java/openjdk/${platform}": None,
      "third_party/ninja:infra/3pp/tools/ninja/${platform}": None,
    },
  },
]
target_os = ["linux"]
target_cpu = ["loong64"]
""")
PY

python3 "$DEPOT_TOOLS/gclient.py" sync -D --no-history --nohooks \
  --ignore-dep-type=cipd -j4
```

### 4.4 构建期源码树修补

以下是最终成功路线中保留的构建树修补。它们只作用于本次 Engine 三方依赖构建树，
不要写入 SDK 发布包里的个人临时文件。

1. BoringSSL 识别 LoongArch64 CPU，否则会报 `Unknown target CPU`：

```bash
target_h="$ENGINE_SRC/flutter/third_party/boringssl/src/include/openssl/target.h"
if [ -f "$target_h" ] && ! grep -q "OPENSSL_LOONGARCH64" "$target_h"; then
  perl -0pi -e 's/#elif defined\(__riscv\) && __SIZEOF_POINTER__ == 8/#elif defined(__loongarch64) || (defined(__loongarch__) \&\& __loongarch_grlen == 64)\n#define OPENSSL_64_BIT\n#define OPENSSL_LOONGARCH64\n#elif defined(__riscv) \&\& __SIZEOF_POINTER__ == 8/' \
    "$target_h"
fi
```

2. glslang 当前 `BUILD.gn` 在 Loong64 系统 GN 下对非常规 `spirv.hpp11` 校验更严，
从 `sources` 列表移除，源码 include 关系不改：

```bash
glslang_gn="$ENGINE_SRC/flutter/third_party/vulkan-deps/glslang/src/BUILD.gn"
if [ -f "$glslang_gn" ]; then
  perl -0pi -e 's/\n\s+"SPIRV\/spirv\.hpp11",//' "$glslang_gn"
fi
```

3. SwiftShader Reactor 使用带 LoongArch64 target 的 LLVM 16 构建描述：

```bash
reactor_gn="$ENGINE_SRC/flutter/third_party/swiftshader/src/Reactor/BUILD.gn"
llvm16_gn="$ENGINE_SRC/flutter/third_party/swiftshader/third_party/llvm-16.0/BUILD.gn"
if [ -f "$reactor_gn" ] && [ -f "$llvm16_gn" ] &&
   grep -q "swiftshader_llvm_loongarch64" "$llvm16_gn"; then
  perl -0pi -e 's#llvm_dir = "\.\./\.\./third_party/llvm-10\.0"#llvm_dir = "../../third_party/llvm-16.0"#' \
    "$reactor_gn"
fi
```

4. libpng 已有 LoongArch LSX 源，但部分上游构建描述没有把源文件接入 Loong64：

```bash
libpng_gn="$ENGINE_SRC/flutter/third_party/libpng/BUILD.gn"
if [ -f "$libpng_gn" ] && ! grep -q "loongarch_lsx_init.c" "$libpng_gn"; then
  perl -0pi -e 's/\n  if \(is_win\) \{/\n  if (current_cpu == "loong64") {\n    sources += [\n      "loongarch\/filter_lsx_intrinsics.c",\n      "loongarch\/loongarch_lsx_init.c",\n    ]\n\n    defines += [ "PNG_LOONGARCH_LSX_OPT=1" ]\n\n    cflags_c += [ "-Wno-unused-variable" ]\n  }\n\n  if (is_win) {/' \
    "$libpng_gn"
fi
```

### 4.5 Engine 构建

```bash
cd "$ENGINE_SRC"
dart_commit="$(git -C "$DART_ROOT" rev-parse HEAD)"

(
  cd flutter
  "$DART_ROOT/out/ReleaseLOONG64/dart-sdk/bin/dart" pub get
)

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

必须保留 `--enable-fontconfig`。发布前验证：

```bash
out="$ENGINE_SRC/out/linux_release_loong64_gtk"
file "$out/libflutter_linux_gtk.so" "$out/gen_snapshot"
"$out/gen_snapshot" --version
ldd "$out/libflutter_linux_gtk.so" | grep fontconfig
```

### 4.6 Flutter cache 和打包

```bash
engine_out="$ENGINE_SRC/out/linux_release_loong64_gtk"
engine_cache="$FLUTTER_ROOT/bin/cache/artifacts/engine/linux-loong64-release"
cache_dir="$FLUTTER_ROOT/bin/cache"

mkdir -p "$engine_cache"
cp -a "$engine_out/libflutter_linux_gtk.so" "$engine_cache/"
cp -a "$engine_out/gen_snapshot" "$engine_cache/"
cp -a "$engine_out/icudtl.dat" "$engine_cache/" 2>/dev/null || true
cp -a "$engine_out/impellerc" "$engine_cache/" 2>/dev/null || true
cp -a "$engine_out/font-subset" "$engine_cache/" 2>/dev/null || true
cp -a "$engine_out/shader_lib" "$engine_cache/" 2>/dev/null || true

if [ ! -d "$engine_cache/flutter_linux" ]; then
  cp -a "$ENGINE_SRC/flutter/shell/platform/linux/public/flutter_linux" \
    "$engine_cache/flutter_linux"
fi

for mode in debug profile; do
  mode_cache="$FLUTTER_ROOT/bin/cache/artifacts/engine/linux-loong64-$mode"
  mkdir -p "$mode_cache"
  cp -a "$engine_cache/libflutter_linux_gtk.so" "$mode_cache/" 2>/dev/null || true
  cp -a "$engine_cache/icudtl.dat" "$mode_cache/" 2>/dev/null || true
  cp -a "$engine_cache/flutter_linux" "$mode_cache/" 2>/dev/null || true
done
```

写入 cache stamp，避免 Flutter tool 请求官方不存在的 Loong64 artifacts：

```bash
engine_revision="$(git -C "$ENGINE_SRC/flutter" rev-parse HEAD)"
engine_revision_date="$(git -C "$ENGINE_SRC/flutter" show -s --format=%cI HEAD)"

mkdir -p "$cache_dir" "$FLUTTER_ROOT/bin/internal" "$cache_dir/pkg"
printf '%s\n' "$engine_revision" > "$cache_dir/engine.stamp"
printf '\n' > "$cache_dir/engine.realm"
printf '%s\n' "$engine_revision" > "$cache_dir/engine-dart-sdk.stamp"
printf '%s\n' "$engine_revision" > "$cache_dir/flutter_sdk.stamp"
printf '%s\n' "$engine_revision" > "$cache_dir/linux-sdk.stamp"
printf '%s\n' "$engine_revision" > "$cache_dir/font-subset.stamp"
printf '%s\n' "$engine_revision" > "$cache_dir/engine_stamp.stamp"

python3 - "$cache_dir/engine_stamp.json" "$engine_revision" "$engine_revision_date" <<'PY'
import json
import sys
import time
from pathlib import Path

path = Path(sys.argv[1])
engine_revision = sys.argv[2]
engine_revision_date = sys.argv[3]
path.write_text(json.dumps({
    "build_time_ms": int(time.time() * 1000),
    "git_revision": engine_revision,
    "git_revision_date": engine_revision_date,
    "content_hash": engine_revision,
}, separators=(",", ":")) + "\n")
PY

rm -rf "$cache_dir/pkg/sky_engine" "$cache_dir/pkg/flutter_gpu"
cp -a "$ENGINE_SRC/flutter/sky/packages/sky_engine" "$cache_dir/pkg/sky_engine"
cp -a "$ENGINE_SRC/flutter/lib/gpu" "$cache_dir/pkg/flutter_gpu"

cat > "$FLUTTER_ROOT/bin/internal/bootstrap.sh" <<EOF
#!/usr/bin/env bash
export FLUTTER_PREBUILT_ENGINE_VERSION="\${FLUTTER_PREBUILT_ENGINE_VERSION:-$engine_revision}"
EOF
chmod +x "$FLUTTER_ROOT/bin/internal/bootstrap.sh"
```

打包：

```bash
cd "$WORKSPACE/flutter-loong64-releases"

FLUTTER_ROOT="$FLUTTER_ROOT" \
DART_ROOT="$DART_ROOT" \
ENGINE_SRC="$ENGINE_SRC" \
ENGINE_OUT="$engine_cache" \
./scripts/package-loong64-release.sh "$RELEASE_ID" "$WORKSPACE/dist/$RELEASE_ID"

cd "$WORKSPACE/dist/$RELEASE_ID"
sha256sum -c SHA256SUMS
```

## 5. Debian 13 QEMU/容器路线

Debian 13 路线以 `flutter-loong64-releases/scripts/ci-qemu-loong64-release.sh`
为准。它是用于 CI 和本地 QEMU 调试的完整脚本，不需要把人工尝试步骤写到文档里。

amd64 主机准备 QEMU：

```bash
docker run --privileged --rm tonistiigi/binfmt --install loong64
docker run --rm --platform linux/loong64 ghcr.io/loong64/debian:trixie uname -m
```

容器内必须设置：

```bash
export DART_TAG=<upstream-dart-sdk-tag>
export RELEASE_ID=<release-version>
export BOOTSTRAP_DART_SDK_URL=<previous-loong64-dart-sdk-archive-url>
export WORKSPACE=<workspace>
```

运行：

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-loong64-releases.git
cd flutter-loong64-releases
./scripts/ci-qemu-loong64-release.sh
```

脚本最终做的事情：

- 安装 `generate-ninja`、`ninja-build`、Clang、GTK、fontconfig、OpenGL/EGL、X11
  开发包；
- 检查 `command -v gn` 和 `command -v ninja`，缺少任一工具则停止；
- 用 gclient 同步 Dart SDK，`--ignore-dep-type=cipd` 跳过官方 Loong64 不存在的
  CIPD 依赖；
- 从上游 Dart tag 创建 release 分支，再把 `Flutter-Dart-loong64/sdk` 上的
  Loong64 patch 按顺序 cherry-pick 进去；
- 如果 patch 与上游 tag 冲突，则跳过本次发布，不产出错误包；
- 下载上一版 Loong64 Dart SDK 作为 bootstrap；
- 构建 Dart SDK、`dartaotruntime`、`gen_snapshot`、`analyze_snapshot`；
- 同步 `Flutter-Dart-loong64/flutter`，并设置 `download_dart_sdk=false`；
- 链接 native `gn`、`ninja` 和本轮 Dart SDK 到 Engine 源码树；
- 应用第 4.4 节的 BoringSSL、glslang、SwiftShader、libpng 构建树修补；
- 使用 `--enable-fontconfig` 构建 Linux GTK Engine；
- 填充 Flutter cache、engine stamps、`sky_engine`、`flutter_gpu`；
- 调用 `package-loong64-release.sh` 输出 Dart SDK、Engine、Flutter SDK archive
  和 `SHA256SUMS`。

Debian 13 路线的产物适合 CI 和普通新世界用户验证。它不能替代 UOS 25 实机上的
OpenGL/LoongGPU 桌面渲染验证，也不能替代 UOS 20 旧世界 ABI 验证。

## 6. UOS 20 旧世界原生路线

旧世界必须在 UOS 20 类系统或等价 old-world sysroot/toolchain 中构建。

### 6.1 系统依赖和 Rust

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
  nasm yasm
```

Rust 应用依赖使用龙芯社区 Rustup 源：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://rust-lang.loongnix.cn/rustup-init.sh | sh
source "$HOME/.cargo/env"
rustup show
rustc -vV
cargo -V
```

安装目录保持 rustup 默认：

```text
$HOME/.cargo
$HOME/.rustup
```

### 6.2 旧世界 GCC 13.4 + binutils 2.42

已验证的旧世界最终链接使用 GCC 13.4 driver + binutils 2.42，目的是避免
UOS 20 系统 linker 在 `libflutter_linux_gtk.so` 中留下动态 `R_LARCH_B26`
分支重定位。工具链源码编译步骤见
[`DEPENDENCY_BUILDING.md`](DEPENDENCY_BUILDING.md#3-旧世界-gcc-134--binutils-242-工具链)。

Engine 构建前设置：

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

### 6.3 Dart SDK 和 Engine 源码树

旧世界 Dart SDK 仍然使用 Loong64 VM 后端构建：

```bash
cd "$DART_ROOT"
mkdir -p buildtools/ninja
ln -sfn "$SYSTEM_GN" buildtools/gn
ln -sfn "$SYSTEM_NINJA" buildtools/ninja/ninja

./tools/build.py \
  -m release \
  -a loong64 \
  --gn-args="use_sysroot=false" \
  create_sdk dartaotruntime gen_snapshot analyze_snapshot

rm -rf "$FLUTTER_ROOT/bin/cache/dart-sdk"
mkdir -p "$FLUTTER_ROOT/bin/cache"
cp -a "$DART_ROOT/out/ReleaseLOONG64/dart-sdk" \
  "$FLUTTER_ROOT/bin/cache/dart-sdk"
```

旧世界路线中最终使用过“干净 Engine 源码树 + 复用已同步 third_party”的整理方式。
公开文档用占位目录表达：

```bash
export ENGINE_WORK=<engine-work-src>
export ENGINE_CLEAN=<engine-clean-src>

mkdir -p "$ENGINE_CLEAN/flutter/third_party"
for entry in "$ENGINE_WORK/flutter/third_party"/*; do
  name="$(basename "$entry")"
  ln -sfn "$entry" "$ENGINE_CLEAN/flutter/third_party/$name"
done

rm -rf "$ENGINE_CLEAN/flutter/third_party/dart" \
  "$ENGINE_CLEAN/flutter/third_party/gn/gn" \
  "$ENGINE_CLEAN/flutter/prebuilts/linux-loong64/dart-sdk"

mkdir -p "$ENGINE_CLEAN/flutter/third_party/gn" \
  "$ENGINE_CLEAN/flutter/prebuilts/linux-loong64"

ln -s "$DART_ROOT" "$ENGINE_CLEAN/flutter/third_party/dart"
ln -s "$SYSTEM_GN" "$ENGINE_CLEAN/flutter/third_party/gn/gn"
ln -s "$DART_ROOT/out/ReleaseLOONG64/dart-sdk" \
  "$ENGINE_CLEAN/flutter/prebuilts/linux-loong64/dart-sdk"
```

如果 Engine 工具脚本需要 content hash helper，而干净源码树没有该文件，补一个只
返回当前 Engine revision 的最小脚本：

```bash
mkdir -p "$FLUTTER_ROOT/bin/internal"
cat > "$FLUTTER_ROOT/bin/internal/content_aware_hash.sh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
engine_src="${ENGINE_SRC:-$(dirname "$0")/../../engine/src}"
git -C "$engine_src/flutter" rev-parse HEAD
EOF
chmod +x "$FLUTTER_ROOT/bin/internal/content_aware_hash.sh"
```

### 6.4 应用旧世界补丁并构建 Engine

```bash
cd "$ENGINE_SRC"
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

旧世界补丁覆盖：

- GCC 构建兼容；
- Engine 共享库静态 GCC runtime 链接；
- 避免 old-world 环境依赖 LSX/LASX-only 源码路径；
- Linux GTK `linux-loong64` artifact plumbing；
- 与当前 Flutter framework 匹配的部分 `dart:ui` ABI 修补。

### 6.5 旧世界验证和打包

```bash
out="$ENGINE_SRC/out/linux_release_loong64_gtk_oldworld"

file "$out/gen_snapshot" "$out/libflutter_linux_gtk.so"
readelf -l "$out/gen_snapshot" | grep interpreter
readelf -rW "$out/libflutter_linux_gtk.so" | grep R_LARCH_B26 || true
ldd "$out/libflutter_linux_gtk.so" | grep fontconfig
```

期望：

- host 可执行文件解释器是 `/lib64/ld.so.1`；
- `libflutter_linux_gtk.so` 不再有动态 `R_LARCH_B26`；
- engine 链接 `fontconfig`；
- 真实 Flutter GTK 应用能显示主窗口。

组装 cache：

```bash
cache="$FLUTTER_ROOT/bin/cache/artifacts/engine"

for mode in linux-loong64 linux-loong64-profile linux-loong64-release; do
  mkdir -p "$cache/$mode"
  cp "$out/libflutter_linux_gtk.so" "$cache/$mode/libflutter_linux_gtk.so"
  cp "$out/gen_snapshot" "$cache/$mode/gen_snapshot"
  cp "$out/icudtl.dat" "$cache/$mode/icudtl.dat"
done
```

验证应用：

```bash
git clone https://github.com/Flutter-Dart-loong64/flutter-linglong-store.git \
  "$WORKSPACE/flutter-linglong-store"
cd "$WORKSPACE/flutter-linglong-store"

flutter --no-version-check pub get
dart run build_runner build --delete-conflicting-outputs
flutter --no-version-check build linux --release --target-platform linux-loong64
```

打包旧世界 SDK：

```bash
cd "$WORKSPACE"
tar -czf flutter-"$RELEASE_ID"-loongarch64-oldworld-uos20.tar.gz flutter
sha256sum flutter-"$RELEASE_ID"-loongarch64-oldworld-uos20.tar.gz \
  > flutter-"$RELEASE_ID"-loongarch64-oldworld-uos20.tar.gz.sha256
```

## 7. 应用打包验证

SDK 发布后至少验证：

```bash
flutter --version --suppress-analytics
dart --version
flutter config --enable-linux-desktop
flutter config --enable-loong64
flutter doctor -v
```

新世界应用：

```bash
flutter build linux --release --target-platform linux-loong64
file build/linux/loong64/release/bundle/*
```

Debian `.deb` 打包应在对应 ABI 环境里完成：

- 新世界 `loong64` 包：UOS 25 / Debian 13 loong64；
- 旧世界 `loongarch64` 包：UOS 20 old-world。

不能用新世界包验证旧世界，也不能用旧世界包验证新世界。
