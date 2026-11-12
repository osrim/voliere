# Research: Atuin Threads in Voliere

This note evaluates Atuin Threads (agent session capture and sync) as an
addition to the `agents` VM. It was written on October 31, 2026, against Atuin
18.24.4, T3 Code at a nightly commit from the same day, and Claude Agent SDK
0.3.276 (the version T3 Code pins). Each claim cites its source. Claims marked
**Hypothesis** weren't verified and need a test.

## Summary

- Threads captures sessions by watching each agent's transcript files, not
  through hooks. Hooks only add Bash commands to shell history.
- T3 Code's Claude and Codex paths both write those transcript files to the
  default locations, so capture should work in Voliere. A test on an `agents`
  VM still has to confirm it.
- One key encrypts everything in an Atuin account, and it sits in a file next
  to a session token. Root on the `agents` VM can read, forge, and delete
  everything in that account.
- Sync covers every record type at once. You can't sync threads without also
  syncing history, so the sandbox needs its own account.
- Synced threads include tool input and output (redacted, clipped to 64 KiB
  per call). They don't include model reasoning.
- A self-hosted server (PostgreSQL, SQLite, or MySQL) stores thread records
  like any other record.

Recommendation: run the capture test first. If it passes, add Atuin with a
dedicated sandbox account and keep the workstation out of that account.

## What Threads is

The [announcement][blog] (October 27, 2026) says Threads ships in Atuin 18.24
and later. It captures Claude Code, Codex, Pi, Omp, and OpenCode sessions.
You can search them across machines, resume them, and continue them in
another agent, for example from Claude Code to Codex. Sync is end-to-end
encrypted and works with the hosted server, a self-hosted server, or no
server. Capture is opt-in. Team sharing is still in development.

Atuin 18.24.4 (October 30, 2026) adds GitHub Copilot CLI sessions
([changelog][changelog]). 18.24.0 shipped on October 15, 2026, so the feature
is about two weeks old, with four patch releases since.

The docs at `docs.atuin.sh/latest/` show a banner saying they describe a
version that isn't the current release, and `docs.atuin.sh/` redirects there.
This note cites the docs from the `v18.24.4` tag of the source repository
instead. The live threads page matched the tagged copy where checked.

Commands and settings ([threads doc][doc-threads], [MCP doc][doc-mcp]):

| Command or setting | Effect |
| ------------------ | ------ |
| `atuin config enable threads` (or `atuin threads enable`) | Sets `capture = true` under `[threads]` and turns on the daemon. |
| `atuin threads import` | Imports sessions that existed before capture was on. |
| `atuin resume` | Opens a picker to resume, fork, or continue a thread in another agent. |
| `atuin mcp` | Runs the MCP server over stdio. |
| `atuin mcp install [agent]` | Registers the MCP server with `claude-code`, `codex`, `opencode`, `pi`, or `omp`. |
| `atuin hook install <agent>` | Records each Bash command an agent runs as a history entry, tagged with the agent's name. |
| `[threads] capture_tools = false` | Keeps only tool names, not their input or output. |

The MCP server exposes `atuin_history`, `atuin_output`,
`atuin_output_search`, `atuin_threads_search`, `atuin_threads_list`, and
`atuin_threads_read`, and serves them over stdio
([`mcp.rs`][src-mcp]). The docs describe all tools as read-only.

## Findings on the open questions

### 1. How Threads captures sessions

**Settled.** The Atuin daemon watches each agent's transcript files. It
doesn't depend on hooks.

- At startup, the capture engine spawns one listener per supported agent
  ([`engine/mod.rs`][src-engine], `SessionCaptureEngine::spawn`).
- The Claude Code listener walks `$CLAUDE_CONFIG_DIR/projects`, or
  `~/.claude/projects`, with a file tree watcher and follows each `.jsonl`
  transcript ([`ccode/session.rs`][src-ccode], `default_root` and
  `CcodeListener::watch`).
- The Codex listener reads `$CODEX_HOME/sessions`, or `~/.codex/sessions`
  ([`codex/session.rs`][src-codex], `default_root`). OpenCode is read from
  its data directory under `$XDG_DATA_HOME` or `~/.local/share/opencode`, Pi
  from its agent directory, and Copilot from `session-state`.
- The daemon resolves these paths from its own environment. It sees only the
  directories of the user it runs as.
