# Packaging

How Zacess Scenes reaches crates.io, Homebrew and winget after a release. Each step is run by a
maintainer on their own machine with their own credentials. No token is ever stored in this
repository or in its workflows.

Before any of these, the release must exist: the tag `vX.Y.Z` pushed, the GitHub release made from
it and [`release.yml`](../.github/workflows/release.yml) finished, so the five archives and their
`.sha256` files are attached. `gh release view vX.Y.Z --repo zacesshq/scenes` lists them.

## In this folder

| File | Does |
| --- | --- |
| [`homebrew/zacess-scenes.rb`](homebrew/zacess-scenes.rb) | The formula for the tap, at the current version with a `SHA256_` placeholder per platform |
| [`winget/`](winget) | The three winget manifests (version, installer and default locale) at the current version, with one placeholder for the Windows zip |
| [`fill-checksums.sh`](fill-checksums.sh) | Downloads the `.sha256` files of a release and writes the formula and the manifests with the sums filled in to a folder of your choice |

The files here keep their placeholders. A release bumps the version in them (the four URLs and
the `version` line in the formula, `PackageVersion`, the zip path and `ReleaseNotesUrl` in the
manifests) in the same pull request as `Cargo.toml`.

## crates.io

The crate is `zacess-scenes`. `Cargo.toml` excludes `.github`, `packaging`, `scripts`, `palette`,
`docs/images` and `docs/frames`, so what is published is the code, the written docs and the
licence.

```sh
# check what would be packaged and that the metadata is complete
cargo publish --dry-run
cargo package --list

# publish with a token read from a file, never typed into the shell history
CARGO_REGISTRY_TOKEN="$(cat /path/to/token)" cargo publish
```

crates.io shows the new version within a minute. The crates.io badge in the README reads it from
there.

## Homebrew

The tap is `zacesshq/homebrew-tap`, which Homebrew reads as `zacesshq/tap`. The formula installs
the release archive for the person's platform rather than building from source, so it needs the
SHA-256 of each of the four Unix archives.

1. Once only, create the tap repository with a `Formula` folder:

   ```sh
   gh repo create zacesshq/homebrew-tap --public --description "Homebrew formulae for Zacess"
   git clone git@github.com:zacesshq/homebrew-tap.git
   mkdir homebrew-tap/Formula
   ```

2. Fill the formula from the release files and copy it into the tap:

   ```sh
   packaging/fill-checksums.sh vX.Y.Z /tmp/packaging-X.Y.Z
   cp /tmp/packaging-X.Y.Z/homebrew/zacess-scenes.rb homebrew-tap/Formula/
   ```

3. Check it from the tap checkout, then commit and push it:

   ```sh
   brew install --formula ./Formula/zacess-scenes.rb
   brew test zacess-scenes
   brew style ./Formula/zacess-scenes.rb
   zacess-scenes --version
   ```

Once the tap holds the formula, the install line is `brew install zacesshq/tap/zacess-scenes` and
`brew upgrade zacess-scenes` follows each later release once its formula is in the tap.

## winget

The package is `Zacess.Scenes`. The installer is the Windows zip from the release, installed as a
portable program with the command alias `zacess-scenes`.

1. Fill the manifests from the release files:

   ```sh
   packaging/fill-checksums.sh vX.Y.Z /tmp/packaging-X.Y.Z
   ```

2. Copy the three files from `/tmp/packaging-X.Y.Z/winget/` into a fork of
   [microsoft/winget-pkgs](https://github.com/microsoft/winget-pkgs) under
   `manifests/z/Zacess/Scenes/X.Y.Z/`.
3. Validate and test them on a Windows machine:

   ```powershell
   winget validate --manifest manifests\z\Zacess\Scenes\X.Y.Z
   winget install --manifest manifests\z\Zacess\Scenes\X.Y.Z
   zacess-scenes --version
   ```

4. Open a pull request to microsoft/winget-pkgs. Their checks take a day or two and the package
   is live once it merges.

Once accepted, the install line is `winget install Zacess.Scenes`. Every later release is submitted
the same way with its own version folder.
