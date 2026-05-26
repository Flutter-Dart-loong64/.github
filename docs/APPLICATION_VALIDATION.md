# Application Validation

Real applications are used to prove the SDK works beyond a hello-world build.
They also expose missing native dependencies, generator issues, packaging
mistakes, ABI mismatches, and desktop runtime bugs.

## Validation Applications

| Application / repository | What it validates |
| --- | --- |
| `flutter-linglong-store` | Flutter Linux desktop build, Riverpod/build_runner generation, Debian packaging, Loong64/new-world architecture detection, old-world package behavior, and release SDK usability. |
| `rustdesk` | Large Rust/C/C++ desktop dependency graph, Rust crate LoongArch support, multimedia/image codec dependencies, Debian packaging, and runtime library compatibility. |
| LocalSend-style Flutter app builds | Independent Flutter application packaging and SDK smoke validation. |

## Architecture Mapping

Application code must distinguish:

- new-world `loong64`;
- old-world `loongarch64`;
- upstream or external package repositories that may use either name.

For new-world UOS 25/Debian 13, package requests should generally use
`loong64` when the package source distinguishes Debian architecture names.
For old-world UOS 20 class systems, use the old-world `loongarch64` track and
do not reuse new-world packages.

## What Counts As A Useful Validation

Good validation includes:

- `flutter pub get`;
- full code generation where the app uses generators;
- `flutter build linux --release --target-platform linux-loong64`;
- package creation, such as `.deb`;
- installing the package on the target system;
- launching the app from a desktop session;
- testing real UI paths, not only process startup;
- checking crash logs and coredumps after interactions.

## Notes From Recent Validation

Recent Loong64 application CI work showed that the SDK release could build
inside Debian 13 QEMU after the application fixed a Riverpod generator
compatibility issue in its global provider source. The SDK itself was usable;
the failure was in the application code generation path for that dependency
stack.

