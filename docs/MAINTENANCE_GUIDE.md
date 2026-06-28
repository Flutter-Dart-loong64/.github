# 长期维护与开发边界

本文定义 `Flutter-Dart-loong64` 的长期维护方案、开发技能分工和边界。目标是让
LoongArch Flutter/Dart 支持能持续跟上上游，而不是依赖某一次本地构建结果。

命令级构建步骤见 [`BUILD_ROUTES.md`](BUILD_ROUTES.md)，依赖源码编译见
[`DEPENDENCY_BUILDING.md`](DEPENDENCY_BUILDING.md)，仓库关系见
[`REPOSITORIES.md`](REPOSITORIES.md)。

## 维护目标

长期维护要同时满足四件事：

- 跟随 `flutter/flutter`、`flutter/engine`、`dart-lang/sdk`、`dart-lang/native`
  上游主线；
- 保持 Loong64 新世界 SDK 可构建、可安装、可运行真实 Flutter Linux 桌面应用；
- 旧世界 `loongarch64` 支持独立管理，不能混进新世界主线；
- 发布包、源码 fork、应用验证和依赖 fork 边界清楚。

项目的核心判断标准不是单个 demo 能启动，而是 SDK、Dart VM、engine artifacts、
Flutter tool、native assets、应用打包这条链路能重复构建和重复验证。

## 现有代码封存不动

当前已经存在的源码分支、release tag 和 GitHub Release 产物按历史状态保留。后续
治理的重点不是回头改写这些历史，而是把它们作为可引用的旧状态，新的开发和发布从
更清楚的控制模型开始。

现有分支和发布保持不动：

- 不删除旧 release。
- 不覆盖旧 asset。
- 不移动已经发布过的 tag。
- 不强推改写已经公开协作的分支。
- 不把旧分支继续当成新开发入口。

旧代码的作用：

- 历史参考；
- bug 对照；
- 补丁来源；
- 回归排查基线；
- 旧版本支持资料。

新开发的入口：

- 干净上游基线；
- LoongArch 独立维护线；
- 明确补丁边界；
- manifest 锁定跨仓库版本；
- merge/cherry-pick 引入官方更新；
- 验证分支确认构建和运行结果；
- 新版本发布。

如果需要记录当前状态，只新增清单文件或文档记录当前 commit、tag、release asset 和
校验信息，不反向修改原有分支和发布。

## 仓库管理原则

核心源码 fork：

- `sdk`：只维护 Dart SDK / Dart VM / `gen_snapshot` / runtime / snapshot / Loong64
  backend 相关改动。
- `engine`：维护独立 Flutter Engine 补丁，便于 review、拆分和同步上游。
- `flutter`：维护 Flutter framework/tool 改动，以及发布 SDK 所需的
  `engine/src` 对齐构建。
- `native`：维护 native assets、FFI target metadata 和 bundling 行为。

发布仓库：

- `flutter-loong64-releases` 只负责新世界 `loong64` SDK 发布、构建说明、自动化
  和发布历史。
- `flutter-loongarch64-releases` 只负责旧世界 `loongarch64` SDK 发布、旧世界补丁
  和 UOS 20 路线记录。

验证和依赖仓库：

- `flutter-linglong-store`、`rustdesk`、`localsend` 等应用用于验证 SDK 和依赖链路。
  应用修复不能反向污染 Flutter SDK 主线。
- `mozjpeg`、`mozjpeg-sys`、`nix` 等依赖 fork 只处理依赖自身的 LoongArch 缺口。
  能上游化的补丁优先保持通用写法。

## 分支策略

现有公开分支：

- `sdk main`
- `engine main`
- `flutter master`
- `native main`

这些分支代表当前已经公开的源码状态。长期维护中不应改写这些分支的公开历史。
后续开发使用 LoongArch 独立维护线，正常追加提交；官方上游只是输入源，通过
merge、cherry-pick 或人工重新适配进入 LoongArch 维护线。

推荐维护线：

| 分支 | 用途 | 历史策略 |
| --- | --- | --- |
| `loong64-main` | 新世界 Loong64 长期开发主线 | 只追加，不改写公开历史 |
| `loong64-next` | 新世界集成验证线 | 只追加，不改写公开历史 |
| `oldworld/loongarch64` | 旧世界长期开发线 | 只追加，不改写公开历史 |
| `release/<version>` | 发布维护分支 | 不改写公开历史 |
| `hotfix/<version>/<issue>` | 已发布版本修复 | 不改写公开历史 |

