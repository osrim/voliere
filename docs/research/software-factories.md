# Research: software factories and the next Voliere design

This note surveys existing software factory and background agent systems, and
asks whether Voliere should stay a custom setup. It was written on November
13, 2026, from three research passes (one by Claude, two by Codex) and spot
checks of the claims the options depend on. Claims marked **Hypothesis**
weren't verified on a primary source and need a test. Claims marked
**Checked** were read again on the primary source for this note.

The survey starts from three problems with the current design:

1. Starting a project needs a workstation: edit the OpenTofu project list,
   apply, create the Coolify app, and turn on Actions.
2. Agents can't work on existing GitHub repositories, and a project that moves
   from Gitea to GitHub leaves the agent workflow behind.
3. Each project carries its own copy of the template's CI, so a change to CI
   or Coolify means touching every repository.

## Summary

- **No existing system replaces Voliere.** None combines Gitea, flat-rate
  subscriptions, your own hardware, and boundaries outside the sandbox. Most
  self-hostable orchestrators put tokens inside the workspace and have no
  egress control. Keep the boundary layer.
- **The custom part can shrink.** Three recent upstream features remove most
  of the manual work without new architecture: Gitea 1.27 scoped workflows,
  the Coolify 4.7 API for Gitea App sources, and T3 Code's scheduled and
  webhook tasks.
- **GitHub access is the one real design change.** GitHub can't scope a token
  to a branch or forbid merging, so the boundary has to come from rulesets, a
  fork, or a broker outside the `agents` VM. Each trades setup work against
  blast radius.
- **The hosted clouds are a ready answer for GitHub only.** Claude Code cloud
  sessions and Codex cloud keep GitHub credentials behind a vendor proxy and
  run on the subscriptions you already pay for. Neither can write to Gitea.
- **Small vendors don't last.** Bloop (Vibe Kanban), Terragon, Coder Tasks,
  Continue, and the open source Daytona all shut down or changed course in
  2026. A control plane from a small vendor is a risk in its own right.

## Existing systems

The table lists the systems closest to Voliere's requirements. The survey
covered about 40. The rest are GitHub-only hosted products or tools without
an orchestrator.

| System | Gitea or Forgejo | Own hardware | Flat-rate plans | Sandbox | Status |
| ------ | ---------------- | ------------ | --------------- | ------- | ------ |
| [T3 Code][t3-source-control] (current) | Yes, through `tea` or `fj` | Yes | Claude and Codex | None, relies on the host | 0.1.x, nine releases in three weeks |
| [Multica][multica] | Yes, per its README | Yes | Both, drives logged-in CLIs | None documented | Very active, source-available license |
| [Gas City][gascity] (successor to Gas Town) | Git only, PR steps need `gh` (**Hypothesis**) | Yes | Both | None by default | Active |
| [OpenHands Agent Canvas][openhands] | Forgejo provider merged in 2025, absent from current docs | Yes | Both, through ACP | Docker, VM sandbox on the roadmap | Very active, MIT |
| [Cyrus][cyrus] | No | Yes | Both | Worktree only | Very active |
| [OpenAI Symphony][symphony-spec] | No | Yes | Codex only | Codex sandbox | No commits since September 2026 |
| [Open-Inspect][open-inspect] | No | Control plane only | Both | Hosted sandbox provider | Active, no releases |
| [Claude Code cloud][claude-web] | No | Team and Enterprise beta only | Claude plan | Vendor VM, credential proxy | Generally available |
| [Codex cloud][codex-cloud] | No | No | ChatGPT plan | Vendor VM, secret proxy | Generally available |
| Copilot agent, Cursor, Devin, Jules | No | No | No | Vendor VM | Generally available |

Three published designs aren't software you can run, but they set the
patterns: [Stripe Minions][stripe-minions], [Ramp Inspect][ramp-inspect], and
the [StrongDM software factory][strongdm].

### Patterns they share

These recur across the systems above. The last column says where Voliere
stands.