- Capture runs in the daemon. The daemon is off by default
  (`daemon.enabled = false`, `daemon.autostart = false`,
  [`settings.rs`][src-settings-defaults]), and thread capture is off by default
  (`capture: false`, [`settings/threads.rs`][src-threads-settings]).

Agent hooks are a separate feature. They record only Bash tool calls as
history entries, through `PreToolUse` and `PostToolUse` events
([agent hooks doc][doc-hooks]).

### 2. Whether T3 Code sessions get captured

**Settled from code, with conditions. Confirm with a test.**

T3 Code drives Claude Code through the Claude Agent SDK and Codex through
`codex app-server`. Both paths write transcripts where Atuin looks.

**Claude Code.**

- T3 builds each turn's options in `makeClaudeQueryOptions`
  ([`ClaudeAdapterV2.ts`][t3-claude-adapter]). It sets neither
  `persistSession` nor `settingSources`.
- In SDK 0.3.276, `persistSession` defaults to `true`, which saves sessions to
  `~/.claude/projects/`. When `settingSources` is omitted, the SDK loads
  user, project, and local settings, as the CLI does (`package/sdk.d.ts` in
  the [SDK tarball][sdk-tgz], lines 1738 to 1745 and 2140 to 2150).
- So T3-driven Claude sessions write transcripts that Atuin can watch, and
  hooks in `~/.claude/settings.json` fire.
- Only T3's capabilities probe turns persistence and hooks off
  (`persistSession: false`, `disableAllHooks: true`,
  [`ClaudeProvider.ts`][t3-claude-provider]). That probe isn't an agent
  session.

**Codex.**

- T3 spawns `codex app-server` and starts each thread with `thread/start`.
  It sends `cwd`, `model`, and `config`, but no `ephemeral` flag
  ([`CodexAdapterV2.ts`][t3-codex-adapter], `codexThreadRuntimeParams`).
- Codex documents `ephemeral` as "Whether the thread is ephemeral and should
  not be materialized on disk" ([`thread_data.rs`][codex-thread-data]), and
  passes the request's value through as a config override
  ([`thread_processor.rs`][codex-thread-processor], line 1331).
- T3 sets `CODEX_HOME` only when a provider instance sets a home path. In
  the "auth overlay" mode, T3 links `sessions` and `archived_sessions` from
  a shadow home back to the shared home
  ([`CodexHomeLayout.ts`][t3-codex-home], `KNOWN_SHARED_DIRECTORIES`). Either
  way, rollouts land in the shared `~/.codex/sessions`.
- Atuin reads Codex's paginated rollout format, which Codex writes since
  0.156 ([`codex/rehydrate.rs`][src-codex-rehydrate]).

**Conditions for capture to work:**

- The Atuin daemon runs as the same user as T3 Code, with the same `HOME`.
- No T3 provider instance sets a custom Claude home path,
  `CLAUDE_CONFIG_DIR`, or `CODEX_HOME` that differs from the daemon's
  environment. T3 supports these per instance
  ([`instanceEnvironment.ts`][t3-instance-env], [`ClaudeHome.ts`][t3-claude-home]).
- The daemon runs as a service. The `agents` VM has no interactive shell to
  start it, so it needs a systemd unit with `autostart = false`
  ([daemon doc][doc-daemon]).

**Hypothesis:** T3's rollback and fork features (`resumeSessionAt`, Codex
thread revert) rewrite transcripts in ways Atuin treats as forks or diverged
branches. Atuin has handling for both, but it's untested with T3.

**Hypothesis:** The Atuin MCP server loads inside T3 turns. T3 doesn't pass
`strictMcpConfig` for Claude turns, and for Codex it overrides only its own
`mcp_servers.t3-code` entry, so user-scope MCP servers should survive. Test
it.

### 3. How sync encryption keys are handled

**Settled.**

- The key file is `key` in Atuin's data directory
  ([`settings.rs`][src-settings], `key_path`). New installs use
  `~/.atuin/data/key` (`$ATUIN_HOME/data`). Older installs keep
  `~/.local/share/atuin/key` ([`dirs.rs`][src-dirs]).
- The sync session token sits next to it in `meta.db`
  ([`settings.rs`][src-settings], `meta.db_path`; [`meta.rs`][src-meta],
  `session_token`).
- The daemon loads one key ([`daemon.rs`][src-daemon], line 487) and uses it
  for both the history store and the threads store ([`lib.rs`][src-daemon-lib],
  lines 137 and 165). Every record type except packfile manifests is
  encrypted with the user's key ([`tag.rs`][src-tag]).