临时分支：

| 分支 | 用途 | 是否允许改写 |
| --- | --- | --- |
| `sync-test/<date>` | 测试一次官方更新能否合入 | 可以删除重建 |
| `patch/<topic>` | 单个补丁开发 | 未公开前可以整理 |
| `try/<topic>` | 构建、二分、远程机器排错 | 可以删除重建 |

旧世界分支：

- `oldworld-loongarch64`

现有的 `oldworld-loongarch64` 保持为旧世界补丁分支。后续如果新建
`oldworld/loongarch64`，应把旧世界支持作为连续追加提交迁移过去，而不是改写旧分支。
旧世界发布说明和补丁文件放在 `flutter-loongarch64-releases`。旧世界分支可以基于
新世界 Loong64 补丁继续追加，但不能把旧世界 ABI、linker、libstdc++ 或 GTK
workaround 直接合进新世界主线。

临时调试分支：

- 用于构建、二分、远程机器排错、依赖验证。
- 不作为长期维护入口。
- 结论沉淀到对应源码仓库、发布仓库或应用仓库后即可删除。

公共分支规则：

- 已发布 tag 不移动。
- 已发布 asset 不覆盖。
- `main`、`master`、`loong64-main`、`loong64-next`、`oldworld/*`、`release/*`、
  `hotfix/*` 不做公开历史改写。
- 官方更新通过 merge commit、cherry-pick 或人工重新适配进入 LoongArch 维护线。
- 不用 rebase 作为公开维护分支的同步方式。

## 上游同步流程

每轮同步按这个顺序做：

1. 拉取上游和 fork 远端。
2. 检查工作区是否干净。
3. 先把未提交的旧世界、本地调试或临时补丁归类。
4. 在临时 `sync-test/<date>` 分支试合官方更新。
5. 选择 merge 官方版本点、cherry-pick 必要官方提交，或人工重新适配冲突文件。
6. 遇到冲突时优先保留上游当前结构，只把 LoongArch 必要差异补回。
7. 跑最小检查：`git diff --check`、相关单测或构建 smoke。
8. 试合通过后，把结果以正常追加提交合入 LoongArch 维护线。
9. 更新 manifest，记录官方来源 revision、LoongArch 维护线 revision 和构建环境。
10. 只有发布事件才进入 SDK 构建和 release 上传。

同步时不做这些事：

- 不把多个无关领域补丁揉成一个大改动。
- 不为了省事改上游格式、排序或无关代码。
- 不把远程机器路径、临时构建目录、个人环境变量写进公开文档。
- 不因为上游同步成功就发布 SDK。
- 不为了让历史看起来干净而改写已公开分支或发布记录。
- 不把 LoongArch 支持长期维护成每次跟随官方 HEAD 重排的一串变基补丁。

## Manifest 和补丁边界

多仓库项目不能只靠“当前分支 HEAD”决定发布。SDK 发布应由 manifest 锁定跨仓库
版本组合。

manifest 至少记录：

```yaml
sources:
  flutter:
    repo: https://github.com/Flutter-Dart-loong64/flutter
    revision: <full-sha>
  sdk:
    repo: https://github.com/Flutter-Dart-loong64/sdk
    revision: <full-sha>
  engine:
    repo: https://github.com/Flutter-Dart-loong64/engine
    revision: <full-sha>
  native:
    repo: https://github.com/Flutter-Dart-loong64/native
    revision: <full-sha>
loongarch_changes:
  flutter: <loongarch-change-range-or-id>
  sdk: <loongarch-change-range-or-id>
  engine: <loongarch-change-range-or-id>
  native: <loongarch-change-range-or-id>
build:
  target: linux-loong64
  system: <build-system>
  toolchain: <toolchain-version>
```

补丁边界用于回答每个改动的生命周期：

- 为什么需要；
- 属于哪个层级；
- 依赖哪些改动；
- 如何验证；
- 是否准备上游化；
- 什么时候可以删除。

