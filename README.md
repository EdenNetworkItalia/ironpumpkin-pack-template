# IronPumpkin pack template

This is a template repository for an [IronPumpkin](https://github.com/EdenNetworkItalia/IronPumpkin)
modpack. One repository builds one server binary. The native mods of the pack are compiled into
the binary. GitHub Actions builds the release binaries for Linux and Windows, so nobody compiles
on their own machine.

The template has no mod and no patch. Built as it is, it gives a vanilla IronPumpkin server at the
commit pinned in `modpack.toml`.

For a worked example, see [ironpumpkin-pack-test](https://github.com/EdenNetworkItalia/ironpumpkin-pack-test):
the same template with one mod.

## Start a pack

1. Select **Use this template** on GitHub to create your repository.
2. In `modpack.toml`, set `[pack] name` to the name of your pack.
3. Run `cargo xtask check`. Commit `bin/Cargo.lock` and `bin/Cargo.lock.commit` if they change.

To move to a newer IronPumpkin, set `[ironpumpkin] commit` to a full 40-character hash. Run
`cargo xtask check` and commit the two lock files again.

## Commands

Run them from the repository root. You need Rust (stable) and git.

| Command                     | What it does                                                                                           |
|-----------------------------|--------------------------------------------------------------------------------------------------------|
| `cargo xtask check`         | Validates `modpack.toml`, fetches IronPumpkin into `.ironpumpkin/`, resolves the mods, applies patches. |
| `cargo xtask build`         | Runs `check` with `--locked`, then builds `target/release/<pack>`.                                      |
| `cargo xtask build --debug` | The same, with a debug build into `target/debug/<pack>`.                                                |
| `cargo xtask name`          | Prints the pack name.                                                                                   |

A release build compiles the whole server. It takes a long time and about 9 GB of disk.

## Add a mod

A mod is a Rust library crate that implements `NativeMod` from the `ironpumpkin-mods` crate. See
`examples/modpack/mods/hello-mod` in the IronPumpkin repository. Add one table per mod to
`modpack.toml`. The key is the package name of the mod crate. Give exactly one source:

```toml
[mods.my-mod]
path = "mods/my-mod"          # a crate in this repository

[mods.other-mod]
git = "https://github.com/someone/other-mod"
rev = "v1.2.0"                # a commit hash or a tag, not a branch
```

The mod crate depends on `ironpumpkin-mods` by the IronPumpkin git URL. The build points that
dependency at the IronPumpkin checkout, so the binary contains one copy of the server. After you
add a mod, run `cargo xtask check` and commit the lock files. The log of the server then shows
`[ironpumpkin] loaded 1 native mod: my-mod`.

## Add a source patch

Use a patch only when no NeoForge event or API covers the need. The mod crate carries the patch.

1. Make `patches/<name>.patch` in the mod crate with `git diff` against the root of the
   IronPumpkin source tree.
2. Write `patches/<name>.md` next to it. It says what the patch does, why a NeoForge event or API
   is not enough, and that the patch is bound to one IronPumpkin commit. The build fails without it.
3. Name that commit in the `Cargo.toml` of the mod:

   ```toml
   [package.metadata.ironpumpkin]
   commit = "<full 40-character commit hash>"
   ```

The build applies the patches with `git apply` before it compiles. It fails when a patch does not
apply, when two mods patch the same lines, or when the patch is written for another commit than
`modpack.toml`. `allow-drift = true` applies such a patch with a warning.

## Release binaries

The `Build modpack` workflow in `.github/workflows/build.yml` does the work:

- A pull request runs `cargo xtask check` and a debug build. It fails when the committed
  `bin/Cargo.lock` does not match `modpack.toml`.
- A tag that starts with `v` builds the release binary on Linux and on Windows and publishes a
  GitHub release:

  ```sh
  git tag v1.0.0 && git push --tags
  ```

  The release has `<pack>-<tag>-x86_64-linux`, `<pack>-<tag>-x86_64-windows.exe` and `SHA256SUMS`.

## More

- [IronPumpkin](https://github.com/EdenNetworkItalia/IronPumpkin): the server, the mod API and the
  full documentation of the pack contract in `blueprint/README.md`.
- [ironpumpkin-pack-test](https://github.com/EdenNetworkItalia/ironpumpkin-pack-test): a pack with
  one mod.

## Licence

The server is licensed under GPL-3.0, and so is this repository. A built binary links the server.
Give the source of the pack to whoever receives the binary. A mod linked into the binary and a
patch to the server are derivative works: distribute them under terms compatible with GPL-3.0.
The `ironpumpkin-mods` API crate is licensed under MIT OR Apache-2.0.
