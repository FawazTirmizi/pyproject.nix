# hacks

This documentation is a guide, for more details see [hacks library reference](../build/hacks.md).

## Using prebuilt packages from Nixpkgs

Sometimes making a package building from source can be difficult, wheels are not available, and Nixpkgs may already contain source-built packages.
In such cases it can be tempting to reuse build outputs from Nixpkgs, just as you would use a binary wheel from PyPI.

For such cases `pyproject.nix` provides an adapter:
``` nix
{ callPackage, pyproject-nix, python3, python3Packages }:
let
  python = python3;

  hacks = callPackage pyproject-nix.build.hacks {};

  overlay = final: prev: {
    # Adapt torch from nixpkgs
    torch = hacks.nixpkgsPrebuilt {
      from = python3Packages.torchWithoutCuda;
      prev = prev.torch;
    };
  };

  pythonSet = (callPackage pyproject-nix.build.packages {
    inherit python;
  }).overrideScope overlay;
in
  pythonSet.mkVirtualenv "torch-venv" {
    torch = [ ];
  }
```

You may also want to filter out certain dependencies, `torch` in particular depends on a number of PyPI packages containing binary shared objects that are already linked by `torch` from nixpkgs.

``` nix
hacks.nixpkgsPrebuilt {
  from = python3Packages.torchWithoutCuda;
  prev = prev.torch.overrideAttrs(old: {
    passthru = old.passthru // {
      dependencies = lib.filterAttrs (name: _: ! lib.hasPrefix "nvidia" name) old.passthru.dependencies;
    };
  });
};
```

## Building Cargo (Rust) packages from source

Rust has its own package manager, Cargo, that expects to be able to download dependencies at build-time.
One way to deal with that is to use [rustPlatform.importCargoLock](https://nixos.org/manual/nixpkgs/stable/#vendoring-of-dependencies).

This mechanism uses IFD (import-from-derivation) on non-local packages.
For background as to why IFD should be avoided see
- [fzakaria - Nix parallelism & Import From Derivation](https://fzakaria.com/2020/10/20/nix-parallelism-import-from-derivation.html).

To adapt the `cryptography` Python package into creating a Rust vendor directory, and use it for building:
``` nix
final: prev: {
  cryptography =
    (hacks.importCargoLock {
      prev = prev.cryptography;
      # Cryptography uses a non-standard location for it's Rust packaging
      cargoRoot = "src/rust";
    });
}
```

In reality, the package still lacks some important metadata, such as native non-Rust dependencies that needs to be supplemented.
Depending on which lock file produced this package it may also need build-systems added.
``` nix
final: prev: {
  cryptography =
    (hacks.importCargoLock {
      prev = prev.cryptography;
      # Cryptography uses a non-standard location for it's Rust packaging
      cargoRoot = "src/rust";
    }).overrideAttrs
      (old: {
        nativeBuildInputs =
          old.nativeBuildInputs
          ++ final.resolveBuildSystem {
            maturin = [ ];
            setuptools = [ ];
            cffi = [ ];
            pycparser = [ ];
          };
        buildInputs = old.buildInputs or [ ] ++ [ pkgs.openssl ];
      });
}
```