旧代码里的有效内容不要整块搬运，应按 Dart VM、Flutter tool、engine、native
assets、旧世界 ABI、硬件 workaround、发布脚本等边界重新拆分为独立提交。

## 提交拆分

提交按能力边界拆分：

- Dart VM Loong64 backend：assembler、disassembler、decoder、codegen、stub、
  runtime entry、snapshot。
- Flutter tool Loong64：host detection、target platform、cache path、artifact name、
  native assets target。
- Engine Linux Loong64：GN、toolchain、sysroot、Linux GTK artifacts、fontconfig。
- Renderer / OpenGL / LoongGPU：单独提交，不和基础 Loong64 target plumbing 混在一起。
- Old-world：单独分支，单独提交，单独说明。
- 依赖库：在依赖 fork 内提交，不放进 Flutter/Dart 主仓库。

一次提交应能说明一个完整目的。为了提交边界干净，可以 amend 同一能力的修复；不要把已经
分清边界的后续工作强行压回旧提交。

## 开发技能分工

Dart VM / SDK：

- LoongArch64 ABI、调用约定、寄存器分配和栈帧布局。
- assembler / disassembler / instruction decoder。
- JIT、AOT、snapshot、object pool、relocation、code patching。
- runtime entry、stub compiler、exception、GC barrier、FFI call。
- `gen_snapshot`、`dart`、`dartaotruntime` 的一致性验证。

Flutter Engine：

- GN build、toolchain、sysroot、Clang/GCC 兼容。
- Linux GTK embedder、accessibility、fontconfig、text rendering。
- Skia / Impeller / OpenGL ES / Vulkan 接入边界。
- `libflutter_linux_gtk.so`、ICU、snapshot data、engine stamp 配套发布。

Flutter tool：

- `HostPlatform`、`TargetPlatform`、artifact path、cache layout。
- `flutter build linux --target-platform linux-loong64`。
- native assets target、bundle layout、Debian packaging 输入。
- SDK archive 的 cache 预填充和版本一致性检查。

Native assets / FFI：

- Dart native asset target metadata。
- FFI library bundling、runtime lookup、architecture naming。
- Rust/C/C++ 依赖在 Loong64 和旧世界上的构建差异。

发布工程：

- 新世界、旧世界、Debian 13 QEMU 的构建路线。
- SHA256、release notes、历史版本记录。
- GitHub Actions 同步、冲突跳过、tag 触发发布。
- 发布包可重复解压、可运行、可构建应用。

应用验证：

- 真实 Flutter Linux 桌面应用构建。
- `.deb` 打包、安装、卸载、desktop file、service hook。
- GTK、OpenGL、字体、图像编解码、FFI 依赖验证。

## 边界判断

属于 Flutter/Dart 核心：

- 目标架构识别；
- Dart VM 后端；
- engine artifacts；
- Flutter tool cache 和 target platform；
- native assets 对 Loong64 的识别；
- SDK 发布结构。

不属于 Flutter/Dart 核心：

- 某个应用自己的业务逻辑；
- 某个系统镜像的本地配置；
- 某台机器的本地环境配置；
- 只为单个应用临时绕过的依赖补丁；
- RustDesk、LocalSend、linglong-store 的应用层 UI 或打包细节。

可以进入依赖 fork：

- 上游依赖缺少 LoongArch 常量、SIMD 后端、build script 分支、target mapping。
- 依赖补丁能独立解释，且不依赖 Flutter SDK 私有路径。

应留在发布仓库：

- 构建命令；
- release asset 命名；
- 历史版本；
- SDK 安装说明；
- CI 和 QEMU 发布流程。

应留在旧世界仓库或旧世界分支：

- `loongarch64` ABI1.0；
- 旧世界 linker/runtime 差异；
- UOS 20 toolchain workaround；
- 旧世界 GTK/link 兼容补丁。

## 版本和产物一致性

发布前必须确认：

- Flutter framework commit、engine commit、Dart SDK commit 能对应。
- `dart`、`frontend_server`、`gen_snapshot`、`dartaotruntime` 来自兼容构建。
- `libflutter_linux_gtk.so`、ICU、snapshot data 和 Flutter cache 目录匹配。
- 新世界产物只标为 `loong64`，旧世界产物只标为 `loongarch64`。
- tag 以版本为主，架构信息放在 release 文档和 asset 文件名中。
- release manifest 记录了所有源码 revision、LoongArch 改动范围和构建环境。

