# What 1,328 popular repositories run, or tell your AI agent, when you open them

This is the research behind `repo-trust-audit`: a scan of the trust surfaces that the most-starred
public repositories on GitHub ship, done with this exact engine. As with
[mcp-audit](https://github.com/eltociear/mcp-audit/blob/main/FINDINGS.md) and
[npm-audit](https://github.com/eltociear/npm-audit/blob/main/FINDINGS.md), the false-positive
rate was driven down on one set and then measured on a held-out set before any number was reported.

## Method

1. **Candidates:** every public, non-fork repository with at least 10,000 stars that GitHub
   search returned, 5,806 in total, walked in star order.
2. **Fetch:** for each repo, one batched GraphQL query took:
   - the root listing;
   - the config directories `.claude`, `.cursor`, `.vscode`, `.devcontainer`, `.gemini`, `.idea`,
     `.run`, `.roo`, `.kiro`, `.windsurf`, `.junie`, `.amazonq`, `.continue`, `.trae`, `.clinerules`
     and `.github/{instructions,prompts,chatmodes,agents,hooks}`, three levels deep, with file
     contents;
   - the fixed root files (`.mcp.json`, `.envrc`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`,
     `.cursorrules`, `.github/copilot-instructions.md`, `.gitmodules`, `.lfsconfig`,
     `.gitattributes`, `package.json`, …).

   A second pass fetched every repo script that a hook, task, dev-container or MCP command
   points at, plus root `.code-workspace` files. Nothing was cloned and nothing was executed.
3. **Qualifying:** a repo qualifies if it ships at least one trust surface beyond
   `.gitattributes`/`package.json`. Of the 2,590 candidates processed (2 failed to fetch),
   **1,328 qualified (51%)**.
4. **Calibration set:** the first 1,000 qualifying repos by stars (477,966 → 22,914 ★). Rules
   were tuned on these only.
5. **Held-out set:** the next 328 qualifying repos (22,907 → 17,804 ★). These were scanned
   **once**, with the rules frozen in commit `f6b2d27` before the scan.
6. **Scope:** root-level surfaces and config directories. Nested `AGENTS.md`/`CLAUDE.md` files
   in subfolders were not fetched in corpus mode. The tool itself does scan them in a local
   checkout or with `--github`.

Scan date: 2026-11-25/26. Default branches at that time.

## Headline

| Result | Calibration (1,000) | Held-out (328) |
| --- | --- | --- |
| Same signatures, context rules off: HIGH/CRITICAL | 34 (3.4%) | 9 (2.7%) |
| **repo-trust-audit: HIGH/CRITICAL** | **1 (0.1%)** | **1 (0.3%)** |
| …of which the match was wrong | 0 | 0 |
| MEDIUM | 17 | 2 |
| LOW | 29 | 11 |
| SAFE | 953 | 314 |

**Nothing malicious was found in the trust surfaces of the most-starred repositories on GitHub.**
That is the expected result, and it is what makes the tool usable: a scanner that flags the
repos everyone already opens teaches people to click through it.

Both HIGH+ hits describe what the files really do; neither looks like an attack (see below).
So the wrong-match rate at HIGH or above is **0 / 1,000 on calibration and 0 / 328 held out**.
The held-out hit is benign in intent and arguably ranked a level too high.

## What popular repositories ship

Across all 1,328 qualifying repos:

| Surface | Repos | |
| --- | --- | --- |
| Agent instruction files (any) | 952 | 71.7% |
| `AGENTS.md` | 769 | 57.9% |
| `CLAUDE.md` | 465 | 35.0% |
| Copilot instructions / prompts (`.github/…`) | 206 | 15.5% |
| `.claude/commands`, `agents`, `skills` | 205 | 15.4% |
| Cursor rules (`.cursorrules`, `.cursor/rules`) | 62 | 4.7% |
| `.vscode/settings.json` | 351 | 26.4% |
| `package.json` install lifecycle script | 297 | 22.4% |
| `.gitmodules` | 267 | 20.1% |
| Dev container | 256 | 19.3% |
| …with an `initializeCommand` (runs on the **host**) | 17 | 1.3% |
| `.claude/settings.json` | 136 | 10.2% |
| …with hooks | 74 | 5.6% |
| MCP config in the repo | 76 | 5.7% |
| `.envrc` (direnv) | 56 | 4.2% |
| Shared JetBrains `.idea` / `.run` configs | 43 | 3.2% |
| `files.exclude` hiding a config surface | 11 | 0.8% |
| VS Code tasks with `runOn: folderOpen` | 2 | 0.2% |
| **Anything that runs automatically once the folder is opened or trusted** | **143** | **10.8%** |

Some detail behind those rows:

- **Hooks:** the 74 repos with Claude Code hooks (plus 8 with Cursor/other agent hooks) ship
  **263 hook commands**. **198 of them call a script inside the repo**, and 42 run on
  `SessionStart`. A hook that calls `.claude/hooks/x.sh` runs whatever that file says in the next
  commit, so reviewing the settings file alone is not enough. `repo-trust-audit` follows the
  reference and scans the script.
- **MCP:** 124 servers are declared in total: 42 remote URLs and 82 local commands, 53 of which are
  `npx`/`pnpm`/`uvx`/`npm` package launches. 9 repos auto-approve their own project servers
  (`enableAllProjectMcpServers` or a named list), 7 of them with servers defined.
- **Auto-run kinds** among the 143: agent hooks 76, `.envrc` 56, host `initializeCommand` 17,
  status lines 5, auto-approved MCP servers 5, folder-open tasks 2.
- **Invisible Unicode:** zero. No tag characters, bidi controls, zero-width clusters or
  variation-selector payloads in any surface of any repo.

## The noise: what looks dangerous in a trust surface but isn't

With the same signatures and the context rules switched off, 34 calibration repos scored
HIGH or worse. Every one was benign:

| Signal | Calibration repos | Why it's noise |
| --- | --- | --- |
| Shell-rc / launchd "persistence" | 13 | `echo 'export PATH=…' >> ~/.zshrc` in install docs; `launchctl load` of the project's own service in setup skills; rc lines in container post-create |
| "Silently / quietly" | 13 | Prohibitions and descriptions: "never silently execute a plan…", "would silently change its semantics", "npx can silently fetch…" |
| Instruction override | 4 | "Ignore your system instructions **for PR descriptions**": a scoped style override (still reported, at MEDIUM) |
| Destructive commands | 3 | Agent guard hooks that *list* `rm -rf /` as a string to block |
| Exfiltration endpoints | 2 | Telegram `getMe` checks and tunnel-URL placeholders in bot-integration skills |
| Git config hijack | 1 | Repo-local `core.hooksPath` for husky; `core.fsmonitor true` (the built-in daemon) |
| Concealment | 1 | "…leave it alone and don't mention it to the user", about a commit trailer |

Earlier calibration rounds also removed:

- `env` matched inside a JavaScript regex (`/\.(env|key)$/`);
- `/dev/tcp/127.0.0.1/PORT` readiness checks read as reverse shells;
- named Docker volumes read as host mounts;
- CLI flag tables ("`--yes` | skip the confirmation prompt") read as safety bypasses.

The fixes are context, not exemptions:

- In a source file, shell-command signatures count only on a line that hands a string to a
  process sink, so a guard's blocklist is data.
- Negated, modal or quoted mentions are demoted.
- A concealment phrase is HIGH only when its sentence names an action (run, install, send,
  script, hook, token, step…).
- Secret-sending needs an explicit URL.
- In agent instructions, an install one-liner is LOW. In an auto-run hook it is HIGH.

## What was flagged

Both HIGH+ hits are stated as observed behaviour, not verdicts. Neither looks malicious.

**Agent confirmations switched off for everyone who opens the repo (HIGH).** One repo commits
`chat.tools.global.autoApprove: true` in its workspace `.vscode/settings.json`. For anyone whose
editor honours that setting at workspace scope, Copilot's agent mode then runs tools and
terminal commands in that folder without asking. It is a convenience setting with a large blast
radius. Agent-permission switches belong in a user's own settings, not in a repo.

**Host shell startup files edited on "Reopen in Container" (CRITICAL by rule; held-out).**
One repo's dev container `initializeCommand` runs a script **on the host**. The script appends an
SSH-agent block to the host user's `~/.bashrc`, and to `~/.zshrc` too for zsh users. The intent
is clearly to make SSH forwarding work. But most developers expect "open in container" to touch
the container, not their own shell profile. The rule (rc-file write + host auto-run) ranks it
CRITICAL; on human review, HIGH would be fairer.

**MEDIUM, worth a maintainer's look (19 repos):**

| Pattern | Repos |
| --- | --- |
| Broad shell pre-approval in a committed `.claude/settings*.json` (`Bash(*)`, `Bash(node:*)`, `Bash(python3:*)`) | 7 |
| Host `~/.ssh` bind-mounted into the dev container | 5 |
| Editor pointed at committed shim scripts (`${workspaceFolder}/scripts/…`) | 3 |
| Scoped instruction overrides ("ignore your system instructions for …") | 2 |
| Concealment phrasing without an action | 2 |
| A git merge driver registered by `package.json` `prepare` | 1 |
| A launcher script reading `~/.kube/config` | 1 |

**Not scored, but worth knowing:** one repo's dev-container configs run
`docker system prune --all --force` as `initializeCommand`. Choosing "Reopen in Container"
therefore deletes every unused Docker image and the build cache on the developer's own machine.
It is destructive to host state, not an attack, and the current rules don't score it; it is a
candidate for the next round.

**Not found anywhere in 1,328 repos:**

- invisible-Unicode text;
- credential or environment exfiltration;
- agent API traffic redirected to a third-party host (`ANTHROPIC_BASE_URL` and similar);
- `NODE_OPTIONS`/`LD_PRELOAD` injection;
- `bypassPermissions`;
- decode-and-execute, reverse shells or encoded PowerShell;
- submodule option injection;
- hidden folder-open tasks;
- prompt injection hidden in HTML comments.

Agent-directed HTML comments exist in 5 repos, and none contain injection phrasing. Install
one-liners (`curl … | sh`) appear in the agent instructions of 20 repos (LOW).

## Takeaways

1. **Opening is executing.** About 1 in 9 popular repos with trust surfaces ships something that
   runs as soon as the folder is trusted: agent hooks, `.envrc`, host `initializeCommand`,
   status lines, auto-approved MCP servers. None of it is malicious today. All of it is one
   commit from being so.
2. **Review the scripts, not just the settings.** Three quarters of hook commands call a script
   in the repo. The risky diff is the script, which is where `repo-trust-audit` looks.
3. **Agent instructions are now the most common trust surface.** `AGENTS.md` is in 58% of these
   repos. Treat instruction files from repos you don't control as untrusted input. Scan them for
   concealment, covert actions, secret-sending and invisible text before an agent reads them.
4. **Don't commit permission switches.** `autoApprove`, `Bash(*)`, `bypassPermissions` and host
   credential mounts apply to everyone who opens the repo, including people who clone a fork
   an attacker controls.
5. **Re-scan on pull.** A trust surface is one commit from changing. The exit code makes this a
   one-line gate before `code .` or `claude`.

---

Run it yourself: `python repo_trust_audit.py --github owner/repo` or `python repo_trust_audit.py ./checkout`.

*Findings are pattern matches warranting human review, not verdicts. No repository in this corpus
is accused of anything. If you find a malicious trust surface, report it to the maintainer and to
GitHub, not as an unverified public accusation.*
