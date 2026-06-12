# nix-wasm

[![Nix Flake](https://img.shields.io/badge/Nix-Flake-blue.svg)](https://nixos.wiki/wiki/Flakes)
> A reproducible development environment for Haskell WASM development using Cabal

Motivation:

To develop a Haskell [WASM](https://webassembly.org/) project using Cabal, you need the 
WASM-specific tools, and a set of packages (project dependencies) compiled for WASM.

Without nix-wasm, you can build a Haskell WASM project with Cabal, it just downloads everything 
from Hackage and builds it locally. This is undesirable, because it leads to unreproducible 
builds: the build might work today and on your system, but if some Hackage packages perform 
breaking updates or if you try to build the project on a different system then the build might 
not work anymore.

The approach in nix-wasm is to use Hackage packages from the nix repository, creating a separate 
derivation for every Haskell package it knows, which is then exposed as a package set that can be 
extended by the user. This results in reproducible builds.

This readme file assumes some familiarity with `cabal` and with `nix flake`.

## Prerequisites

- A .cabal file of your Haskell project
- Cabal must not be configured to download from remote repositories such as Hackage; in the Cabal 
configuration file (typically `~/.config/cabal/config`) you should have the following lines 
commented out:
    ```
    -- repository hackage.haskell.org
      -- url: http://hackage.haskell.org/
    ```
- You must have [Nix](https://nixos.org/download.html) installed with **flakes enabled**.

## Usage

### Running directly

```sh
nix develop
```

This way you enter into a shell with the WASM tool-set in the execution path
(e.g. `wasm32-wasi-cabal`, ...)

Note: at the time of writing, `nix develop` fails with the error:

```
       > Configuring ghc-trace-events-0.1.2.10...
       > Error: [Cabal-8010]
       > Encountered missing or private dependencies:
       >     base >=4.8 && <4.22 (installed: 4.22.0.0)
       > CallStack (from HasCallStack):
       >   dieWithException, called at libraries/Cabal/Cabal/src/Distribution/Simple/Configure.hs:1620:11 in Cabal-3.16.0.0-b738:Distribution.Simple.Configure
```

This happens because the Haskell package set of ghc 9.14 is unstable in nixos-unstable 
(TODO: pin flake.lock to a compatible package set for GHC 9.14?)


### Integrating in your project flake

First, include the repository in your flake:

```nix
inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    nix-wasm.url = "github:ners/nix-wasm";
    # ... Other inputs ...
};
```

Then, define an entry into `outputs.devShells` for your shell; for example, suppose your cabal file is
named `wasm-project.cabal`:

```nix
outputs = 
  {
    # ... other outputs...
    devShells.wasm-project-shell = 
      let
        wasmPkgs = inputs.nix-wasm.legacyPackages.${system};
        mkPackage = ps: ps.callCabal2nix "wasm-project" ./ {};
      in
        pkgs.mkShell {
          inputsFrom = [
            # make available wasm tools e.g. wasm32-wasi-cabal
            (wasmPkgs.haskell.packages.ghc914.shellFor {
              packages = ps: [(mkPackage ps)];
              nativeBuildInputs = with wasmPkgs; [
                cabal-install
              ];
            })
            # make available general Haskell tools such as cabal or haskell-language-server.
            # the purpose is to provide haskell-language-server for your editor.
            (pkgs.haskell.packages.ghc912.shellFor {
              packages = ps: [(mkPackage ps)];
              nativeBuildInputs = with pkgs; [
                cabal-install
                pkgs.haskell.packages.ghc912.haskell-language-server
                haskellPackages.implicit-hie
                hpack
              ];
            })
          ];

          packages = let
            wasmCabalWrapper = pkgs.writeShellScriptBin "wasm-cabal" ''
              if [ $# -eq 0 ]; then
                  echo "Error: No cabal command provided."
                  echo "Usage: wasm-cabal <command> [args...]"
                  echo "Example: wasm-cabal build my-app-frontend"
                  exit 1
              fi
              COMMAND="$1"
              shift
              wasm32-wasi-cabal "$COMMAND" "$@"
            '';
            wasmDistribute = pkgs.writeShellScriptBin "wasm-distribute" ''
              CABAL_TARGET="$1"
              TARGET_DIR="$2"
              BIN_NAME=$(basename "$0")

              # parameter validation
              [[ -z "$CABAL_TARGET" ]] && { echo "Usage: $BIN_NAME <cabal-target> <target-dir>"; exit 1; }
              [ -d "$TARGET_DIR" ] && [ -w "$TARGET_DIR" ] || { echo "Error, missing target directory. usage: $BIN_NAME <cabal-entry> <target-dir>"; exit 1; }

              # compile and copy to target dir
              WASM_FILE=$(wasm-cabal list-bin -v0 "$CABAL_TARGET")
              if [ $? -ne 0 ]; then
                  echo "Error: wasm-cabal failed to locate the target for $CABAL_TARGET" >&2
                  exit 1
              fi
              GHC_LIBDIR=$(wasm32-wasi-ghc --print-libdir)
              "$GHC_LIBDIR"/post-link.mjs --input "$WASM_FILE" --output "$TARGET_DIR"/ghc_wasm_jsffi.js
              cp -v "$WASM_FILE" "$TARGET_DIR"
            '';
          in [
            wasmCabalWrapper
            wasmDistribute
          ];
          shellHook = ''
            echo "Welcome to the WASM development shell"
          '';
        };
  };
```

Notice that for the non-WASM Haskell tools, the language server, the `ghc912` package set is used.
This means that the language server will build your project against this ghc version, so be 
conscious of potential differences between versions.

Finally, enter your shell using:

```sh
nix develop .#wasm-project-shell
```

The example above also provides two additional scripts in your development shell:

- `wasm-cabal` is a convenience wrapper around `wasm32-wasi-cabal`
- `wasm-distribute` prepares your binary for the web and copies it to e.g. a `static/` directory,
    alongside `ghc_wasm_jsffi.js` which must be loaded in the browser also.
    If your cabal executable is e.g. `executable my-project-executable` then you'd use
    `wasm-distribute my-project-executable ./static/`

You use these scripts to test your WASM build and to deploy the WASM file on your web server.
For instructions on how to include these in the browser see the 
[ghc documentation](https://ghc.gitlab.haskell.org/ghc/doc/users_guide/wasm.html#the-javascript-api)
