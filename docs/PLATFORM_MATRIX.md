# Platform and Build Matrix

## Summary

| Environment | Purpose | Architecture name | Expected result |
| --- | --- | --- | --- |
| UOS 25 / deepin 25 native LoongArch64 | Primary new-world SDK and engine validation | `loong64` / `loongarch64` host, Debian package arch usually `loong64` | Build and run native Flutter Linux desktop applications. |
| Debian 13 `loong64` container/QEMU | CI/package build and repeatable application packaging | `loong64` | Build Loong64 app bundles and `.deb` packages in GitHub Actions or local Docker/QEMU. |
| UOS 20 old-world LoongArch64 | Old-world SDK and application validation | `loongarch64` | Build/run old-world-compatible Flutter SDK and app packages. |

## New-World

New-world LoongArch systems generally use:

- Debian architecture name: `loong64`
- GNU triplet: `loongarch64-linux-gnu`
- ABI: LP64D
- dynamic linker: `/lib64/ld-linux-loongarch-lp64d.so.1`
- Flutter target platform: `linux-loong64`

Use `flutter-loong64-releases` for this track.

## Old-World

Old-world LoongArch systems generally use:

- architecture name: `loongarch64`
- old-world ABI family
- dynamic linker: `/lib64/ld.so.1`
- Flutter target platform exposed by the patched tool: `linux-loong64`

Use `flutter-loongarch64-releases` for this track.

Old-world and new-world binaries are not compatible.

## Debian 13 QEMU

Debian 13 `loong64` under QEMU is useful for CI because it gives a repeatable
system image. It is slower than native hardware and may expose toolchain/cache
bootstrap problems earlier than native machines.

It is appropriate for:

- building Loong64 Flutter application bundles;
- building Debian packages for application projects;
- checking that the released SDK archive can bootstrap in a clean environment;
- validating that scripts do not rely on local machine paths.

It is not a complete substitute for:

- GPU/rendering validation;
- desktop integration validation;
- old-world ABI validation;
- performance testing.