- Each record gets a random content key, wrapped with the master key
  (PASETO v4 envelope encryption, [`paseto_v4.rs`][src-paseto]). Anyone with
  the master key decrypts every record in the account.
- The server authenticates API calls with the session token alone
  ([`router.rs`][src-router], `UserAuth`). With that token, a client can
  upload records and call `DELETE /api/v0/store`, which deletes every record
  the account has on the server ([`store.rs`][src-store-handler],
  [`db/mod.rs`][src-db], `DELETE_STORE_BY_USER`). Deleting the account itself
  needs the password since 18.24.3 ([changelog][changelog]).

So root on the `agents` VM can read every thread and history entry in the
account, upload forged records, and wipe the server copy. The deletion path
is verified for the open source server only. **Hypothesis:** the hosted
server behaves the same.

### 4. Separate sync, and tool output

**Threads sync can't be separated from history sync. Settled.**

- The `[threads]` settings have no sync switch
  ([`settings/threads.rs`][src-threads-settings]).
- Sync diffs every (host, tag) series between client and server, with no
  filter by tag ([`record/sync/mod.rs`][src-sync], `diff` and `operations`).
- The only switches are global: `auto_sync`, or not logging in at all.

**Stored threads include tool output. Settled.**

- `capture_tools` defaults to `true`
  ([`settings/threads.rs`][src-threads-settings]). With it on, each tool
  call's input and output is kept, with secrets redacted and each payload
  clipped to 64 KiB ([`tools.rs`][src-tools], `ToolCapture`).
- Capture sanitizes each message, then pushes it, encrypted, into the record
  store that sync uploads ([`session_capture/mod.rs`][src-capture], lines
  406 and 451).
- Sanitizing keeps user and assistant text, tool calls, and edit diffs. It
  replaces model reasoning with a marker and drops other roles' text
  ([`session_capture/mod.rs`][src-capture], `sanitize`).
- Calls that read a sensitive file (`.env`, SSH keys, any path in
  `sensitive_files`), or run a command that prints a credential, keep no
  output ([threads doc][doc-threads]).

The threads doc is inconsistent here. Its "What's captured" section says a
thread resumed on another machine has its tool calls back as they were. Its
section on switching branches says a branch from sync carries "not the other
machine's tool input and output". The code above stores payloads in the
synced records. The note doesn't resolve why the switch path differs.

### 5. What a self-hosted server needs

**Settled.**

- `atuin-server` is a separate binary or container. It needs PostgreSQL,
  SQLite, or MySQL (MySQL is tier two) ([server setup doc][doc-server],
  [`Cargo.toml`][src-server-cargo]).
- Registration is closed by default (`open_registration = false`), and it
  listens on `127.0.0.1:8888` by default.
- The server stores records without knowing their contents. It accepts any
  tag, including `ai-session`, through the generic `/api/v0/record`
  endpoints ([`router.rs`][src-router], [`tag.rs`][src-tag]).
- The open source server doesn't advertise packfile support, and the client
  falls back to loose records for that case
  ([`record/sync/mod.rs`][src-sync], `sync_remote`). Thread records aren't
  packfiles, so they sync normally.
- The default `max_record_size` is 1 GiB, far above a thread record
  ([server `settings.rs`][src-server-settings]).
- The client's default `sync_address` is `https://api.atuin.sh`
  ([`settings.rs`][src-settings-address]).

## Where it helps

