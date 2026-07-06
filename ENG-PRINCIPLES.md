# Engineering Principles

> A living document. Codified rules for how we build. Each principle is stated as an imperative so it can be checked, enforced, and argued about.

---

## 1. Typing

- **ALWAYS use strong typing.** No stringly-typed data, no untyped blobs, no "we'll validate it later." If a value has a shape, that shape has a type.

- **ALWAYS define types in one language.** Rust is the source of truth. Every other language's types are *generated* from the Rust definitions by repo-wide codegen scripts — never hand-written, never allowed to drift.

- **ALWAYS present types in a human-readable form.** Codegen also emits a Markdown (and/or script) rendering of every type, so a reader can understand the data model without reading Rust.

- **ALWAYS comment every field.** A field without a comment is a bug. The comment explains *what it means* and *why it exists*, not just restates the name.

- **ALWAYS make illegal states unrepresentable.** Prefer enums/unions over booleans-plus-conventions. If two fields can't both be set, model that in the type, not in a runtime check.

- **ALWAYS use single-key-encoded JSON for unions.** A tagged value is `{ "VariantName": { ...payload } }` — the variant is the single top-level key, not a `"type"` discriminator field alongside the payload.

- **ALWAYS use `snake_case` for JSON field names**, except single-key type names and union variant names, which are `CamelCase`.

- **ALWAYS strict-reject unknown fields on deserialization** (`deny_unknown_fields`), virtually everywhere — config files very much included. An unknown key is a hard, loud error: typos and drift die instantly instead of silently becoming "field never set." Since all types are codegen'd from one Rust source of truth, drift *is* a bug. Lenient parsing is reserved for the rare explicit boundary you don't control (e.g. a third-party API) — liberal with humans (§2), strict with machines.

- **Do NOT version types by default.** No `FooV1`/`FooV2`, no version numbers in type names. Prefer clean code and short type names over compatibility verbosity. Breaking changes are generally allowed — the codebase moves forward.

- **Backwards compatibility is opt-in, not the default.** It matters only for data stored long-term, and only when compatibility is *explicitly* important for that data. In those specific cases, keep the old shape deserializable; the rest of the time, break freely and migrate the code.

## 2. CLI

- **ALWAYS make CLI output human-friendly by default:** color, bold, aligned, copy-paste-ready, complete but not verbose. Optimize for a human reading it in a terminal.

- **ALWAYS support `--json`**, emitting two-space-indented JSON. `--json` is *on by default when the output is not a TTY* (piped, redirected, captured), so scripts and pipelines get machine-readable output for free.

- **Machine output is a single-key union document — success and failure alike.** Per §1, every machine-mode result is `{ "VariantName": { ...payload } }`, and the variant names are request-specific where that adds meaning: `{ "SuccessfullyAdded": { ... } }`, `{ "NotFound": { ... } }`, not one generic wrapper for everything. Scripts branch on the top-level key; the exit code remains the authoritative success/failure signal. The extra jq path segment is the price of a self-describing, one-worldview output — pay it.

- **ALWAYS make commands self-descriptive:**
  - `help <command>` prints help for that command.
  - Bare `help`, `--help`, and invoking with no arguments all print comprehensive help.
  - Every command, flag, and subcommand carries its own one-line description.

- **`help` recommends the canonical command; the CLI is liberal in what it accepts.** Help output always shows the one canonical form. At the same time, accept non-canonical, abbreviated, or simplified invocations whenever they *unambiguously* mean what the caller intended — resolve to the obvious command rather than erroring on a technicality. Only refuse when the intent is genuinely ambiguous. When a non-canonical invocation is resolved, print the canonical form to stderr as a gentle recommendation — in dim/grey when color is on, so it reads as a nudge, not a warning — and humans learn the real spelling for free.

