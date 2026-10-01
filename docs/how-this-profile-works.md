# How lsp_rust works

This repository is a Loom language profile for `rust-analyzer`. It holds
no code. It holds one `extension.toml` that tells
[Loom](https://github.com/Roasbeef/loom) how to run the server in its
jail, a small Cargo crate in `fixture/` that the server loads, and a CI
workflow that proves the two agree. This document walks through each of
them so you can read `extension.toml` without having Loom's source open.

For the machinery behind it, see Loom's
[language-server architecture](https://github.com/Roasbeef/loom/blob/main/docs/architecture/lsp.md)
and [ADR-016, language profiles](https://github.com/Roasbeef/loom/blob/main/docs/adr/016-language-profiles.md).
The short version: Loom speaks the Language Server Protocol and knows no
language. A server runs project code (`rust-analyzer` runs `cargo`, build
scripts and proc macros), so Loom runs it inside a sandbox, and everything
that sandbox must grant, plus the few facts a language spells differently,
comes from a profile like this one. Rust is in the first-party set
because Loom's original defaults could not serve it: it qualifies names
with `::`.

## Reading path and design rules

This is the only design document in the repository, because the profile
is one short TOML file and a split into principles and architecture would
repeat itself. Read `extension.toml` first, then `fixture/`, then
`.github/workflows/check.yml`; this document explains each in that order.
Three rules shaped the file.

### Never make anything under ~/.cargo writable

A project's build scripts and proc macros run inside the server's jail.
One that could write under `~/.cargo` could plant Cargo configuration or
edit crate sources that later build on the host. So the profile grants
`~/.rustup` and `~/.cargo/registry` read-only and nothing writable, and
Loom mounts only the directories of the files the `rust-analyzer` link
leads through, never the whole `~/.cargo` prefix (which would expose a
registry token). The cost, that a crate needs a committed `Cargo.lock`, is
paid in the fixture and the README.

### Say in data what the resolver cannot say in code

Rust qualifies names with `::` and often writes a leading `crate::`. The
first is a key, `qualifier_separators`. The second names no directory, so
no key can match it, and the profile tells the model to leave it out with
a `hint`. A convention the resolver cannot express is a sentence to the
model, not a new mechanism in Loom.

### Put the proof beside the claim

A profile asserts that a server loads a project and answers inside the
jail. The `[[check]]`s and the fixture are that assertion made
executable, and CI runs them against a real Loom, so a claim in
`extension.toml` that stops being true fails a build instead of a
user's session.
The `println!` reference in the fixture exists to prove the registry
grant: lose the grant and that site disappears.

## extension.toml, key by key

### `[extension]`

`name = "lsp_rust"` is what `loomd ext check` takes, and `tier =
"profile"` says this extension is data only. A profile may declare
language-server tables and checks. It may not declare tools, hooks or a
network policy, and Loom refuses the manifest if it does. `version`,
`description` and `license` are ordinary metadata.

### `[lsp.rust]`

The table name, `rust`, is the server's name inside Loom. A `loom.toml`
table with the same name replaces this one whole, never field by field.

`command = ["rust-analyzer"]` is the argv Loom executes. It is never a
shell string. A bare name is looked up on the daemon's `PATH`. With
rustup, `rust-analyzer` in `~/.cargo/bin` is a link to `rustup`, so the
jail follows the link and mounts the directory of each file it passes
through, read-only. Mounting the whole `~/.cargo` prefix would put
`credentials.toml`, a registry token, inside the jail, so Loom does not.

`extensions = [".rs"]` claims every `.rs` file for this server. A file
has exactly one owning server, so two profiles claiming `.rs` conflict
and Loom refuses both.

`root_markers = ["Cargo.toml"]` chooses the project: the nearest ancestor
directory of a file that holds a `Cargo.toml` is the crate root, and it
is the root the server is started on. A file whose real location lies
outside that root is refused before any request is sent.

`project` is not set, so it defaults to `"read-only"`, and nothing in
this profile is writable. Measurement found nothing needed to be. With a
committed `Cargo.lock`, `cargo metadata` writes nothing. `cargo check`,
which `rust-analyzer` runs for build scripts and proc macros, cannot
write `target/` and says so. A crate without build scripts or proc macros
is fully analysed; one with them loses what they generate.

`language_id = "rust"` is set because the default, the first extension
without its dot, would be `rs`, and `rust-analyzer` opens documents as
`rust`.

`qualifier_separators = ["::"]` is why Rust is here. Loom splits a
qualified name on `.` by default, and Rust writes `util::greet`. With this
key the qualifier is `util`, and it must end the definition's file path
without its extension (`src/util.rs`).

`hint` is one line, appended once to `lsp_definition`'s description in
sessions that configure this server. Rust code often spells a path with a
leading `crate::`, which names no directory, so the resolver cannot match
it. Rather than add a key for that, the hint tells the model to write
`util::greet`.

`readable = ["~/.rustup", "~/.cargo/registry"]` grants two read-only
roots, each of which was measured:

- `~/.rustup` holds the toolchains every rustup proxy dispatches to.
  Without it, the proxy fails with "could not create home directory:
  '~/.rustup': Read-only file system".
- `~/.cargo/registry` holds the standard library's own dependencies
  (`hashbrown`, `libc` and a few more), which `rust-analyzer` resolves to
  load `std`. The jail has no network, so they must already be there.
  Without them, the call inside `println!`, a `std` macro, is never
  found: the references check loses `src/main.rs:4`.

Both are read-only. Nothing under `~/.cargo` is ever writable, since a
build script that could write there could plant Cargo configuration or
crate sources that later run on the host. The README says how to
populate the registry once, online.

There is no `cache_env` and no `env`: the project is read-only and the
server needs nothing from the daemon's environment. The profile assumes
rustup's default layout. If `RUSTUP_HOME` or `CARGO_HOME` point elsewhere,
copy the table into `loom.toml` with those directories instead.

### `[[check]]`

A check is one question asked of a running server through the same door
Loom's tools use, and a list of the sites the answer must equal. Sites
are fixture-relative `path:line`, compared as a set, so two hits on one
line count once and order does not matter. This profile has two:

- **`definition` of `util::greet`, expecting `src/util.rs:1`.** It proves
  that the server starts under the jail's policy, that `::` splits the
  name, and that the qualifier `util` narrows the candidates to the right
  module.
- **`references` of `greet`, asked from `src/util.rs` line 1, expecting
  `src/util.rs:1`, `src/main.rs:4` and `src/main.rs:5`.** The call on line
  4 sits inside `println!`, so this check passes only when `std` loaded
  completely, which is what the registry grant is for. The check names a
  `path` and `line`, so it also proves the narrowed form of a query.

Both checks also depend on a behaviour specific to this server.
`rust-analyzer` answers requests while it is still loading the workspace,
and answers them with empty results instead of errors. Loom therefore
waits, after it starts a server, until the server's work-done progress
has been quiet for 300 ms, bounded at a minute. Measured through the jail
on this kind of crate, the server was quiet about 3.5 to 3.75 seconds
after the handshake. Without the wait, the same definition answers empty.

## The fixture

`fixture/` is a Cargo crate with no dependencies and a committed
`Cargo.lock`, so `cargo metadata` has nothing to write. `src/util.rs`
declares `greet` on line 1. `src/main.rs` declares `mod util;` and calls
`util::greet()` on lines 4 and 5, the first inside `println!`. The checks
assert line numbers, so editing a fixture file means updating `expect`.

## What the CI does

`.github/workflows/check.yml` runs on pushes to `main`, pull requests and
manual dispatch. Its single job proves the profile against a real Loom:

1. **Check out two repositories.** This one goes into `profile/` and Loom
   into `loom/`, at the revision `LOOM_REV`, so the tree that is later
   installed is exactly this repository.
2. **Install the build tools.** Erlang/OTP, Gleam and rebar3 build and
   run `loomd`, and Go builds the sandbox helper.
3. **Prepare the runner for the sandbox.** Install `bubblewrap` (the
   namespace and mount work) and `ripgrep` (what a bare-name question
   searches with). Lift Ubuntu 24.04's AppArmor restriction on
   unprivileged user namespaces, and delegate a cgroup v2 base to the
   helper with probes that prove the delegation is real.
4. **Build Loom.** `make -C loom sandbox` builds the helper, and `make -C
   loom server-shipment` builds `loomd`, retrying Hex fetches.
5. **Install `rust-analyzer` and the standard library's sources.** Loom's
   `.github/scripts/install_rust_analyzer.sh` pins the toolchain and
   fetches `std`'s dependencies into the Cargo registry while the runner
   still has a network, since the jail will not.
6. **Install the profile and check it.** `loomd ext install` on the
   `profile/` checkout, then `loomd ext check lsp_rust`. The job passes
   exactly when that command exits 0.

### Re-proving against a newer Loom

`LOOM_REV` in the workflow's `env` block is the Loom revision the profile
is proven against: a commit SHA on Loom's `main`, starting at the merge
that landed language-server support (loom#680). To re-prove the
profile against a newer Loom, change that one value, push, and read the
`check lsp_rust` job. A failure prints a `FAIL` line naming the expected
and the actual sites. To do the same by hand, build `loomd` from that
Loom and run the last step's two commands.