不要混用：

- UOS 25 构建的 SDK 和 UOS 20 运行环境；
- Debian 13 QEMU 验证结果和 UOS 25 GPU/桌面实机结论；
- standalone `engine` 产物和不匹配的 `flutter/engine/src` revision；
- 不同 Dart revision 的 `gen_snapshot` 和 runtime。

## 验证分层

最低验证：

- `git diff --check`
- 相关单元测试或脚本 dry run
- `dart --version`
- `flutter --version`
- `flutter doctor -v`

SDK 验证：

- `flutter config --enable-linux-desktop`
- `flutter config --enable-loong64`
- `flutter create` 或已有应用 `flutter pub get`
- `flutter build linux --release --target-platform linux-loong64`

应用验证：

- `flutter-linglong-store` 构建、安装、启动、页面切换。
- RustDesk 构建和依赖动态库加载。
- LocalSend 构建、安装和基本运行。

平台验证：

- UOS 25 / deepin 25：新世界实机桌面、字体、OpenGL。
- Debian 13：QEMU/容器构建和 CI。
- UOS 20：旧世界 ABI、旧世界 SDK、旧世界应用包。

## 发布规则

发布条件：

- 上游 SDK tag 或明确的人工发布需求。
- 源码 fork 已完成 LoongArch 维护线验证或记录了跳过原因。
- manifest 已固定源码 revision、LoongArch 改动范围和构建环境。
- 对应平台至少完成一次 SDK smoke 和一个真实应用构建。
- release notes 写清版本、构建系统、产物、校验和、已知限制。

不发布的情况：

- 只是普通上游同步。
- 只有应用仓库改动。
- 只有依赖 fork 改动，还没有进入 SDK 验证。
- 旧世界补丁变化但没有在旧世界系统构建验证。

发布后规则：

- 不移动 tag。
- 不覆盖 asset。
- 不改写 manifest 关键元数据。
- 需要修复时发布新版本号。

## 文档维护

文档只记录最终可复现路线和明确结论。探索过程、失败尝试、临时机器路径不写入公开
文档。

文档分工：

- `.github/docs`：项目级说明、长期维护规则、仓库关系、平台矩阵。
- `flutter-loong64-releases`：新世界安装、构建、发布、历史版本。
- `flutter-loongarch64-releases`：旧世界安装、构建、补丁、历史版本。
- 应用仓库：只写应用自身构建和打包说明。
- 依赖 fork：只写依赖自身的 LoongArch 支持说明。

每次修改后至少更新一处记录：

- 源码 fork：提交信息说明改动边界。
- 发布仓库：发布或构建路线变化。
- `.github`：跨仓库关系、维护策略或平台矩阵变化。

## 维护检查清单

同步源码 fork：

- 工作区干净。
- 已确认上游默认分支。
- 已确认本地未提交补丁归属。
- LoongArch 维护线只保留 Loong64 必要差异。
- manifest 已记录 LoongArch 维护线源码 revision。
- 旧世界补丁留在 `oldworld-loongarch64`。

修改核心补丁：

- 改动属于对应仓库职责。
- 不破坏上游格式和无关代码。
- 新增 target 名称覆盖 tool、engine、native assets。
- 有最小测试或真实构建验证。

发布 SDK：

- 版本号、Dart revision、engine revision 明确。
- manifest 已固定。
- SDK cache 完整。
- SHA256 已生成。
- release notes 和历史记录已更新。
- 新世界和旧世界产物没有混放。

处理应用问题：

- 先判断是 SDK、engine、Dart VM、系统 ABI、依赖库还是应用自身问题。
- 能复现再归类。
- 应用 workaround 不直接进入 Flutter/Dart 主线。
- 依赖缺口优先在依赖 fork 内解决。

## 优先级

优先级从高到低：

1. 让源码 fork 可持续跟随上游。
2. 保证新世界 SDK 可构建和真实应用可运行。
3. 保证旧世界支持独立可复现。
4. 把通用依赖补丁推向依赖 fork 或上游。
5. 扩展 CI、QEMU 和自动发布能力。
6. 优化性能、SIMD、GPU 后端和更多应用生态。