- **Liberal acceptance is for humans only — machines get the canonical syntax told back to them.** When run by a machine (no TTY, or `--json`), a non-canonical invocation does NOT run — no exceptions, not even for exact, never-ambiguous synonyms: the CLI refuses to proceed and responds with an error stating the correct canonical syntax (as a single-key JSON error, per below), until the caller provides the canonical command. This keeps aliases and abbreviations out of scripts and harnesses, where they would rot: an abbreviation that is unambiguous today can become ambiguous when a new subcommand lands.

- **ALWAYS separate streams: data → stdout, everything else → stderr.** Logs, progress, prompts, and diagnostics go to stderr so that `cmd | jq` (and any pipeline) sees clean data on stdout, never polluted. In machine mode the error document *is* data — it goes to stdout (per below) — and stderr stays quiet by default: nothing lands there beyond §9 liveness output, unless a verbose mode explicitly asks for more.

- **ALWAYS format errors the same way as output.** Errors respect the same toggle as everything else: plain, human-friendly text for a terminal; two-space-indented, single-key-encoded JSON — `{ "Error": { ... } }`, or a more specific variant per the union rule above — whenever `--json` is on or the output is not a terminal. A harness or automation engine must always see clean JSON — never a stack trace on stdout, never plain text where JSON was promised. In machine mode a failed command still emits exactly one JSON document *on stdout*, plus its non-zero exit code — never bare prose to stderr while stdout stays empty. Whatever captures stdout always gets something parseable.

- **Warnings land in BOTH channels.** Warnings go to stderr for humans *and* are folded into the JSON document as a `warnings` array in machine mode. A warning only on stderr is invisible to most scripts; one only in the JSON is invisible to humans.

- **Exit codes MAY be custom, but MUST be documented VERY cleanly.** Beyond the basics (`0` ok, `1` failure, `2` usage), a command may define its own codes — but every code is documented, and `help exitcodes` prints the complete table.

- **Exit codes draw the distinctions that change the caller's next move.** Separate "bad credential" (remedy: log in again) from "valid credential, missing permission" (re-login can never fix it) — collapsing these sends users into re-login loops. Give "quota/limit exhausted" its own code so callers branch to wait-or-upgrade instead of blind retry. Mirror the classification in a machine-readable `stage` field inside the JSON error document (`"auth"`, `"permission"`, `"quota"`, `"usage"`, `"request"`), so scripts can branch on stage even where codes overlap.

- **Exit `130` on SIGINT; die quietly on SIGPIPE.** Ctrl+C is not a crash — no stack trace, exit `128+2`. Piping into `head` is not an error either: a broken pipe ends the program silently, never with a traceback.

- **Machine output is a public API.** For a published CLI, the exit-code table and the shape of the `--json` output are exactly the kind of long-lived surface where §1's opt-in backwards compatibility is opted *into*: changing what a code or a field means is a breaking change — version it, document it, and spell out what scripts must do differently.

- **A CLI over an API curates; it never proxies.** Each command reduces the server response to what the user asked for — a write returns its id and a compact summary, a query returns the answer. Server internals (SQL, execution plans, intermediate trees) are stripped in BOTH output modes; `--verbose` adds a small *whitelist* of extra context (a trace id, a deep-link URL), it is not a "show everything" switch. If a field is internal, no flag reveals it.

- **Never trust the HTTP status code alone.** If the API's response body carries its own error channel, reject a non-empty `errors[]` even on HTTP 200, before touching the payload. Treat a non-JSON body from a JSON API as an error, and include the first ~200 characters of the body in the message, so the failure is debuggable from CLI output alone.

- **Async operations return an id immediately; `wait` absorbs the polling loop.** Synchronous is the default — the simple path is the short one; `--no-wait` returns a job id, and a separate wait/status command polls to a terminal state. That command accepts MANY ids at once, polls them concurrently (bounded worker pool), preserves input order in the output, and exits with the *worst* individual outcome (timeout > failed > ok). One failing id never kills the batch: per-item errors ride inline in that item's result row — partial results are results. Poll against a monotonic-clock deadline and mark timed-out items explicitly.

