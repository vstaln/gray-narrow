<div align="center">
  <img alt="gray-narrow" src="assets/icon.svg" width="120" height="120" />
  <h1>gray-narrow</h1>
  <p><strong>Capability narrowing for delegated gray children. A child can only shrink its parent's tool grant.</strong></p>
  <p>
    <a href="https://gray.alignment.id">Website</a> ·
    <a href="https://gray.alignment.id/plugins/gray-narrow">Store</a> ·
    <a href="https://github.com/vstaln/gray-narrow">Source</a> ·
    <a href="https://github.com/vstaln/gray">gray</a>
  </p>
  <p>
    <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-1c1c20?style=flat-square&labelColor=0a0a0b" /></a>
    <a href="https://www.rust-lang.org"><img alt="Built with Rust" src="https://img.shields.io/badge/built%20with-rust-1c1c20?style=flat-square&labelColor=0a0a0b&logo=rust&logoColor=d4a373" /></a>
    <a href="https://gray.alignment.id/plugins/gray-narrow"><img alt="gray plugin" src="https://img.shields.io/badge/gray-plugin-1c1c20?style=flat-square&labelColor=0a0a0b&color=7aa2f7" /></a>
  </p>
</div>

<br/>

```bash
gray plugin install gray-narrow
```

Capability narrowing for delegated gray children — a port of pi-daddy's grant
model (MIT). A delegation may only ever **shrink** the tool set it hands down:
a child can never exceed its parent's grant, and every grant decision is
recorded in an append-only ledger.

```
gray narrow run --tools "read,grep,bash:git*" 'summarize this repo'
gray subagents run --tools "read,grep" 'find the entry point'   (integration)
```

## How caps reach a child

Gray core has **no** tool-allowlist flag on `gray -p` (verified on the current
CLI), and no manifest-subset mechanism. The channels that do exist:

- **Env**: `gray-subagents` spawns children as `gray -p <task> --json` with
  `env = os.environ + GRAY_*` — `GRAY_*` variables pass straight through.
  `gray-narrow` carries the grant as `GRAY_NARROW_GRANT` (comma-separated
  capability ids), plus `GRAY_NARROW_DEPTH`/`GRAY_NARROW_MAX_DEPTH`/
  `GRAY_NARROW_RUN` for delegation depth and ledger correlation.
- **Enforcement**: `tool/before` (protocol-1.1 wire) sees every tool call's
  `{name, args, session}` and answers `{decision:"deny", reason}`. Because an
  installed plugin sidecar runs inside *every* gray process — including
  `gray -p` children — the env-carried grant is enforceable inside the child.
- `gray-subagents` calls `gray-narrow plan` before spawning when `--tools` or
  a profile `tools:` frontmatter field is present (or whenever a
  `GRAY_NARROW_GRANT` would be inherited): it resolves the child's effective
  grant against the parent's, writes the ledger event, and returns the exact
  `GRAY_NARROW_*` env to set. Resolution happens **parent-side** — the child
  can never mint itself a wider grant.

## Capability grammar

| form | meaning |
|---|---|
| `read` or `tool:read` | the `read` tool, any args |
| `bash:git*` | `bash` calls whose command matches the glob `git*` |
| `read:/tmp/*` | `read` constrained to paths under `/tmp` |
| `agent:scout` / `agent:*` | may spawn that agent profile / any profile |
| `*` or `tool:*` | every tool (root-equivalent wildcard) |

Arg-globs match the call's *primary argument* (`command` for bash, `path` for
file tools, `url`/`query` for web tools, etc). No commas/whitespace/newlines
inside an id — the grant is a comma-separated env var and ids are the ledger's
audit fields.

## Semantics (ported from pi-daddy)

- `resolve()` is the whole security surface — a pure function:
  `effective = requested ∩ expandSubsumed(parentGrant)`, with every rejection
  reported (`denied` / `gated_blocked`), never silently dropped.