| Pattern | Who does it | Voliere today |
| ------- | ----------- | ------------- |
| The issue tracker or chat is the control plane. A thin poller or webhook receiver starts work. | Symphony, Cyrus, Multica, Minions, Inspect, OpenHands | A spec typed into a T3 thread. |
| One isolated workspace per task. | Everyone. Worktree, process sandbox, container, or VM. | Worktrees inside one VM. |
| A credential proxy outside the sandbox. | Claude Code cloud, Codex cloud, Coder. Rare in open source. | Tokens sit on the `agents` VM, limited by Gitea permissions. |
| An egress allowlist. | The hosted clouds, Stripe, Coder. | Open internet, private ranges blocked. |
| The agent writes its own environment setup. | Devin, Codex cloud, Jules | A template repository. |
| A pull request is the only output, on one branch, with no merge right. | Copilot, Claude Code cloud, Ramp | True in manual merge mode, by instruction only. |
| Deterministic gates around the agent, with capped CI retries. | Stripe blueprints, the Gas Town Refinery | CI plus `review-bot`. |
| CI and review feedback go back to the session that owns the change. | Claude auto-fix, Cyrus, T3 Code `watch_pull_request` | The worker waits on CI. |
| Central configuration in place of CI files in each repository. | Copilot, Claude environments, the Refinery | A copy per repository. |
| OTLP as the telemetry export. | Claude Code, Codex, T3 Code, Gas Town, OpenHands | Not collected. |

### Subscription terms

Every option that adds autonomy leans on flat-rate plans, so the terms matter.

- **Anthropic** documents headless use on a plan through `claude setup-token`
  ([authentication][claude-auth]). Its limits "assume ordinary, individual
  usage" ([legal and compliance][claude-legal]), and it doesn't define that
  term. In 2026 it announced a move of Agent SDK and `claude -p` usage to
  separate metered credits, then paused it ([support article][claude-sdk-plan]).
  **Hypothesis:** the pause could end, which would put T3 Code's Claude
  workers on metered billing.
- **OpenAI** lists T3 Code as an integration in its
  [Sign in with ChatGPT][codex-signin] program for locally hosted apps. It
  recommends API keys for automation and documents plan login on a trusted
  private runner as an advanced path ([CI authentication][codex-ci-auth]).

OpenAI is the more permissive vendor for an unattended, single-operator
setup. Keep the role assignment in `site.yaml` easy to change.

## Findings by problem

### Starting a project

- **Gitea gives no safe way for agents to create repositories.** Org-level
  inherited branch protection exists only in
  [Gitea Enterprise][gitea-ent-protection]. In the open source edition,
  protection stays per repository and the creator stays admin.
- **Coolify can create the app through its API.** `POST
  /api/v1/applications/private-gitea-app` creates an application from a Gitea
  App source. The Gitea App integration needs Coolify 4.5.0, and the API
  endpoints need 4.7.0. An organization webhook covers every repository of
  the owner. **Checked** on the [Coolify Gitea page][coolify-gitea].
- **Coolify doesn't comment on Gitea pull requests.** The Gitea page mentions
  neither PR comments nor commit statuses. **Checked.** Its
  [preview documentation][coolify-previews] reserves status updates for
  GitHub App sources.
- **T3 Code adds a project from its UI or CLI**, with no file to edit.

### Working on GitHub

- **Tokens can't be scoped to a branch.** A fine-grained token or a GitHub App
  installation token selects repositories and permissions, nothing finer
  ([installation tokens][gh-install-token]).
- **Pushing and merging share one permission.** The merge API needs
  `Contents: write`, the same permission as a push
  ([merge API][gh-merge-api]). Only a ruleset or a proxy keeps an agent from
  merging.
- **Workflow files have their own permission.** A token without `Workflows:
  write` can't change `.github/workflows`. It can still change a test or a
  build script that an existing workflow runs, so pull request jobs from
  agents must run without secrets ([secure use][gh-secure-use]).