- **`version` and `status` work offline.** The two questions support always asks — *what build is this?* and *is there a credential, where, for whom?* — must be answerable with no network.

- **Document what the CLI does NOT do.** Keep a "not this tool's job" table, each row pointing at where that job IS done (web UI, raw API). A deliberate absence documented is a support ticket avoided; an undocumented absence reads as a bug.

- **Color by default for humans; easy to turn off.** A human at a terminal gets color. `NO_COLOR` (env), `--color=never`, or `--color=no` all disable it. Non-terminal output is never colored.

- **Learn from the best.** Model the CLI's ergonomics on `git`, `gh` (GitHub), `yt` (YouTrack), and `aws` (esp. `aws s3`): subcommand structure, discoverable help, consistent flags, sensible defaults, and machine-readable escape hatches.

## 3. Testing

- **ALWAYS test the code.** No exceptions.

- **If code is not unit-tested, ALWAYS comment HOW it is tested** — in the code, right where the untested logic lives.

- **Prefer an "executable Markdown harness" over long shell scripts.** Instead of a sprawling `.sh`, write an `UPPER-DASH-CASED.md` file describing what to do and what the expected result is, so a single harness run exercises it end-to-end. These `.md` files are *executable by design*.
  - **Run by a script** — as a pre-push hook — plus a GitHub Action gate, so the harness blocks both push and merge.
  - **Deterministic where it can be, LLM-judged where it must be.** Mechanical checks run deterministically; an agent also judges the prose assertions, so the check *legitimately fails and blocks progress* when a rule is not actually satisfied.
  - **Verbose yet compact.** Written so humans understand it *and* LLMs spend fewer tokens and run faster — dense, not padded.
  - This doc only outlines the recommendation; the concrete mechanics are left to each end-user repo.
  - **The human-perspective variant is codified in §12** — what a human must configure by hand, and the executable harness that proves that configuration sufficient.

- **ALWAYS gate on tests in BOTH GitHub CI and pre-push hooks.** Committing locally stays free and fast; the full gate runs once, when code is about to leave the machine (`pre-push`), and again as a required check before merge (CI). Document both gates in `CONTRIBUTING.md`.

- **The gate passes with ZERO warnings — in debug AND release builds, and in the debug AND release TEST builds.** All four build flavors are warning-free, with the test code held to the same bar as the code it tests. For Rust that is `cargo clippy --workspace --all-targets -- -D warnings`, and the same command again with `--release` (`--all-targets` is what pulls the tests, benches, and examples in). A warning that survives one merge is background noise by the next; the only warning count that stays readable is zero.

- **The gate NEVER publishes — but it ALWAYS proves that publishing would work.** If the repo has anything publishable (crates.io, PyPI, npm, …), releasing stays a deliberate human act, never a side effect of a green build. The gate instead runs the dry run of publishing — for Rust `cargo package --workspace`, which also resolves not-yet-published sibling crates against the locally built packages, so it works in the window after a version bump and before the release; for Python `python -m build` plus `twine check` — and requires it to BUILD and to be WARNING-FREE. A warning at packaging time means the published artifact would silently differ from the repo's intent (files that do not ship, targets that get dropped, metadata that does not validate), so it fails the gate exactly like a failing test.

## 4. Web

- **Web UIs are 100% functional, neat-looking, and minimalistic.** No dead buttons, no decorative clutter. Every element does something.

- **The heuristic is binary: EITHER it is trivial — and vanilla JS does miracles — OR it is WASM.** There is no middle ground. Anything with real logic or state is strongly-typed Rust compiled to WASM. Anything trivial is a few lines of plain vanilla JS. Nothing lives in between (no jQuery-era sprawl, no half-typed JS "apps").

- **Raw `wasm-bindgen` is the house default.** No framework (Leptos/Yew/Dioxus) unless a repo has a specific reason — go straight to `wasm-bindgen`.

