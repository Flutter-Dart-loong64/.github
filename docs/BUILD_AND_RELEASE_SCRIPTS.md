# Build and Release Scripts

This document explains the script roles across the project. Script names may
move over time; the release repositories are the source of truth for exact
current commands.

Source-build steps for Rust/Cargo/vcpkg/mozjpeg/nix/application dependencies
are documented in [`DEPENDENCY_BUILDING.md`](DEPENDENCY_BUILDING.md).

## Source Sync Automation

`flutter-loong64-releases` owns the cross-repository sync workflow.

Expected behavior:

- Rebase `sdk`, `engine`, `flutter`, and `native` fork branches onto their
  upstream branches.
- Skip a repository if the rebase conflicts.
- Do not publish SDK archives merely because a fork branch was rebased.
- Build/publish only for release events that are explicitly configured, such
  as upstream Dart SDK tags or manually approved release work.

This keeps ordinary upstream sync separate from release publishing.

## New-World Release Build

The new-world release flow publishes:

- Dart SDK archive
- Flutter Engine Linux GTK Loong64 archive
- Flutter SDK archive with matching cache artifacts
- `SHA256SUMS`
- release notes/history

Important build constraints:

- Dart SDK, engine, `gen_snapshot`, and Flutter tool revisions must match.
- Engine should be built with `--enable-fontconfig`.
- Flutter tool cache must contain `linux-loong64` artifacts.
- Release tags should use version-only naming where possible; asset filenames
  carry architecture markers such as `linux-loong64`.

## Old-World Release Build

The old-world release flow lives in `flutter-loongarch64-releases`.

It stores:

- old-world build instructions;
- old-world engine patch;
- old-world SDK release archive;
- SHA256 checksum;
- bring-up history.

Old-world releases should be built and validated on old-world-compatible
systems/toolchains.

## Application Package Helpers

`flutter-linglong-store` contains application packaging scripts used to validate
the SDK:

| Script | Role |
| --- | --- |
| `build/scripts/build-loong64-in-container.sh` | Runs Loong64 application builds inside a `linux/loong64` Debian container/QEMU environment. Downloads and bootstraps the Loong64 Flutter SDK release. |
| `build/scripts/install-loong64-build-deps.sh` | Installs container build dependencies such as C/C++ toolchains, CMake, Ninja, GTK development headers, and packaging tools. |
| `build/scripts/build-linux-bundle.sh` | Creates a disposable source copy, runs version updates, `flutter pub get`, code generation, and `flutter build linux`. |
| `build/scripts/package-bundle.sh` | Produces compressed Linux bundle artifacts. |
| `build/scripts/package-deb.sh` | Produces Debian packages for the target architecture/channel. |

Application scripts are validation tooling. They should not be mistaken for the
core Flutter SDK build system.

## GitHub Actions Notes

Loong64 GitHub Actions usually need:

- QEMU setup for `loong64`;
- a Loong64 container image;
- current helper scripts copied into older release source commits when patching
  historical nightly releases;
- release SDK cache bootstrap because Google Flutter infra does not publish
  official Loong64 artifacts;
- clear separation between "sync source forks" and "publish a release".