- **Subsumption**: `tool:bash` confers `read`, `write`, `edit`, `grep`,
  `find`, `ls` (a shell can do all of them). Used in grant resolution and the
  `subsumed_by` audit field — *not* in call-time enforcement, where the
  allowlist is literal.
- **Gated**: `tool:bash` is gated by default — a bash-holding child can run
  `env -u GRAY_NARROW_GRANT gray -p` and escape governance entirely. Granting
  it (or a glob covering it) needs `--approve 'tool:bash'`. Denied and
  unapproved-gated requests refuse the whole delegation (fail closed).
- **Depth**: `GRAY_NARROW_DEPTH`/`GRAY_NARROW_MAX_DEPTH` (default 2). A
  malformed bound fails closed — delegation is refused outright.
- **Ungoverned sessions**: no `GRAY_NARROW_GRANT` → `tool/before` allows
  everything (governance is opt-in). The parent grant for a fresh delegation
  is `tool:*`.
- **Ledger**: append-only JSONL at `~/.gray/narrow-ledger.jsonl`
  (`GRAY_NARROW_LEDGER` overrides). `grant` events record every
  parent → child decision including refusals; `deny` events record every
  enforcement-time block.

## Commands

- `gray narrow run [--tools CSV] [--agent PROFILE] [--approve CSV] [--model M] [--effort E] [--cwd D] [--name N] 'task'` — spawn a detached `gray -p` child under the resolved grant (log + record under `~/.gray/narrow/runs/`).
- `gray narrow plan --request CSV [--agent P] [--approve CSV] [--via V]` — resolve and print `{env, effective, denied, …}` as JSON; the seam `gray-subagents` calls.
- `gray narrow grants` — this process's effective grant and depth.
- `gray narrow status` — narrow-spawned runs and liveness.
- `gray narrow ledger [--lines N]` — tail the ledger.
- `/narrow grants|status|ledger|help` — same, in-session.
- `narrow_delegate` tool — the model-facing spawn: `{task, tools?, agent?, name?, approve?}`.

## Install / setup

```sh
cargo build --release && cargo install --path .   # or: put target/release/gray-narrow on PATH
gray plugin install /path/to/gray-narrow          # registers sidecar + `gray narrow` CLI
```

Enforcement is only live in processes where the plugin is installed — the
sidecar is what answers `tool/before`. `gray subagents run --tools` resolves
`gray-narrow` via `GRAY_NARROW_BIN` or PATH; without it an explicit `--tools`
request is refused (fail closed), while grant *inheritance* still passes
through env.

## Limits — what core support would fix

- **Env is forgeable.** A child holding `bash` can relaunch
  `env -u GRAY_NARROW_GRANT gray -p …` — that is exactly why `bash` is gated.
  Real containment needs gray core to enforce a tool allowlist (e.g. a
  `--tools`/`--no-tools` flag on `gray -p`, like pi) so the allowlist is set
  at spawn, not read from forgeable env.
- **Grant authenticity**: a signed/HMAC grant token, or core-owned cap state
  (the way `GRAY_SUBAGENTS_ACTIVE` is only a convention), would let a child
  prove rather than merely carry its grant.
- **Plugin presence is the enforcement.** If gray-narrow is not installed in
  the child, nothing denies. `gray subagents run --tools` fails closed only
  when the *parent* lacks the binary; core-side enforcement would not depend
  on plugin install state.
- **Grant ≠ tool surface.** The grant intersects the child's actual tools at
  call time; it cannot add tools (a `tools-minimal` child only ever has
  `bash`).
- Deny events are logged per call; a model retry-loop grows the ledger
  linearly (append-only, small lines).

## Tags

`gray` `plugin` `narrow` `rust`

---
Part of the [gray](https://github.com/vstaln/gray) plugin ecosystem, the open-source AI agent harness. <https://gray.alignment.id>