- **WASM is fine everywhere, including public pages.** A multi-hundred-KB WASM blob is not a problem; do not contort the architecture to shave bytes off first paint.

- **Prefer a shared, minimal component and design-token set over per-repo reinvention.** Keep it small, keep it neat, but reuse it — don't hand-roll a new button and a new palette in every repo.

## 5. Error Handling & Logging

- **Panic only on true programmer bugs.** `panic!` / `unwrap()` / `expect()` are for invariant violations that should be impossible — never for runtime-fallible operations (I/O, parse, network, user input), which always return `Result`. In non-test code, `unwrap()`/`expect()` are allowed only where the invariant is provable, and then with a message that says *why* it holds.

- **Use `thiserror` and `anyhow` liberally — don't be shy.** Typed error enums (`thiserror`) where callers branch on variants; `anyhow` with context where they don't. Reach for whichever fits. This is the Linux Way: every project and sub-project should do one thing and do it well, with errors scoped tightly to that job.

- **Logs are structured yet compact — full JSON is overkill.** A log line is a compact structured record, not a pretty-printed JSON object. Logs are read mostly by agents, so keep them dense and machine-friendly — while staying human-understandable too.

- **ALWAYS provide configurable verbosity levels** (`error` / `warn` / `info` / `debug` / `trace`). `info` is the default and must stay quiet enough to read; anything spammy belongs at `debug` or lower.

- **Logging respects the same modes as the CLI: human (TTY) vs. `--json` vs. neither.** The agent-vs-human distinction matters for logs exactly as it does for command output — pretty and colored for a human at a terminal, compact-structured for agents and pipelines.

## 6. Git, Commits & Pull Requests

- **Linear history only.** No merge commits — rebase onto `main`. History reads as a single straight line.

- **Commit messages are short, complete sentences.** Each starts with a capital letter and ends with a period. Use `` `backtick` `` quotes so identifiers, literals, and anything that must be fixed-font renders as fixed-font.

- **No `Co-Authored-By` trailers.** Commits are not co-attributed — not to agents, not otherwise.

- **PR-only with required green gates is *recommended*, not mandated.** This doc recommends merging through PRs with the §3 CI and pre-push gates required before merge, but does not insist — each repo may decide.

- **Keep PRs small and single-purpose.** The detailed policy is handled separately, but the principle stands: a PR should be reviewable in one sitting.

## 7. Dependencies & Builds

- **Lightweight, Linux-way dependencies are the way to go.** Reach for a well-established, small, single-purpose crate rather than reinventing it — the bar is quality and maintenance, not dependency count. Don't be shy, but stay lightweight.

- **Commit and pin `Cargo.lock`, and update it regularly.** Every build — local, CI, agent — resolves to the same versions. Refresh the lockfile on a regular cadence rather than letting it rot.

- **Monorepos are the way to go.** One repo, many small crates. Sub-projects are ideally carved into separate crates over time — potentially published independently to crates.io — each doing one thing well.

- **Inside the monorepo, break freely; publishing is the act of opting into compatibility.** Unpublished crates follow §1 — no versioning ceremony, breaking changes allowed. Once a crate is published to crates.io, a breaking change MUST bump the minor version (not the patch version): patch bumps are always safe to take, a minor bump signals "read the diff."

- **No mandated task runner — keep it simple.** Rust tests run with `cargo test`, and always run release-mode tests too (`cargo test --release`). Beyond that, other kinds of tests are fine and expected: shell scripts, UI / Playwright, and the `.md`-based harness tests (§3). Don't force everything through one `make`/`just` entry point.

- **Don't over-engineer the toolchain.** No elaborate MSRV / edition / nightly policy — this is nowhere near that complex. Use stable Rust and move on.

## 8. Config & Secrets

- **Config is strongly-typed JSON.** It deserializes once, at startup, into a Rust struct — a malformed config fails loudly and immediately rather than `None`-ing its way through the program. **Make illegal states unrepresentable. Use single-key JSON encoding.** Everything in §1 applies to config.

