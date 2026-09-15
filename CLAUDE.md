# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

`AGENTS.md` is a symlink to this file and holds the same agent contract.

## Commands

Every task runs through a [`just`](https://just.systems) recipe. Do not bypass a
recipe with an ad hoc `cargo` or tool command. If a recurring task has no
recipe, add one under `just/` before using it. Run `just` to list all recipes.

```sh
just build                 # cargo build --all-targets
just release               # release build
just test                  # cargo test --lib, then --doc
just coverage               # cargo llvm-cov, writes lcov.info
just fmt                   # dprint fmt (Markdown, Rust, TOML, YAML)
just fmt_check             # dprint check
just lint "-- -D warnings" # alias of just clippy
just doc                   # cargo doc --no-deps with RUSTDOCFLAGS="-D warnings"
just deny                  # cargo deny check
just scan_secrets          # trufflehog filesystem
just check                 # the full local quality gate
just setup_githooks        # point core.hooksPath at .githooks
just changelog_preview 0.2.0
just changelog 0.2.0
just publish "--dry-run --allow-dirty"
```

`just check` is the required gate before declaring work done. It chains
`fmt_check`, Clippy with warnings denied, `doc`, `deny`, and `test`.

All tests are plain unit and doc tests; nothing here needs a container or a
network connection to run.

If a required tool is missing, say so. Never claim a check passed or silently
swap in a weaker command.

## Architecture

remotefs-memory is a [remotefs](https://github.com/remotefs-rs/remotefs-rs)
client implementation backed entirely by an in-memory tree, useful for tests
and simulations. It is a library-only crate (`src/lib.rs`, crate name
`remotefs_memory`) with no binaries or examples.

- **One client.** `MemoryFs` in `src/lib.rs` implements `remotefs::RemoteFs`
  (remotefs 1) over an `orange_trees::Tree<PathBuf, Inode>` (`FsTree`) held
  behind `Arc<Mutex<FsTree>>` so every operation takes `&self`; the connection
  flag is an `AtomicBool`. There is no working directory: every path must be
  absolute and is validated with `remotefs::path::ensure_absolute` before the
  connection check. `rename` and `copy` clone the whole subtree and rewrite
  its ids; `remove_dir_all` drops the subtree in one call without following
  symlinks. `exec` is unsupported.
- **Inode.** `src/inode.rs` defines `Inode`, the value stored at each tree
  node: `Metadata` plus optional file content (`None` for directories).
  `Inode::dir`, `Inode::file`, and `Inode::symlink` are the only ways to
  construct one.
- **Streams.** `src/stream.rs` holds `MemoryReader` (a seekable cursor over a
  ranged snapshot of the file) and `MemoryWriter` (stages bytes in a cursor
  and commits content, size, and `modified` into the tree on `finish`). The
  inode is inserted when the stream opens; a dropped writer discards the
  staged bytes and logs at `debug` level.
- **uid/gid.** `get_uid`/`get_gid` are pluggable
  `Box<dyn Fn() -> u32 + Send + Sync>` closures (default: always `0`),
  overridable via `MemoryFs::with_get_uid`/`with_get_gid`, since an in-memory
  filesystem has no real owner to read from.
- **Command layer.** `Justfile` is a thin importer. Each recipe group lives in
  its own file under `just/` (`build`, `test`, `code_check`, `changelog`,
  `publish`) and carries a `[group(...)]` attribute so `just --list` stays
  organized. Recipes take an `args=""` passthrough rather than hard-coding
  flags.
- **Formatting is dprint, not cargo fmt.** `dprint.json` owns Markdown, TOML,
  and YAML, and delegates `.rs` files to nightly rustfmt through its exec
  plugin (`--edition 2024`, matching this crate's `package.edition`).
  `rustfmt.toml` uses nightly-only options (`imports_granularity`,
  `group_imports`), which is why nightly is required. Always format with
  `just fmt`.
- **Release path.** Commits follow Conventional Commits and `cliff.toml` turns
  them into `CHANGELOG.md`. Publishing goes through `just publish`
  (`cargo publish --locked`); version bumps live in `Cargo.toml`.
- **Supply-chain policy.** `deny.toml` is strict: license allowlist,
  `yanked = "deny"`, `unmaintained = "all"`, wildcard versions denied, and
  crates.io as the only allowed source. It runs with `all-features = true`.

## Conventions

- Toolchain is pinned to Rust 1.98.1 (`rust-toolchain.toml`). `package.edition`
  in `Cargo.toml` is 2024 and `package.rust-version` is 1.89; do not bump
  either as part of unrelated changes.
- Public library items need canonical rustdoc, including a runnable example.
  `just test` runs doctests, and `just doc` denies warnings.
- Keep `Cargo.toml` dependency and feature entries alphabetically sorted, with
  bare minimal versions.
- Conventional Commits, imperative and lower-case. No agent attribution,
  session links, or agent `Co-Authored-By` lines.
- Do not stage planning state. `docs/superpowers/`, `.superpowers/`, and
  `.claude/plans/` are gitignored and dprint-excluded.
- After editing a Markdown file that contains a table, run
  `fmt-md-tables -i <file>`.
- After any change under `.github/workflows/`, run `zizmor .github/workflows`
  until it exits clean. Pin actions to a full commit SHA with the matching tag
  in a trailing comment, declare least-privilege permissions, and set
  `persist-credentials: false` on checkout.
