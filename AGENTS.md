# AGENTS.md

This repository is a **Flatpak packaging manifest** for [WeylusCommunityEdition](https://github.com/electronstudio/WeylusCommunityEdition), not the app source.

## What this repo contains

- `io.github.electronstudio.WeylusCommunityEdition.json` — top-level Flatpak build manifest.
- `fltk-rs-modules.json` — nested manifest module for `fltk-rs` build dependencies (pango, harfbuzz, cairo, etc. and their transitive deps).
- `Cargo.toml.patch` — patch to replace `lto = true` with `lto = "thin"` in upstream, because full LTO causes multi-minute link timeouts on CI.
- `generated-sources-cargo.json` — vendored Rust crates for `flatpak-builder` offline builds.
- `io.github.electronstudio.WeylusCommunityEdition.metainfo.xml` — AppStream metadata, because the upstream release tarball does not contain it.

## Removed (no longer present)

- `build.rs.patch` — deleted. The old upstream `build.rs` ran `npm install` / `pnpm install` online inside the sandbox. Current upstream does not do this.
- `generated-sources-node.json` — deleted along with `build.rs.patch`; offline node dependencies are no longer needed.

## Upstream source

Source is fetched during build from the upstream GitHub release:
```
https://github.com/electronstudio/WeylusCommunityEdition/archive/refs/tags/2026.5.22.tar.gz
```

## Build environment

- **Runtime:** `org.gnome.Platform` 50
- **SDK:** `org.gnome.Sdk`
- **SDK extensions:** `org.freedesktop.Sdk.Extension.rust-stable`, `org.freedesktop.Sdk.Extension.node20`, `org.freedesktop.Sdk.Extension.llvm22`
- **Build command:** `cargo b --features "ffmpeg-system,fltk/use-wayland" --release`
- **Pre-built deps:** x264 (shared), ffmpeg (minimal shared build with `--disable-everything`)

## Why fltk-rs-modules.json is huge and fragile

The `fltk-rs` Rust crate links against system libraries (pango, cairo, harfbuzz, etc.). The GNOME 50 runtime *has* these, but only as shared libraries without static archives or certain development headers. `fltk-rs-modules.json` rebuilds them from source so `fltk-rs` can statically link or find the right headers.

The current chain is:

| Module | Purpose | Notes |
|---|---|---|
| gmp, mpfr, mpc | GCC bootstrap math | Only needed to build GCC's C++ runtime |
| libsupc++ | C++ standard library static build | **This is the slowest module.** Custom `buildsystem: simple` builds only `all-gcc` + `all-target-libstdc++-v3` to avoid compiling the full GCC runtime suite. See `fltk-rs-modules.json` for the exact commands. |
| pango, pcre, harfbuzz, cairo | Core text/font/graphics dependencies | All use meson; harfbuzz has `--buildtype=release` plus many feature disables (tests, docs, utilities, glib, gobject, cairo, chafa, icu). Pango gets `{xft, freetype, cairo}=enabled`. |
| libpanel, libdex, sysprof | GNOME transitive deps | Only needed because meson dependency resolution in the build pulls them in. sysprof is configured with `-Dsysprofd=none`. |

**Do not remove these without verifying fltk-rs can link against the runtime's shared libraries.** We already checked; it cannot.

## Flatpak-specific constraints

- Network access is **disabled** inside the build sandbox. All Rust crates are vendored (`generated-sources-cargo.json`). Node packages are fetched as a single `.tgz` archive (TypeScript compiler).
- `CARGO_NET_OFFLINE=true` is set in the Weylus module's `env`.
- `RUSTFLAGS` and `CFLAGS`/`CXXFLAGS` are heavily overridden to point to `/app/include/*` and force-link the rebuilt pango/glib/cairo/pcre2 static libraries. **Do not simplify these without testing the build.** They are the result of trial-and-error against the actual linker errors fltk-rs produces.

## Build optimizations applied

| Optimization | Where | Why |
|---|---|---|
| `-j4` in `make-args` | top-level `build-options` | Parallel C/C++ compilation (GitHub Actions runners have ~4 vCPUs) |
| `CARGO_BUILD_JOBS=4` | Weylus module `env` | Parallel Rust crate compilation |
| `--buildtype=release` | all meson modules in `fltk-rs-modules.json` | Skips debug symbols and assertions |
| `--disable-everything` + selective enables | ffmpeg `config-opts` | Weylus's `encode_video.c` only needs `libx264` encoder, `mp4` muxer, `scale`/`buffersrc`/`buffersink` filters, and `file` protocol. Before this change ffmpeg compiled ~200 decoders and ~150 demuxers. |
| Custom libsupc++ commands | `fltk-rs-modules.json` | Builds only GCC compiler + libstdc++-v3, skipping libsanitizer, libada, libgomp, etc. |
| `Cargo.toml.patch` | Weylus module sources | ThinLTO instead of full LTO; dramatically faster final link. |

## What `cleanup` removes

The top-level manifest has a `cleanup` array that strips build artifacts from the final Flatpak:

```json
"/include",
"/share/man", "/share/doc", "/share/info", "/share/help",
"/share/gir-1.0", "/share/vala", "/share/gcc-15.2.0",
"*.a", "*.la"
```

This saves ~30–50 MB from the bundle. `cleanup` runs **after** all modules finish building, so it does not interfere with compile-time header/static-lib needs.

## Manifest-only gotchas

- **Do not put `ccache: true` in the manifest.** It is a CLI flag (`flatpak-builder --ccache`), not a manifest property. Setting it in JSON causes the `flatpak-builder-lint` manifest check to fail with a JSON schema validation error.
- **No `generated-sources-node.json`** in the sources list. Upstream's current `build.rs` does not run `npm install`; it expects `tsc` to already be available, which we provide via the `typescript.tgz` file source.
- **Do not upgrade the upstream version** without also regenerating `generated-sources-cargo.json` and verifying the `Cargo.toml.patch` still applies cleanly.

## Typical edit workflow

1. Update the upstream archive URL + checksum in the manifest when a new release drops.
2. Regenerate `generated-sources-cargo.json` with flatpak-builder-tools (e.g., `flatpak-cargo-generator.py`) based on the new upstream `Cargo.lock`.
3. Verify `Cargo.toml.patch` still applies cleanly to the new upstream `Cargo.toml`.
4. If the upstream build process changes (new npm/pnpm usage, new system deps), the manifest may need a matching change.

## Local build commands

```bash
# Build locally (expect 2+ hours first time)
flatpak-builder --install --user --force-clean builddir io.github.electronstudio.WeylusCommunityEdition.json

# Run from builddir without installing
flatpak-builder --run builddir io.github.electronstudio.WeylusCommunityEdition.json weylus

# Lint the manifest
flatpak-builder-lint manifest io.github.electronstudio.WeylusCommunityEdition.json

# Clean local artifacts after build
rm -rf builddir repo .flatpak-builder
```

## Flathub submission

Before submitting or updating on Flathub, run the four linter checks:

```bash
flatpak-builder-lint manifest io.github.electronstudio.WeylusCommunityEdition.json
flatpak-builder-lint appstream io.github.electronstudio.WeylusCommunityEdition.metainfo.xml
flatpak-builder-lint builddir builddir/
flatpak-builder-lint repo repo/
```

The manifest check must pass before Flathub will accept the PR.