- **Config sources layer in a fixed precedence: defaults → config file → env vars → CLI flags**, each overriding the last (flags win). The config file *path* is itself given by a flag or env var, with a sensible default location. Document this precedence in the CLI's own `help` so the resolution order is discoverable.

- **Config files are JSON** — easier, and consistent with the single serialization worldview of §1.

- **Secrets: the hard rule.** Secrets never live in the repo, never in committed config, and are never logged. They come from env vars or a secrets manager, and are held in a redacting wrapper type that hides the value on `Debug` / log output — so a secret can't accidentally be printed (pairs with §5: logs are read by agents).

- **`.env` is for local dev only, and git-ignored.** Both `.env` and `.env.*` are `.gitignore`'d. Use them for local development; never commit them, and never rely on them in production.

- **Upward discovery walks MUST have a ceiling.** If a config or credential file is discovered by walking up from the current directory, the walk stops at `$HOME` — it never climbs into `/tmp` or `/`, where a stray planted file could be picked up. An explicitly chosen out-of-home start dir may be honored; an unbounded walk is a security bug, not a convenience.

- **A stored credential is bound to the target it was issued for.** If the credential comes from a file on disk but the caller points the tool at a different target, refuse: "refusing to use the stored credential (bound to X) against Y." Never silently send a stored secret somewhere other than where it was saved for. Symmetrically, an explicitly supplied credential (flag or env) is never silently paired with a target found on disk — fall back to the built-in default instead, and warn once per invocation about which target will actually be used. Normalize targets before comparing (semantic equality, not string equality), so equivalent spellings don't produce false mismatches.

- **Secret files: `0600` before bytes, atomic always.** Restrict permissions BEFORE any secret bytes land in the file, and write it atomically (tempfile + rename). If the file sits inside a git repo and is not gitignored, warn at write time with the exact line to add to `.gitignore` — prevent the leak at write time, not in the postmortem. A malformed config file must not break a command that supplied its own credentials: resolve the file lazily, only when actually needed.

## 9. External Processes & Liveness

- **Every external tool can fail — treat every invocation as fallible.** Harnesses, LLMs, Docker containers, network services, build tools: anything outside the process boundary can hang, die, or lie. No external call is assumed to succeed; every one has an explicit error path *and* a liveness bound.

- **ALWAYS show signs of life: emit output at least every 10–15 seconds.** Anything we build that can run inside a script produces *some* output semi-frequently, so the caller can tell it is alive. In a TTY, show a proper progress bar — if it's downloading or building, track actual progress. Not a TTY: print a periodic status line so the external caller (script, harness, agent) sees it is not hanging. Per §2, all of this goes to stderr, never stdout.

- **ALWAYS enforce a hard no-output watchdog on everything we invoke.** Default 5 minutes; 10 for known-slow jobs, but 5 is the default. The watchdog measures *silence*, not total runtime: a process that keeps producing output keeps running, and a process that produces no output whatsoever within the window is killed — hard stop, no exceptions — and an error is journaled stating that no output was received from that process within N minutes. The two rules reinforce each other: because our tools emit output every 10–15 seconds, silence *is* a reliable death signal.

## 10. Agents & Destructive Operations

- **Humans govern credentials and destruction; agents get the rest.** AI agents *will* drive every tool we build — let that sharpen the security posture, not loosen it. Read paths are agent-friendly by default; the narrow set of human-only operations is explicit and deliberate.

- **Secrets never enter the agent transcript, a URL, or stdout.** Interactive login is a browser handoff: the CLI sends the human to the web to authenticate, receives a short-lived signed code over a localhost callback (never the credential itself), redeems it over a secure channel, and writes the credential straight to a locked-down file (§8). Print only a prefix of the credential, ever. Write down the threat table (secret-in-URL, code replay, CSRF, wrong page) and the mitigation for each.

- **Agents may OBSERVE limits but never LIFT them.** Reading quota and usage is free; raising a limit routes through a human in the web UI. A limit the agent can raise itself is not a limit.