- **Rulesets enforce branch limits on GitHub's side.** They can restrict
  creation and updates on every branch outside an agent namespace, with a
  bypass list ([rulesets][gh-rulesets]). They're available on public
  repositories on the Free plan, and on private repositories from GitHub Pro.
- **A short-lived token isn't a boundary.** Root on the `agents` VM uses the
  token until it expires, and gets another if the minting service trusts the
  VM.
- **A fork is the simplest strong boundary for public repositories.** Fork
  pull requests get no secrets and a read-only token. `pull_request_target`
  workflows break that boundary ([GitHub's guidance][gh-prt]). Private forks
  inherit upstream permissions and aren't a boundary.
- **The Claude Code cloud proxy is the reference design.** It keeps the GitHub
  credential outside the VM and rejects pushes to other repositories
  ([sandboxing][claude-sandboxing]). No self-hostable release of that proxy
  was found, so building one is custom work.
- **Gitea push mirrors force-push** and overwrite the destination
  ([mirrors][gitea-mirror]). Two-way sync between Gitea and GitHub isn't
  workable. Moving a project carries history, and not issues or pull requests
  ([GitHub Importer][gh-importer]).
- **Coolify's GitHub App source posts preview links on pull requests**, which
  the Gitea source doesn't ([GitHub App setup][coolify-gh-app]).
- **T3 Code handles several forges at once.** Its
  [source control page][t3-source-control] lists GitHub, GitLab, Forgejo, and
  Gitea, with per-server logins. Auto-merge and stacks are documented for
  GitHub and not for Gitea. **Checked.** **Hypothesis:** one T3 server works
  with a GitHub project and a Gitea project in the same session. The page
  implies it and doesn't state it.

### Maintaining 50 projects

- **Gitea 1.27 scoped workflows remove the CI copies.** One source repository
  holds workflows in `.gitea/scoped_workflows`. You register it for the org,
  and its workflows run in every repository of the org, with that
  repository's runners, secrets, and token. A required scoped workflow can't
  be turned off by the repository and gates the merge. `schedule` and
  `workflow_run` triggers aren't supported. **Checked** on the
  [scoped workflows page][gitea-scoped].
- **Reusable workflows also work in 1.27**, as `uses:
  org/repo/.gitea/workflows/x.yml@ref` on the same instance
  ([1.27 release notes][gitea-127]). They still need a caller file in each
  repository.
- **Runners and secrets already exist at org level**
  ([runners][gitea-runners], [secrets][gitea-secrets]).
- **Templates don't turn on Actions by default.** `DEFAULT_REPO_UNITS`
  includes Actions and `DEFAULT_TEMPLATE_REPO_UNITS` doesn't
  ([configuration][gitea-config]). **Hypothesis:** this is why each new
  repository needs Actions turned on by hand, and one setting fixes it.

### Autonomy

- **T3 Code tasks run on a schedule or on a webhook.** Webhook tasks support
  signature checks, filters, deduplication, and coalescing. A public URL
  needs a T3 Connect managed tunnel. The docs warn that the webhook body is
  untrusted text the agent may still follow. **Checked** on the
  [project settings page][t3-project-settings].
- **Symphony's specification is the reference for a queue.** It polls a
  tracker, dispatches eligible issues with bounded concurrency, and retries
  with backoff ([specification][symphony-spec]). It has no Gitea adapter and
  runs Codex only.
- **Gitea has no merge queue.** It has scheduled auto-merge and an "update
  branch" API ([merge API][gitea-merge-api]). **Hypothesis:** serial merges
  need either a rule in `AGENTS.md` or a small controller.

### Observability

- **Claude Code and Codex both export OpenTelemetry.** Claude Code exports
  sessions, tokens, estimated cost, commits, pull requests, and tool
  decisions ([monitoring][claude-monitoring]). Codex exports requests, tool
  decisions, and token counts ([configuration][codex-otel]).
- **T3 Code exports OTLP and has a usage page** with an "estimated
  API-equivalent cost" ([observability][t3-observability]).
- **Telemetry from the `agents` VM can be forged or dropped** by anyone with
  root there. The signals an attacker can't forge come from outside the VM:
  Gitea events, CI results, and Coolify deploy records.
- **Single-binary backends exist.** [OpenObserve][openobserve] and
  [SigNoz][signoz] both publish a Claude Code guide. A community
  [Grafana dashboard][grafana-dash] covers Claude Code's metrics.

## Where the current design is weaker than it reads

- **`review-bot` isn't an independent control.** Both bot tokens sit on the
  `agents` VM, so root there can approve its own change. The design already
  accepts this inside the agents org. It matters as soon as the same reviewer
  gates a GitHub repository.
- **Manual merge mode rests on instructions.** Branch protection lets
  `agent-bot` merge in both modes.

## Options

Each group is one decision. Options run from most locked down to most
convenient.

### A. Build or adopt

| Option | You gain | You give up |
| ------ | -------- | ----------- |
| A1. Keep Voliere and shrink the custom part (options B to F) | The boundaries, Gitea, flat-rate plans | You maintain the glue |
| A2. Keep Voliere for Gitea, use the hosted clouds for GitHub | No new GitHub credential on your host, no new component | Two orchestrators, no T3 lead on GitHub work, vendor lock on that half |
| A3. Replace T3 Code with Multica or OpenHands inside the `agents` VM | A board UI and built-in issue triggers | A second migration, an unverified Gitea path, no gain in isolation |
| A4. Go hosted and GitHub only, retire the host | No infrastructure | Gitea, previews on your hardware, and control of the boundary |

### B. How agents reach GitHub

For every option, root on the `agents` VM can read the code of each
repository the agents work on and can open a malicious pull request. The
options differ in what else it can do.

| Option | What a root attacker can also do | Your work | New components |
| ------ | -------------------------------- | --------- | -------------- |
| B1. Today: pull mirror in, you promote by hand | Nothing on GitHub | Every promotion | None |
| B2. Hosted clouds for GitHub repositories | Nothing from your host. The vendor proxy limits pushes to attached repositories, and not to a branch | Install the vendor's GitHub App once, add a ruleset per repository | None |
| B3. Bot fork, public repositories only | Rewrite the bot's fork | Create the fork, review and merge each pull request | A bot account or app |
| B4. Broker on a separate VM, one branch per task | Push to the task's branch until the grant expires | Approve a grant per task, review and merge | A custom git-aware proxy |
| B5. Token service on a separate VM, one-hour GitHub App tokens, plus rulesets | Use the token for an hour on the selected repositories. Rulesets hold it to the agent branches | Install the app and a ruleset per repository, review and merge | A small token service |
| B6. Fine-grained token on the `agents` VM, plus rulesets | The same as B5 until you revoke the token | A ruleset per repository, rotate the token | None |

B4 keeps the token itself away from the attacker. B5 and B6 let the attacker
hold a token, and GitHub's rulesets limit what it does. All of B3 to B6 need
three rules on each GitHub repository: you approve every merge, agent pull
request jobs get no secrets, and deploy secrets sit behind a protected
environment or outside GitHub Actions.

### C. Which forge holds what

| Option | You gain | You give up |
| ------ | -------- | ----------- |
| C1. Gitea only (today) | One workflow, fully private | GitHub projects stay out of reach |
| C2. Gitea for private incubation, GitHub for existing and public projects, with a scripted "graduate" step | Agents keep working after a project goes public | Two sets of rules, and issues and pull requests don't move |
| C3. GitHub only, Gitea retired | One forge, preview comments from Coolify, rulesets | Self-hosted privacy, and CI moves to GitHub runners or a self-hosted runner that must never serve a public repository |

### D. Starting a project in a pinch

| Option | You gain | You give up |
| ------ | -------- | ----------- |
| D1. A pool of spare projects. OpenTofu keeps a few empty repositories with their Coolify apps ready, and you top the pool up later | Instant start from any device, no new component, no new credential | Placeholder names until you rename them from a workstation |
| D2. GitOps. You merge a one-line pull request to a config repository from your phone, and a trusted runner applies it | Reviewable history, any project name | A runner outside the agents org that holds the OpenTofu credentials and state passphrase |
| D3. A provisioner service outside the `agents` VM that accepts a project name and a template name | Start from a phone, or from the lead agent | An always-on service that holds Gitea admin and Coolify tokens. If agents may call it, an attacker can use up its quota |

D1 is a suggestion from this note, not a pattern from the survey.

### E. Fleet maintenance

| Option | You gain | You give up |
| ------ | -------- | ----------- |
| E1. Required scoped workflows, plus Coolify apps created through the API | No CI files in projects, one edit changes the fleet | One bad edit breaks the fleet, no scheduled jobs |
| E2. Thin caller workflows pinned to a shared reusable workflow, updated by Renovate | Staged rollout of each CI change | A file and an update pull request in each repository |
| E3. The template as it is, updated with `multi-gitter` | No change in design | A pull request in each repository for each change |

### F. Autonomy and observability

| Option | You gain | You give up |
| ------ | -------- | ----------- |
| F1. A scheduled T3 task picks up issues labelled ready | Work starts from a Gitea issue on any device, no inbound route | Polling delay, plan usage while idle |
| F2. A Gitea webhook starts a T3 task | Immediate start | A new route from Gitea to the `agents` VM, or a T3 Connect tunnel |
| F3. A queue controller outside the `agents` VM, after Symphony | Leases, retries, concurrency and spend caps an attacker can't lift | A custom service |
| F4. OTLP from T3 Code and the agent CLIs to one backend | Cost, sessions, and stuck work in one place | A service and a route to it, and an attacker can forge the data |
| F5. A status board fed by Gitea and Coolify webhooks | Signals the agents can't forge | A custom collector |

With F1 to F3, an issue's text becomes agent input. On a public GitHub
repository, start work only from issues you wrote or labelled yourself.

## Recommendation

1. Take E1 first. It's the largest saving and changes no boundary.
2. Take D1 now, and D2 once the pool feels cramped.
3. Take C2 with B5 for GitHub. Use B2 in the meantime, since it needs no
   work on the host. B4 is worth building only if a one-hour token limited
   by rulesets isn't a small enough exposure for you.
4. Take F1 and F4, then add F5 if the forged-telemetry gap matters in
   practice.
5. Stay with A1. Look again at Multica and OpenHands if T3 Code's release
   pace becomes a burden.

## Open decisions

| Decision | Default guess | What the answer changes |
| -------- | ------------- | ----------------------- |
| Build or adopt (A) | A1 | A2 drops the GitHub work on the host. A3 and A4 replace the orchestrator or the host. |
| GitHub access (B) | B5 | B2 or B3 need no new component. B4 adds a custom proxy. B6 puts a long-lived token on the VM. |
| Forge layout (C) | C2 | C3 retires Gitea and moves CI. C1 leaves problem 2 open. |
| Private repositories on GitHub | Yes, with GitHub Pro | Without Pro, rulesets don't apply to private repositories, so B5 and B6 lose their branch limit there. |
| Project start (D) | D1 | D2 and D3 each add a trusted machine with admin credentials. |
| Fleet CI (E) | E1 | E2 keeps a file in each repository in exchange for staged rollouts. |
| May the lead agent start work without you (F) | Yes, from issues you label | A no keeps the T3 thread as the only entry point and drops F1 to F3. |
| Where telemetry lives | The `deploy` VM | Off the host adds a Tailscale ACL row. The `deploy` VM loses the data on a rebuild. |

## Tests before committing

- Register a scoped workflow source for the agents org and confirm a required
  workflow gates a merge on `pilot-notes`.
- Confirm that `agent-bot` can't post a commit status that satisfies a
  required check.
- Create a Coolify app through the 4.7 API from a Gitea App source, and
  confirm a pull request gets a preview.
- Add a GitHub project to the T3 server next to a Gitea project, and run one
  issue through each.
- On a scratch GitHub repository, confirm that a ruleset holds a GitHub App
  token to `agent/**` branches and that the token can't merge.

## Gaps

- Multica's forge authentication and isolation model weren't verified. Its
  docs site wasn't reachable.
- Whether OpenHands still ships the Forgejo provider wasn't verified.
- The two Codex research passes couldn't read local files, so they worked
  from a description of the design and not from `design.md`.
- Short quotes came through a summarizing fetch tool. Check exact wording on
  the source before quoting it elsewhere.

[t3-source-control]: https://github.com/pingdotgg/t3code/blob/main/docs/user/source-control.md
[t3-project-settings]: https://github.com/pingdotgg/t3code/blob/main/docs/user/project-settings.md
[t3-observability]: https://github.com/pingdotgg/t3code/blob/main/docs/operations/observability.md
[multica]: https://github.com/multica-ai/multica
[gascity]: https://github.com/gastownhall/gascity
[openhands]: https://github.com/OpenHands/OpenHands
[cyrus]: https://github.com/cyrusagents/cyrus
[symphony-spec]: https://github.com/openai/symphony/blob/main/SPEC.md
[open-inspect]: https://github.com/open-inspect/background-agents
[claude-web]: https://code.claude.com/docs/en/claude-code-on-the-web
[claude-auth]: https://code.claude.com/docs/en/authentication
[claude-legal]: https://code.claude.com/docs/en/legal-and-compliance
[claude-sdk-plan]: https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan
[claude-sandboxing]: https://www.anthropic.com/research/claude-code-sandboxing
[claude-monitoring]: https://code.claude.com/docs/en/monitoring-usage
[codex-cloud]: https://learn.chatgpt.com/docs/environments/cloud-environments
[codex-signin]: https://learn.chatgpt.com/docs/sign-in-with-chatgpt
[codex-ci-auth]: https://learn.chatgpt.com/docs/auth/ci-cd-auth
[codex-otel]: https://developers.openai.com/codex/config-advanced/
[stripe-minions]: https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents
[ramp-inspect]: https://builders.ramp.com/post/why-we-built-our-background-agent
[strongdm]: https://factory.strongdm.ai/
[gitea-scoped]: https://docs.gitea.com/1.27/usage/actions/scoped-workflows/
[gitea-127]: https://blog.gitea.com/release-of-1.27.0/
[gitea-runners]: https://docs.gitea.com/1.27/usage/actions/runner/
[gitea-secrets]: https://docs.gitea.com/1.27/usage/actions/secrets/
[gitea-config]: https://docs.gitea.com/1.27/administration/config-cheat-sheet/
[gitea-mirror]: https://docs.gitea.com/1.27/usage/repository/repo-mirror/
[gitea-merge-api]: https://docs.gitea.com/api/operations/repo-merge-pull-request/
[gitea-ent-protection]: https://docs.gitea.com/enterprise/features/inheritable-branch-protection/
[coolify-gitea]: https://coolify.io/docs/applications/sources/gitea
[coolify-previews]: https://coolify.io/docs/applications/deployments/preview-deployments
[coolify-gh-app]: https://coolify.io/docs/applications/sources/github/app
[gh-install-token]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app
[gh-merge-api]: https://docs.github.com/en/rest/pulls/pulls#merge-a-pull-request
[gh-secure-use]: https://docs.github.com/en/actions/reference/security/secure-use
[gh-rulesets]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets
[gh-prt]: https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target
[gh-importer]: https://docs.github.com/en/migrations/importing-source-code/using-github-importer/about-github-importer
[openobserve]: https://openobserve.ai/docs/integration/ai/claude-code-tracing/
[signoz]: https://signoz.io/docs/claude-code-monitoring/
[grafana-dash]: https://grafana.com/grafana/dashboards/25255-claude-code-metrics-prometheus/