- **Sessions survive host rebuilds.** Today a rebuild loses every session's
  intent and dead ends. With threads synced off the host, they come back
  after `atuin login` on the new VM. This protects against accidents, not
  against an attacker: root on the VM can wipe the server copy (see
  [question 3](#3-how-sync-encryption-keys-are-handled)).
- **Agents can recall earlier work through MCP.** A worker can search why an
  earlier attempt was dropped. The reviewer can read the worker's prompts,
  replies, commands, and command output as well as the diff. The reviewer
  can't read the worker's reasoning, because capture keeps only a marker for
  it.
- **Hooks give a per-agent log of Bash commands.** Root can stop or forge
  it, so it's a debugging aid, not a forensic trail.

## Where it conflicts with the design

1. **The key and token sit on the `agents` VM.** A root attacker gets read,
   write, and delete access to the whole account. Sharing the account with
   your workstation would expose your shell history and your own agent
   sessions. It would also let the attacker plant forged history entries and
   threads in your workstation's search. A dedicated sandbox account is
   mandatory.
2. **Resume and continue open an injection channel out of the sandbox.**
   Resuming a sandbox thread on your workstation feeds content an attacker
   can control into an agent that holds your credentials. Treat it like
   promoting code.
3. **The sync server's location matters.**
   - Hosted needs only internet egress, which the Proxmox firewall already
     allows.
   - Self-hosted on the tailnet needs a new Tailscale ACL row from
     `tag:voliere`.
   - Self-hosted on the Voliere host is pointless, since a rebuild wipes it.
4. **Threads will contain secrets agents print.** The built-in patterns
   redact `Authorization: Bearer` values and values assigned to names that
   contain `token` or `secret` ([`structural.rs`][src-structural]).
   **Hypothesis:** they catch the Coolify API token in a request header and
   the `CF-Access-Client-Secret` header, but not a bare token printed on its
   own. Add both tokens' files to `sensitive_files`, add `redact_patterns`
   for their formats, and test.
5. **Atuin adds a credential to the VM.** The key and session token belong in
   the design's [credentials table](../design.md#credentials-on-the-host).
   Recovery must change too: after a compromise, treat every thread in the
   account as read and possibly forged. Deleting the account and registering
   a new one is the clean reset.
6. **Shell integration on the VM would sync your own commands.** If you
   install `atuin init` in a shell on the `agents` VM, your SSH commands sync
   into the sandbox account too. Install the daemon, hooks, and MCP only.
7. **It overlaps with T3 Code's own threads.** T3 keeps its threads in
   `~/.t3/userdata/statev2.sqlite`, also on the disposable VM
   ([`config.ts`][t3-config], [`os-jank.ts`][t3-os-jank]). T3 also scans the
   agents' transcript directories itself ([`AgentSessionScanner.ts`][t3-scanner]).
   Atuin adds search, off-host sync, and cross-agent continuation. It doesn't
   preserve T3's orchestration state.
8. **It's immature.** 18.24.0 is two weeks old, with four patch releases and
   many capture fixes since. Pin the version, as the design pins T3 Code.

## Recommendation

1. Run the capture test on an `agents` VM:
   1. Install Atuin 18.24.4 or later, and run the daemon as a systemd unit
      under the T3 Code user.
   2. Set `[threads] capture = true`, and install the hooks and MCP for
      Claude Code and Codex.
   3. Run one worker thread through T3 Code with each agent.
   4. Check that `atuin threads list` shows both sessions and that
      `atuin search --author '$all-agent' -- ''` shows their commands.
   5. Check that a T3 thread can call `atuin_threads_search`.
   6. Print a dummy Coolify token and a dummy `CF-Access-Client-Secret`
      header, and check what the thread stores.
2. If the test passes, add Atuin to the Ansible role for the `agents` VM:
   - A dedicated sandbox account, with the key and login delivered as
     secrets.
   - Threads, hooks, and MCP turned on for every agent CLI.
   - `sensitive_files` and `redact_patterns` for the sandbox's tokens.
   - No shell integration.
3. Keep the workstation out of that account by default.

## Open decisions

| Decision | Default guess | What the answer changes |
| -------- | ------------- | ----------------------- |
| Sync server | Hosted (`api.atuin.sh`). | Self-hosted off the host adds a Tailscale ACL row from `tag:voliere`, a server and database to run and back up, and control over store deletion. |
| Workstation access | None. | Read-only access through a separate Atuin config (its own `ATUIN_HOME` logged in to the sandbox account) gets documented as a boundary crossing in the design, like promotion. The workstation's own account stays separate either way. |
| When to add it to `docs/design.md` | After the T3 capture test. | Adding it now as optional means documenting a component that may not work under T3 Code. |

## Doc inconsistencies found

- `docs.atuin.sh/` redirects to `latest/`, which says it isn't the current
  release.
- The [sync guide][doc-sync] says Atuin syncs once an hour by default. The
  code defaults `sync_frequency` to 5 minutes
  ([`settings.rs`][src-settings-defaults]).
- The threads doc disagrees with itself on whether tool input and output
  sync (see [question 4](#4-separate-sync-and-tool-output)).

## Sources

Atuin source at tag `v18.24.4` (commit `b7ccdaf6b71f648d096de24b66cc8c7821d73351`),
T3 Code at commit `e2a6267d3063ddd25f38fdb59a467f44ed65ae37`, and Codex at
commit `67f0994b532b8740f14b5b738af7b7b2a915b1d6`.

[blog]: https://blog.atuin.sh/threads-agent-sessions
[changelog]: https://github.com/atuinsh/atuin/blob/v18.24.4/CHANGELOG.md
[doc-threads]: https://github.com/atuinsh/atuin/blob/v18.24.4/docs/docs/ai/threads.md
[doc-mcp]: https://github.com/atuinsh/atuin/blob/v18.24.4/docs/docs/ai/mcp.md
[doc-hooks]: https://github.com/atuinsh/atuin/blob/v18.24.4/docs/docs/guide/agent-hooks.md
[doc-daemon]: https://github.com/atuinsh/atuin/blob/v18.24.4/docs/docs/reference/daemon.md
[doc-server]: https://github.com/atuinsh/atuin/blob/v18.24.4/docs/docs/self-hosting/server-setup.md
[doc-sync]: https://github.com/atuinsh/atuin/blob/v18.24.4/docs/docs/guide/sync.md
[src-engine]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-daemon/src/session_capture/engine/mod.rs#L53
[src-capture]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-daemon/src/session_capture/mod.rs#L667-L708
[src-tools]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-daemon/src/session_capture/tools.rs#L44-L52
[src-ccode]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-common/src/harnesstools/ccode/session.rs#L43-L46
[src-codex]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-common/src/harnesstools/codex/session.rs#L44-L46
[src-codex-rehydrate]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-common/src/harnesstools/codex/rehydrate.rs#L33-L37
[src-threads-settings]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-client/src/settings/threads.rs#L98-L140
[src-settings]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-client/src/settings.rs#L1524-L1540
[src-settings-defaults]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-client/src/settings.rs#L1595-L1648
[src-settings-address]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-client/src/settings.rs#L58
[src-dirs]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-common/src/dirs.rs#L318-L372
[src-meta]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-client/src/meta.rs#L181
[src-daemon]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-daemon/src/daemon.rs#L487
[src-daemon-lib]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-daemon/src/lib.rs#L137-L165
[src-paseto]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-common/src/encryption/paseto_v4.rs#L754
[src-tag]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-domain/src/record/tag.rs#L16-L46
[src-sync]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-client/src/record/sync/mod.rs#L240-L252
[src-structural]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-common/src/secrets/structural.rs
[src-mcp]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-ai/src/mcp.rs#L287-L348
[src-router]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-server/src/router.rs#L27-L135
[src-store-handler]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-server/src/handlers/v0/store.rs
[src-db]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-server/src/db/mod.rs#L200
[src-server-settings]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-server/src/settings.rs#L103
[src-server-cargo]: https://github.com/atuinsh/atuin/blob/v18.24.4/crates/atuin-server/Cargo.toml#L55
[sdk-tgz]: https://registry.npmjs.org/@anthropic-ai/claude-agent-sdk/-/claude-agent-sdk-0.3.276.tgz
[t3-claude-adapter]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/apps/server/src/orchestration-v2/Adapters/ClaudeAdapterV2.ts#L1461-L1615
[t3-claude-provider]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/apps/server/src/provider/ClaudeProvider.ts#L198-L215
[t3-claude-home]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/apps/server/src/provider/Drivers/ClaudeHome.ts
[t3-codex-adapter]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/apps/server/src/orchestration-v2/Adapters/CodexAdapterV2.ts#L1323-L1352
[t3-codex-home]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/apps/server/src/provider/Drivers/CodexHomeLayout.ts#L18-L30
[t3-instance-env]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/packages/provider-core/src/server/instanceEnvironment.ts
[t3-config]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/apps/server/src/config.ts#L144-L148
[t3-os-jank]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/apps/server/src/os-jank.ts#L126
[t3-scanner]: https://github.com/pingdotgg/t3code/blob/e2a6267d3063ddd25f38fdb59a467f44ed65ae37/apps/server/src/project/AgentSessionScanner.ts#L1240-L1290
[codex-thread-data]: https://github.com/openai/codex/blob/67f0994b532b8740f14b5b738af7b7b2a915b1d6/codex-rs/app-server-protocol/src/protocol/v2/thread_data.rs#L225-L226
[codex-thread-processor]: https://github.com/openai/codex/blob/67f0994b532b8740f14b5b738af7b7b2a915b1d6/codex-rs/app-server/src/request_processors/thread_processor.rs#L1331