- **Destructive operations come in two explicit tiers.** Scriptable mutations with blast radius get a `--dry-run` sibling and a separate `--confirm-destructive` acknowledgment flag — preview and consent are flags, not an interactive prompt that breaks scripting. Truly irreversible, human-only operations additionally require an interactive TTY and a typed confirmation; point automation at a deliberate, auditable alternative instead. Either way, the command stays fully documented in `help` — self-descriptiveness (§2) is not negotiable, and the gate is the protection, not obscurity.

## 11. Markdown

- **NEVER hard-wrap Markdown.** One line per paragraph or bullet — no fixed-column wrapping, no manual line breaks mid-sentence. Wrapping is the reader's (or renderer's) job, and hard wraps produce noisy diffs where a one-word edit reflows an entire paragraph. Let the editor soft-wrap; keep the source one logical line per thought. Markdown is a first-class artifact here — human-readable type renderings (§1) and executable harnesses (§3) are all Markdown — so its formatting is a codified rule, not a matter of taste.

- **Separate blocks with a blank line.** Every heading, paragraph, list, and code block is surrounded by blank lines, so structure survives every renderer and stays legible in raw form.

- **Name Markdown files `UPPER-DASH-CASE.md`.** The stem is upper-case, dash-separated — `ARCHITECTURE.md`, `ENG-PRINCIPLES.md`, not `architecture.md` — and the `.md` extension stays lowercase. These files are load-bearing artifacts (executable harnesses per §3, docs, design notes); the shouting name makes them stand out in a file listing and signals "read this." (Conventional community filenames like `README.md`, `LICENSE`, `CONTRIBUTING.md` already follow this and are kept as-is.)

## 12. Human Config & Skills

- **Everything only a human can do lives in `HUMAN-CONFIG.md`.** Our tools forward credentials and configuration that already exist on the host; per §10 they never perform a login themselves. The repo therefore documents its human-only surface in one dedicated `HUMAN-CONFIG.md`: one section per external tool, each stating what to install, how to log in, what artifact the login leaves behind, and *precisely* what the software probes for — file paths, env vars, keychain items — matched to the code, never to wishful memory. Close the doc with a quick-verification table: one shell one-liner per tool, exit `0` meaning "this tool will be seen as available." Optional tools are marked optional; tools the software does not yet integrate are documented honestly as groundwork, never implied to work.

- **Pair `HUMAN-CONFIG.md` with an executable human-perspective test.** A §3-style executable Markdown harness (e.g. `TEST-LOCAL-AGENTS-CREDENTIALS.md`) proves that what the human configured is *sufficient*: numbered steps run from a clean checkout, a **Predict** line per step stating the expected observable outcome, and a per-item **PASS / SKIPPED / FAIL** rubric plus an overall verdict. Configuration the human deliberately left out yields SKIPPED, never FAIL — the test measures sufficiency of what was set up, not completeness of everything that could be. The Markdown *is* the harness; helper scripts stay thin or absent.

- **The human-perspective test is itself a skill.** Wire the same Markdown as a skill in the local repository, so any coding agent — on any harness — can execute it end to end and report the rubric. "Have your agent run `TEST-<WHAT>.md`" is the entire test-execution story, identical for a human following along, an agent invoked by hand, and a CI gate.

- **Skills live once, under `.skills/`; harnesses reach them through symlink aliases.** Each skill is a self-contained directory — `.skills/<name>/SKILL.md` plus its scripts — complete on its own and deployable by copying that directory alone. The per-harness discovery locations (`.claude/skills`, `.codex/skills`, `.cursor/skills`, `.opencode/skills`, `.agents/skills`) are symlinks to `../.skills`, never copies: one source of truth, every harness sees the same skill, and supporting a new harness costs exactly one symlink. See `dkorolev/beautiful-skills` for the authoring pattern and `scsh installskills` for the wiring that sets the aliases up in a consumer repo.
