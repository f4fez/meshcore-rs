# CLAUDE.md

Project-specific context for Claude Code sessions on this repo, derived from this project's own
CI config, `Makefile`, and past PR review history (`git log`) — not invented.

## Project overview

`meshcore-rs` — Rust library for communicating with MeshCore companion radio nodes over serial,
TCP or BLE (each behind an optional feature flag: `serial`, `tcp`, `ble`; all three are default).
Maintained by Andrew Mackenzie (`andrewdavidmackenzie`); PRs are reviewed by Andrew and by
CodeRabbit (automated review bot). Published to crates.io.

`fez-mesh-controller` (the sibling project) depends on this crate via a local
`[patch.crates-io]` path override, for features not yet released — so a fix made here often
exists to unblock that project too.

## Before opening a PR: local verification checklist

Run **`make pr`** (= `make checks tests`) — this is the project's own canonical pre-PR command,
covering everything CI checks except code coverage upload and the multi-OS build matrix:

- `make format` — `cargo fmt --all -- --check`
- `make clippy` — `cargo clippy --tests --no-deps --all-features --all-targets`
- `make publish` — `cargo build --release && cargo publish --no-verify --allow-dirty` (dry-run-style sanity check)
- `make udeps` — `cargo +nightly udeps` (**not installed locally** — install with `cargo install cargo-udeps` before relying on this, or flag the gap to the user)
- `make todos` — greps for `TODO` in `*.rs`; CI's `todo-check` job does the same and a past PR review explicitly flagged a leftover TODO — don't leave any in code you're submitting.
- `make debug` / `make release` — `cargo build [--release]`
- `make test` — `cargo test`

`make features` (`cargo check-all-features`) is in the full `make all` target but the
`cargo-all-features` plugin providing it is **not installed locally** either.

### Jonesy — panic-point analysis

A custom static analyzer (also by Andrew, `andrewdavidmackenzie/jonesy`) that finds panic points
in the built lib via DWARF debug info, runs as a required PR check (`.github/workflows/jonesy.yml`,
annotates the PR diff). Installed locally at `~/.cargo/bin/jonesy` (v0.10.0). Run it after any
change:

```sh
cargo build && jonesy
```

Configured via `jonesy.toml` (project-wide `allow`s for `expect`/`capacity`/`format`/
`async_resumed`, plus per-function `bounds`/`overflow` allowances for specific `parsing.rs`
functions) and inline `// jonesy:allow(<cause>)` comments for one-off suppressions. New panic
points introduced by a change should be fixed (bounds-checked access, `checked_*`/`saturating_*`
arithmetic) rather than blanket-allowed, unless there's a specific reason a panic there is
acceptable — in which case allow it explicitly (config or inline) with a comment explaining why,
matching the existing entries' style.

### Code coverage

CI (`build_and_test.yml`) generates coverage via `grcov` → `lcov` → uploads to Codecov with
`fail_ci_if_error: true` — a broken/missing coverage upload fails the build. `grcov`/`lcov`
aren't installed locally, but **`cargo-llvm-cov` is** (a faster local equivalent) — use it to
check coverage on new code before opening a PR:

```sh
cargo llvm-cov --summary-only            # per-file coverage %
cargo llvm-cov report --show-missing-lines   # exact uncovered line numbers
```

Check that the lines you *added or changed* aren't in the missed-lines list for their file —
don't chase the repo's overall coverage number (some files, e.g. `meshcore/ble.rs`, are
essentially untested today and that's out of scope for an unrelated change).

## Patterns from past PR review feedback (CodeRabbit + Andrew)

Concrete, observed in `git log` (`9259549`, `b9aec3c`, `fc99b98`) — not general best-practice
guessing:

- **Bounds/validation gaps that cause silent misbehavior get flagged hard.** E.g. a truncated
  advertisement's location field was silently parsed past its end, misaligning the subsequent
  `name` field, instead of the parse cleanly failing. Prefer failing loudly (`Option`/`Result`)
  over reading past a boundary. Add a regression test for the truncated-input case specifically.
- **Missing upper-bound validation on a length/count parameter before it's used to build a
  request** was flagged (`pubkey_prefix_length` needed an explicit `<= 32` check). Validate
  caller-supplied sizes before using them, don't rely on the callee to reject them gracefully.
- **`tokio::time::sleep`-based test synchronization is flagged as flaky.** Replace fixed-delay
  sleeps with an explicit readiness signal (e.g. poll with a short `tokio::time::timeout` loop
  until the expected precondition is observed) rather than hoping a delay is long enough.
- **Doc comments must stay consistent with the actual behavior and with other docs describing
  the same thing** — a stale comment describing old semantics (contradicting a corrected doc
  elsewhere in the same PR) was caught and had to be fixed.
- **Example usage comments must include required `--features` flags** — an example's doc header
  was missing `--features serial` in its `cargo run` invocation.
- **No stray `TODO`s** — CI's `todo-check` job (and past review) flags these; resolve or remove
  before submitting, don't leave a `TODO` comment as a placeholder.

## Recently found & fixed here (context for related work)

`remove_contact`/`export_contact` (`src/commands/base.rs`) sent only a 6-byte public-key prefix
despite their own doc comments documenting the wire format as requiring the full 32-byte key —
present since the crate's initial commit, never caught because no test exercised either function
with an actual key. Fixed to use `Destination::public_key()` (erroring cleanly if only a prefix
is available), matching the already-correct pattern in `send_login`/`send_logout`/
`send_binary_req`. Verified against a real node via the new `examples/add_remove_contact.rs`.
Lesson: when adding a `Destination`-taking command, check the doc comment's declared wire format
against what's actually sent — `.prefix()` (6 bytes) and `.public_key()` (32 bytes, `Option`)
are easy to confuse, and nothing catches the mismatch without a test using a real key/prefix.
