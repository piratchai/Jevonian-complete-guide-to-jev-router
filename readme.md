# Jevonian + jev-gateway on Windows (CMD): the complete replication guide

This guide rebuilds, step by step, a working setup of two open-source projects on one Windows PC:

- **[Jevonian](https://github.com/xinyao27/jevonian) 0.1.7:** a local model router. Its brain, **Jev**, picks the model and the
  thinking effort for every turn.
- **[jev-gateway](https://github.com/vinilana/jev-gateway) 0.4.3:** a local gateway. It asks Jev **which tool** the agent
  should call next.

Four coding agents use them: **Qwen Code, Kilo, OpenCode and Claude Code**. Each tool is one plain CMD command with its own
router, its own web page, and its own status window. Every file, command and expected output below comes from
the real build of **2026-09-25** (in `D:\learn\JevAI`). Nothing is paraphrased: the files are embedded exactly as installed.

**Rule followed throughout:** use the official projects as they are, and change them as little as possible, so an official
upgrade can't silently break the setup. jev-gateway is **not modified at all**. Jevonian gets three small patches, applied
again automatically on every start. Everything else is configuration and a few Node scripts around them.
[What is official and what was changed](#4-what-is-official-and-what-was-changed) lists every difference.

## At a glance

| You type in CMD | Goes through | Port | The model comes from | Web page |
|---|---|---|---|---|
| `qwen` | Jevonian | 8793 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8793/logs |
| `kilo` | Jevonian | 8795 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8795/logs |
| `opencode` | Jevonian | 8799 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8799/logs |
| `claude` | Jevonian (its official `jevonian launch claude`) | 8797 | Anthropic, through your Claude subscription login | http://127.0.0.1:8797/logs |
| `jev-claude` | jev-gateway → Jevonian | 8789 → 8797 | Anthropic, through your Claude subscription login | http://127.0.0.1:8789/dashboard?peers=none |
| `jev-opencode` | jev-gateway → Jevonian | 8791 → 8799 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8791/dashboard?peers=none |
| `jev …` | manages all of the above: `start`, `stop`, `status`, `test`, `windows`, `dashboards`, `logs` | | | |

Verified on 2026-09-25 (see [Proof of working](#15-proof-of-working)):
- Both Jev brain channels work on all four routers: 8/8.
- All 23 tiers are served by the configured model at the configured effort: 23/23.
- All six commands answer, and all six read a file through their router: 12/12.
- For every tool, the client, its status window and its web page show the same requests.

## Contents

1. [What you will build](#1-what-you-will-build)
2. [Accounts, keys and providers](#2-accounts-keys-and-providers)
3. [Prerequisites](#3-prerequisites)
4. [What is official and what was changed](#4-what-is-official-and-what-was-changed)
5. [Build it, step by step](#5-build-it-step-by-step)
6. [Everyday use](#6-everyday-use)
7. [Status windows](#7-status-windows)
8. [Web pages: one per tool](#8-web-pages-one-per-tool)
9. [Test results: comparing the three views of every tool](#9-test-results-comparing-the-three-views-of-every-tool)
10. [Can Jevonian and jev-gateway share one folder?](#10-can-jevonian-and-jev-gateway-share-one-folder)
11. [Known behaviours and limits](#11-known-behaviours-and-limits)
12. [Troubleshooting](#12-troubleshooting)
13. [Upgrading and rolling back](#13-upgrading-and-rolling-back)
14. [Moving, stopping and uninstalling](#14-moving-stopping-and-uninstalling)
15. [Proof of working](#15-proof-of-working)
- [Appendix A: the command layer, every file in full](#appendix-a-the-command-layer-every-file-in-full)
- [Appendix B: exact diff of the patched Jevonian against the official 0.1.7](#appendix-b-exact-diff-of-the-patched-jevonian-against-the-official-017)

---

## 1. What you will build

Two sibling folders, each holding one official project, installed locally (no `npm -g`), plus small scripts around them:

```text
D:\learn\JevAI\
├─ jevonian\                       Jevonian: 4 routers, one per tool, and the CMD commands
│  ├─ jev.js                       `jev`: start | stop | restart | status | test | logs | windows | dashboards | install | uninstall
│  ├─ kilo.js  qwen.js  opencode.js  claude.js     the CMD commands for the four tools (through Jevonian)
│  ├─ lib\                         shared code: targets.js, common.js, launch.js, monitor.js (the status window)
│  ├─ credentials\                 qwen-alibaba-credential.txt, typesafe-ai-credential.txt, vercel-ai-gateway-credential.txt
│  ├─ jev-router-qwen\             package.json, start.js, patch-jevonian-waf.mjs, patch-jevonian-effort.mjs,
│  │                               config\config.json, .qwen\settings.json            (port 8793)
│  ├─ jev-router-kilo\             the same files + .kilo\kilo.json                    (port 8795)
│  ├─ jev-router-claude\           the same files + patch-jevonian-haiku.mjs           (port 8797)
│  ├─ jev-router-opencode\         the same files + opencode.json                      (port 8799)
│  └─ generated: run\ (status-window logs, test output), jev.doskey, autorun.backup.json,
│                jev-router-*\node_modules\, data\ (ledger, dashboards' data), logs\ (serve.log)
└─ jev-gateway\                    jev-gateway: the official package, unmodified
   ├─ package.json                 "jev-gateway": "0.4.3"
   ├─ gateway-env.js               the settings passed to the official launchers (documented variables only)
   ├─ jev-claude.js  jev-opencode.js     the CMD commands for the two gateway tools
   ├─ credentials\typesafe-ai-credential.txt
   └─ generated: node_modules\     (its logs and pid files go to %USERPROFILE%\.jev-gateway, the official place)
```

How a request travels:

```text
qwen          ─► Jevonian :8793 ─┐
kilo          ─► Jevonian :8795 ─┼─ Jev picks the tier (model + effort) ─► Alibaba Cloud Model Studio (Token Plan)
opencode      ─► Jevonian :8799 ─┘
claude        ─► Jevonian :8797 ─── Jev picks the tier (model + effort) ─► Anthropic, with your Claude Code login (OAuth)

jev-claude    ─► jev-gateway :8789 (Jev picks the TOOL) ─► Jevonian :8797 (Jev picks the tier) ─► Anthropic
jev-opencode  ─► jev-gateway :8791 (Jev picks the TOOL) ─► Jevonian :8799 (Jev picks the tier) ─► Alibaba
```

Each Jevonian router also binds its **port + 1** (8794, 8796, 8798, 8800). That's its tunnel surface, and it answers only with a
Jevonian API key, which this setup never creates. Everything listens on **127.0.0.1** only.

### The tiers

**Qwen Code, Kilo and OpenCode** (Alibaba Token Plan). The same six tiers in all three routers:

| Tier (model id) | Used for | Model | Effort sent |
|---|---|---|---|
| `jevonian/chat` | short replies, acknowledgements | deepseek-v4.1-flash | low |
| `jevonian/small` | a small, well-scoped change | qwen3.8-flash | low |
| `jevonian/execute` ("Medium task") | typical implementation / debugging | qwen3.8-flash | high |
| `jevonian/large` | big multi-file work, hard debugging | qwen3.8-max | xhigh |
| `jevonian/utility` | summaries, lookups, small mechanical edits | qwen3.7-plus | medium |
| `jevonian/plan` | architecture, planning, hard reasoning | glm-5.3 | high (glm-5.3 accepts only low, high and max) |

**Claude Code** (`claude` and `jev-claude`):

| Tier (model id) | Used for | Model | Effort sent | Price per million tokens in / out |
|---|---|---|---|---|
| `jevonian/plan` | architecture, planning, hard reasoning | Claude Opus 5.5 | xhigh | $4 / $20 |
| `jevonian/heavy` | large multi-file work | Claude Opus 5.5 | high | $4 / $20 |
| `jevonian/execute` | implementation, debugging, tool loops | Claude Sonnet 5 | high | $2 / $10 |
| `jevonian/utility` | summaries, lookups, small edits | Claude Sonnet 5 | medium | $2 / $10 |
| `jevonian/chat` | short replies | Claude Haiku 4.5 | low (a 2,048-token thinking budget: Haiku has no effort setting) | $1 / $5 |

`jevonian/auto` (the default in every command) lets Jev pick the tier for each turn. `jevonian/<tier>` pins one.

Prices are Anthropic's list prices (platform.claude.com/docs/en/about-claude/pricing, read 2026-09-25). With a Claude
subscription you pay the plan, not per token, and the dashboard's "cost" column is Jevonian's *estimate* at list prices.

### Jevonian or jev-gateway: which one?

| | **Jevonian** | **jev-gateway** |
|---|---|---|
| Jev decides | **which model and effort** serve this turn | **which tool** the agent should call next |
| Saves by | sending easy turns to cheaper models | steering the tool choice; the model is unchanged |
| Tools here | Qwen Code, Kilo, OpenCode, Claude Code | Claude Code and OpenCode only |
| Commands | `qwen`, `kilo`, `opencode`, `claude` | `jev-claude`, `jev-opencode` (they **also** go through Jevonian) |

**Use the plain commands (Jevonian) by default.** Tier routing is where the savings are.

Use `jev-claude` / `jev-opencode` to *add* Jev's tool routing on top. jev-gateway's own README says to expect better tool
picks on large tool lists, not lower cost, and to measure with `--routing off` / `on` (baseline mode on its dashboard).

---

## 2. Accounts, keys and providers

You need **three API keys and one subscription login**. Nothing else: no Anthropic API key, no OpenRouter key, no Kilo,
OpenCode or Qwen account, and no Jevonian API key.

| # | Account / provider | What it's used for here | Used by | Where the key goes | File format |
|---|---|---|---|---|---|
| 1 | **Alibaba Cloud Model Studio, Token Plan** (subscription; endpoint `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`) | The models for Qwen Code, Kilo and OpenCode: deepseek-v4.1-flash, qwen3.8-flash, qwen3.7-plus, qwen3.8-max, glm-5.3 | the qwen, kilo and opencode routers | `jevonian\credentials\qwen-alibaba-credential.txt` | a line `API Key: sk-…` (other lines are ignored) |
| 2 | **TypeSafe** (typesafe.ai): the Jev decision model `jev-latest` | Jevonian's brain (picks the tier) **and** jev-gateway's tool picks | all 4 routers + both gateways | `jevonian\credentials\typesafe-ai-credential.txt` **and** `jev-gateway\credentials\typesafe-ai-credential.txt` (the same key) | the key alone |
| 3 | **Vercel AI Gateway** (`typesafe-ai/jev`, the same Jev model resold) | Jevonian's **fallback** brain, used only if TypeSafe fails | all 4 routers | `jevonian\credentials\vercel-ai-gateway-credential.txt` | the key alone |
| 4 | **Claude Pro or Max subscription**, logged in to Claude Code | The models for Claude Code: Opus 5.5, Sonnet 5, Haiku 4.5 | the claude router | nothing to copy: the router reads Claude Code's own login (`%USERPROFILE%\.claude\.credentials.json`) and refreshes it when needed | — |

- **TypeSafe:** get a key at https://typesafe.ai.
- **Vercel AI Gateway:** https://vercel.com/dashboard/ai-gateway/api-keys.
- **Alibaba Token Plan:** the Model Studio console of your Token Plan subscription (the international / Singapore endpoint above).
- **Claude:** run `claude` once and use `/login` with your claude.ai account.

**Placeholder keys you will see in the files are not secrets:**
- `local-no-key`: Qwen Code, Kilo and OpenCode must send *some* key. Jevonian accepts any key on 127.0.0.1 while no Jevonian
  key exists, and this setup never creates one.
- `jevonian-local`: the loopback token Jevonian's own `jevonian launch claude` gives Claude Code.
- `OPENAI_API_KEY=local-no-key`: jev-opencode forwards the client's key to its upstream, which is Jevonian.

**How the keys are read (never written into a config file):**
- **Each router's `start.js`** reads the three files in `jevonian\credentials\` into three environment variables of its own process.
  `config.json` only names them:
  - `ALIBABA_TOKENPLAN_API_KEY` (`providers[].apiKeyEnv`)
  - `TYPESAFE_API_KEY` (`brains[0].apiKeyEnv`)
  - `AI_GATEWAY_API_KEY` (`brains[1].apiKeyEnv`)
- **`jev-gateway\gateway-env.js`** reads `jev-gateway\credentials\typesafe-ai-credential.txt` into `TYPESAFE_API_KEY` for the
  official launcher. An existing `TYPESAFE_API_KEY` in your environment wins.
- **No key is ever printed.** `jev test brains` proves they work by asking Jevonian's own "Test channel" endpoint.
- Keep both `credentials\` folders out of any git repository.

**In this build** the key files were copied from an older setup's folder by a script that never printed them:
- `qwen-alibaba-credential.txt` and `typesafe-ai-credential.txt` as they were.
- `vercel-ai-gateway-credential.txt` from the `AI_GATEWAY_API_KEY=` line of that folder's `.env`. The old
  `vercel-credential.txt` held a *different* value, which was not the key in use.

---

## 3. Prerequisites

- **Windows 10/11 and CMD.** Windows Terminal is fine; the status windows open in it.
- **No administrator rights needed.** `jev install` writes one per-user registry value (see step 7).
- **Node.js 22.15 or newer, with npm.** jev-gateway needs 22.15+, and Jevonian needs 22+.
- **The four clients on your `PATH`, and Claude Code logged in** with your Claude subscription.

Check them **before** step 7. Afterwards `claude`, `kilo`, `opencode` and `qwen` become the routed commands, and the real
ones are `claude-direct`, `kilo-direct`, `opencode-direct` and `qwen-direct`.

```bat
node --version
npm --version
claude --version
opencode --version
kilo --version
qwen --version
```

Versions used for this guide (2026-09-25), and how they were installed on this PC. Any install method works, as long as
the command is on `PATH`:

| Tool | Version | Installed at |
|---|---|---|
| Node.js / npm | v22.23.2 / 10.9.8 | `C:\nvm4w\nodejs` (nvm for Windows) |
| Claude Code | 2.1.282 | `%USERPROFILE%\.local\bin\claude.exe` (Claude Code's native installer) |
| OpenCode | 2.0.15 | `%USERPROFILE%\.bun\bin\opencode.exe` (bun global) |
| Kilo (Kilo Code CLI) | 7.7.9 | `%USERPROFILE%\.bun\bin\kilo.exe` (bun global) |
| Qwen Code | 0.24.4 | `C:\nvm4w\nodejs\qwen.cmd` (npm global) |
| jevonian | 0.1.7 (latest on npm and on GitHub `main`) | installed locally in step 3 |
| jev-gateway | 0.4.3 (latest on npm) | installed locally in step 4 |

Claude Code must be logged in with a **Claude Pro or Max** account. Run `claude`, type `/login`, and follow it. The Claude
router uses that login and has no API key of its own.

**Free ports:** 8789, 8791 (jev-gateway) and 8793 to 8800 (Jevonian: each router uses its port and port+1). Check with
`netstat -ano | findstr ":879 :880"`. Nothing should be listening yet.

---

## 4. What is official and what was changed

This section is the full list. If a behaviour is not listed here, it is the official default of that project.

### 4.1 Jevonian: official defaults vs. this setup

| Topic | Official Jevonian 0.1.7 (its README and docs) | This setup | Why |
|---|---|---|---|
| Install | `npm install --global jevonian` | `npm install` inside each of the four `jev-router-*` folders, `"jevonian": "0.1.7"` exact | No global installs. Each router is pinned and upgraded on its own. |
| Number of routers | one, for everything | **four**, one per tool (qwen, kilo, claude, opencode) | One web page, ledger and status window per tool. The Claude router serves different models. |
| Start | `jevonian` (on Windows: `serve` in the foreground) | `node jev-router-<tool>\start.js` → official `cli.mjs serve --foreground`, started in the background by `jev start` or by a CMD command | Keys, paths and patches are handled first. |
| Port | 8787 (+8788 tunnel surface) | qwen **8793**, kilo **8795**, claude **8797**, opencode **8799** (each +1) | 8787 to 8791 are jev-gateway's default ports. |
| Config file | `~/.config/jevonian/config.json` | `jev-router-<tool>\config\config.json` (env `JEVONIAN_CONFIG`) | Self-contained; nothing in your home folder. |
| Provider / brain keys | pasted on the Providers page, stored in `~/.config/jevonian/credentials.json` | environment variables named in `config.json` (`apiKeyEnv`), set by `start.js` from `jevonian\credentials\*.txt`. `JEVONIAN_CREDENTIALS` points at `config\credentials.json`, which is never created. | Keys stay out of the config and out of the home folder. |
| Data: ledger, request bodies, pricing, quota | `~/.local/share/jevonian/` | `jev-router-<tool>\data\` (env `JEVONIAN_DATA_DIR`, `JEVONIAN_LEDGER`, `JEVONIAN_UPDATE_STATE`) | Per-tool Logs page and status window. |
| Browser on start | opens the dashboard | `JEVONIAN_NO_OPEN=1`. The CMD commands open the router's page once, when they had to start it. | `jev start` doesn't open four tabs at once. |
| Providers | added on the Providers page (presets) | written in `config.json`. **Alibaba routers:** `alibaba-tokenplan`, type `openai`, Token Plan base URL, `billing: subscription`. **Claude router:** `claude-subscription` and `claude-subscription-haiku`, type `anthropic`, `auth: oauth`, `oauthSource: claude-code`, `injectStreamUsage`, an `anthropic-beta` header list. | Official provider fields. The Haiku provider carries a shorter beta list Haiku accepts. |
| Routes (tiers) | built-ins `plan`, `execute`, `utility`, `chat` | **Alibaba:** plan, execute (label "Medium task"), utility, chat, **small**, **large**. **Claude:** plan, execute, utility, chat, **heavy**. Each has one model. The Claude chat route pins its model to `claude-subscription-haiku` with the official per-model `providers` map. | Your tier design. |
| Effort | the brain picks it (`brainPicksEffort: true`), or `defaultEffort`. A level the client sets is never overridden. | `brainPicksEffort: false`. Each route carries its own `"effort"` (**patch**), and every Claude route adds `"forceEffort": true` (**patch**). | "small = flash at low, execute = flash at high" can't be expressed officially, and Claude Code always sends its own level. |
| `capacities` | from the models.dev catalog | stated for every model (context, max output, accepted efforts); glm-5.3 accepts only `low`/`high`/`max`, Haiku `low`/`medium`/`high` | Correct clamping. |
| Brains | none until you add one (`brains: []`) | 1) `typesafe` (`TYPESAFE_API_KEY`, 8 s timeout, `minConfidence` 0.6); 2) `vercel` (`AI_GATEWAY_API_KEY`, 20 s) | Failover only: the second is asked only if the first fails. |
| Jevonian API keys (`sk-jev-…`) | created on the Keys page; once one exists every `/v1` call needs it | **none created**. The clients send placeholders, and the routers listen on `127.0.0.1` only. | Local-only use. The port+1 tunnel surface stays locked. |
| Connecting Claude Code | "Connect Claude" rewrites `~/.claude/settings.json`, or `jevonian launch claude` | only the official **`jevonian launch claude`** (per session, writes nothing) | Your global Claude settings stay untouched. |
| Model auto-sync | on for OAuth providers | Claude router: `"modelSync": {"enabled": false}`. Alibaba routers: official default (API-key providers don't sync). | `config.json` stays exactly as written. |
| Tunnel | opt-in | off (the Claude config states `"tunnel": {"enabled": false}`; others use the default, off) | Nothing public. |
| Quota guard, session TTL | on; 720 min | on (Claude: `lowPercent` 10); 720 min | Same as official. |
| `baselineModel` (for "savings") | the priciest model | glm-5.3 (Alibaba routers); claude-sonnet-5 (Claude router) | A realistic comparison. |
| **Code** | as published | `node_modules\jevonian\dist\cli.mjs` is patched by three scripts on every start: +34/−6 lines (Alibaba routers), +65/−6 (Claude router). No other file changes. | See 4.2 and [Appendix B](#appendix-b-exact-diff-of-the-patched-jevonian-against-the-official-017). |

### 4.2 The three Jevonian patches

All three are small Node scripts next to `start.js`, run by it **on every start**. Each one:
- **Checks before writing:** it looks for its own marker comment. If the marker is there, it prints `already applied` and stops.
- **Refuses if the code is different:** if the code it expects is missing, or found more than once, it prints `pattern … found N times`,
  writes nothing, and `start.js` refuses to start the router.
- **Survives reinstalls:** `npm install` restores the official file, and the next start patches it again.

| Patch | Router(s) | What it changes | Why it is needed on 0.1.7 |
|---|---|---|---|
| `patch-jevonian-waf.mjs` (marker `waf-safe-patch v2`) | all four | Wraps the "brain state" sent to Jev in a function that defangs text Cloudflare's WAF blocks: `\|` → `│`, `/etc/` → `/ etc/`, `<script` → `‹script`, `../` → `.. /` | Coding sessions contain paths and shell pipes. Without it, the TypeSafe endpoint's WAF can reject the brain call, and routing fails. |
| `patch-jevonian-effort.mjs` (5 edits, markers `routing-effort-patch v1, v1b, v2, v3, v3b`) | all four | **v1:** keeps `"effort"` when a route is read from config. **v1b:** for Jev-routed turns (`jevonian/auto`), the chosen route's effort wins. **v2:** pinned turns (`jevonian/<tier>`) also send the route's effort; officially they send none. **v3:** keeps `"forceEffort": true`. **v3b:** when the route has `forceEffort`, the client's own level is ignored for that turn. | Official routes have no effort field. Without v3/v3b, Claude Code's own "high" wins on every tier. Proven by the same-folder test (section 10): an unpatched router sent the chat tier at "medium" instead of "low". |
| `patch-jevonian-haiku.mjs` (marker `haiku-safe-patch v1`) | claude only | Before a request goes to a model without adaptive thinking (Haiku 4.5): removes `output_config` and `context_management`, turns adaptive thinking into a thinking budget for the effort (low = 2,048 tokens), and drops mid-conversation `system` messages | Claude Code sends all of these, and Haiku 4.5 rejects each with HTTP 400 ("context_management: Extra inputs are not permitted"). |

The exact source of each script is in [step 3](#step-3-the-four-jevonian-routers). The exact resulting code change is
[Appendix B](#appendix-b-exact-diff-of-the-patched-jevonian-against-the-official-017).

### 4.3 jev-gateway: official defaults vs. this setup

**The jev-gateway code is not modified.** After `npm install` it was compared byte for byte with
`npm pack jev-gateway@0.4.3`: identical. Every difference is a documented setting or argument, passed by
`jev-gateway\gateway-env.js`, `jev-claude.js` and `jev-opencode.js`.

| Topic | Official jev-gateway 0.4.3 | This setup | Why |
|---|---|---|---|
| Install | `npm install -g jev-gateway` | `npm install` in `D:\learn\JevAI\jev-gateway`, `"jev-gateway": "0.4.3"` | No global installs. |
| Launchers | `jev-claude`, `jev-opencode` on `PATH` | the same official `node_modules\jev-gateway\bin\jev-claude.mjs` / `jev-opencode.mjs`, run by our `jev-claude.js` / `jev-opencode.js` (CMD macros) | They start the Jevonian router first and open a status window. |
| Jev key | asked on first run, saved to `~/.jev-gateway/.env` (`--setup`) | `TYPESAFE_API_KEY` from `jev-gateway\credentials\typesafe-ai-credential.txt` and `JEV_PROVIDER=typesafe`, as environment variables | Documented variables. No prompt, and the key stays in the folder. |
| Claude Code upstream | `https://api.anthropic.com/v1` | `JEV_CLAUDE_UPSTREAM_BASE_URL=http://127.0.0.1:8797/v1` (the Jevonian Claude router) | Tier and effort for jev-claude too. |
| Claude Code environment | only `ANTHROPIC_BASE_URL` | plus exactly what Jevonian's own `jevonian launch claude` sets: `ANTHROPIC_AUTH_TOKEN=jevonian-local`, `ANTHROPIC_API_KEY=""`, Opus/Sonnet → `jevonian/auto`, Haiku → `jevonian/utility`, subagents → `jevonian/auto`, display names, `CLAUDE_CODE_ATTRIBUTION_HEADER=0`, error reporting, feedback and survey off. Plus `--model jevonian/auto` unless you pass a model. | Jevonian needs its virtual model names. |
| OpenCode upstream / model | `https://api.openai.com/v1`, model `gpt-5` | `JEV_OPENCODE_UPSTREAM_BASE_URL=http://127.0.0.1:8799/v1`, `JEV_OPENCODE_MODEL=jevonian/auto` (OpenCode selects `jev-gateway/jevonian/auto`) | The Jevonian OpenCode router picks the model. |
| `OPENAI_API_KEY` | your OpenAI key | `local-no-key` | Forwarded to Jevonian, which needs none on loopback. |
| OpenCode version | tested with v1 (1.18.31); "v2 is out of scope" | OpenCode **2.0.15** plus `--standalone` and `PWD` (the same two fixes the plain `opencode` command uses) | Works in practice; see [known behaviours](#11-known-behaviours-and-limits). |
| Ports | claude 8789, opencode 8791 | the same | Official defaults. |
| Dashboard | `/dashboard`: one page with every gateway it finds | the same page. Our commands open `?peers=none` (an official option), so each tool has its own page. | One web page per tool. |
| Logs, pid files | `%USERPROFILE%\.jev-gateway\` | the same | Official. |

### 4.4 The clients: nothing global is changed

| Client | How it's pointed at its router | What is *not* touched |
|---|---|---|
| Qwen Code | command-line flags on every run: `--auth-type openai --openai-base-url http://127.0.0.1:8793/v1 --openai-api-key local-no-key -m jevonian/auto` | `~/.qwen/settings.json` |
| Kilo | `KILO_CONFIG_CONTENT` = `jev-router-kilo\.kilo\kilo.json` (a provider named `jevonian` → `http://127.0.0.1:8795/v1`), plus `-m jevonian/jevonian/auto` and `PWD` | your Kilo config and login |
| OpenCode | `OPENCODE_CONFIG_CONTENT` = `jev-router-opencode\opencode.json` (provider `jevonian` → `:8799/v1`), plus `--standalone`, `PWD`, `-m jevonian/jevonian/auto` | `~/.config/opencode` |
| Claude Code | the official `jevonian launch claude` (environment for that one session only) | `~/.claude/settings.json` |

The `*-direct` commands (`claude-direct`, `kilo-direct`, `opencode-direct`, `qwen-direct`) run the real clients with no router, exactly as before.

### 4.5 What was added around the two projects

None of this is part of Jevonian or jev-gateway. It's a thin layer, all Node, with no `.bat` files:

| File | What it does |
|---|---|
| `jevonian\jev-router-*\start.js` | Reads the keys, points Jevonian at the router folder, applies the patches, runs the official `serve`. |
| `jevonian\kilo.js`, `qwen.js`, `opencode.js`, `claude.js` | The CMD commands. They start the router if it's down, open the status window, and run the real client in *your* folder with the wiring from 4.4. |
| `jev-gateway\jev-claude.js`, `jev-opencode.js`, `gateway-env.js` | The gateway CMD commands: start the Jevonian router and the gateway, then run the official launcher with the settings from 4.3. |
| `jevonian\lib\*.js` | Shared code: the list of the six servers, start/stop, the launcher, and the status window. |
| `jevonian\jev.js` | `jev`: start, stop, restart, status, logs, windows, dashboards, `test`, and `install` / `uninstall` of the CMD commands. |
| `jevonian\jev.doskey` + one CMD AutoRun registry value | Make `kilo`, `qwen`, `opencode`, `claude`, `jev-claude`, `jev-opencode`, `jev` and the `*-direct` commands exist in every new CMD window (see [step 7](#step-7-install-the-cmd-commands)). |

---

## 5. Build it, step by step

Every command below is for **CMD**. `D:\learn\JevAI` is the folder used here. Another folder works too:
- The scripts find everything relative to themselves.
- The only absolute paths are the ones `jev install` writes into `jev.doskey`.
- The two folders must stay siblings (`…\jevonian` and `…\jev-gateway`), or set `JEV_GATEWAY_DIR` / `JEVONIAN_DIR`.

### Step 1: create the folders

```bat
mkdir D:\learn\JevAI\jevonian\credentials
mkdir D:\learn\JevAI\jevonian\lib
mkdir D:\learn\JevAI\jevonian\jev-router-qwen\config     D:\learn\JevAI\jevonian\jev-router-qwen\.qwen
mkdir D:\learn\JevAI\jevonian\jev-router-kilo\config     D:\learn\JevAI\jevonian\jev-router-kilo\.kilo
mkdir D:\learn\JevAI\jevonian\jev-router-claude\config
mkdir D:\learn\JevAI\jevonian\jev-router-opencode\config
mkdir D:\learn\JevAI\jev-gateway\credentials
```

### Step 2: the key files

Create them with Notepad, so the keys never land in your shell history:

```bat
notepad D:\learn\JevAI\jevonian\credentials\qwen-alibaba-credential.txt
notepad D:\learn\JevAI\jevonian\credentials\typesafe-ai-credential.txt
notepad D:\learn\JevAI\jevonian\credentials\vercel-ai-gateway-credential.txt
notepad D:\learn\JevAI\jev-gateway\credentials\typesafe-ai-credential.txt
```

| File | Content |
|---|---|
| `qwen-alibaba-credential.txt` | `API Key: sk-…` (your Token Plan key; other lines, such as console links, are ignored) |
| `typesafe-ai-credential.txt` (both copies) | the TypeSafe key alone |
| `vercel-ai-gateway-credential.txt` | the Vercel AI Gateway key alone |

### Step 3: the four Jevonian routers

Each router folder is self-contained: its own `package.json`, its own copy of the official package, config, patches and data.

**3a. `package.json` and the official package.** One `package.json` per router, with the router's name in `"name"`.
This is Kilo's:

```json
{
  "name": "jev-router-kilo",
  "version": "1.0.0",
  "description": "Self-contained Jevonian router + Kilo config",
  "dependencies": {
    "jevonian": "0.1.7"
  }
}
```

(The others: `"name": "jev-router-qwen"`, `"jev-router-claude"`, `"jev-router-opencode"`; `"jevonian": "0.1.7"` in all four,
exact, with no `^`.)

```bat
cd /d D:\learn\JevAI\jevonian\jev-router-qwen      && npm install --no-audit --no-fund
cd /d D:\learn\JevAI\jevonian\jev-router-kilo      && npm install --no-audit --no-fund
cd /d D:\learn\JevAI\jevonian\jev-router-claude    && npm install --no-audit --no-fund
cd /d D:\learn\JevAI\jevonian\jev-router-opencode  && npm install --no-audit --no-fund
```

Expected output, each time: `added 15 packages in 2s`. `node_modules\jevonian\package.json` should say `"version": "0.1.7"`.

**3b. `config\config.json`: providers, tiers, effort, brains.** The three Alibaba routers use the same file, except for
`"port"`: qwen **8793**, kilo **8795**, opencode **8799**. This is the Qwen router's:

<details><summary><code>jevonian\jev-router-qwen\config\config.json</code> (kilo: port 8795, opencode: port 8799, otherwise identical)</summary>

```json
{
  "listen": {
    "host": "127.0.0.1",
    "port": 8793
  },
  "defaultProvider": "alibaba-tokenplan",
  "providers": [
    {
      "name": "alibaba-tokenplan",
      "type": "openai",
      "baseUrl": "https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1",
      "apiKeyEnv": "ALIBABA_TOKENPLAN_API_KEY",
      "billing": "subscription",
      "models": [
        "deepseek-v4.1-flash",
        "qwen3.8-flash",
        "qwen3.7-plus",
        "glm-5.3",
        "qwen3.8-max"
      ]
    }
  ],
  "routing": {
    "mode": "auto",
    "routings": [
      {
        "id": "plan",
        "label": "Plan",
        "description": "architecture, design, multi-file planning, hard reasoning before code",
        "models": [
          "glm-5.3"
        ],
        "effort": "high"
      },
      {
        "id": "execute",
        "label": "Medium task",
        "description": "typical implementation or debugging across a few files, tool loops",
        "models": [
          "qwen3.8-flash"
        ],
        "effort": "high"
      },
      {
        "id": "utility",
        "label": "Utility",
        "description": "summaries, lookups, small mechanical edits",
        "models": [
          "qwen3.7-plus"
        ],
        "effort": "medium"
      },
      {
        "id": "chat",
        "label": "Chat",
        "description": "short conversational replies, acknowledgements",
        "models": [
          "deepseek-v4.1-flash"
        ],
        "effort": "low"
      },
      {
        "id": "small",
        "label": "Small task",
        "description": "a small, well-scoped change: one file or a few lines, a quick fix or single command",
        "models": [
          "qwen3.8-flash"
        ],
        "effort": "low"
      },
      {
        "id": "large",
        "label": "Large task",
        "description": "large or heavy work: big multi-file changes, hard debugging, maximum reasoning",
        "models": [
          "qwen3.8-max"
        ],
        "effort": "xhigh"
      }
    ],
    "sessionTtlMinutes": 720,
    "baselineModel": "glm-5.3",
    "brainPicksEffort": false,
    "capacities": {
      "qwen3.8-flash": {
        "contextWindow": 983616,
        "maxOutput": 65536,
        "efforts": [
          "low",
          "medium",
          "high",
          "xhigh"
        ]
      },
      "qwen3.7-plus": {
        "contextWindow": 983616,
        "maxOutput": 65536,
        "efforts": [
          "low",
          "medium",
          "high",
          "xhigh"
        ]
      },
      "glm-5.3": {
        "contextWindow": 200000,
        "maxOutput": 65536,
        "efforts": [
          "low",
          "high",
          "max"
        ]
      },
      "deepseek-v4.1-flash": {
        "contextWindow": 1000000,
        "maxOutput": 65536,
        "efforts": [
          "low",
          "medium",
          "high",
          "xhigh"
        ]
      },
      "qwen3.8-max": {
        "contextWindow": 983616,
        "maxOutput": 65536,
        "efforts": [
          "low",
          "medium",
          "high",
          "xhigh"
        ]
      }
    },
    "brains": [
      {
        "channel": "typesafe",
        "apiKeyEnv": "TYPESAFE_API_KEY",
        "minConfidence": 0.6,
        "timeoutMs": 8000
      },
      {
        "channel": "vercel",
        "apiKeyEnv": "AI_GATEWAY_API_KEY",
        "minConfidence": 0.6,
        "timeoutMs": 20000
      }
    ]
  }
}
```

</details>

<details><summary><code>jevonian\jev-router-claude\config\config.json</code></summary>

```json
{
  "listen": {
    "host": "127.0.0.1",
    "port": 8797
  },
  "defaultProvider": "claude-subscription",
  "providers": [
    {
      "name": "claude-subscription",
      "type": "anthropic",
      "baseUrl": "https://api.anthropic.com/v1",
      "auth": "oauth",
      "oauthSource": "claude-code",
      "billing": "subscription",
      "models": [
        "claude-opus-5-5",
        "claude-sonnet-5",
        "claude-fable-5-1",
        "claude-haiku-4-5-20251001"
      ],
      "injectStreamUsage": true,
      "headers": {
        "anthropic-beta": "claude-code-20250219,context-1m-2025-08-07,interleaved-thinking-2025-05-14,thinking-token-count-2026-05-13,context-management-2025-06-27,prompt-caching-scope-2026-01-05,mid-conversation-system-2026-04-07,mid-conversation-tool-changes-2026-07-01,advisor-tool-2026-03-01,effort-2025-11-24"
      }
    },
    {
      "name": "claude-subscription-haiku",
      "type": "anthropic",
      "baseUrl": "https://api.anthropic.com/v1",
      "auth": "oauth",
      "oauthSource": "claude-code",
      "billing": "subscription",
      "models": [
        "claude-haiku-4-5-20251001"
      ],
      "injectStreamUsage": true,
      "headers": {
        "anthropic-beta": "claude-code-20250219,interleaved-thinking-2025-05-14,thinking-token-count-2026-05-13"
      }
    }
  ],
  "tunnel": {
    "enabled": false,
    "provider": "cloudflare"
  },
  "routing": {
    "mode": "auto",
    "routings": [
      {
        "id": "plan",
        "label": "Plan",
        "description": "architecture, design, multi-file planning, hard reasoning",
        "models": [
          "claude-opus-5-5"
        ],
        "effort": "xhigh",
        "forceEffort": true
      },
      {
        "id": "execute",
        "label": "Execute",
        "description": "implementation, debugging, tool loops",
        "models": [
          "claude-sonnet-5"
        ],
        "effort": "high",
        "forceEffort": true
      },
      {
        "id": "utility",
        "label": "Utility",
        "description": "summaries, lookups, small mechanical edits",
        "models": [
          "claude-sonnet-5"
        ],
        "effort": "medium",
        "forceEffort": true
      },
      {
        "id": "chat",
        "label": "Chat",
        "description": "short conversational replies, acknowledgements",
        "models": [
          "claude-haiku-4-5-20251001"
        ],
        "providers": {
          "claude-haiku-4-5-20251001": [
            "claude-subscription-haiku"
          ]
        },
        "effort": "low",
        "forceEffort": true
      },
      {
        "id": "heavy",
        "label": "Heavy",
        "description": "large tasks, big multi-file work, maximum reasoning",
        "models": [
          "claude-opus-5-5"
        ],
        "effort": "high",
        "forceEffort": true
      }
    ],
    "sessionTtlMinutes": 720,
    "baselineModel": "claude-sonnet-5",
    "capacities": {
      "claude-opus-5-5": {
        "contextWindow": 1000000,
        "maxOutput": 128000,
        "efforts": [
          "low",
          "medium",
          "high",
          "xhigh",
          "max"
        ]
      },
      "claude-sonnet-5": {
        "contextWindow": 1000000,
        "maxOutput": 128000,
        "efforts": [
          "low",
          "medium",
          "high",
          "xhigh",
          "max"
        ]
      },
      "claude-haiku-4-5-20251001": {
        "contextWindow": 200000,
        "maxOutput": 65536,
        "efforts": [
          "low",
          "medium",
          "high"
        ]
      }
    },
    "quotaGuard": {
      "enabled": true,
      "lowPercent": 10
    },
    "brains": [
      {
        "channel": "typesafe",
        "apiKeyEnv": "TYPESAFE_API_KEY",
        "timeoutMs": 8000,
        "minConfidence": 0.6
      },
      {
        "channel": "vercel",
        "apiKeyEnv": "AI_GATEWAY_API_KEY",
        "timeoutMs": 20000,
        "minConfidence": 0.6
      }
    ],
    "brainPicksEffort": false
  },
  "modelSync": {
    "enabled": false,
    "intervalMinutes": 720
  }
}
```

</details>

What the fields mean (all official Jevonian fields except `effort` and `forceEffort`, which need the effort patch):

| Field | Meaning here |
|---|---|
| `listen` | `127.0.0.1` and the router's port; Jevonian also opens port+1. |
| `providers[]` | **Alibaba:** one OpenAI-compatible provider, key from the environment variable in `apiKeyEnv`, `billing: "subscription"` (a flat-rate plan), and the model ids it may serve. **Claude:** two OAuth providers reading Claude Code's login. `claude-subscription-haiku` sends only the beta headers Haiku accepts. |
| `routing.mode: "auto"` | `jevonian/auto` asks Jev; `jevonian/<id>` pins a route. |
| `routing.routings[]` | The tiers. `models` is a fallback chain (one model each here). `effort` is the level sent (patch). `forceEffort: true` overrides the client's own level (patch; Claude routes only). The Claude `chat` route's `providers` map keeps Haiku on the Haiku provider (it must be a **map**, not an array). |
| `brainPicksEffort: false` | Jev picks only the route; the effort comes from the route. |
| `capacities` | Context window, max output, and the effort levels each model accepts. A route's effort is clamped to these. |
| `brains[]` | TypeSafe first, Vercel only if TypeSafe fails. `minConfidence` 0.6: below it the pick is still used, but marked `jev-low-confidence`. |
| `sessionTtlMinutes`, `baselineModel`, `quotaGuard` | Session stickiness, the savings baseline, and skipping providers that ran out of quota. |
| `tunnel`, `modelSync` (Claude) | Tunnel off. Model auto-sync off, so the file is never rewritten. |

**3c. The patches.** Copy `patch-jevonian-waf.mjs` and `patch-jevonian-effort.mjs` into **all four** router folders, and
`patch-jevonian-haiku.mjs` into **`jev-router-claude` only**. You don't run them yourself: `start.js` does, on every start.

<details><summary><code>patch-jevonian-waf.mjs</code> (all four routers)</summary>

```js
// Idempotent patch for jevonian 0.1.7+ — defangs attack-looking text (paths, script tags,
// pipes, path traversal) in the brain-state payload so Cloudflare WAF doesn't reject it.
// Shell-command redaction is built into 0.1.7+ (formatToolCallForBrain); this covers the rest.
// Run: node patch-jevonian-waf.mjs
import { readFileSync, writeFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const file = join(dirname(fileURLToPath(import.meta.url)), "node_modules/jevonian/dist/cli.mjs");
let src = readFileSync(file, "utf8");
const MARK = "/* waf-safe-patch v2 */";
if (src.includes(MARK)) { console.log("WAF patch: already applied"); process.exit(0); }

const HOOK_FROM = "\tconst brainState = {";
const HOOK_TO = `\t${MARK}\n\tconst brainState = wafSafeState({`;
if (!src.includes(HOOK_FROM)) { console.error("WAF patch: brainState start not found"); process.exit(1); }
const endFrom = "\t\t...constraints\n\t};\n\tconst applyVerdict";
if (!src.includes(endFrom)) { console.error("WAF patch: brainState end not found"); process.exit(1); }
src = src.replace(HOOK_FROM, HOOK_TO);
src = src.replace(endFrom, "\t\t...constraints\n\t});\n\tconst applyVerdict");

src += `
${MARK}
function wafSafeText(s) {
	return s.replace(/\\|/g, "│").replace(/\\/etc\\//gi, "/ etc/").replace(/<(\\/?)script/gi, "‹$1script").replace(/\\.\\.\\//g, ".. /");
}
function wafSafeState(value) {
	if (typeof value === "string") return wafSafeText(value);
	if (Array.isArray(value)) return value.map(wafSafeState);
	if (value && typeof value === "object") {
		const out = {};
		for (const [k, v] of Object.entries(value)) out[k] = wafSafeState(v);
		return out;
	}
	return value;
}
`;
writeFileSync(file, src);
console.log("WAF patch: applied");
```

</details>

<details><summary><code>patch-jevonian-effort.mjs</code> (all four routers)</summary>

```js
// Idempotent patch for jevonian 0.1.7 — per-routing `effort`, plus opt-in `forceEffort`.
//
// Upstream, a routing entry only carries id/label/description/models/providers, and the thinking
// level sent is: brain pick -> x-jevonian-effort header -> routing.defaultEffort, clamped to what
// the model supports. That cannot express "small = qwen3.8-flash at low, medium = qwen3.8-flash
// at high": both would share one per-model setting. With this patch a routing may declare
//   { "id": "small", ..., "models": ["qwen3.8-flash"], "effort": "low" }
// and that level wins whenever that routing serves the turn — routed by Jev (jevonian/auto) or
// pinned (jevonian/<id>, which upstream sends with no level at all). It is still clamped to
// `capacities.<model>.efforts`.
//
// A level the CLIENT set itself (reasoning_effort / thinking / output_config.effort) is still never
// overridden (upstream rule) — unless the routing also says  "forceEffort": true . Claude Code
// always sends its own output_config.effort (even with no effortLevel configured), so per-tier
// effort for Claude Code needs forceEffort; OpenCode/Kilo/Qwen send none, so they don't.
//
// Each edit carries its own marker, so the script is safe to re-run and to extend.
// Run: node patch-jevonian-effort.mjs   (start.js runs it on every start; npm install undoes it)
import { readFileSync, writeFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const file = join(dirname(fileURLToPath(import.meta.url)), "node_modules", "jevonian", "dist", "cli.mjs");
let src = readFileSync(file, "utf8");

const edits = [
  {
    // 1) Keep `effort` when a routing entry is parsed from config.json.
    mark: "/* routing-effort-patch v1 */",
    from: "\t\tmodels,\n\t\t...providers ? { providers } : {}\n\t};\n}\n/**\n* Build the routings list",
    to: (mark) => "\t\tmodels,\n\t\t...providers ? { providers } : {},\n\t\t" + mark + "\n\t\t...typeof value.effort === \"string\" && isReasoningEffort(value.effort) ? { effort: value.effort } : {}\n\t};\n}\n/**\n* Build the routings list",
  },
  {
    // 2) Jev-routed turns (jevonian/auto): the chosen routing's effort wins.
    mark: "/* routing-effort-patch v1b */",
    from: "\tconst appliedEffort = clampEffort(wanted ?? requestedEffort ?? defaultEffort, ",
    to: (mark) => "\t" + mark + "\n\tconst routingEffort = config.routing.routings.find((entry) => entry.id === phase)?.effort;\n\tconst appliedEffort = clampEffort(routingEffort ?? wanted ?? requestedEffort ?? defaultEffort, ",
  },
  {
    // 3) Pinned turns (jevonian/<id> or an x-jevonian-phase header): upstream returns early here
    //    with no effort at all; send the routing's level (then the header, then defaultEffort).
    mark: "/* routing-effort-patch v2 */",
    from: "\t\t\tvirtual: true,\n\t\t\trouted: true,\n\t\t\treason,\n\t\t\tsession\n\t\t};\n\t}\n\tconst brains = config.routing.brains;",
    to: (mark) =>
      "\t\t\tvirtual: true,\n\t\t\trouted: true,\n\t\t\treason,\n\t\t\tsession,\n\t\t\t" + mark + "\n" +
      "\t\t\t...(() => {\n" +
      "\t\t\t\tconst pinned = clampEffort(config.routing.routings.find((entry) => entry.id === phase)?.effort ?? headerEffort(headers) ?? brainEffort(config.routing.defaultEffort), effectiveCapabilities(picked.model, config.routing.capacities?.[picked.model]).efforts);\n" +
      "\t\t\t\treturn pinned ? { effort: pinned } : {};\n" +
      "\t\t\t})()\n" +
      "\t\t};\n\t}\n\tconst brains = config.routing.brains;",
  },
  {
    // 4) Keep the opt-in `forceEffort: true` flag when a routing entry is parsed (needs edit 1).
    mark: "/* routing-effort-patch v3 */",
    from: "? { effort: value.effort } : {}\n\t};\n}\n/**\n* Build the routings list",
    to: (mark) => "? { effort: value.effort } : {},\n\t\t" + mark + "\n\t\t...value.forceEffort === true ? { forceEffort: true } : {}\n\t};\n}\n/**\n* Build the routings list",
  },
  {
    // 5) When the serving routing has forceEffort and an effort, ignore the client's own level for
    //    this turn, so withEffort() writes the routing's level (output_config.effort, or a thinking
    //    budget on legacy models such as Haiku 4.5).
    mark: "/* routing-effort-patch v3b */",
    from: "\t\t\tconst clientEffort = clientEffortOf(body, clientKind);",
    to: (mark) =>
      "\t\t\t" + mark + "\n" +
      "\t\t\tconst forcedRouting = config.routing.routings.find((entry) => entry.id === decision.phase);\n" +
      "\t\t\tconst clientEffort = forcedRouting?.forceEffort === true && forcedRouting.effort && decision.effort ? void 0 : clientEffortOf(body, clientKind);",
  },
];

let applied = 0;
for (const edit of edits) {
  if (src.includes(edit.mark)) continue;
  const count = src.split(edit.from).length - 1;
  if (count !== 1) {
    console.error(`Effort patch: pattern for ${edit.mark} found ${count} times (expected 1) — refusing, nothing written`);
    process.exit(1);
  }
  src = src.replace(edit.from, edit.to(edit.mark));
  applied++;
}
if (applied === 0) { console.log("Effort patch: already applied"); process.exit(0); }
writeFileSync(file, src);
console.log(`Effort patch: applied (${applied} edit${applied === 1 ? "" : "s"})`);
```

</details>

<details><summary><code>patch-jevonian-haiku.mjs</code> (<code>jev-router-claude</code> only)</summary>

```js
// Idempotent patch — makes legacy-thinking models (Haiku 4.5 and older, i.e. every model
// anthropicThinkingSupport() classifies as `adaptive: false`) able to serve Claude Code
// requests. Claude Code always sends `output_config.effort`, adaptive `thinking`,
// `context_management`, and mid-conversation `role:"system"` messages; Haiku 4.5 rejects
// each with HTTP 400. This sanitizes the outgoing body for exactly those models at
// withEffort(), which every outgoing Anthropic body passes through. Sonnet/Opus untouched.
// (Verbatim from the guide, section 7a. Verified still needed on jevonian 0.1.7, 2026-09-25:
//  without it the chat tier fails with 400 "context_management: Extra inputs are not permitted".)
//
// Run: node patch-jevonian-haiku.mjs   (re-run safe; restart the router afterwards)
import { readFileSync, writeFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const file = join(dirname(fileURLToPath(import.meta.url)), "node_modules", "jevonian", "dist", "cli.mjs");
let src = readFileSync(file, "utf8");
const MARK = "/* haiku-safe-patch v1 */";
if (src.includes(MARK)) { console.log("Haiku patch: already applied"); process.exit(0); }

// 1) Hook withEffort: sanitize the body for legacy-thinking targets before anything else.
const HOOK_FROM = "function withEffort(body, effort, wire, clientEffort) {";
const HOOK_TO = `${HOOK_FROM}
\t${MARK}
\tif (wire === "anthropic") body = haikuSafeBody(body);`;
if (!src.includes(HOOK_FROM)) { console.error("Haiku patch: withEffort not found"); process.exit(1); }
if (src.split(HOOK_FROM).length > 2) { console.error("Haiku patch: withEffort appears more than once — refusing"); process.exit(1); }
src = src.replace(HOOK_FROM, HOOK_TO);

// 2) Append the sanitizer (module scope, so it can call anthropicThinkingSupport/effortBudget).
src += `
${MARK}
function haikuSafeBody(body) {
\ttry {
\t\tif (!body || typeof body !== "object" || typeof body.model !== "string") return body;
\t\tconst support = anthropicThinkingSupport(body.model);
\t\tif (!support || support.adaptive !== false) return body;
\t\tconst next = { ...body };
\t\tconst level = (typeof next.output_config === "object" && next.output_config !== null
\t\t\t&& typeof next.output_config.effort === "string") ? next.output_config.effort : void 0;
\t\tdelete next.output_config;
\t\tdelete next.context_management;
\t\tconst thinking = typeof next.thinking === "object" && next.thinking !== null ? next.thinking : null;
\t\tif (thinking && thinking.type === "adaptive") {
\t\t\tif (level) {
\t\t\t\tnext.thinking = { type: "enabled", budget_tokens: effortBudget(level) };
\t\t\t} else {
\t\t\t\tdelete next.thinking;
\t\t\t}
\t\t} else if (thinking && thinking.type === "disabled") {
\t\t\tdelete next.thinking;
\t\t}
\t\tif (Array.isArray(next.messages)) {
\t\t\tnext.messages = next.messages.filter((m) => !(m && m.role === "system"));
\t\t}
\t\treturn next;
\t} catch {
\t\treturn body;
\t}
}
`;
writeFileSync(file, src);
console.log("Haiku patch: applied");
```

</details>

**3d. `start.js`:** the same file in all four router folders.

<details><summary><code>jev-router-*\start.js</code></summary>

```js
// start.js — starts THIS Jevonian router (run by `jev start <name>`, or by a CMD command when the router is down).
//   1. Reads the API keys from ..\credentials\ into the environment variables that config\config.json names
//      with "apiKeyEnv", so no key is ever written into config.json.
//   2. Points Jevonian at this folder (config, credentials, data) instead of its defaults in your home folder.
//   3. Re-applies this folder's patch-jevonian-*.mjs (idempotent): `npm install` restores the official
//      dist\cli.mjs, so a patch can never be silently missing.
//   4. Runs the official `jevonian serve --foreground`.
// The four jev-router-*\start.js files are identical; the router's port and tiers come from config\config.json.
const { existsSync, readFileSync } = require("fs");
const { basename, join, resolve } = require("path");
const { spawnSync, spawn } = require("child_process");

const ROUTER_DIR = __dirname;
const CREDENTIALS_DIR = resolve(ROUTER_DIR, "..", "credentials");

function readCredential(file) {
  try { return readFileSync(join(CREDENTIALS_DIR, file), "utf8"); } catch { return ""; }
}

// Alibaba Cloud Model Studio, Token Plan: the file holds a line "API Key: sk-...".
const alibabaKey = (readCredential("qwen-alibaba-credential.txt").match(/^API Key:\s*(.+)$/m) || [, ""])[1].replace(/\s+/g, "");
// TypeSafe (Jev, the routing brain) and Vercel AI Gateway (Jev again, the fallback channel): the key alone.
const typesafeKey = readCredential("typesafe-ai-credential.txt").replace(/\s+/g, "");
const aiGatewayKey = readCredential("vercel-ai-gateway-credential.txt").replace(/\s+/g, "");

for (const [name, value] of [
  ["ALIBABA_TOKENPLAN_API_KEY", alibabaKey], // providers[].apiKeyEnv (Qwen, Kilo and OpenCode routers)
  ["TYPESAFE_API_KEY", typesafeKey], //         routing.brains[0].apiKeyEnv
  ["AI_GATEWAY_API_KEY", aiGatewayKey], //      routing.brains[1].apiKeyEnv
]) {
  if (!value) console.error(`WARNING: ${name} is empty (check ${CREDENTIALS_DIR})`);
  process.env[name] = value;
}

// Official Jevonian environment variables (docs/configuration.md): everything lives in this folder.
process.env.JEVONIAN_CONFIG = join(ROUTER_DIR, "config", "config.json");
process.env.JEVONIAN_CREDENTIALS = join(ROUTER_DIR, "config", "credentials.json"); // only used if you add a key on the dashboard
process.env.JEVONIAN_DATA_DIR = join(ROUTER_DIR, "data");
process.env.JEVONIAN_LEDGER = join(ROUTER_DIR, "data", "ledger.jsonl");
process.env.JEVONIAN_UPDATE_STATE = join(ROUTER_DIR, "data", "update.json");
process.env.JEVONIAN_NO_OPEN = "1"; // the CMD commands open the dashboard themselves when they start a router

const pkg = JSON.parse(readFileSync(join(ROUTER_DIR, "node_modules", "jevonian", "package.json"), "utf8"));
const port = JSON.parse(readFileSync(process.env.JEVONIAN_CONFIG, "utf8")).listen.port;
console.log(`[start] ${basename(ROUTER_DIR)}: jevonian ${pkg.version}, port ${port}, node ${process.version}, ${new Date().toISOString()}`);

// Re-apply this folder's patches on every start. Each router carries only the patches it needs:
//   patch-jevonian-waf.mjs    keeps Cloudflare's WAF from rejecting brain calls        (all routers)
//   patch-jevonian-effort.mjs per-routing "effort" (+ opt-in "forceEffort") in config  (all routers)
//   patch-jevonian-haiku.mjs  lets Haiku 4.5 serve Claude Code requests                (Claude router only)
const PATCHES = ["patch-jevonian-waf.mjs", "patch-jevonian-effort.mjs", "patch-jevonian-haiku.mjs"].filter((p) => existsSync(join(ROUTER_DIR, p)));
for (const patch of PATCHES) {
  const result = spawnSync(process.execPath, [join(ROUTER_DIR, patch)], { stdio: "inherit", windowsHide: true });
  if (result.status !== 0) {
    console.error(`${patch} failed — refusing to start with an unpatched bundle.`);
    process.exit(1);
  }
}

// Start the router (official CLI). Not detached: stopping start.js also stops the server.
const cli = join(ROUTER_DIR, "node_modules", "jevonian", "dist", "cli.mjs");
const child = spawn(process.execPath, [cli, "serve", "--foreground"], { stdio: "inherit", windowsHide: true });
child.on("exit", (code) => process.exit(code ?? 0));
```

</details>

**3e. The client configs.** They're read by the CMD commands (Kilo, OpenCode), or used when you run the real client inside
the router folder (Qwen's `/model` list):

<details><summary><code>jev-router-kilo\.kilo\kilo.json</code> (injected as <code>KILO_CONFIG_CONTENT</code>)</summary>

```json
{
  "$schema": "https://kilo.ai/config.json",
  "provider": {
    "jevonian": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Jevonian :8795 (Jev router -> Alibaba)",
      "options": {
        "baseURL": "http://127.0.0.1:8795/v1",
        "apiKey": "local-no-key"
      },
      "models": {
        "jevonian/auto": {
          "name": "Jev Auto (Jev picks tier + effort each turn)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/chat": {
          "name": "Jev Chat = deepseek-v4.1-flash, effort low (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/small": {
          "name": "Jev Small = qwen3.8-flash, effort low (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/execute": {
          "name": "Jev Medium = qwen3.8-flash, effort high (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/large": {
          "name": "Jev Large = qwen3.8-max, effort xhigh (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/utility": {
          "name": "Jev Utility = qwen3.7-plus, effort medium (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/plan": {
          "name": "Jev Plan = glm-5.3, effort high (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        }
      }
    }
  },
  "model": "jevonian/jevonian/auto"
}
```

</details>

<details><summary><code>jev-router-opencode\opencode.json</code> (injected as <code>OPENCODE_CONFIG_CONTENT</code>)</summary>

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "jevonian": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Jevonian :8799 (Jev router -> Alibaba)",
      "options": {
        "baseURL": "http://127.0.0.1:8799/v1",
        "apiKey": "local-no-key"
      },
      "models": {
        "jevonian/auto": {
          "name": "Jev Auto (Jev picks tier + effort each turn)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/chat": {
          "name": "Jev Chat = deepseek-v4.1-flash, effort low (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/small": {
          "name": "Jev Small = qwen3.8-flash, effort low (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/execute": {
          "name": "Jev Medium = qwen3.8-flash, effort high (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/large": {
          "name": "Jev Large = qwen3.8-max, effort xhigh (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/utility": {
          "name": "Jev Utility = qwen3.7-plus, effort medium (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        },
        "jevonian/plan": {
          "name": "Jev Plan = glm-5.3, effort high (pinned)",
          "limit": {
            "context": 200000,
            "output": 65536
          },
          "tool_call": true
        }
      }
    }
  },
  "model": "jevonian/jevonian/auto"
}
```

</details>

<details><summary><code>jev-router-qwen\.qwen\settings.json</code> (for <code>qwen-direct</code> run inside <code>jev-router-qwen</code>)</summary>

```json
{
  "modelProviders": {
    "openai": [
      {
        "id": "jevonian/auto",
        "name": "Jev Auto (Jev picks tier + effort each turn) — Jevonian :8793",
        "baseUrl": "http://127.0.0.1:8793/v1",
        "envKey": "JEV_ROUTER_API_KEY",
        "description": "Jev router :8793 -> Alibaba Token Plan (chat/small/medium/large/utility/plan)"
      },
      {
        "id": "jevonian/chat",
        "name": "Jev Chat = deepseek-v4.1-flash, effort low (pinned) — Jevonian :8793",
        "baseUrl": "http://127.0.0.1:8793/v1",
        "envKey": "JEV_ROUTER_API_KEY"
      },
      {
        "id": "jevonian/small",
        "name": "Jev Small = qwen3.8-flash, effort low (pinned) — Jevonian :8793",
        "baseUrl": "http://127.0.0.1:8793/v1",
        "envKey": "JEV_ROUTER_API_KEY"
      },
      {
        "id": "jevonian/execute",
        "name": "Jev Medium = qwen3.8-flash, effort high (pinned) — Jevonian :8793",
        "baseUrl": "http://127.0.0.1:8793/v1",
        "envKey": "JEV_ROUTER_API_KEY"
      },
      {
        "id": "jevonian/large",
        "name": "Jev Large = qwen3.8-max, effort xhigh (pinned) — Jevonian :8793",
        "baseUrl": "http://127.0.0.1:8793/v1",
        "envKey": "JEV_ROUTER_API_KEY"
      },
      {
        "id": "jevonian/utility",
        "name": "Jev Utility = qwen3.7-plus, effort medium (pinned) — Jevonian :8793",
        "baseUrl": "http://127.0.0.1:8793/v1",
        "envKey": "JEV_ROUTER_API_KEY"
      },
      {
        "id": "jevonian/plan",
        "name": "Jev Plan = glm-5.3, effort high (pinned) — Jevonian :8793",
        "baseUrl": "http://127.0.0.1:8793/v1",
        "envKey": "JEV_ROUTER_API_KEY"
      }
    ]
  },
  "env": {
    "JEV_ROUTER_API_KEY": "local-no-key"
  },
  "security": {
    "auth": {
      "selectedType": "openai"
    }
  },
  "model": {
    "name": "jevonian/auto"
  },
  "$version": 4
}
```

</details>

- **Kilo and OpenCode model names:** `jevonian/jevonian/auto` means provider `jevonian`, model `jevonian/auto`.
- **Qwen model names:** plain `jevonian/auto`.
- **The `limit` values** (200,000 context, 65,536 output) are what the clients plan with. The router still sends each turn to the model the tier names.

### Step 4: jev-gateway (official, unmodified)

```json
{
  "name": "jev-gateway-local",
  "private": true,
  "description": "Official jev-gateway (github.com/vinilana/jev-gateway) installed locally and unmodified. jev-claude.js and jev-opencode.js run its official launchers with documented settings.",
  "dependencies": {
    "jev-gateway": "0.4.3"
  }
}
```

```bat
cd /d D:\learn\JevAI\jev-gateway && npm install --no-audit --no-fund
```

Expected output: `added 3 packages in 2s`. The official launchers are now `node_modules\jev-gateway\bin\jev-claude.mjs` and
`jev-opencode.mjs`. Add the three small files below; they only pass documented settings to those launchers:

<details><summary><code>jev-gateway\gateway-env.js</code></summary>

```js
// Environment for the OFFICIAL jev-gateway launchers (no code changes to jev-gateway itself).
// Every jev-gateway variable here is documented in its README; the ANTHROPIC_* / CLAUDE_CODE_* ones are
// Claude Code's own and mirror exactly what Jevonian's official `jevonian launch claude` sets.
//
//   jev-claude   : Claude Code -> jev-gateway :8789 (Jev picks the TOOL) -> Jevonian :8797 (tier + effort) -> Anthropic
//   jev-opencode : OpenCode    -> jev-gateway :8791 (Jev picks the TOOL) -> Jevonian :8799 (tier + effort) -> Alibaba
"use strict";
const { readFileSync } = require("fs");
const { join } = require("path");

/** The Jev key for the gateway's tool decisions: $TYPESAFE_API_KEY, else credentials\typesafe-ai-credential.txt. */
function typesafeKey() {
  if (process.env.TYPESAFE_API_KEY) return process.env.TYPESAFE_API_KEY;
  try {
    return readFileSync(join(__dirname, "credentials", "typesafe-ai-credential.txt"), "utf8").replace(/\s+/g, "");
  } catch {
    return "";
  }
}

// What `jevonian launch claude` sets for Claude Code (Jevonian dist/claude-code-*.mjs, claudeCodeEnv()).
const CLAUDE_VIA_JEVONIAN = {
  ANTHROPIC_AUTH_TOKEN: "jevonian-local", // loopback sentinel Jevonian accepts; Jevonian itself uses your Claude login (OAuth)
  ANTHROPIC_API_KEY: "",
  ANTHROPIC_DEFAULT_OPUS_MODEL: "jevonian/auto",
  ANTHROPIC_DEFAULT_SONNET_MODEL: "jevonian/auto",
  ANTHROPIC_DEFAULT_HAIKU_MODEL: "jevonian/utility",
  CLAUDE_CODE_SUBAGENT_MODEL: "jevonian/auto",
  ANTHROPIC_DEFAULT_OPUS_MODEL_NAME: "Jevonian Auto",
  ANTHROPIC_DEFAULT_SONNET_MODEL_NAME: "Jevonian Auto",
  ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME: "Jevonian Utility",
  ANTHROPIC_CUSTOM_MODEL_OPTION: "jevonian/auto",
  ANTHROPIC_CUSTOM_MODEL_OPTION_NAME: "Jevonian Auto",
  CLAUDE_CODE_ATTRIBUTION_HEADER: "0",
  DISABLE_ERROR_REPORTING: "1",
  DISABLE_FEEDBACK_COMMAND: "1",
  CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY: "1",
};

function gatewayEnv(client) {
  const env = { TYPESAFE_API_KEY: typesafeKey(), JEV_PROVIDER: "typesafe" };
  if (client === "claude") {
    // jev-gateway forwards to the Jevonian Claude router instead of api.anthropic.com.
    Object.assign(env, { JEV_CLAUDE_UPSTREAM_BASE_URL: "http://127.0.0.1:8797/v1" }, CLAUDE_VIA_JEVONIAN);
  }
  if (client === "opencode") {
    Object.assign(env, {
      JEV_OPENCODE_UPSTREAM_BASE_URL: "http://127.0.0.1:8799/v1", // the Jevonian OpenCode router
      JEV_OPENCODE_MODEL: "jevonian/auto", //                        OpenCode selects jev-gateway/jevonian/auto
      OPENAI_API_KEY: "local-no-key", //                             forwarded upstream; Jevonian needs none on loopback
    });
  }
  return env;
}

/** Put the gateway env into this process, so the official launcher (and what it spawns) inherit it. */
function applyGatewayEnv(client) {
  Object.assign(process.env, gatewayEnv(client));
}

module.exports = { gatewayEnv, applyGatewayEnv };
```

</details>

<details><summary><code>jev-gateway\jev-claude.js</code></summary>

```js
#!/usr/bin/env node
// `jev-claude` in CMD (doskey macro, see `jev install`) -> the OFFICIAL jev-gateway launcher
// (node_modules\jev-gateway\bin\jev-claude.mjs), unmodified, chained to the Jevonian Claude router:
//   Claude Code -> jev-gateway :8789 (Jev picks the TOOL) -> Jevonian :8797 (Jev picks tier + effort) -> Anthropic
// Same tiers and effort as `claude` (plan Opus 5.5 xhigh, heavy Opus 5.5 high, execute Sonnet 5 high,
// utility Sonnet 5 medium, chat Haiku 4.5 low). Settings: gateway-env.js (documented env only).
//   jev-claude --dangerously-skip-permissions
//   jev-claude --model jevonian/plan -p "..."                    pin a tier
//   jev-claude --status | --dashboard | --stop | --routing off   official launcher flags
// The status window and "start it if it is down" logic are shared with the Jevonian commands
// (..\jevonian\lib; set JEVONIAN_DIR if the jevonian folder is somewhere else).
"use strict";
const { join, resolve } = require("path");
const { applyGatewayEnv } = require("./gateway-env");

const JEVONIAN = process.env.JEVONIAN_DIR ? resolve(process.env.JEVONIAN_DIR) : resolve(__dirname, "..", "jevonian");
const { launch, hasModelFlag } = require(join(JEVONIAN, "lib", "launch"));
const { TARGETS } = require(join(JEVONIAN, "lib", "targets"));

const LAUNCHER_FLAGS = new Set(["--gateway-help", "--print-config", "--start", "--stop", "--status", "--logs", "--dashboard", "--routing", "--setup"]);

applyGatewayEnv("claude");
const t = TARGETS["gateway-claude"];
launch({
  key: "gateway-claude",
  exe: process.execPath,
  display: `jev-claude (official jev-gateway ${t.bin})`,
  wire: (argv) => ({
    // Launcher flags go to the official launcher untouched; otherwise default the model to jevonian/auto.
    args: [t.bin, ...(LAUNCHER_FLAGS.has(argv[0]) || hasModelFlag(argv) ? argv : ["--model", "jevonian/auto", ...argv])],
    env: {},
    how: "official jev-claude, JEV_CLAUDE_UPSTREAM_BASE_URL=http://127.0.0.1:8797/v1 (Jevonian), Claude Code env as `jevonian launch claude`",
  }),
});
```

</details>

<details><summary><code>jev-gateway\jev-opencode.js</code></summary>

```js
#!/usr/bin/env node
// `jev-opencode` in CMD (doskey macro, see `jev install`) -> the OFFICIAL jev-gateway launcher
// (node_modules\jev-gateway\bin\jev-opencode.mjs), unmodified. This wrapper only sets documented env vars
// (gateway-env.js) and arguments:
//   JEV_OPENCODE_UPSTREAM_BASE_URL = the Jevonian OpenCode router :8799, JEV_OPENCODE_MODEL = jevonian/auto
//   -> OpenCode -> jev-gateway :8791 (Jev picks the TOOL) -> Jevonian :8799 (Jev picks MODEL + EFFORT) -> Alibaba
// plus two OpenCode 2.x needs the official launcher (tested on OpenCode v1) does not cover: --standalone
// (skip a background service started elsewhere with another config) and PWD.
//   jev-opencode                         interactive
//   jev-opencode run "fix the test"      one-shot (in scripts add  < nul)
//   jev-opencode --status | --dashboard | --stop     official launcher flags
"use strict";
const { join, resolve } = require("path");
const { applyGatewayEnv } = require("./gateway-env");

const JEVONIAN = process.env.JEVONIAN_DIR ? resolve(process.env.JEVONIAN_DIR) : resolve(__dirname, "..", "jevonian");
const { launch, withStandalone } = require(join(JEVONIAN, "lib", "launch"));
const { TARGETS } = require(join(JEVONIAN, "lib", "targets"));

const LAUNCHER_FLAGS = new Set(["--gateway-help", "--print-config", "--start", "--stop", "--status", "--logs", "--dashboard", "--routing", "--setup"]);

applyGatewayEnv("opencode");
process.env.PWD = process.cwd();
const t = TARGETS["gateway-opencode"];
launch({
  key: "gateway-opencode",
  exe: process.execPath,
  display: `jev-opencode (official jev-gateway ${t.bin})`,
  wire: (argv) => ({
    // Launcher flags (--status, --dashboard, ...) go to the official launcher untouched.
    args: [t.bin, ...(LAUNCHER_FLAGS.has(argv[0]) ? argv : withStandalone(argv))],
    env: {},
    how: "official jev-opencode, JEV_OPENCODE_UPSTREAM_BASE_URL=http://127.0.0.1:8799/v1 (Jevonian), JEV_OPENCODE_MODEL=jevonian/auto, --standalone",
  }),
});
```

</details>

### Step 5: the command layer

Create `jevonian\jev.js`, `jevonian\kilo.js`, `qwen.js`, `opencode.js`, `claude.js`, and `jevonian\lib\targets.js`,
`common.js`, `launch.js`, `monitor.js`. [Appendix A](#appendix-a-the-command-layer-every-file-in-full) has every file in
full. What they do:

- **`lib\targets.js`:** the six servers: ports, URLs, folders, window titles.
- **`lib\common.js`:**
  - Health checks.
  - Starting a router: `node start.js`, in the background, output to `logs\serve.log`.
  - Starting a gateway: the official `--start`, with the settings from `gateway-env.js`.
  - Stopping: a router's process tree, or a gateway's official `--stop`.
  - Opening a web page, the status windows, and running a client.
- **`lib\launch.js`:** what every CMD command does:
  1. Start the router (and, for a gateway, the Jevonian router behind it) if it's down.
  2. Open or reuse the status window.
  3. Print port, URL, web page, provider and tiers.
  4. Run the real client in your current folder.
- **`lib\monitor.js`:** the status window (section 7).
- **`kilo.js` / `qwen.js` / `opencode.js` / `claude.js`:** the wiring from 4.4.
- **`jev.js`:** `jev start | stop | restart | status | logs | windows | dashboards | test | install | uninstall`.

Check the syntax of everything:

```bat
cd /d D:\learn\JevAI
for %f in (jevonian\*.js jevonian\lib\*.js jevonian\jev-router-qwen\start.js jev-gateway\*.js) do node --check "%f"
```

No output means no errors.

### Step 6 (optional): prove the installs are the official ones

```bat
mkdir %TEMP%\jevpack && cd /d %TEMP%\jevpack
npm pack jevonian@0.1.7 jev-gateway@0.4.3
mkdir jevonian jev-gateway
tar -xzf jevonian-0.1.7.tgz -C jevonian
tar -xzf jev-gateway-0.4.3.tgz -C jev-gateway
fc /b jevonian\package\dist\cli.mjs D:\learn\JevAI\jevonian\jev-router-kilo\node_modules\jevonian\dist\cli.mjs
fc /b jev-gateway\package\dist\app.js D:\learn\JevAI\jev-gateway\node_modules\jev-gateway\dist\app.js
```

- **Before the first start:** both comparisons print `FC: no differences encountered`. In this build, all four Jevonian
  copies and jev-gateway were identical to the tarballs, file for file.
- **After the first start:** the Jevonian `cli.mjs` differs by exactly the patch edits ([Appendix B](#appendix-b-exact-diff-of-the-patched-jevonian-against-the-official-017)),
  and jev-gateway stays identical.

### Step 7: install the CMD commands

```bat
node D:\learn\JevAI\jevonian\jev.js install
```

Output from this build (it replaced the entry of the older copy this setup was moved from):

```text
[jev] AutoRun set (replaced 1 older jev.doskey entry); previous value saved to D:\learn\JevAI\jevonian\autorun.backup.json
[jev] macros written to D:\learn\JevAI\jevonian\jev.doskey
[jev] Open a NEW CMD window, then type:  kilo | qwen | opencode | claude   (Jevonian)
[jev]                                   jev-claude | jev-opencode     (jev-gateway -> Jevonian)   jev status
```

It writes two things:

1. **`D:\learn\JevAI\jevonian\jev.doskey`:** plain-text CMD macros (`doskey`), with no `.bat` files:

   ```text
   kilo=node "D:\learn\JevAI\jevonian\kilo.js" $*
   qwen=node "D:\learn\JevAI\jevonian\qwen.js" $*
   opencode=node "D:\learn\JevAI\jevonian\opencode.js" $*
   claude=node "D:\learn\JevAI\jevonian\claude.js" $*
   jev-claude=node "D:\learn\JevAI\jev-gateway\jev-claude.js" $*
   jev-opencode=node "D:\learn\JevAI\jev-gateway\jev-opencode.js" $*
   jev=node "D:\learn\JevAI\jevonian\jev.js" $*
   claude-direct="C:\Users\PIRATCHAI.K\.local\bin\claude.exe" $*
   opencode-direct="C:\Users\PIRATCHAI.K\.bun\bin\opencode.exe" $*
   kilo-direct="C:\Users\PIRATCHAI.K\.bun\bin\kilo.exe" $*
   qwen-direct="C:\nvm4w\nodejs\qwen.cmd" $*
   ```

2. **The per-user registry value `HKCU\Software\Microsoft\Command Processor\AutoRun`.** CMD runs it in every new window:

   ```text
   if exist "D:\learn\JevAI\jevonian\jev.doskey" doskey /macrofile="D:\learn\JevAI\jevonian\jev.doskey"
   ```

   - An AutoRun value you already had is kept; the fragment is appended with `&`.
   - The previous value is saved to `autorun.backup.json`.
   - Entries that load a `jev.doskey` from another folder (an older copy) are replaced.
   - `jev uninstall` removes exactly this fragment.

Open a **new** CMD window: `doskey /macros` lists the commands. A doskey macro file has no comment syntax, so it has no
header line; a `;` line would print "Invalid macro definition." in every window.

**Limits of doskey macros:** they work at the **interactive CMD prompt** only. They don't work in `.bat` files, in PowerShell,
or with `cmd /d`. There, run the file directly, for example `node D:\learn\JevAI\jevonian\kilo.js run "…"`. The commands
behave the same; the macro only saves typing.

### Step 8: start everything

```bat
jev start
```

Output from this build, the first start after the installs (5.7 seconds in total):

```text
[jev] qwen         :8793  started
[jev] kilo         :8795  started
[jev] claude       :8797  started
[jev] opencode     :8799  started
[jev] jev-claude   :8789  started
[jev] jev-opencode :8791  started
```

You don't have to run `jev start`: every CMD command starts what it needs. Each router's log starts like this
(`jev logs qwen`); on the first start after an install, every patch says `applied`, and later `already applied`:

```text
[start] jev-router-qwen: jevonian 0.1.7, port 8793, node v22.23.2, 2026-09-25T06:01:06.890Z
WAF patch: applied
Effort patch: applied (5 edits)
models: up to date
public surface on http://127.0.0.1:8794 (only /v1, key required)
jevonian listening on http://127.0.0.1:8793/
providers: alibaba-tokenplan
routing: auto (models: jevonian/auto, jevonian/plan, jevonian/execute, jevonian/utility, jevonian/chat, jevonian/small, jevonian/large)
```

The Claude router also prints `Haiku patch: applied` and `models: auto-sync disabled`. Then check with `jev status`:

```text
  name           command        port   state  status window   web page
  qwen           qwen           8793   UP     open            http://127.0.0.1:8793/  (+ /logs)
  kilo           kilo           8795   UP     open            http://127.0.0.1:8795/  (+ /logs)
  claude         claude         8797   UP     open            http://127.0.0.1:8797/  (+ /logs)
  opencode       opencode       8799   UP     open            http://127.0.0.1:8799/  (+ /logs)
  jev-claude     jev-claude     8789   UP     open            http://127.0.0.1:8789/dashboard?peers=none
  jev-opencode   jev-opencode   8791   UP     open            http://127.0.0.1:8791/dashboard?peers=none

  Upstreams: qwen, kilo, opencode -> Alibaba Token Plan; claude -> Anthropic (your Claude Code login)
             jev-claude -> Jevonian claude :8797; jev-opencode -> Jevonian opencode :8799
  CMD commands: installed (new CMD windows have them)
```

### Step 9: test everything

```bat
jev test
```

It runs three groups, and the output is also saved to `jevonian\run\`:

1. **`jev test brains`:** both Jev channels on every router, through Jevonian's own "Test channel" endpoint.
2. **`jev test tiers`:** every tier of every router, pinned:
   - **Alibaba:** each tier is called over HTTP, and the model and effort Jevonian *actually sent* (`x-jevonian-model` /
     `x-jevonian-effort`) must equal the config.
   - **Claude:** a subscription login only serves Claude Code itself, so each tier runs through `jevonian launch claude -p`
     and is read back from the router's ledger.
3. **`jev test clients`:** each of the six commands, one-shot, from `%TEMP%\jev-test`:
   - a reply with a marker, like `QWEN_OK`
   - "Read the file notes.txt …", which needs a tool call; the file holds a random `JEV-NOTES-…` line
   - it checks the answer, that the router logged the request, and (gateways) that the gateway saw it

   The full client output goes to `run\test-output\`. Add `set JEV_TEST_WINDOWS=1` first to watch it in the status windows.

The real results of this build:

```text

== brains: POST /api/brain/test on each router (the dashboard's Test channel button) ==
  PASS  qwen      typesafe  TypeSafe (direct) / jev-latest, 698 ms
  PASS  qwen      vercel    Vercel AI Gateway / typesafe-ai/jev, 1268 ms
  PASS  kilo      typesafe  TypeSafe (direct) / jev-latest, 642 ms
  PASS  kilo      vercel    Vercel AI Gateway / typesafe-ai/jev, 817 ms
  PASS  claude    typesafe  TypeSafe (direct) / jev-latest, 300 ms
  PASS  claude    vercel    Vercel AI Gateway / typesafe-ai/jev, 632 ms
  PASS  opencode  typesafe  TypeSafe (direct) / jev-latest, 697 ms
  PASS  opencode  vercel    Vercel AI Gateway / typesafe-ai/jev, 642 ms

== 8/8 passed ==
```

```text

== tiers: each pinned tier must be served by its first model at its configured effort ==
  result  router    tier       expected                          got
  PASS    qwen      plan       glm-5.3 / high                    glm-5.3 / high (200)
  PASS    qwen      execute    qwen3.8-flash / high              qwen3.8-flash / high (200)
  PASS    qwen      utility    qwen3.7-plus / medium             qwen3.7-plus / medium (200)
  PASS    qwen      chat       deepseek-v4.1-flash / low         deepseek-v4.1-flash / low (200)
  PASS    qwen      small      qwen3.8-flash / low               qwen3.8-flash / low (200)
  PASS    qwen      large      qwen3.8-max / xhigh               qwen3.8-max / xhigh (200)
  PASS    kilo      plan       glm-5.3 / high                    glm-5.3 / high (200)
  PASS    kilo      execute    qwen3.8-flash / high              qwen3.8-flash / high (200)
  PASS    kilo      utility    qwen3.7-plus / medium             qwen3.7-plus / medium (200)
  PASS    kilo      chat       deepseek-v4.1-flash / low         deepseek-v4.1-flash / low (200)
  PASS    kilo      small      qwen3.8-flash / low               qwen3.8-flash / low (200)
  PASS    kilo      large      qwen3.8-max / xhigh               qwen3.8-max / xhigh (200)
  PASS    claude    plan       claude-opus-5-5 / xhigh           claude-opus-5-5 / xhigh (200)
  PASS    claude    execute    claude-sonnet-5 / high            claude-sonnet-5 / high (200)
  PASS    claude    utility    claude-sonnet-5 / medium          claude-sonnet-5 / medium (200)
  PASS    claude    chat       claude-haiku-4-5-20251001 / low   claude-haiku-4-5-20251001 / low (200)
  PASS    claude    heavy      claude-opus-5-5 / high            claude-opus-5-5 / high (200)
  PASS    opencode  plan       glm-5.3 / high                    glm-5.3 / high (200)
  PASS    opencode  execute    qwen3.8-flash / high              qwen3.8-flash / high (200)
  PASS    opencode  utility    qwen3.7-plus / medium             qwen3.7-plus / medium (200)
  PASS    opencode  chat       deepseek-v4.1-flash / low         deepseek-v4.1-flash / low (200)
  PASS    opencode  small      qwen3.8-flash / low               qwen3.8-flash / low (200)
  PASS    opencode  large      qwen3.8-max / xhigh               qwen3.8-max / xhigh (200)

== 23/23 passed ==
```

```text

== clients: each CMD command, one-shot, in C:\Users\PIRATC~1.K\AppData\Local\Temp\jev-test ==
  result  command        prompt  answer                     router saw (phase model effort)             gateway saw (mode tool)
  PASS    qwen           marker  QWEN_OK                    chat:deepseek-v4.1-flash:low                
  PASS    qwen           read    JEV-NOTES-2E8530           chat:deepseek-v4.1-flash:low, chat:deepseek-v4.1-flash:low 
  PASS    kilo           marker  KILO_OK                    chat:deepseek-v4.1-flash:low                
  PASS    kilo           read    JEV-NOTES-2E8530           utility:qwen3.7-plus:medium, utility:qwen3.7-plus:medium 
  PASS    claude         marker  CLAUDE_OK                  utility:claude-sonnet-5:medium, chat:claude-haiku-4-5-20251001:low 
  PASS    claude         read    JEV-NOTES-2E8530           utility:claude-sonnet-5:medium, utility:claude-sonnet-5:medium, utility:claude-sonnet-5:medium 
  PASS    opencode       marker  OPENCODE_OK                chat:deepseek-v4.1-flash:low, chat:deepseek-v4.1-flash:low 
  PASS    opencode       read    JEV-NOTES-2E8530           utility:qwen3.7-plus:medium                 
  PASS    jev-claude     marker  JEV_CLAUDE_OK              utility:claude-sonnet-5:medium, chat:claude-haiku-4-5-20251001:low passthrough:-, passthrough:(no_tool_needed)
  PASS    jev-claude     read    JEV-NOTES-2E8530           utility:claude-sonnet-5:medium, utility:claude-sonnet-5:medium, utility:claude-sonnet-5:medium passthrough:-, hint:Read, passthrough:(no_tool_needed)
  PASS    jev-opencode   marker  JEV_OPENCODE_OK            chat:deepseek-v4.1-flash:low                none:(no_tool_needed), passthrough:-
  PASS    jev-opencode   read    JEV-NOTES-2E8530           utility:qwen3.7-plus:medium, utility:qwen3.7-plus:medium, utility:qwen3.7-plus:medium passthrough:(read), none:(no_tool_needed), passthrough:-
  (full client output: D:\learn\JevAI\jevonian\run\test-output)

== 12/12 passed ==
```

- **"router saw":** what the Jevonian router logged for that prompt.
- **"gateway saw":** jev-gateway's tool decision for each request: `passthrough`, `hint` (Claude), `none` (Jev: no tool
  needed), or `(read)` / `Read` (the tool Jev picked).
- **Two routes per Claude prompt:** Claude Code makes a small background call, sent as its "Haiku" model, which
  `jevonian launch claude` maps to `jevonian/utility` (Sonnet 5, medium).
- **`jev-opencode` read:** see [known behaviours](#11-known-behaviours-and-limits) for why it shows `(read)` and not `forced`.

### Step 10: open the status windows and the web pages

```bat
jev windows
jev dashboards
```

- **`jev windows`** opens six windows: `JEVONIAN - QWEN - :8793`, `JEVONIAN - KILO - :8795`, `JEVONIAN - CLAUDE - :8797`,
  `JEVONIAN - OPENCODE - :8799`, `JEV-GATEWAY - CLAUDE - :8789`, `JEV-GATEWAY - OPENCODE - :8791`.
- **`jev dashboards`** opens the six web pages in your browser (section 8).
- Closing a window never stops a router. The CMD commands open their own window anyway.

---

## 6. Everyday use

Open a new CMD window, go to your project, and type the command. It starts whatever isn't running, opens the tool's status
window (and, the first time, its web page), prints where it's connected, then runs the real client **in your folder**:

```bat
cd /d D:\path\to\your\project

qwen                                     :: interactive; one-shot:  qwen -p "fix the failing test"
kilo                                     :: interactive; one-shot:  kilo run "fix the failing test"
opencode                                 :: interactive; one-shot:  opencode run "fix the failing test"
claude --dangerously-skip-permissions    :: one-shot:  claude -p "fix the failing test"

jev-claude --dangerously-skip-permissions   :: Claude Code through jev-gateway -> Jevonian
jev-opencode                                :: OpenCode through jev-gateway -> Jevonian; one-shot: jev-opencode run "…"
```

**Pin a tier** (skip Jev's choice):

| Tool | Command |
|---|---|
| Qwen Code | `qwen -m jevonian/large` |
| Kilo | `kilo -m jevonian/jevonian/plan` |
| OpenCode | `opencode -m jevonian/jevonian/small` |
| Claude Code | `claude --model jevonian/plan` (plan, heavy, execute, utility, chat) |
| jev-claude | `jev-claude --model jevonian/heavy` |

**Manage everything:**

| Command | Does |
|---|---|
| `jev status` | Up/down, status window and web page of all six. Also whether the CMD commands are installed. |
| `jev start` / `jev stop` / `jev restart` `[name]` | All six, or one (`qwen`, `kilo`, `claude`, `opencode`, `jev-claude`, `jev-opencode`). A gateway also starts the Jevonian router behind it. Stop does the gateways first. |
| `jev logs <name>` | The last 40 lines of that router's `logs\serve.log`, or the gateway's `%USERPROFILE%\.jev-gateway\<client>.log`. |
| `jev windows` / `jev windows close` | Open or close all six status windows. |
| `jev dashboards` | Open all six web pages in the browser. |
| `jev test [brains\|tiers\|clients] [name]` | The checks from step 9. |
| `jev-claude --status` / `--dashboard` / `--routing off` / `--routing on` / `--stop` | The **official** jev-gateway launcher flags, passed through unchanged (the same for `jev-opencode`). |

**After a reboot** nothing runs until your first command, which starts what it needs in about 2 seconds. There is no
Windows service and no autostart.

**Where things run:**
- The routers and gateways run in the background, with no window of their own.
- Their logs: `jevonian\jev-router-<tool>\logs\serve.log` and `%USERPROFILE%\.jev-gateway\<client>.log`.

Setting environment variables before a command:

| Variable | Effect |
|---|---|
| `JEV_OPEN_DASHBOARD=0` | Don't open web pages. |
| `JEV_NO_WINDOW=1` | Don't open a status window (scripts). |
| `JEV_GATEWAY_DIR` / `JEVONIAN_DIR` | The folders aren't siblings. |

---

## 7. Status windows

Every command opens (or reuses) one console window per server, titled like `JEVONIAN - KILO - :8795`. Real content of the
Kilo window during the test run:

```text
==============================================================================
  JEVONIAN - KILO - :8795   Jevonian router for kilo
==============================================================================
  Status      : ONLINE   {"ok":true,"sessions":1,"routing":"auto"}
  Port        : 8795   (Jevonian also binds 8796 for its tunnel surface)
  URL         : http://127.0.0.1:8795/v1
  Web page    : http://127.0.0.1:8795/    Logs: http://127.0.0.1:8795/logs
  Provider    : Alibaba Cloud Model Studio - Token Plan  (https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1)
  Jev brain   : TypeSafe Jev (fallback: Vercel AI Gateway) picks the tier each turn
  Tiers (model, effort):
      jevonian/plan          glm-5.3                      high
      jevonian/execute       qwen3.8-flash                high
      jevonian/utility       qwen3.7-plus                 medium
      jevonian/chat          deepseek-v4.1-flash          low
      jevonian/small         qwen3.8-flash                low
      jevonian/large         qwen3.8-max                  xhigh
  Ledger      : D:\learn\JevAI\jevonian\jev-router-kilo\data\ledger.jsonl
  Window log  : D:\learn\JevAI\jevonian\run\jevonian-kilo.monitor.log
==============================================================================
  Live requests (this window only watches; closing it does not stop the router)
  time      phase      model                       effort  status  cost      latency  requested (brain)
  13:09:32  chat       deepseek-v4.1-flash         low     200     $0.0000    7367ms  jevonian/auto (jev)
  13:09:50  utility    qwen3.7-plus                medium  200     $0.0120    3519ms  jevonian/auto (jev-low-confidence)
  13:09:53  utility    qwen3.7-plus                medium  200     $0.0131    1989ms  jevonian/auto (jev)
```

| Part | Where it comes from |
|---|---|
| Header | `lib\targets.js` + the router's `config.json` (tiers and effort; `(forced)` marks `forceEffort`) |
| Jevonian lines | the router's ledger, `data\ledger.jsonl`: the same records as its web page's **Logs**, with the same time, model, phase, effort, status, cost and latency |
| jev-gateway lines | the gateway's official `/dashboard/events`: the same data as its web page (mode, tool Jev picked, confidence, tokens) |
| `(jev)` / `(jev-low-confidence)` | Jev chose the tier; "low-confidence" means below `minConfidence` 0.6, and the choice is still used |
| `(the last 8 before this window opened)` | recent history, dimmed, so a window opened *after* a request still shows it |

**The window refreshes itself.** An older version printed its header once, so after a config change or a restart it showed
stale tiers (for example `(default)` effort and heavy = Sonnet) or an old gateway, and a restarted gateway's requests never appeared.
Now:
- A changed `config.json`, or a router coming back, redraws the header.
- A **restarted gateway** is followed from its new start. Its request numbers begin again at 1, which the official page
  handles the same way, with the `startedAt` field.
- Rows the gateway replays from its log file on restart are not shown again.

Real content of the `JEV-GATEWAY - CLAUDE` window across a restart:

```text
  13:11:02  passthrough  jevonian/auto            29  200     (no_tool_needed)             1.00   48945 / 15
  13:16:48  router on :8789 is not answering
  ---- 13:16:50  gateway restarted at 13:16:49: header refreshed ----
==============================================================================
  JEV-GATEWAY - CLAUDE - :8789   jev-gateway for claude
==============================================================================
  Status      : ONLINE   {"status":"ok","pid":82480,"upstream":"http://127.0.0.1:8797/v1","jev":"typesafe"}
  Port        : 8789
  URL         : http://127.0.0.1:8789
  Web page    : http://127.0.0.1:8789/dashboard?peers=none    Both gateways: http://127.0.0.1:8789/dashboard
  Provider    : Jevonian Claude router :8797 (tier + effort) -> Anthropic API (api.anthropic.com) - your Claude Code login (OAuth), no API key
  Jev brain   : TypeSafe Jev picks the next TOOL (official jev-gateway); the model is not changed
  Tiers (model, effort):
      jevonian/auto          -> Jevonian :8797, which picks: 
        via jevonian/plan    claude-opus-5-5              xhigh (forced)
        via jevonian/execute claude-sonnet-5              high (forced)
        via jevonian/utility claude-sonnet-5              medium (forced)
        via jevonian/chat    claude-haiku-4-5-20251001    low (forced)
        via jevonian/heavy   claude-opus-5-5              high (forced)
  Log         : C:\Users\PIRATCHAI.K\.jev-gateway\claude.log
  Window log  : D:\learn\JevAI\jevonian\run\gateway-claude.monitor.log
==============================================================================
  Live requests (this window only watches; closing it does not stop the router)
  time      mode         model                 tools  status  tool / reason                conf   in / out
  13:16:51  router on :8789 is back online
  13:16:59  passthrough  jevonian/utility          0  200     - no_tools                   -      1205 / 23
  13:16:59  passthrough  jevonian/auto            29  200     (no_tool_needed)             1.00   34999 / 51
```

- **Text copy:** every window also writes what it prints to `jevonian\run\<target>.monitor.log` (plain text), so you can
  check afterwards what it showed.
- **Closing a window never stops its router.** `jev windows close` closes all six.

---

## 8. Web pages: one per tool

Each of the six servers is its own web server on 127.0.0.1:

| Tool | Server | Page | What it shows |
|---|---|---|---|
| `qwen` | Jevonian | http://127.0.0.1:8793/logs (Overview at `/`) | every request: time, model, provider, phase (tier), **effort**, status, cost, latency; "details" shows the prompt and Jev's decision |
| `kilo` | Jevonian | http://127.0.0.1:8795/logs | the same |
| `claude` | Jevonian | http://127.0.0.1:8797/logs | the same; also `jev-claude`'s turns, which pass through it |
| `opencode` | Jevonian | http://127.0.0.1:8799/logs | the same; also `jev-opencode`'s turns |
| `jev-claude` | jev-gateway | http://127.0.0.1:8789/dashboard?peers=none | the gateway's status (Routing / Passthrough only / Idle), Jev's calls, latency and confidence, tokens, why requests were not routed, and every request with its mode and tool |
| `jev-opencode` | jev-gateway | http://127.0.0.1:8791/dashboard?peers=none | the same |
| both gateways | jev-gateway | http://127.0.0.1:8789/dashboard (or `:8791/dashboard`) | the official combined page, with both cards |

- **jev-gateway's page is only at `/dashboard`.** Its root `http://127.0.0.1:8789/` answers **404 "Not found"** by design.
- **Without `?peers=none`**, the page also shows every other gateway it finds on the official ports, so `:8789/dashboard`
  and `:8791/dashboard` look identical. `?peers=none` is jev-gateway's own option for "only this gateway".
- **Harmless console errors:** on the combined page, `ERR_CONNECTION_REFUSED` for 8787/8788/8790 (the page probes the
  standalone, Gemini and Codex ports, which aren't used here), and `favicon.ico` 404.
- **"Passthrough only"** on a gateway card isn't an error. Jev said no tool was needed, its confidence was below 0.7, or
  the request had no tools.
- **Ports 8794, 8796, 8798, 8800 aren't pages:** they are the routers' tunnel surface, and answer 401.

---

## 9. Test results: comparing the three views of every tool

Every tool has three "tabs" to look at:
- **the client's own answer**
- **its status window**
- **its web page**

They were compared for the final `jev test clients` run of 2026-09-25 (13:08 to 13:12):
- The client answers come from `run\test-output\`.
- The status window lines come from `run\*.monitor.log`.
- The web page rows were read in the browser (one tab per server, with Playwright).

Result: for every tool, the status window and the web page show the **same rows**: the same time, tier, model, effort,
status, cost and latency (or gateway mode, tool, confidence and tokens).

**Through Jevonian:**

| Tool, prompt | Client answer | Status window = web page (time · tier · model · effort · status · latency) |
|---|---|---|
| `qwen`, marker | `QWEN_OK` | 13:08:57 · chat · deepseek-v4.1-flash · low · 200 · 2868 ms |
| `qwen`, read notes.txt | `JEV-NOTES-2E8530` | 13:09:08 · chat · deepseek · low · 200 · 2145 ms; 13:09:11 · chat · deepseek · low · 200 · 2641 ms |
| `kilo`, marker | `KILO_OK` | 13:09:32 · chat · deepseek-v4.1-flash · low · 200 · 7367 ms |
| `kilo`, read | `JEV-NOTES-2E8530` | 13:09:50 · utility · qwen3.7-plus · medium · 200 · 3519 ms (jev-low-confidence); 13:09:53 · utility · qwen3.7-plus · medium · 200 · 1989 ms |
| `claude`, marker | `CLAUDE_OK` | 13:09:57 · utility · claude-sonnet-5 · medium · 200 · 1429 ms (Claude Code's background call); 13:09:59 · chat · claude-haiku-4-5 · low · 200 · 2615 ms |
| `claude`, read | `JEV-NOTES-2E8530` | 13:10:04 · utility · sonnet-5 · medium (background); 13:10:08 · utility · sonnet-5 · medium · 5751 ms; 13:10:11 · utility · sonnet-5 · medium · 2332 ms |
| `opencode`, marker | `OPENCODE_OK` | 13:10:28 · chat · deepseek · low · 4084 ms (title); 13:10:29 · chat · deepseek · low · 5021 ms |
| `opencode`, read | `JEV-NOTES-2E8530` | 13:10:45 · utility · qwen3.7-plus · medium · 200 · 2083 ms. The tool-call turn before it has **no row in either view**; see [known behaviours](#11-known-behaviours-and-limits). |

**Through jev-gateway**, where each prompt shows up in **two** servers:

| Tool, prompt | Client answer | Gateway window = gateway page (mode · tool · confidence · tokens in/out) | Jevonian window = Jevonian page |
|---|---|---|---|
| `jev-claude`, marker | `JEV_CLAUDE_OK` | 13:10:51 passthrough · no_tools (background call) · 1206/25; 13:10:51 passthrough · (no_tool_needed) · 0.99 · 35002/72 | :8797, 13:10:53 · utility · sonnet-5 · medium · 2066 ms; 13:10:54 · chat · haiku-4-5 · low · 1478 ms |
| `jev-claude`, read | `JEV-NOTES-2E8530` | 13:10:57 passthrough · no_tools; **13:10:58 hint · Read · 0.99** · 48825/84; 13:11:02 passthrough · (no_tool_needed) · 1.00 · 48945/15 | :8797, 13:10:59 · utility · 1388 ms; 13:11:02 · utility · 3217 ms; 13:11:04 · utility · 1949 ms (all sonnet-5 · medium) |
| `jev-opencode`, marker | `JEV_OPENCODE_OK` | 13:11:21 **none** · (no_tool_needed) · 1.00 · 8941/14; 13:11:21 passthrough · no_tools (title) | :8799, 13:11:23 · chat · deepseek · low · 1409 ms |
| `jev-opencode`, read | `JEV-NOTES-2E8530` | **13:11:42 passthrough · (read) · 0.98 · upstream_rejected_forced** · 12507/61; 13:11:46 none · 0.99 · 11942/75; 13:11:42 passthrough · no_tools | :8799, **13:11:43 · utility · qwen3.7-plus · medium · 400** · 869 ms; 13:11:46 · utility · 200 · 2272 ms; 13:11:48 · utility · 200 · 2400 ms |

How to read the differences:
- **Clock differences:** a gateway row's time is when the gateway *received* the request. The Jevonian row is 1 to 2 s later,
  after the gateway's Jev call (≈0.3 to 1 s) and Jevonian's own Jev call.
- **Background rows:** `jevonian/utility` rows with no tools are Claude Code's small background call, and "title" rows are
  OpenCode's session-title call. Both are normal client behaviour.
- **`hint · Read`:** Jev told Claude Code "use the Read tool" as a one-line suggestion. With Claude Code, jev-gateway only
  hints, because thinking is on. Claude Code then read the file.
- **`upstream_rejected_forced` + the 400:** see [known behaviours](#11-known-behaviours-and-limits). The official gateway
  recovered by itself, and the answer was right.

Screenshots of all seven pages from this run: `plan6-1-qwen-8793-logs.png` … `plan6-7-both-gateways-8789.png` (kept next to
the guide's working folder, `.playwright-mcp\`).

---

## 10. Can Jevonian and jev-gateway share one folder?

**Yes. Both official packages install and run from one folder with no conflict.** They're still kept in two folders here,
by choice:
- Each project gets its own version pin and upgrade.
- Each folder shows clearly what belongs to which project.

**What was checked in their code:**
- **Names:** different package names, and different commands (`jevonian` vs `jev-claude`, `jev-codex`, `jev-gemini`, `jev-opencode`).
- **Settings:** neither reads a `.env` from the working folder. Jevonian reads `~/.config/jevonian` unless `JEVONIAN_*`
  variables say otherwise. jev-gateway reads `~/.jev-gateway/.env`, plus a source checkout's own `.env` only when run from source.
- **State:** jev-gateway's state (pid, log) is always in `%USERPROFILE%\.jev-gateway`, whatever folder it runs from.

**What was tested:**
- One scratch folder, with `jevonian@0.1.7` and `jev-gateway@0.4.3` both installed in it.
- A Jevonian router on 8811, and the official `jev-opencode --start` gateway on 8815 → 8811, both run from that folder.
- One request with a tool, sent through both.

Real output:

```text
1. npm install jevonian@0.1.7 + jev-gateway@0.4.3 into ONE folder
   added 16 packages in 6s exit 0
   npm ls: +-- jev-gateway@0.4.3 | `-- jevonian@0.1.7 (exit 0)
   bins: jev-claude, jev-codex, jev-gemini, jev-opencode, jevonian
2. start Jevonian :8811 and jev-gateway (official jev-opencode --start) :8815 -> :8811, both from this folder
   Jevonian :8811 UP
   gateway --start: jev-opencode: router up on http://127.0.0.1:8815 → http://127.0.0.1:8811/v1 (logs: C:\Users\PIRATCHAI.K\.jev-gateway\opencode.log)
   gateway :8815 health: {"status":"ok","pid":45052,"upstream":"http://127.0.0.1:8811/v1","jev":"typesafe"}
3. one request with a tool through gateway :8815 -> Jevonian :8811 -> Alibaba
   HTTP 200 in 3213 ms; reply: "SAME_FOLDER_OK"
   gateway said: mode=none reason=null
   Jevonian said: model=deepseek-v4.1-flash phase=chat effort=medium
   gateway dashboard events: 10 (none 200, passthrough 200, none 200, passthrough 200, passthrough 200, none 200, passthrough 200, passthrough 200, none 200, none 200)
   Jevonian ledger entries: 1 (chat deepseek-v4.1-flash 200)
4. files written in the shared folder (node_modules excluded):
   jevonian-config.json
   jevonian-data\bodies\10442a7c-a4f3-4336-8bbd-196ccfd12e0d.json
   jevonian-data\bodies\7949b118-8bad-422e-9312-08d5c3685492.json
   jevonian-data\leaderboard.json
   jevonian-data\ledger.jsonl
   jevonian-data\model-sync.json
   jevonian-data\pricing.json
   jevonian-data\update.json
   jevonian.log
   package-lock.json
   package.json
   + jev-gateway wrote its pid/log to %USERPROFILE%\.jev-gateway (official location, not this folder)
5. stop both
   gateway --stop: jev-opencode: stopped router (pid 45052).
   :8811 down  :8815 down
```

- **The effort patch, proven:** that router ran the **unpatched** official package, and it sent the chat tier at `medium`,
  not the configured `low`.
- **The request count:** the gateway's "10 events" include rows it replayed from its log file
  (`%USERPROFILE%\.jev-gateway\opencode.log`); only one was new.

**Real conflicts, all about ports:**
1. **Jevonian's default port 8787 is jev-gateway's standalone-server port**, and the combined gateway page probes it. Give
   Jevonian another port (here 8793 to 8800).
2. **Jevonian always binds its port + 1** (the tunnel surface). The first attempt of this test put the gateway on 8812 =
   8811 + 1, and it failed with `EADDRINUSE`. Never give a gateway a Jevonian router's port + 1.
3. The official launchers keep **one pid file per client** (`%USERPROFILE%\.jev-gateway\<client>.pid`). Run one gateway
   per client at a time; this test ran with the real gateways stopped.

---

## 11. Known behaviours and limits

These are real behaviours found while testing this build. None of them breaks a tool.

**1. Claude Code's own effort setting doesn't change the tiers.**
- Claude Code always sends its own effort level (default "high").
- With `"forceEffort": true` on every Claude route, the router's level wins. Claude Code's `/effort` has no effect here.
- To change a tier's effort, edit `jev-router-claude\config\config.json`, then run `jev restart claude`.

**2. Haiku 4.5 has no effort setting.** The chat tier's "low" becomes a 2,048-token thinking budget: Jevonian's official
mapping, with the Haiku patch.

**3. Claude Code's background calls run on the utility tier.**
- Claude Code makes a small background call per prompt, sent as its "Haiku" model.
- Jevonian's official `jevonian launch claude` maps that model to `jevonian/utility`, which is **Sonnet 5 at medium** here.
- That's the extra utility row per Claude prompt in the logs.

**4. jev-opencode: Jev's forced tool pick is rejected by Qwen thinking models, and the gateway recovers.**

What happens:
1. When Jev is sure which tool OpenCode should call, jev-gateway sets `tool_choice` to that tool (mode `forced`).
2. The Jevonian OpenCode router sends the turn to a Qwen model **with thinking on** (every tier has an effort).
3. Alibaba rejects it: HTTP 400, *"The tool_choice parameter does not support being set to required or object in thinking mode"*.
4. jev-gateway then resends the original request by itself (reason `upstream_rejected_forced`), and the turn succeeds.

The cost is one failed request (≈1 s). Seen in section 9: a 400 row on :8799, then a 200.

Why it isn't changed:
- Changing it would mean either turning off thinking or patching jev-gateway, and both go against official-first.
- jev-claude isn't affected: with Claude Code the gateway only *hints* (mode `hint`).

**5. A one-shot run can be missing a row in Logs and in the status window.**
- Jevonian writes a request's ledger line when the reply stream ends.
- `opencode run` (and some title calls) can exit the moment the answer is printed, cutting the stream first, so that
  request has no row.
- The request *was* routed and answered: its Jev brain calls are in the ledger (`"kind": "brain"`), and `jev test`
  reports it as `(routed: N brain calls, no request line)`.
- Interactive sessions don't exit mid-stream. This is official Jevonian behaviour and isn't patched.

**6. Kilo shares sessions to its cloud by default.**
- `kilo run` prints a link like `https://app.kilo.ai/s/…`: Kilo's own session sharing to app.kilo.ai, not part of this setup.
- To turn it off, add `"share": "disabled",` to `jev-router-kilo\.kilo\kilo.json`. It was tested: no link is printed.
- It was **not** turned off in this build, because it's your Kilo account's behaviour.

**7. Kilo and OpenCode take their project folder from `PWD`.**
- If `PWD` is set (Git Bash and some tools set it), Kilo / OpenCode 2.x use it instead of the real current folder.
- In this build's first test run, Kilo looked for `notes.txt` in the wrong folder.
- `kilo.js` and `opencode.js` (and `jev-opencode.js`) now set `PWD` to the folder you run them in.

**8. OpenCode 2.x runs a background service.** One started from another folder keeps *that* folder's config
("Model unavailable"). The commands always pass `--standalone`. jev-gateway officially supports OpenCode v1; with the
same two fixes, `jev-opencode` works on 2.0.15.

**9. `jevonian/auto` picks per turn, and the pick can vary.** In the tests, "read notes.txt" went to `chat` on Qwen but to
`utility` on Kilo, OpenCode and Claude. That's Jev's choice (with its confidence), not a fault. Pin a tier when you need a
fixed model.

**10. Messages you can ignore in Claude Code:**
- **"jevonian/auto isn't described by this version's model catalog … 200k":** Claude Code doesn't know the virtual model and
  assumes 200k tokens, which suits the Haiku tier.
- **"claude.ai connectors are disabled…":** a custom base URL turns off claude.ai cloud connectors in routed sessions. Your
  own MCP servers still load.

**11. Interactive typing wasn't automated.** The tests use each tool's one-shot mode. Interactive sessions use exactly the
same command, router and wiring. Typing into the TUIs from a script was blocked by this PC's endpoint protection, and
wasn't worked around.

---

## 12. Troubleshooting

First check `jev status` (what's up), `jev logs <name>` (why a router or gateway didn't start), and `jev test brains`
(the keys).

| Symptom | Cause | Fix |
|---|---|---|
| `kilo`, `claude`… open the plain client, or "is not recognized" | The CMD window was opened before `jev install`, or you're in PowerShell / a `.bat` (no doskey macros there) | Open a **new** CMD window, or run `node D:\learn\JevAI\jevonian\kilo.js …` |
| "Invalid macro definition." in every new CMD window | A line in `jev.doskey` that isn't `name=command` (e.g. a comment) | Run `jev install` again (it rewrites the file) |
| `[jev] qwen did not start on :8793` | A patch refused ("pattern … found N times"), a key is missing, or the port is taken | `jev logs qwen`; `netstat -ano \| findstr ":8793"` |
| `WARNING: ALIBABA_TOKENPLAN_API_KEY is empty` in `serve.log` | A key file is missing or has no `API Key:` line | Fix `jevonian\credentials\` (step 2), then `jev restart` |
| "Jev brain unavailable: all 2 configured brain(s) failed" | Both TypeSafe and Vercel failed (keys, network, a transient error) | `jev test brains`; check access to `api.typesafe.ai`; retry |
| A patch prints "pattern … found 0 times" | The Jevonian version changed the code the patch targets. Nothing was written, and the router refuses to start. | **Don't force it.** See [Upgrading](#13-upgrading-and-rolling-back). |
| Effort shows `default`, or the client's level, instead of the tier's | Alibaba: the effort patch is missing. Claude: the route lacks `"forceEffort": true`. | `findstr /c:"routing-effort-patch v3b" node_modules\jevonian\dist\cli.mjs` in the router folder; `jev restart` |
| HTTP 400 on Claude chat turns: `context_management: Extra inputs are not permitted` | Haiku was served without the Haiku patch, or the chat route's `providers` is an array | Put `patch-jevonian-haiku.mjs` in `jev-router-claude\`. `providers` must be a **map**: `{"claude-haiku-4-5-20251001": ["claude-subscription-haiku"]}`. |
| jev-opencode rows `upstream_rejected_forced` + a 400 on :8799 | Expected: see [known behaviour 4](#11-known-behaviours-and-limits) | Nothing to do; the gateway replays the original request |
| Kilo: "Add credits to continue, or switch to a free model" | Kilo ignored the config's default model and used its own cloud default | The `kilo` command always adds `-m jevonian/jevonian/auto`; with your own `-m`, name a `jevonian/jevonian/…` model |
| Kilo / OpenCode can't find a file in your folder | `PWD` points elsewhere (see [known behaviour 7](#11-known-behaviours-and-limits)) | The commands set `PWD`. When running the real client, `set PWD=%CD%` first. |
| OpenCode: "Model unavailable: jevonian/jevonian/auto" | A background OpenCode service holds another folder's config | The commands add `--standalone`. By hand: `opencode run --standalone …`. |
| `opencode run` hangs with nothing in the ledger (scripts) | `run` reads stdin when it isn't a terminal | Add `< nul` |
| Qwen: "Persisted model.baseUrl no longer matches" | Qwen cached an older selection | Harmless; pick `jevonian/auto` once with `/model` |
| jev-gateway: "router on :8789 forwards to …, expected …" | A gateway started with other settings still owns the port | `jev stop jev-claude` (or `jev-claude --stop`), then run the command again |
| `EADDRINUSE` when starting a gateway or router | Something owns the port, e.g. a Jevonian router's port + 1 | `netstat -ano \| findstr ":8791"`; never put a gateway on a router's port + 1 |
| `http://127.0.0.1:8789/` shows "Not found" | By design: jev-gateway's page is `/dashboard` | Open `http://127.0.0.1:8789/dashboard?peers=none` |
| "No Jevonian API key exists yet" | You opened a router's port + 1 (tunnel surface) | Use the router's own port |
| A status window shows old information | It was opened by an older version of `monitor.js` | `jev windows close`, then `jev windows` |
| `npm install` fails with "resource busy" in `node_modules\jevonian` | A running router or a file watcher (IDE) holds the folder | `jev stop <name>` first; close the IDE's view of that folder |
| Tests look fine, but did the router really get it? | Reply text alone proves nothing | The router's page / status window must show a new row, or `jev test` reports it |

---

## 13. Upgrading and rolling back

### Jevonian (one router at a time; each has its own copy)

1. **Read what changed:** `npm view jevonian version`, then https://github.com/xinyao27/jevonian/releases. Look for changes to
   routing or effort (`withEffort`, `clientEffortOf`, `parseRoutingEntry`), the brain state (WAF patch), and Claude
   Code / Haiku handling. **If a release adds per-route effort, a client-effort override, or Haiku handling officially,
   delete that patch file instead of updating it.**
2. **Stop the router:** `jev stop kilo`.
3. **Install, exact version, in that folder only:**
   `cd /d D:\learn\JevAI\jevonian\jev-router-kilo && npm install jevonian@X.Y.Z --save-exact`.
4. **Start it:** `jev start kilo`, then `jev logs kilo`. Each patch prints `applied`.
   - **If one prints "pattern … found 0 times":** the router did not start and nothing was written. Find the new code shape with
     `findstr /n "withEffort brainState clientEffortOf parseRoutingEntry" node_modules\jevonian\dist\cli.mjs`, update that
     patch's search text, or roll back.
5. **Test:** `jev test tiers kilo` (every tier), `jev test clients kilo`, and for the Claude router `jev test tiers claude`,
   which also covers Haiku.
6. **Roll back:** `npm install jevonian@0.1.7 --save-exact` in that folder, then `jev restart kilo`.

### jev-gateway

```bat
jev stop jev-claude
jev stop jev-opencode
cd /d D:\learn\JevAI\jev-gateway && npm install jev-gateway@X.Y.Z --save-exact
jev test clients jev-claude
jev test clients jev-opencode
```

Nothing in it is patched. Check its CHANGELOG for renamed settings (`JEV_CLAUDE_UPSTREAM_BASE_URL`,
`JEV_OPENCODE_UPSTREAM_BASE_URL`, `JEV_OPENCODE_MODEL`, `JEV_PROVIDER`) or changed default ports, and for OpenCode v2 support.

### The clients

Updating Claude Code, OpenCode, Kilo or Qwen Code needs nothing here. Run `jev test clients` afterwards.

---

## 14. Moving, stopping and uninstalling

- **Stop everything:** `jev stop` (gateways first, then routers), then `jev windows close`.
- **Remove the CMD commands:** `jev uninstall`. It removes only its own AutoRun fragment, and a previous AutoRun value stays.
- **Move the setup:**
  1. `jev stop` and `jev uninstall`.
  2. Move both folders (keep them siblings).
  3. Run `node <new place>\jevonian\jev.js install`.
  4. Delete the old folder's `node_modules` (they are recreated) and run `npm install` in each package folder. Starting fresh is the clean way.
- **Uninstall completely:**
  1. `jev stop`, `jev windows close`, `jev uninstall`.
  2. Delete `D:\learn\JevAI\jevonian` and `D:\learn\JevAI\jev-gateway`.
  3. Optionally delete `%USERPROFILE%\.jev-gateway` (the gateway's logs).
  4. Nothing else was written on your PC.

### How this build was moved (2026-09-25)

This setup previously ran from `D:\learn\gemini-mcp\agy-opencode-jev` (folders `jevnonian`, `jev-gateway`, `cli`). The move:

1. **Stopped everything that setup had started:**
   - its six status windows
   - a Claude Code window opened for testing
   - both gateways (official `--stop`) and the four routers
   - two leftover OpenCode background servers from earlier tests (`opencode serve --port 4096` and `opencode serve --service`)
   - It checked that ports 8787 to 8800 and 4096 were free.
2. **Built the new folders:**
   - Copied the configs, patches and client configs.
   - Rewrote the scripts for the new layout: `lib\` instead of `cli\`, and one `jev.js` instead of two.
   - Copied the keys (step 2).
3. **Fresh `npm install`** in all five package folders, verified byte-identical to the official npm tarballs.
4. **`jev install`**, which replaced the old AutoRun entry. Then `jev start`, `jev test` and the browser checks.
5. **Kept each old ledger** as `jev-router-<tool>\data\ledger.before-move-2026-09-25.jsonl`. The web pages read only `ledger.jsonl`,
   so they start empty. Removed the old `jevnonian`, `jev-gateway` and `cli` folders.

---

## 15. Proof of working

Built and verified **2026-09-25** on Windows 11 Pro, Node v22.23.2, npm 10.9.8.

| Component | Version | Check |
|---|---|---|
| jevonian | 0.1.7 (npm latest) | 4 installs identical to the npm tarball before the first start; afterwards only `dist/cli.mjs` differs, by the patch edits (Appendix B) |
| jev-gateway | 0.4.3 (npm latest) | identical to the npm tarball, unmodified |
| Claude Code / OpenCode / Kilo / Qwen Code | 2.1.282 / 2.0.15 / 7.7.9 / 0.24.4 | all answered through their routers |

| Test | Result |
|---|---|
| First start after fresh installs | all six up in 5.7 s; every patch `applied` (WAF, Effort 5 edits, Haiku on Claude) |
| `jev test brains` | **8/8**: TypeSafe (`jev-latest`, ≈0.7 s) and the Vercel AI Gateway fallback (`typesafe-ai/jev`, ≈0.65 s) on all four routers |
| `jev test tiers` | **23/23**: every Alibaba tier (3 routers × 6) and every Claude tier (5) served by its configured model at its configured effort |
| `jev test clients` | **12/12**: `qwen`, `kilo`, `opencode`, `claude`, `jev-claude`, `jev-opencode` each answered a marker and read a file through their router; the gateways logged Jev's tool decision (`hint · Read` for Claude, `(read)` / `none` for OpenCode) |
| The three views | For every tool, the status window and the web page showed the same rows (section 9) |
| Restart | The `JEV-GATEWAY - CLAUDE` window followed a gateway restart and kept showing new requests (section 7) |
| Same folder | Both official packages installed and ran from one folder; one request went gateway → Jevonian → Alibaba (section 10) |
| Browser | All seven pages opened in one tab each and matched the status windows (section 8) |

---

## Appendix A: the command layer, every file in full

These are the exact files used in this build. Put them at the paths shown.

<details><summary><code>jevonian\jev.js</code> (the <code>jev</code> command: manage, test, install the CMD commands)</summary>

```js
#!/usr/bin/env node
// jev — manage this setup (D:\learn\JevAI\jevonian and its sibling ..\jev-gateway) and install the CMD commands.
//
//   node jev.js install        make these work in every NEW CMD window (doskey macros + CMD AutoRun; no .bat files):
//                                kilo, qwen, opencode, claude    -> Jevonian routers
//                                jev-claude, jev-opencode        -> official jev-gateway launchers -> Jevonian
//                                jev                             -> this script
//                                claude-direct, opencode-direct, kilo-direct, qwen-direct -> the real client, no router
//   jev uninstall              undo it (restores any AutoRun value you had before)
//   jev status                 every router/gateway: port, up/down, status window, web page
//   jev start   [name]         start routers + gateways (default: all six; a gateway also starts its Jevonian router)
//   jev stop    [name]         stop them (default: all six)
//   jev restart [name]
//   jev logs    <name>         the last 40 lines of that router's / gateway's log
//   jev windows [close]        open (or close) every status window
//   jev dashboards             open every web page in the browser
//   jev test [brains|tiers|clients] [name]   check everything against the configs (default: all three)
// name = qwen | kilo | claude | opencode | jev-claude | jev-opencode
//
// How the plain commands work without .bat/.cmd files: CMD "doskey" macros (jev.doskey, written by `install`),
// loaded by the per-user CMD AutoRun registry value HKCU\Software\Microsoft\Command Processor\AutoRun.
// They work at the interactive CMD prompt only (not in .bat scripts, PowerShell, or VS Code terminals running
// PowerShell). There, run the files directly: node D:\learn\JevAI\jevonian\kilo.js ...
"use strict";
const { spawnSync } = require("child_process");
const { existsSync, mkdirSync, readFileSync, writeFileSync } = require("fs");
const { tmpdir } = require("os");
const { join } = require("path");
const { GATEWAY, JEVONIAN, RUN_DIR, TARGETS, keyOf } = require("./lib/targets");
const common = require("./lib/common");
const { health, ensureUp, stopTarget, openDashboard, readConfig, statusWindowPid, openStatusWindow, closeStatusWindow, findExecutable, sleep } = common;

const MACROS = join(JEVONIAN, "jev.doskey");
const BACKUP = join(JEVONIAN, "autorun.backup.json");
const FRAGMENT = `if exist "${MACROS}" doskey /macrofile="${MACROS}"`;
const REG = "HKCU:\\Software\\Microsoft\\Command Processor";
const JEVONIAN_KEYS = Object.keys(TARGETS).filter((k) => TARGETS[k].kind === "jevonian");
const GATEWAY_KEYS = Object.keys(TARGETS).filter((k) => TARGETS[k].kind === "gateway");

// ---- install / uninstall: doskey macros + CMD AutoRun --------------------------------------------------------
function ps(script, env = {}) {
  const r = spawnSync("powershell.exe", ["-NoProfile", "-NonInteractive", "-Command", script], { encoding: "utf8", env: { ...process.env, ...env }, windowsHide: true });
  if (r.status !== 0) throw new Error((r.stderr || r.stdout || "powershell failed").trim());
  return (r.stdout || "").trim();
}
const readAutoRun = () => ps(`$v=(Get-ItemProperty -Path '${REG}' -Name AutoRun -ErrorAction SilentlyContinue).AutoRun; if ($null -ne $v) { [Console]::Out.Write($v) }`);
function writeAutoRun(value) {
  if (value) ps(`if (-not (Test-Path '${REG}')) { New-Item -Path '${REG}' | Out-Null }; Set-ItemProperty -Path '${REG}' -Name AutoRun -Value $env:JEV_AUTORUN`, { JEV_AUTORUN: value });
  else ps(`Remove-ItemProperty -Path '${REG}' -Name AutoRun -ErrorAction SilentlyContinue`);
}
const parts = (autorun) => autorun.split(" & ").filter((p) => p.trim());
// Windows paths are case-insensitive: a shell started in "d:\" must still match an AutoRun written from "D:\".
const isOurs = (part) => part.toLowerCase().includes(MACROS.toLowerCase());
const isAnyJevMacroFile = (part) => /doskey \/macrofile=".*\\jev\.doskey"/i.test(part);

function install() {
  const bin = (name) => findExecutable(name) ?? name;
  const n = (p) => `node "${p}" $*`;
  const macros = [
    `kilo=${n(join(JEVONIAN, "kilo.js"))}`,
    `qwen=${n(join(JEVONIAN, "qwen.js"))}`,
    `opencode=${n(join(JEVONIAN, "opencode.js"))}`,
    `claude=${n(join(JEVONIAN, "claude.js"))}`,
    `jev-claude=${n(join(GATEWAY, "jev-claude.js"))}`,
    `jev-opencode=${n(join(GATEWAY, "jev-opencode.js"))}`,
    `jev=${n(join(JEVONIAN, "jev.js"))}`,
    `claude-direct="${bin("claude")}" $*`,
    `opencode-direct="${bin("opencode")}" $*`,
    `kilo-direct="${bin("kilo")}" $*`,
    `qwen-direct="${bin("qwen")}" $*`,
  ];
  // doskey reads the macro file in the console's code page: keep it plain ASCII. It has no comment syntax
  // (a ";" line prints "Invalid macro definition." in every new CMD window), so no header line.
  writeFileSync(MACROS, macros.join("\r\n") + "\r\n", "latin1");

  const current = readAutoRun();
  // Drop fragments that load a jev.doskey from another folder (an older copy of this setup).
  const kept = parts(current).filter((p) => isOurs(p) || !isAnyJevMacroFile(p));
  const removed = parts(current).length - kept.length;
  if (kept.some(isOurs) && !removed) {
    console.log(`[jev] AutoRun already loads ${MACROS}`);
  } else {
    writeFileSync(BACKUP, JSON.stringify({ savedAt: new Date().toISOString(), previous: current || null }, null, 2));
    writeAutoRun([...kept.filter((p) => !isOurs(p)), FRAGMENT].join(" & "));
    console.log(`[jev] AutoRun set${removed ? ` (replaced ${removed} older jev.doskey entr${removed === 1 ? "y" : "ies"})` : ""}; previous value saved to ${BACKUP}`);
  }
  console.log(`[jev] macros written to ${MACROS}`);
  console.log("[jev] Open a NEW CMD window, then type:  kilo | qwen | opencode | claude   (Jevonian)");
  console.log("[jev]                                   jev-claude | jev-opencode     (jev-gateway -> Jevonian)   jev status");
}

function uninstall() {
  const kept = parts(readAutoRun()).filter((p) => !isOurs(p)).join(" & ");
  writeAutoRun(kept);
  console.log(`[jev] AutoRun ${kept ? `restored to: ${kept}` : "removed"} - new CMD windows no longer have the jev commands.`);
}

// ---- status / start / stop / logs --------------------------------------------------------------------------
async function status() {
  console.log("  name           command        port   state  status window   web page");
  for (const [key, t] of Object.entries(TARGETS)) {
    const up = await health(t);
    const win = statusWindowPid(key) ? "open" : "-";
    console.log(`  ${t.name.padEnd(13)}  ${t.command.padEnd(13)}  ${String(t.port).padEnd(5)}  ${(up ? "UP" : "down").padEnd(5)}  ${win.padEnd(14)}  ${t.dashboard}${t.kind === "jevonian" ? "  (+ /logs)" : ""}`);
  }
  console.log("\n  Upstreams: qwen, kilo, opencode -> Alibaba Token Plan; claude -> Anthropic (your Claude Code login)");
  console.log("             jev-claude -> Jevonian claude :8797; jev-opencode -> Jevonian opencode :8799");
  let installed = false;
  try {
    installed = parts(readAutoRun()).some(isOurs);
  } catch {}
  console.log(`  CMD commands: ${installed ? "installed (new CMD windows have them)" : `not installed - run: node "${join(JEVONIAN, "jev.js")}" install`}`);
}

function selectKeys(name, fallback) {
  if (!name || name === "all") return fallback;
  const key = keyOf(name);
  if (!key) {
    console.error(`[jev] unknown name "${name}". Use: ${Object.values(TARGETS).map((t) => t.name).join(" | ")}`);
    process.exit(1);
  }
  return [key];
}

async function start(name) {
  for (const key of selectKeys(name, [...JEVONIAN_KEYS, ...GATEWAY_KEYS])) {
    const t = TARGETS[key];
    const was = await health(t);
    const ok = was || (await ensureUp(t, { open: false }));
    console.log(`[jev] ${t.name.padEnd(12)} :${t.port}  ${was ? "already running" : ok ? "started" : `DID NOT START - see: jev logs ${t.name}`}`);
    if (!ok) process.exitCode = 1;
  }
}

async function stop(name) {
  // Gateways first: they forward to the Jevonian routers.
  for (const key of selectKeys(name, [...GATEWAY_KEYS, ...JEVONIAN_KEYS])) {
    const t = TARGETS[key];
    if (!(await health(t))) {
      console.log(`[jev] ${t.name.padEnd(12)} :${t.port}  was not running`);
      continue;
    }
    const stopped = await stopTarget(t);
    console.log(`[jev] ${t.name.padEnd(12)} :${t.port}  ${stopped ? "stopped" : "STILL ANSWERING"}`);
    if (!stopped) process.exitCode = 1;
  }
}

function logs(name) {
  const t = TARGETS[selectKeys(name, [])[0] ?? ""];
  if (!t) return console.log("usage: jev logs <name>");
  if (!existsSync(t.log)) return console.log(`[jev] no log yet: ${t.log}`);
  console.log(`== ${t.name} (${t.log}) ==`);
  console.log(readFileSync(t.log, "utf8").trim().split("\n").slice(-40).join("\n"));
}

// ---- test ---------------------------------------------------------------------------------------------------
function pad(s, n) {
  return String(s ?? "").padEnd(n);
}

function ledgerSince(target, sinceMs, { brain = false } = {}) {
  if (!existsSync(target.ledger)) return [];
  return readFileSync(target.ledger, "utf8")
    .split("\n")
    .map((l) => {
      try {
        return JSON.parse(l);
      } catch {
        return undefined;
      }
    })
    .filter((e) => e && (e.kind === "brain") === brain && Date.parse(e.ts) >= sinceMs);
}

async function gatewayEventsSince(target, sinceMs) {
  try {
    const r = await fetch(`http://127.0.0.1:${target.port}/dashboard/events?since=0`, { signal: AbortSignal.timeout(3000) });
    return (await r.json()).events.filter((e) => Date.parse(e.time) >= sinceMs);
  } catch {
    return [];
  }
}

/** Brains: Jevonian's own "Test channel" (POST /api/brain/test) for TypeSafe and the Vercel fallback. */
async function testBrains(keys, results) {
  console.log("\n== brains: POST /api/brain/test on each router (the dashboard's Test channel button) ==");
  for (const key of keys.filter((k) => TARGETS[k].kind === "jevonian")) {
    const t = TARGETS[key];
    for (const [channel, apiKeyEnv] of [["typesafe", "TYPESAFE_API_KEY"], ["vercel", "AI_GATEWAY_API_KEY"]]) {
      let j;
      try {
        const r = await fetch(`http://127.0.0.1:${t.port}/api/brain/test`, {
          method: "POST",
          headers: { "content-type": "application/json" },
          body: JSON.stringify({ channel, apiKeyEnv }),
          signal: AbortSignal.timeout(30000),
        });
        j = await r.json();
      } catch (e) {
        j = { ok: false, error: e.message };
      }
      const pass = j.ok === true;
      results.push({ test: "brain", target: t.name, item: channel, pass });
      console.log(`  ${pass ? "PASS" : "FAIL"}  ${pad(t.name, 9)} ${pad(channel, 9)} ${pass ? `${j.channel} / ${j.model}, ${j.latencyMs} ms` : j.error}`);
    }
  }
}

/** Tiers: every routing of every router, pinned (jevonian/<id>); the model and effort actually sent must match. */
async function testTiers(keys, results) {
  console.log("\n== tiers: each pinned tier must be served by its first model at its configured effort ==");
  console.log("  result  router    tier       expected                          got");
  for (const key of keys.filter((k) => TARGETS[k].kind === "jevonian")) {
    const t = TARGETS[key];
    const cfg = readConfig(t);
    for (const r of cfg.routing.routings) {
      const want = { model: r.models[0], effort: r.effort ?? "(default)" };
      let got;
      if (t.client !== "claude") {
        try {
          const res = await fetch(`${t.apiBase}/chat/completions`, {
            method: "POST",
            headers: { "content-type": "application/json", authorization: "Bearer local-no-key" },
            body: JSON.stringify({ model: `jevonian/${r.id}`, messages: [{ role: "user", content: "Reply with exactly: ok" }], max_tokens: 2048 }),
            signal: AbortSignal.timeout(120000),
          });
          await res.text();
          got = { model: res.headers.get("x-jevonian-model"), effort: res.headers.get("x-jevonian-effort") ?? "(none)", status: res.status };
        } catch (e) {
          got = { model: "-", effort: "-", status: e.message };
        }
      } else {
        // Claude's subscription login only serves Claude Code itself, so each tier is checked through the client
        // (Jevonian's official `launch claude`), then read back from the router's ledger.
        const since = Date.now();
        spawnSync(process.execPath, [join(t.dir, "node_modules", "jevonian", "dist", "cli.mjs"), "launch", "claude", "--model", `jevonian/${r.id}`, "--", "-p", "Reply with exactly: ok", "--output-format", "text"], {
          cwd: testDir(),
          input: "",
          encoding: "utf8",
          timeout: 300000,
          windowsHide: true,
          env: { ...process.env, JEVONIAN_CONFIG: t.config, JEVONIAN_CREDENTIALS: join(t.dir, "config", "credentials.json"), JEVONIAN_DATA_DIR: join(t.dir, "data") },
        });
        const e = ledgerSince(t, since).filter((x) => x.requestedModel === `jevonian/${r.id}`).pop();
        got = e ? { model: e.model, effort: e.effort ?? "(none)", status: e.status } : { model: "-", effort: "-", status: "no ledger entry" };
      }
      const pass = got.status === 200 && got.model === want.model && got.effort === want.effort;
      results.push({ test: "tier", target: t.name, item: r.id, pass, want, got });
      console.log(`  ${pass ? "PASS" : "FAIL"}    ${pad(t.name, 9)} ${pad(r.id, 9)}  ${pad(`${want.model} / ${want.effort}`, 32)}  ${got.model} / ${got.effort} (${got.status})`);
    }
  }
}

function testDir() {
  const dir = join(tmpdir(), "jev-test");
  mkdirSync(dir, { recursive: true });
  return dir;
}

/** Clients: every CMD command, one-shot, from a scratch folder: a marker reply, then a read-a-file task. */
async function testClients(keys, results) {
  const dir = testDir();
  const secret = `JEV-NOTES-${Math.random().toString(16).slice(2, 8).toUpperCase()}`;
  writeFileSync(join(dir, "notes.txt"), `${secret}\nThis file is only here for jev test.\n`);
  const outDir = join(RUN_DIR, "test-output");
  mkdirSync(outDir, { recursive: true });
  const cmds = {
    "jevonian-qwen": (p) => [join(JEVONIAN, "qwen.js"), "-p", p],
    "jevonian-kilo": (p) => [join(JEVONIAN, "kilo.js"), "run", p],
    "jevonian-opencode": (p) => [join(JEVONIAN, "opencode.js"), "run", p],
    "jevonian-claude": (p) => [join(JEVONIAN, "claude.js"), "-p", p, "--dangerously-skip-permissions"],
    "gateway-opencode": (p) => [join(GATEWAY, "jev-opencode.js"), "run", p],
    "gateway-claude": (p) => [join(GATEWAY, "jev-claude.js"), "-p", p, "--dangerously-skip-permissions"],
  };
  console.log(`\n== clients: each CMD command, one-shot, in ${dir} ==`);
  console.log("  result  command        prompt  answer                     router saw (phase model effort)             gateway saw (mode tool)");
  for (const key of keys) {
    const t = TARGETS[key];
    const router = t.kind === "gateway" ? TARGETS[t.upstreamKey] : t;
    const marker = `${t.command.replace(/-/g, "_").toUpperCase()}_OK`;
    for (const [label, prompt, expect] of [
      ["marker", `Reply with exactly: ${marker}`, marker],
      ["read", "Read the file notes.txt in the current folder and reply with only its first line.", secret],
    ]) {
      const since = Date.now();
      const r = spawnSync(process.execPath, cmds[key](prompt), {
        cwd: dir,
        input: "",
        encoding: "utf8",
        timeout: 300000,
        windowsHide: true,
        env: { ...process.env, JEV_NO_WINDOW: process.env.JEV_TEST_WINDOWS === "1" ? "" : "1", JEV_OPEN_DASHBOARD: "0" },
      });
      const secs = ((Date.now() - since) / 1000).toFixed(1);
      await sleep(1500); // the ledger line is written when the reply finishes
      const answer = (r.stdout || "").replace(/\x1b\[[0-9;]*m/g, "").trim();
      writeFileSync(join(outDir, `${t.command}-${label}.txt`), `$ ${t.command} ${JSON.stringify(prompt)}\n--- stdout ---\n${r.stdout ?? ""}\n--- stderr ---\n${r.stderr ?? ""}\n--- exit ${r.status} after ${secs}s ---\n`);
      const seen = ledgerSince(router, since);
      // A one-shot client can exit before Jevonian writes the request line (it is written when the stream
      // ends); the Jev brain calls for that request are still logged, which proves the router routed it.
      const brainCalls = seen.length ? [] : ledgerSince(router, since, { brain: true });
      const lastSeen = seen.map((e) => `${e.phase}:${e.model}:${e.effort ?? "-"}`).join(", ") || (brainCalls.length ? `(routed: ${brainCalls.length} brain calls, no request line)` : "(nothing)");
      let gw = "";
      if (t.kind === "gateway") {
        const ev = await gatewayEventsSince(t, since);
        gw = ev.map((e) => `${e.mode}:${e.tool ?? (e.jev?.choice ? `(${e.jev.choice})` : "-")}`).join(", ") || "(nothing)";
      }
      const pass = r.status === 0 && answer.includes(expect) && seen.length + brainCalls.length > 0 && (t.kind !== "gateway" || gw !== "(nothing)");
      results.push({ test: "client", target: t.command, item: label, pass, secs, answer: answer.split("\n").pop()?.slice(0, 60), router: lastSeen, gateway: gw });
      console.log(`  ${pass ? "PASS" : "FAIL"}    ${pad(t.command, 13)}  ${pad(label, 6)}  ${pad(answer.split("\n").pop()?.slice(0, 26), 26)} ${pad(lastSeen, 43)} ${gw}`);
    }
  }
  console.log(`  (full client output: ${outDir})`);
}

async function test(what, name) {
  const keys = selectKeys(name, [...JEVONIAN_KEYS, ...GATEWAY_KEYS]);
  for (const key of new Set(keys.flatMap((k) => (TARGETS[k].kind === "gateway" ? [TARGETS[k].upstreamKey, k] : [k])))) {
    if (!(await health(TARGETS[key]))) {
      console.error(`[jev] ${TARGETS[key].name} is not running - start it first: jev start`);
      process.exit(1);
    }
  }
  const results = [];
  const all = !what || what === "all";
  if (all || what === "brains") await testBrains(keys, results);
  if (all || what === "tiers") await testTiers(keys, results);
  if (all || what === "clients") await testClients(keys, results);
  const passed = results.filter((r) => r.pass).length;
  console.log(`\n== ${passed}/${results.length} passed ==`);
  mkdirSync(RUN_DIR, { recursive: true });
  writeFileSync(join(RUN_DIR, "test-results.json"), JSON.stringify({ at: new Date().toISOString(), results }, null, 2));
  if (passed !== results.length) process.exitCode = 1;
}

// ---- main ---------------------------------------------------------------------------------------------------
(async () => {
  const [cmd, arg, arg2] = process.argv.slice(2);
  if (cmd === "install") return install();
  if (cmd === "uninstall") return uninstall();
  if (cmd === "status" || !cmd) return status();
  if (cmd === "start") return start(arg);
  if (cmd === "stop") return stop(arg);
  if (cmd === "restart") return (await stop(arg), start(arg));
  if (cmd === "logs") return logs(arg);
  if (cmd === "test") return ["brains", "tiers", "clients", "all"].includes(arg) ? test(arg, arg2) : test(undefined, arg);
  if (cmd === "windows") {
    for (const key of Object.keys(TARGETS)) console.log(`${TARGETS[key].title}: ${arg === "close" ? (closeStatusWindow(key) ? "closed" : "was not open") : openStatusWindow(key)}`);
    return;
  }
  if (cmd === "dashboards") {
    for (const t of Object.values(TARGETS)) console.log(`${t.name.padEnd(13)} ${t.dashboard}${openDashboard(t) ? "  (opened)" : ""}`);
    return;
  }
  console.log("usage: jev install|uninstall|status|start|stop|restart|logs|windows|dashboards|test [brains|tiers|clients] [name]");
  process.exit(1);
})();
```

</details>

<details><summary><code>jevonian\kilo.js</code></summary>

```js
#!/usr/bin/env node
// `kilo` in CMD (doskey macro, see `jev install`) -> Kilo through the Jevonian router :8795.
//   kilo                              interactive, in the folder you are in
//   kilo run "fix the test"           one-shot
//   kilo -m jevonian/jevonian/plan    pin a tier
// The router's client config (jev-router-kilo\.kilo\kilo.json) is injected with KILO_CONFIG_CONTENT, so
// nothing in your own Kilo config is changed. Kilo ignores a config's default `model` for its own cloud
// default (minimax/..., "Add credits"), so `-m jevonian/jevonian/auto` is added unless you pass your own -m.
"use strict";
const { readFileSync } = require("fs");
const { join } = require("path");
const { launch, withModel, hasModelFlag } = require("./lib/launch");

launch({
  key: "jevonian-kilo",
  exe: "kilo",
  wire: (argv) => ({
    args: withModel(argv, "jevonian/jevonian/auto"),
    // PWD: like OpenCode, Kilo takes its project folder from PWD when it is set (Git Bash and some tools set it),
    // so it is pinned to the folder you are really in.
    env: { KILO_CONFIG_CONTENT: readFileSync(join(__dirname, "jev-router-kilo", ".kilo", "kilo.json"), "utf8"), PWD: process.cwd() },
    how: `KILO_CONFIG_CONTENT = jev-router-kilo\\.kilo\\kilo.json, PWD, -m ${hasModelFlag(argv) ? "(yours)" : "jevonian/jevonian/auto"}`,
  }),
});
```

</details>

<details><summary><code>jevonian\qwen.js</code></summary>

```js
#!/usr/bin/env node
// `qwen` in CMD (doskey macro, see `jev install`) -> Qwen Code through the Jevonian router :8793.
//   qwen                     interactive, in the folder you are in
//   qwen -p "fix the test"   one-shot
//   qwen -m jevonian/plan    pin a tier
// Qwen reads .qwen\settings.json only from the folder it starts in, so the router is passed on the command
// line (nothing in your own Qwen settings is changed). The API key is a placeholder: Jevonian answers only
// on 127.0.0.1 and no Jevonian key exists. For a /model picker listing all 7 Jev models, run qwen-direct
// inside jev-router-qwen\ (it reads jev-router-qwen\.qwen\settings.json).
"use strict";
const { launch, hasModelFlag } = require("./lib/launch");

launch({
  key: "jevonian-qwen",
  exe: "qwen",
  wire: (argv) => ({
    args: ["--auth-type", "openai", "--openai-base-url", "http://127.0.0.1:8793/v1", "--openai-api-key", "local-no-key", ...(hasModelFlag(argv) ? [] : ["-m", "jevonian/auto"]), ...argv],
    env: {},
    how: `--auth-type openai --openai-base-url http://127.0.0.1:8793/v1 -m ${hasModelFlag(argv) ? "(yours)" : "jevonian/auto"}`,
  }),
});
```

</details>

<details><summary><code>jevonian\opencode.js</code></summary>

```js
#!/usr/bin/env node
// `opencode` in CMD (doskey macro, see `jev install`) -> OpenCode through the Jevonian router :8799.
//   opencode                          interactive, in the folder you are in
//   opencode run "fix the test"       one-shot (in scripts add  < nul  — `run` reads a piped stdin)
//   opencode -m jevonian/jevonian/plan   pin a tier
// The router's client config (jev-router-opencode\opencode.json) is injected with OPENCODE_CONFIG_CONTENT, so
// ~/.config/opencode is never changed. OpenCode 2.x quirks handled here: --standalone (a background service
// started elsewhere may hold another config -> "Model unavailable"), PWD (this build resolves its config
// against it), and -m (default jevonian/jevonian/auto).
"use strict";
const { readFileSync } = require("fs");
const { join } = require("path");
const { launch, withModel, withStandalone, hasModelFlag } = require("./lib/launch");

launch({
  key: "jevonian-opencode",
  exe: "opencode",
  wire: (argv) => ({
    args: withStandalone(withModel(argv, "jevonian/jevonian/auto")),
    env: { OPENCODE_CONFIG_CONTENT: readFileSync(join(__dirname, "jev-router-opencode", "opencode.json"), "utf8"), PWD: process.cwd() },
    how: `OPENCODE_CONFIG_CONTENT = jev-router-opencode\\opencode.json, --standalone, PWD, -m ${hasModelFlag(argv) ? "(yours)" : "jevonian/jevonian/auto"}`,
  }),
});
```

</details>

<details><summary><code>jevonian\claude.js</code></summary>

```js
#!/usr/bin/env node
// `claude` in CMD (doskey macro, see `jev install`) -> Claude Code through the Jevonian router :8797, using
// Jevonian's OWN launcher:  jevonian launch claude [--model M] -- <claude args>
//   claude --dangerously-skip-permissions
//   claude -p "fix the test" --dangerously-skip-permissions
//   claude --model jevonian/plan            pin a tier (jevonian/plan|heavy|execute|utility|chat)
// The official launcher sets ANTHROPIC_BASE_URL / ANTHROPIC_AUTH_TOKEN and remaps Opus/Sonnet/Haiku onto
// jevonian/* for this one session (like `ollama launch claude`); ~/.claude/settings.json is never touched.
// Tier + effort are decided by the router (jev-router-claude\config\config.json, "forceEffort": true):
//   plan Opus 5.5 xhigh | heavy Opus 5.5 high | execute Sonnet 5 high | utility Sonnet 5 medium | chat Haiku 4.5 low
"use strict";
const { join } = require("path");
const { launch } = require("./lib/launch");

const R = join(__dirname, "jev-router-claude");

/** Lift --model / -m out of the user's args: `launch claude` takes it before the `--`. */
function splitModel(argv) {
  const rest = [];
  let model;
  for (let i = 0; i < argv.length; i++) {
    const a = argv[i];
    if (a === "--model" || a === "-m") {
      model = argv[++i];
      continue;
    }
    if (a.startsWith("--model=")) {
      model = a.slice(8);
      continue;
    }
    rest.push(a);
  }
  return { model, rest };
}

launch({
  key: "jevonian-claude",
  exe: process.execPath,
  display: "jevonian launch claude (official) -> claude.exe",
  wire: (argv) => {
    const { model, rest } = splitModel(argv);
    return {
      // Everything for Claude Code goes after `--`: `launch claude` drops flags placed before it.
      args: [join(R, "node_modules", "jevonian", "dist", "cli.mjs"), "launch", "claude", ...(model ? ["--model", model] : []), "--", ...rest],
      env: {
        JEVONIAN_CONFIG: join(R, "config", "config.json"),
        JEVONIAN_CREDENTIALS: join(R, "config", "credentials.json"),
        JEVONIAN_DATA_DIR: join(R, "data"),
      },
      how: `jevonian launch claude --model ${model ?? "jevonian/auto"} (tier + effort decided by the router)`,
    };
  },
});
```

</details>

<details><summary><code>jevonian\lib\targets.js</code></summary>

```js
// Every server this setup runs, in one place. Used by the CMD commands, the status windows (monitor.js)
// and `jev`. Folders are found relative to this file:
//   D:\learn\JevAI\jevonian      this folder (4 Jevonian routers)
//   D:\learn\JevAI\jev-gateway   its sibling (official jev-gateway); override with JEV_GATEWAY_DIR
//
//   Jevonian (official xinyao27/jevonian 0.1.7, one local install per router; each also binds port+1):
//     qwen :8793   kilo :8795   claude :8797   opencode :8799
//   jev-gateway (official vinilana/jev-gateway 0.4.3, unmodified, its default ports):
//     jev-claude :8789 -> Jevonian :8797 -> Anthropic      jev-opencode :8791 -> Jevonian :8799 -> Alibaba
"use strict";
const { homedir } = require("os");
const { join, resolve } = require("path");

const JEVONIAN = resolve(__dirname, "..");
const GATEWAY = process.env.JEV_GATEWAY_DIR ? resolve(process.env.JEV_GATEWAY_DIR) : resolve(JEVONIAN, "..", "jev-gateway");
const GATEWAY_BIN = join(GATEWAY, "node_modules", "jev-gateway", "bin");
const RUN_DIR = join(JEVONIAN, "run");
const ALIBABA = "Alibaba Cloud Model Studio - Token Plan";
const ALIBABA_URL = "https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1";
const ANTHROPIC = "Anthropic API (api.anthropic.com) - your Claude Code login (OAuth), no API key";

const jevonian = (client, port, provider) => ({
  kind: "jevonian",
  name: client, // `jev start qwen`
  command: client, // the CMD command
  client,
  port,
  dir: join(JEVONIAN, `jev-router-${client}`),
  config: join(JEVONIAN, `jev-router-${client}`, "config", "config.json"),
  health: `http://127.0.0.1:${port}/healthz`,
  apiBase: client === "claude" ? `http://127.0.0.1:${port}` : `http://127.0.0.1:${port}/v1`,
  dashboard: `http://127.0.0.1:${port}/`,
  logs: `http://127.0.0.1:${port}/logs`,
  logsLabel: "Logs",
  provider,
  brain: "TypeSafe Jev (fallback: Vercel AI Gateway) picks the tier each turn",
  ledger: join(JEVONIAN, `jev-router-${client}`, "data", "ledger.jsonl"),
  log: join(JEVONIAN, `jev-router-${client}`, "logs", "serve.log"),
  title: `JEVONIAN - ${client.toUpperCase()} - :${port}`,
});

const gateway = (client, port, provider) => ({
  kind: "gateway",
  name: `jev-${client}`,
  command: `jev-${client}`,
  client,
  port,
  health: `http://127.0.0.1:${port}/health`,
  apiBase: client === "claude" ? `http://127.0.0.1:${port}` : `http://127.0.0.1:${port}/v1`,
  // The official gateway serves its page only at /dashboard (/ is 404). By default that page also shows the
  // other gateways it finds; ?peers=none (official) keeps it to this tool's gateway.
  dashboard: `http://127.0.0.1:${port}/dashboard?peers=none`,
  logs: `http://127.0.0.1:${port}/dashboard`,
  logsLabel: "Both gateways",
  provider,
  brain: "TypeSafe Jev picks the next TOOL (official jev-gateway); the model is not changed",
  upstreamKey: `jevonian-${client}`, // the Jevonian router this gateway forwards to
  bin: join(GATEWAY_BIN, `jev-${client}.mjs`),
  log: join(homedir(), ".jev-gateway", `${client}.log`), // official location
  title: `JEV-GATEWAY - ${client.toUpperCase()} - :${port}`,
});

const TARGETS = {
  "jevonian-qwen": jevonian("qwen", 8793, `${ALIBABA}  (${ALIBABA_URL})`),
  "jevonian-kilo": jevonian("kilo", 8795, `${ALIBABA}  (${ALIBABA_URL})`),
  "jevonian-claude": jevonian("claude", 8797, ANTHROPIC),
  "jevonian-opencode": jevonian("opencode", 8799, `${ALIBABA}  (${ALIBABA_URL})`),
  "gateway-claude": gateway("claude", 8789, `Jevonian Claude router :8797 (tier + effort) -> ${ANTHROPIC}`),
  "gateway-opencode": gateway("opencode", 8791, `Jevonian OpenCode router :8799 (tier + effort) -> ${ALIBABA}`),
};

/** "kilo" | "jev-claude" | "jevonian-kilo" | "gateway-claude" -> the target key, or undefined. */
function keyOf(name) {
  if (TARGETS[name]) return name;
  return Object.keys(TARGETS).find((k) => TARGETS[k].name === name);
}

module.exports = { JEVONIAN, GATEWAY, GATEWAY_BIN, RUN_DIR, ALIBABA_URL, TARGETS, keyOf };
```

</details>

<details><summary><code>jevonian\lib\common.js</code></summary>

```js
// Shared helpers: health checks, starting/stopping routers and gateways, tier tables, the status window,
// and running a client binary.
"use strict";
const { spawn, spawnSync } = require("child_process");
const { closeSync, existsSync, mkdirSync, openSync, readFileSync, rmSync, writeFileSync } = require("fs");
const { extname, join } = require("path");
const { GATEWAY, RUN_DIR, TARGETS } = require("./targets");

const MONITOR = join(__dirname, "monitor.js");
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function health(target) {
  try {
    const r = await fetch(target.health, { signal: AbortSignal.timeout(1500) });
    return r.ok ? await r.json() : undefined;
  } catch {
    return undefined;
  }
}

/** Environment for the official jev-gateway launchers (defined in ..\jev-gateway\gateway-env.js). */
function gatewayEnv(client) {
  return require(join(GATEWAY, "gateway-env.js")).gatewayEnv(client);
}

/** The pid listening on 127.0.0.1:<port>, from `netstat -ano` (Windows). */
function pidOnPort(port) {
  const r = spawnSync("netstat", ["-ano", "-p", "tcp"], { encoding: "utf8", windowsHide: true });
  for (const line of (r.stdout || "").split(/\r?\n/)) {
    const m = line.trim().split(/\s+/);
    if (m.length >= 5 && m[3] === "LISTENING" && m[1].endsWith(`:${port}`)) return Number(m[4]);
  }
  return undefined;
}

function killTree(pid) {
  if (!Number.isInteger(pid) || pid <= 0) return false;
  return spawnSync("taskkill", ["/PID", String(pid), "/T", "/F"], { stdio: "ignore", windowsHide: true }).status === 0;
}

/** Start a Jevonian router in the background: node <router>\start.js, output appended to <router>\logs\serve.log. */
function startJevonian(target) {
  mkdirSync(join(target.dir, "logs"), { recursive: true });
  const log = openSync(target.log, "a");
  const child = spawn(process.execPath, [join(target.dir, "start.js")], {
    cwd: target.dir,
    detached: true,
    windowsHide: true,
    stdio: ["ignore", log, log],
  });
  closeSync(log);
  child.unref();
  writeFileSync(join(target.dir, "logs", "router.pid"), String(child.pid));
}

/** Start a jev-gateway with its OFFICIAL launcher (`jev-<client>.mjs --start`) and the documented env. */
function startGateway(target) {
  spawnSync(process.execPath, [target.bin, "--start"], {
    stdio: ["ignore", "ignore", "inherit"],
    env: { ...process.env, ...gatewayEnv(target.client) },
    windowsHide: true,
  });
}

/**
 * Make sure a router/gateway answers; start it with node if it does not. A gateway first needs the
 * Jevonian router it forwards to. When this call had to START it, the dashboard is opened once in the
 * default browser (JEV_OPEN_DASHBOARD=0 turns that off). Resolves true when it answers.
 */
async function ensureUp(target, { open = true } = {}) {
  if (await health(target)) return true;
  if (target.kind === "gateway" && !(await ensureUp(TARGETS[target.upstreamKey], { open }))) return false;
  if (target.kind === "jevonian") startJevonian(target);
  else startGateway(target);
  for (let i = 0; i < 80; i++) {
    if (await health(target)) {
      if (open) openDashboard(target);
      return true;
    }
    await sleep(250);
  }
  return false;
}

/** Stop a router/gateway. Jevonian: the start.js process tree, then whatever still owns the port. */
async function stopTarget(target) {
  if (target.kind === "gateway") {
    spawnSync(process.execPath, [target.bin, "--stop"], { stdio: "inherit", env: { ...process.env, ...gatewayEnv(target.client) }, windowsHide: true });
    return !(await health(target));
  }
  const pf = join(target.dir, "logs", "router.pid");
  if (existsSync(pf)) killTree(Number(readFileSync(pf, "utf8")));
  rmSync(pf, { force: true });
  for (let i = 0; i < 20 && (await health(target)); i++) await sleep(250);
  if (await health(target)) killTree(pidOnPort(target.port)); // e.g. started some other way
  for (let i = 0; i < 20 && (await health(target)); i++) await sleep(250);
  return !(await health(target));
}

/**
 * Open a dashboard in the default browser. Called when a command had to START that router/gateway
 * (so once per start, not on every command), and by `jev dashboards`. JEV_OPEN_DASHBOARD=0 turns it off.
 */
function openDashboard(target) {
  if (process.env.JEV_OPEN_DASHBOARD === "0" || process.platform !== "win32") return false;
  spawn(process.env.ComSpec || "cmd.exe", ["/d", "/s", "/c", `"start "" "${target.dashboard}""`], { detached: true, stdio: "ignore", windowsVerbatimArguments: true }).unref();
  console.error(`[jev] started ${target.title.split(" - ").slice(0, 2).join(" ")} -> opened its web page ${target.dashboard}`);
  return true;
}

/** Jevonian config.json of a router, or undefined. */
function readConfig(target) {
  try {
    return JSON.parse(readFileSync(target.config, "utf8"));
  } catch {
    return undefined;
  }
}

/**
 * [{ tier, models, effort }] — Jevonian: from its config.json. jev-gateway has no model tiers (Jev picks the
 * tool); both gateways here send jevonian/auto to their Jevonian router, so the tiers shown are that router's.
 */
function tiersOf(target) {
  if (target.kind === "gateway") {
    const upstream = TARGETS[target.upstreamKey];
    const downstream = tiersOf(upstream).map((t) => ({ ...t, tier: "  via " + t.tier }));
    const head = target.client === "claude" ? "jevonian/auto" : "jev-gateway/jevonian/auto";
    return [{ tier: head, models: `-> Jevonian :${upstream.port}, which picks:`, effort: "" }, ...downstream];
  }
  const cfg = readConfig(target);
  if (!cfg) return [];
  return cfg.routing.routings.map((r) => ({
    tier: `jevonian/${r.id}`,
    label: r.label,
    models: r.models.join(", "),
    effort: `${r.effort ?? "(default)"}${r.forceEffort ? " (forced)" : ""}`,
  }));
}

const pidFile = (key) => join(RUN_DIR, `${key}.monitor.pid`);
const monitorLog = (key) => join(RUN_DIR, `${key}.monitor.log`);

function alive(pid) {
  try {
    process.kill(pid, 0);
    return true;
  } catch {
    return false;
  }
}

/** pid of the status window's monitor process, if that window is open. */
function statusWindowPid(key) {
  try {
    const pid = Number(readFileSync(pidFile(key), "utf8"));
    return Number.isInteger(pid) && pid > 0 && alive(pid) ? pid : undefined;
  } catch {
    return undefined;
  }
}

/**
 * One status window per router: a new console window titled e.g. "JEVONIAN - KILO - :8795" running
 * monitor.js (header + live request feed). Reused while it is open; re-opened after it is closed.
 * JEV_NO_WINDOW=1 skips it (scripts).
 */
function openStatusWindow(key) {
  if (process.env.JEV_NO_WINDOW === "1" || process.platform !== "win32") return "skipped";
  if (statusWindowPid(key)) return "reused";
  mkdirSync(RUN_DIR, { recursive: true });
  const target = TARGETS[key];
  const line = `"start "${target.title}" cmd /k node "${MONITOR}" ${key}"`;
  spawn(process.env.ComSpec || "cmd.exe", ["/d", "/s", "/c", line], {
    detached: true,
    stdio: "ignore",
    windowsVerbatimArguments: true,
  }).unref();
  return "opened";
}

/** Close a status window (its monitor process and the console window around it). */
function closeStatusWindow(key) {
  const pid = statusWindowPid(key);
  if (!pid) return false;
  const parent = spawnSync("powershell.exe", ["-NoProfile", "-NonInteractive", "-Command", `(Get-CimInstance Win32_Process -Filter "ProcessId=${pid}").ParentProcessId`], { encoding: "utf8", windowsHide: true });
  const cmdPid = Number((parent.stdout || "").trim());
  killTree(Number.isInteger(cmdPid) && cmdPid > 0 ? cmdPid : pid);
  rmSync(pidFile(key), { force: true });
  return true;
}

/** First real executable for `name` on PATH, preferring .exe over .cmd (never a bare sh script). */
function findExecutable(name) {
  const probe = spawnSync("where", [name], { encoding: "utf8", windowsHide: true });
  const hits = (probe.stdout || "").split(/\r?\n/).map((s) => s.trim()).filter(Boolean);
  return hits.find((p) => extname(p).toLowerCase() === ".exe") ?? hits.find((p) => extname(p).toLowerCase() === ".cmd");
}

function cmdQuote(arg) {
  if (arg === "") return '""';
  if (!/[\s"&|<>^()%!]/.test(arg)) return arg;
  return `"${arg.replace(/"/g, '""')}"`;
}

/** Run a client binary in this console and folder; resolves with its exit code. */
function runClient(exe, args, env) {
  const isCmd = extname(exe).toLowerCase() === ".cmd";
  const child = isCmd
    ? spawn(process.env.ComSpec || "cmd.exe", ["/d", "/s", "/c", `"${[cmdQuote(exe), ...args.map(cmdQuote)].join(" ")}"`], {
        stdio: "inherit",
        env: { ...process.env, ...env },
        windowsVerbatimArguments: true,
      })
    : spawn(exe, args, { stdio: "inherit", env: { ...process.env, ...env } });
  // Ctrl+C belongs to the client (same console); the launcher just waits for it to exit.
  process.on("SIGINT", () => {});
  return new Promise((resolve) => {
    child.on("error", (e) => {
      console.error(`[jev] could not run ${exe}: ${e.message}`);
      resolve(127);
    });
    child.on("exit", (code, signal) => resolve(signal ? 1 : code ?? 0));
  });
}

module.exports = {
  health,
  gatewayEnv,
  pidOnPort,
  ensureUp,
  stopTarget,
  openDashboard,
  readConfig,
  tiersOf,
  pidFile,
  monitorLog,
  statusWindowPid,
  openStatusWindow,
  closeStatusWindow,
  findExecutable,
  runClient,
  sleep,
};
```

</details>

<details><summary><code>jevonian\lib\launch.js</code></summary>

```js
// The one launcher behind every CMD command (kilo, qwen, opencode, claude, jev-claude, jev-opencode):
//   1. start the router/gateway with node if it is down (a gateway also starts the Jevonian router behind it)
//   2. open (or reuse) its status window: port, URL, provider, tiers + effort, live request feed
//   3. print the same facts here, then run the real client in this window and folder
"use strict";
const { isAbsolute } = require("path");
const { TARGETS } = require("./targets");
const { ensureUp, health, tiersOf, openStatusWindow, findExecutable, runClient } = require("./common");

/**
 * @param {object} spec
 * @param {string} spec.key   key in TARGETS, e.g. "jevonian-kilo"
 * @param {string} spec.exe   client binary name on PATH (e.g. "kilo"), or an absolute path
 * @param {string} [spec.display]  what to call the client in the banner (defaults to the binary)
 * @param {(argv: string[]) => { args: string[], env: Record<string,string>, how: string }} spec.wire
 */
async function launch(spec) {
  const target = TARGETS[spec.key];
  if (!(await ensureUp(target))) {
    console.error(`[jev] ${target.name} did not start on :${target.port}. Check: jev logs ${target.name}`);
    process.exit(1);
  }
  const up = await health(target);
  const windowState = openStatusWindow(spec.key);

  const exe = isAbsolute(spec.exe) ? spec.exe : findExecutable(spec.exe);
  if (!exe) {
    console.error(`[jev] ${spec.exe} is not on PATH.`);
    process.exit(127);
  }
  const { args, env, how } = spec.wire(process.argv.slice(2));

  const line = "=".repeat(78);
  const out = (s = "") => console.error(s);
  out(line);
  out(`  ${target.title}   (${target.kind === "jevonian" ? "Jevonian router" : "jev-gateway"})`);
  out(line);
  out(`  Port        : ${target.port}${target.kind === "jevonian" ? `  (+${target.port + 1} tunnel surface)` : ""}   [${up ? "ONLINE" : "?"}]`);
  out(`  URL         : ${target.apiBase}`);
  out(`  Web page    : ${target.dashboard}   ${target.logsLabel}: ${target.logs}`);
  out(`  Provider    : ${target.provider}`);
  out(`  Jev brain   : ${target.brain}`);
  out(`  Tiers       :`);
  for (const t of tiersOf(target)) out(`      ${t.tier.padEnd(22)} ${t.models.padEnd(28)} ${t.effort ? "effort " + t.effort : ""}`);
  out(`  Client      : ${spec.display ?? exe}   in ${process.cwd()}`);
  out(`  Wiring      : ${how}`);
  out(`  Status win  : ${windowState === "opened" ? `opened  "${target.title}"` : windowState === "reused" ? `already open  "${target.title}"` : "off (JEV_NO_WINDOW=1)"}`);
  out(line);

  process.exit(await runClient(exe, args, env));
}

const hasModelFlag = (argv) => argv.some((a) => a === "-m" || a === "--model" || a.startsWith("--model="));

/** Insert `-m <model>` after a leading subcommand (e.g. `run`), or first for the TUI, unless -m was given. */
function withModel(argv, model, subcommands = ["run"]) {
  if (hasModelFlag(argv)) return argv;
  return subcommands.includes(argv[0]) ? [argv[0], "-m", model, ...argv.slice(1)] : ["-m", model, ...argv];
}

/** OpenCode 2.x: --standalone (skip a background service started elsewhere with another config). */
function withStandalone(argv, subcommands = ["run"]) {
  if (argv.includes("--standalone")) return argv;
  return subcommands.includes(argv[0]) ? [argv[0], "--standalone", ...argv.slice(1)] : ["--standalone", ...argv];
}

module.exports = { launch, withModel, withStandalone, hasModelFlag };
```

</details>

<details><summary><code>jevonian\lib\monitor.js</code> (the status window)</summary>

```js
#!/usr/bin/env node
// The status window each CMD command opens:  node lib\monitor.js <target-key>
// Header: port, URL, web page, provider, Jev brain, tiers + effort. Then one line per request, live:
//   Jevonian     tails <router>\data\ledger.jsonl       (the same records as the web page's Logs)
//   jev-gateway  polls http://127.0.0.1:<port>/dashboard/events   (the same data as its dashboard)
// The header is redrawn when the router restarts or its config.json changes, so it never shows stale tiers,
// and a restarted gateway is followed from its new start (its request numbers start over).
// Every line is also written to run\<target>.monitor.log (plain text), so what the window showed can be
// checked later. Closing this window does NOT stop the router.
"use strict";
const fs = require("fs");
const { RUN_DIR, TARGETS } = require("./targets");
const { health, tiersOf, pidFile, monitorLog } = require("./common");

const key = process.argv[2];
const target = TARGETS[key];
if (!target) {
  console.error(`usage: node monitor.js <${Object.keys(TARGETS).join("|")}>`);
  process.exit(1);
}

// JEV_MONITOR_PREVIEW=1: print to this console only; leave the real window's pid and log files alone.
const PREVIEW = Boolean(process.env.JEV_MONITOR_PREVIEW);
const LOG = monitorLog(key);
if (!PREVIEW) {
  fs.mkdirSync(RUN_DIR, { recursive: true });
  fs.writeFileSync(pidFile(key), String(process.pid));
  fs.writeFileSync(LOG, "");
  process.on("exit", () => fs.rmSync(pidFile(key), { force: true }));
}
for (const sig of ["SIGINT", "SIGTERM", "SIGHUP"]) process.on(sig, () => process.exit(0));

const C = { dim: "\x1b[2m", b: "\x1b[1m", g: "\x1b[32m", r: "\x1b[31m", y: "\x1b[33m", c: "\x1b[36m", x: "\x1b[0m" };
const ANSI = /\x1b\[[0-9;]*m/g;
function out(text = "") {
  console.log(text);
  if (!PREVIEW) {
    try {
      fs.appendFileSync(LOG, text.replace(ANSI, "") + "\n");
    } catch {}
  }
}
const clock = (iso) => (iso ? new Date(iso).toLocaleTimeString("en-GB") : "--:--:--");
const now = () => clock(new Date().toISOString());
const isJevonian = target.kind === "jevonian";

async function header(note) {
  const up = await health(target);
  const line = "=".repeat(78);
  if (note) out(`${C.y}  ---- ${now()}  ${note}: header refreshed ----${C.x}`);
  out(C.c + line + C.x);
  out(`${C.b}  ${target.title}${C.x}   ${isJevonian ? "Jevonian router" : "jev-gateway"} for ${target.client}`);
  out(C.c + line + C.x);
  out(`  Status      : ${up ? `${C.g}ONLINE${C.x}` : `${C.r}DOWN${C.x}`}   ${up ? JSON.stringify(up).slice(0, 100) : ""}`);
  out(`  Port        : ${C.b}${target.port}${C.x}${isJevonian ? `   (Jevonian also binds ${target.port + 1} for its tunnel surface)` : ""}`);
  out(`  URL         : ${C.b}${target.apiBase}${C.x}`);
  out(`  Web page    : ${target.dashboard}    ${target.logsLabel}: ${target.logs}`);
  out(`  Provider    : ${target.provider}`);
  out(`  Jev brain   : ${target.brain}`);
  out(`  Tiers (model, effort):`);
  for (const t of tiersOf(target)) out(`      ${t.tier.padEnd(22)} ${t.models.padEnd(28)} ${t.effort}`);
  out(`  ${isJevonian ? `Ledger      : ${target.ledger}` : `Log         : ${target.log}`}`);
  if (!PREVIEW) out(`  Window log  : ${LOG}`);
  out(C.c + line + C.x);
  out(`${C.dim}  Live requests (this window only watches; closing it does not stop the router)${C.x}`);
  out(
    isJevonian
      ? `${C.dim}  time      phase      model                       effort  status  cost      latency  requested (brain)${C.x}`
      : `${C.dim}  time      mode         model                 tools  status  tool / reason                conf   in / out${C.x}`,
  );
}

// ---- Jevonian: the ledger (one JSON line per request; brain calls are separate "brain" lines) --------------
function statusCell(status, ok, old) {
  const cell = String(status ?? "-").padEnd(6);
  return old ? cell : `${ok ? C.g : C.r}${cell}${C.x}`;
}

function jevonianLine(e, old = false) {
  const cost = typeof e.costUsd === "number" ? `$${e.costUsd.toFixed(4)}` : "-";
  const latency = e.latencyMs ? `${e.latencyMs}ms` : "-";
  const tail = `${e.requestedModel ?? ""}${e.brain ? ` (${e.brain})` : ""}`;
  const text = `  ${clock(e.ts)}  ${String(e.phase ?? "-").padEnd(9)}  ${String(e.model ?? "-").padEnd(26)}  ${String(e.effort ?? "-").padEnd(6)}  ${statusCell(e.status, e.status === 200, old)}  ${cost.padEnd(8)}  ${latency.padStart(7)}  ${tail}`;
  return old ? `${C.dim}${text}${C.x}` : text;
}

function parseLedger(text) {
  return text
    .split("\n")
    .map((raw) => {
      try {
        return JSON.parse(raw);
      } catch {
        return undefined;
      }
    })
    .filter((e) => e && e.kind !== "brain");
}

function tailLedger() {
  let offset = 0;
  let carry = "";
  if (fs.existsSync(target.ledger)) {
    const size = fs.statSync(target.ledger).size;
    const from = Math.max(0, size - 256 * 1024);
    const fd = fs.openSync(target.ledger, "r");
    const buf = Buffer.alloc(size - from);
    fs.readSync(fd, buf, 0, buf.length, from);
    fs.closeSync(fd);
    const recent = parseLedger(buf.toString("utf8")).slice(-8);
    if (recent.length) {
      out(`${C.dim}  (the last ${recent.length} before this window opened)${C.x}`);
      for (const e of recent) out(jevonianLine(e, true));
    }
    offset = size;
  }
  setInterval(() => {
    if (!fs.existsSync(target.ledger)) return;
    const fd = fs.openSync(target.ledger, "r");
    try {
      const { size } = fs.fstatSync(fd);
      if (size < offset) offset = 0; // a new ledger
      if (size === offset) return;
      const buf = Buffer.alloc(size - offset);
      fs.readSync(fd, buf, 0, buf.length, offset);
      offset = size;
      const lines = (carry + buf.toString("utf8")).split("\n");
      carry = lines.pop() ?? "";
      for (const e of parseLedger(lines.join("\n"))) out(jevonianLine(e));
    } finally {
      fs.closeSync(fd);
    }
  }, 1000);
}

// ---- jev-gateway: the official dashboard feed -------------------------------------------------------------
function gatewayLine(e, old = false) {
  const ok = e.status === 200 || e.status === undefined;
  const tool = e.tool ?? (e.jev?.choice ? `(${e.jev.choice})` : "-");
  const why = `${tool}${e.reason && !tool.includes(e.reason) ? " " + e.reason : ""}`;
  const confidence = e.confidence ?? e.jev?.confidence; // passthrough rows carry Jev's answer under e.jev
  const conf = typeof confidence === "number" ? confidence.toFixed(2) : "-";
  const tokens = e.usage ? `${e.usage.input} / ${e.usage.output}` : "-";
  const text = `  ${clock(e.time)}  ${String(e.mode ?? "-").padEnd(11)}  ${String(e.model ?? "-").padEnd(20)}  ${String(e.tools ?? "-").padStart(5)}  ${statusCell(e.status, ok, old)}  ${why.padEnd(27)}  ${conf.padEnd(5)}  ${tokens}`;
  return old ? `${C.dim}${text}${C.x}` : text;
}

function pollGateway() {
  let since = 0;
  let startedAt;
  const tick = async () => {
    try {
      const r = await fetch(`http://127.0.0.1:${target.port}/dashboard/events?since=${since}`, { signal: AbortSignal.timeout(3000) });
      const j = await r.json();
      if (j.router.startedAt !== startedAt) {
        const first = startedAt === undefined;
        startedAt = j.router.startedAt;
        if (first) {
          const recent = j.events.slice(-8);
          if (recent.length) {
            out(`${C.dim}  (the last ${recent.length} before this window opened)${C.x}`);
            for (const e of recent) out(gatewayLine(e, true));
          }
          since = j.router.recorded;
        } else {
          // The gateway restarted: its request numbers start over (the official page does the same).
          since = 0;
          await header(`gateway restarted at ${clock(startedAt)}`);
        }
        return;
      }
      for (const e of j.events) {
        since = Math.max(since, e.seq);
        // A restarted gateway replays its log once; those rows are older than startedAt (the official page dims them).
        if (Date.parse(e.time) >= Date.parse(startedAt)) out(gatewayLine(e));
      }
    } catch {
      // Gateway restarting or down: the watcher below says so.
    }
  };
  tick();
  setInterval(tick, 2000);
}

// ---- up / down / changed ----------------------------------------------------------------------------------
function identityOf(h) {
  if (!h) return "down";
  if (!isJevonian) return `up ${h.pid} ${h.upstream}`;
  let mtime = 0;
  try {
    mtime = fs.statSync(target.config).mtimeMs;
  } catch {}
  return `up ${mtime}`;
}

async function watch() {
  let last = identityOf(await health(target));
  setInterval(async () => {
    const id = identityOf(await health(target));
    if (id === last) return;
    const prev = last;
    last = id;
    if (id === "down") return out(`${C.r}  ${now()}  router on :${target.port} is not answering${C.x}`);
    if (prev === "down") {
      out(`${C.g}  ${now()}  router on :${target.port} is back online${C.x}`);
      if (isJevonian) await header("router restarted"); // a gateway's header is redrawn by pollGateway
      return;
    }
    if (isJevonian) await header("config.json changed");
  }, 3000);
}

(async () => {
  process.title = target.title;
  await header();
  if (isJevonian) tailLedger();
  else pollGateway();
  watch();
})();
```

</details>

The jev-gateway files (`gateway-env.js`, `jev-claude.js`, `jev-opencode.js`), `start.js`, the patches and the configs are
in [step 3](#step-3-the-four-jevonian-routers) and [step 4](#step-4-jev-gateway-official-unmodified).

---

## Appendix B: exact diff of the patched Jevonian against the official 0.1.7

`diff -u` of `node_modules\jevonian\dist\cli.mjs`:
- **Left:** the official 0.1.7 file, saved right after `npm install` and identical to the npm tarball.
- **Right:** the file in `jev-router-claude` after the first start.

It has all three patches: 8 hunks, +65/−6 lines. The Alibaba routers have the same diff without the Haiku additions
(7 hunks, +34/−6). **No other file in the package is changed.**

<details><summary>The full diff</summary>

```diff
--- jevonian 0.1.7 dist/cli.mjs (official)
+++ jev-router-claude (patched)
@@ -3565,7 +3565,11 @@
 		label: label.trim(),
 		description,
 		models,
-		...providers ? { providers } : {}
+		...providers ? { providers } : {},
+		/* routing-effort-patch v1 */
+		...typeof value.effort === "string" && isReasoningEffort(value.effort) ? { effort: value.effort } : {},
+		/* routing-effort-patch v3 */
+		...value.forceEffort === true ? { forceEffort: true } : {}
 	};
 }
 /**
@@ -8396,7 +8400,12 @@
 			virtual: true,
 			routed: true,
 			reason,
-			session
+			session,
+			/* routing-effort-patch v2 */
+			...(() => {
+				const pinned = clampEffort(config.routing.routings.find((entry) => entry.id === phase)?.effort ?? headerEffort(headers) ?? brainEffort(config.routing.defaultEffort), effectiveCapabilities(picked.model, config.routing.capacities?.[picked.model]).efforts);
+				return pinned ? { effort: pinned } : {};
+			})()
 		};
 	}
 	const brains = config.routing.brains;
@@ -8491,7 +8500,8 @@
 		return first ? leaderboardViewFor(first.model) : void 0;
 	}));
 	const includeBenchmarkHints = benchmarksCoverage !== "none";
-	const brainState = {
+	/* waf-safe-patch v2 */
+	const brainState = wafSafeState({
 		last_user_message: lastUserMessage(body, kind),
 		...lastAssistant ? { last_assistant_message: lastAssistant.slice(0, 500) } : {},
 		...goal ? { session_goal: goal } : {},
@@ -8512,7 +8522,7 @@
 			benchmarks_coverage: benchmarksCoverage
 		} : {},
 		...constraints
-	};
+	});
 	const applyVerdict = (verdict, channel, source) => {
 		confidence = verdict.confidence;
 		brain = source;
@@ -8648,7 +8658,9 @@
 	if (chosenCandidate?.cache.state === "hot") reason = `${reason}:cache-hot`;
 	if (chosenCandidate?.cache.state === "stale") reason = `${reason}:cache-stale`;
 	const wanted = brainPicksEffort ? brainEffort(best?.verdict.effort) : void 0;
-	const appliedEffort = clampEffort(wanted ?? requestedEffort ?? defaultEffort, effectiveCapabilities(chosen.model, config.routing.capacities?.[chosen.model]).efforts);
+	/* routing-effort-patch v1b */
+	const routingEffort = config.routing.routings.find((entry) => entry.id === phase)?.effort;
+	const appliedEffort = clampEffort(routingEffort ?? wanted ?? requestedEffort ?? defaultEffort, effectiveCapabilities(chosen.model, config.routing.capacities?.[chosen.model]).efforts);
 	const effortNote = wanted && appliedEffort && wanted !== appliedEffort ? `clamped "${wanted}" to "${appliedEffort}"` : requestedEffort && appliedEffort && requestedEffort !== appliedEffort ? `requested "${requestedEffort}", model supports "${appliedEffort}"` : void 0;
 	if (effortNote) reason = `${reason}:effort-clamped`;
 	store.set(session, {
@@ -12225,6 +12237,8 @@
 * defaults to thinking would otherwise keep thinking.
 */
 function withEffort(body, effort, wire, clientEffort) {
+	/* haiku-safe-patch v1 */
+	if (wire === "anthropic") body = haikuSafeBody(body);
 	const target = stripForeignEffort(body, wire);
 	if (wire === "anthropic") {
 		if (!effort || clientEffort) return normalizeAnthropicThinking(target);
@@ -12589,7 +12603,9 @@
 			body
 		});
 		const bodyFor = (wire) => {
-			const clientEffort = clientEffortOf(body, clientKind);
+			/* routing-effort-patch v3b */
+			const forcedRouting = config.routing.routings.find((entry) => entry.id === decision.phase);
+			const clientEffort = forcedRouting?.forceEffort === true && forcedRouting.effort && decision.effort ? void 0 : clientEffortOf(body, clientKind);
 			const maxOutput = effectiveCapabilities(decision.model, config.routing.capacities?.[decision.model]).maxOutput;
 			if (wire === "anthropic" && bridgeToAnthropic) return bridgedAnthropicBody(clientKind === "responses" ? responsesToChatRequest(body, decision.model) : body, {
 				model: decision.model,
@@ -15138,3 +15154,48 @@
 await main();
 //#endregion
 export {};
+
+/* waf-safe-patch v2 */
+function wafSafeText(s) {
+	return s.replace(/\|/g, "│").replace(/\/etc\//gi, "/ etc/").replace(/<(\/?)script/gi, "‹$1script").replace(/\.\.\//g, ".. /");
+}
+function wafSafeState(value) {
+	if (typeof value === "string") return wafSafeText(value);
+	if (Array.isArray(value)) return value.map(wafSafeState);
+	if (value && typeof value === "object") {
+		const out = {};
+		for (const [k, v] of Object.entries(value)) out[k] = wafSafeState(v);
+		return out;
+	}
+	return value;
+}
+
+/* haiku-safe-patch v1 */
+function haikuSafeBody(body) {
+	try {
+		if (!body || typeof body !== "object" || typeof body.model !== "string") return body;
+		const support = anthropicThinkingSupport(body.model);
+		if (!support || support.adaptive !== false) return body;
+		const next = { ...body };
+		const level = (typeof next.output_config === "object" && next.output_config !== null
+			&& typeof next.output_config.effort === "string") ? next.output_config.effort : void 0;
+		delete next.output_config;
+		delete next.context_management;
+		const thinking = typeof next.thinking === "object" && next.thinking !== null ? next.thinking : null;
+		if (thinking && thinking.type === "adaptive") {
+			if (level) {
+				next.thinking = { type: "enabled", budget_tokens: effortBudget(level) };
+			} else {
+				delete next.thinking;
+			}
+		} else if (thinking && thinking.type === "disabled") {
+			delete next.thinking;
+		}
+		if (Array.isArray(next.messages)) {
+			next.messages = next.messages.filter((m) => !(m && m.role === "system"));
+		}
+		return next;
+	} catch {
+		return body;
+	}
+}
```

</details>
