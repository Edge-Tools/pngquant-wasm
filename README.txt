pngquant-wasm — Corresponding Source mirror
==================================================

This repository publishes the Corresponding Source for the WebAssembly
build of pngquant (license: GPL-3.0-or-later) used in edgetools.io.

Contents
  build/      our build recipe: Dockerfile + helper scripts/config/patches.
              Rebuild with:  docker build build/
  upstream/   the exact upstream source archive(s) the build fetched,
              byte-identical and sha256-verified (see below).

Upstream sources:
  pngquant-3.0.3.crate
    https://static.crates.io/crates/pngquant/pngquant-3.0.3.crate
    sha256 68a12bdd8825f9989f4ee9a6ab0b42727dae57728b939ef63453366697a07232
