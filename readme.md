# Jevonian + jev-gateway on Windows (CMD): the complete replication guide

This guide rebuilds, step by step, a working setup of two open-source projects on one Windows PC:

- **[Jevonian](https://github.com/xinyao27/jevonian) 0.1.7:** a local model router. Its brain, **Jev**, picks the model and the
  thinking effort for every turn.
- **[jev-gateway](https://github.com/vinilana/jev-gateway) 0.4.3:** a local gateway. It asks Jev **which tool** the agent
  should call next.

Four coding agents use them: **Qwen Code, Kilo, OpenCode and Claude Code**. Each tool is available both directly through its own Jevonian router and through jev-gateway for tool steering. Each tool has its own router, its own web page, and its own status window. Every file, command and expected output below comes from the real build of **2026-09-25** (in `D:\learn\JevAI`). Nothing is paraphrased: the files are embedded exactly as installed.

**Customizations & Enhancements:**
- Jevonian gets three small patches, applied again automatically on every start.
- jev-gateway is enhanced over the base repository with:
  1. **Full Kilo Code and Qwen Code support** (`jev-kilo` on port 8785 and `jev-qwen` on port 8787).
  2. **Served Model Column Display** from [PR #50](https://github.com/vinilana/jev-gateway/pull/50) by `@piratchai`, showing the concrete served model instead of generic aliases like `jevonian/auto`.
[What is official and what was changed](#4-what-is-official-and-what-was-changed) lists every enhancement in detail.

## At a glance

| You type in CMD | Goes through | Port | The model comes from | Web page |
|---|---|---|---|---|
| `qwen` | Jevonian | 8793 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8793/logs |
| `kilo` | Jevonian | 8795 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8795/logs |
| `opencode` | Jevonian | 8799 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8799/logs |
| `claude` | Jevonian (its official `jevonian launch claude`) | 8797 | Anthropic, through your Claude subscription login | http://127.0.0.1:8797/logs |
| `jev-kilo` | jev-gateway → Jevonian | 8785 → 8795 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8785/dashboard?peers=none |
| `jev-qwen` | jev-gateway → Jevonian | 8787 → 8793 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8787/dashboard?peers=none |
| `jev-claude` | jev-gateway → Jevonian | 8789 → 8797 | Anthropic, through your Claude subscription login | http://127.0.0.1:8789/dashboard?peers=none |
| `jev-opencode` | jev-gateway → Jevonian | 8791 → 8799 | Alibaba Cloud Model Studio, Token Plan | http://127.0.0.1:8791/dashboard?peers=none |
| `jev …` | manages all of the above: `start`, `stop`, `status`, `test`, `windows`, `dashboards`, `logs` | | | |

Verified on 2026-09-25 (see [Proof of working](#15-proof-of-working)):
- Both Jev brain channels work on all four routers: 8/8.
- All 23 tiers are served by the configured model at the configured effort: 23/23.
- All 8 commands answer, and all 8 read a file through their router: 16/16.
- All 8 server targets (4 routers + 4 gateways) verified live and operational in `jev status`.
- For every tool, the client, its status window and its web page show the same requests.

## Contents

0. [Quickstart TL;DR (5-minute setup)](#0-quickstart-tldr-5-minute-setup)
1. [What you will build](#1-what-you-will-build)
2. [Accounts, keys and providers](#2-accounts-keys-and-providers)
3. [Prerequisites](#3-prerequisites)
4. [What is official and what was changed](#4-what-is-official-and-what-was-changed)
   - 4.1 [Jevonian: official defaults vs. this setup](#41-jevonian-official-defaults-vs-this-setup)
   - 4.2 [The three Jevonian patches](#42-the-three-jevonian-patches)
   - 4.3 [jev-gateway: official defaults vs. this setup](#43-jev-gateway-official-defaults-vs-this-setup)
   - 4.4 [Jev-Gateway Customizations & Enhancements (Over the Base Repository)](#44-jev-gateway-customizations--enhancements-over-the-base-repository)
   - 4.5 [The clients: nothing global is changed](#45-the-clients-nothing-global-is-changed)
   - 4.6 [What was added around the two projects](#46-what-was-added-around-the-two-projects)
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
16. [Jev-Router for Claude Code — Complete Setup, Customization & Standalone Architecture](#16-jev-router-for-claude-code--complete-setup-customization--standalone-architecture)
   - 16.1 [Architecture & Direct Integration: Why Claude Code Differs](#161-architecture--direct-integration-why-claude-code-differs)
   - 16.2 [The 6-Tier Model & Reasoning Effort Routing Matrix](#162-the-6-tier-model--reasoning-effort-routing-matrix)
   - 16.3 [Downloading & Setting Up Jev-Gateway & gargpratyush-jev-router](#163-downloading--setting-up-jev-gateway--gargpratyush-jev-router)
   - 16.4 [Customizing Jev-Router for Claude Code](#164-customizing-jev-router-for-claude-code)
   - 16.5 [Enabling Direct Execution in Plain Regular Windows CMD](#165-enabling-direct-execution-in-plain-regular-windows-cmd)
   - 16.6 [Configuration, Credentials & Environment Isolation](#166-configuration-credentials--environment-isolation)
   - 16.7 [Workspace-Scoped MCP Servers vs. Global Pollution](#167-workspace-scoped-mcp-servers-vs-global-pollution)
   - 16.8 [Running Dedicated Dashboards on Separate Ports](#168-running-dedicated-dashboards-on-separate-ports)
   - 16.9 [Replicating to Any New Workspace (e.g. graduated_project)](#169-replicating-to-any-new-workspace-eg-graduated_project)
- [Appendix A: the command layer, every file in full](#appendix-a-the-command-layer-every-file-in-full)
- [Appendix B: exact diff of the patched Jevonian against the official 0.1.7](#appendix-b-exact-diff-of-the-patched-jevonian-against-the-official-017)

---

## 0. Quickstart TL;DR (5-minute setup)

If you already understand the architecture and want to get the routers and gateways running immediately on your machine:

1. **Pick a root folder (`%JEVAI%`):**
   Choose any directory (e.g. `D:\learn\JevAI` or `C:\Users\<user>\JevAI`). Two sibling folders will live here: `%JEVAI%\jevonian` and `%JEVAI%\jev-gateway`.
2. **Add the 3 API key files:**
   - `%JEVAI%\jevonian\credentials\qwen-alibaba-credential.txt` → `API Key: sk-...` (Alibaba Model Studio Token Plan)
   - `%JEVAI%\jevonian\credentials\typesafe-ai-credential.txt` and `%JEVAI%\jev-gateway\credentials\typesafe-ai-credential.txt` → the raw key alone (TypeSafe `jev-latest`)
   - `%JEVAI%\jevonian\credentials\vercel-ai-gateway-credential.txt` → the raw key alone (`AI_GATEWAY_API_KEY` fallback from Vercel AI Gateway)
3. **Verify prerequisites & client logins:**
   - Node.js 22.15+ & npm installed.
   - Globally installed tools on `PATH`: `qwen`, `kilo`, `opencode`, `claude`.
   - **Crucial:** Run `claude` once and log in with `/login` to store your OAuth session in `%USERPROFILE%\.claude\.credentials.json`.
4. **Install & start:**
   - Run `npm install --no-audit --no-fund` in each of the 4 `jev-router-*` folders and `jev-gateway`.
   - Run `node %JEVAI%\jevonian\jev.js install` (installs CMD doskey macros & AutoRun).
   - *If using PowerShell or VS Code terminal:* Add the PowerShell functions to your profile (see [PowerShell integration](#powershell-and-vs-code-terminal-integration)).
   - Run `jev start` (or start via `node %JEVAI%\jevonian\jev.js start`).
5. **Verify:**
   - `jev status` → all 6 servers must report `UP`.
   - `jev test brains` → `8/8` passed.
   - `jev test clients` → `12/12` passed.
   - Open the web dashboards: [http://127.0.0.1:8793/logs](http://127.0.0.1:8793/logs) (Qwen), [http://127.0.0.1:8797/logs](http://127.0.0.1:8797/logs) (Claude), or [http://127.0.0.1:8789/dashboard](http://127.0.0.1:8789/dashboard) (Gateways).

---

## 1. What you will build

Two sibling folders, each holding one official project, installed locally (no `npm -g`), plus small scripts around them:

```text
%JEVAI%\                           (D:\learn\JevAI in the original build; see step 0)
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
└─ jev-gateway\                    jev-gateway: with Kilo/Qwen additions & PR #50 served-model display
   ├─ package.json                 "jev-gateway": "0.4.3"
   ├─ gateway-env.js               environment mappings for claude, opencode, kilo, and qwen
   ├─ start-gateway.js             background daemon manager (ports 8785, 8787, 8789, 8791)
   ├─ bin\                         CLI launcher scripts:
   │  ├─ jev-kilo.mjs              launcher for Kilo Code CLI (port 8785)
   │  ├─ jev-qwen.mjs              launcher for Qwen Code CLI (port 8787)
   │  ├─ jev-claude.mjs            official launcher for Claude Code (port 8789)
   │  └─ jev-opencode.mjs          official launcher for OpenCode (port 8791)
   ├─ jev-kilo.js  jev-qwen.js     CMD commands for Kilo & Qwen gateway
   ├─ jev-claude.js  jev-opencode.js CMD commands for Claude & OpenCode gateway
   ├─ credentials\typesafe-ai-credential.txt
   └─ generated: node_modules\     (logs and pid files in %USERPROFILE%\.jev-gateway)
```

How a request travels:

```text
qwen          ─► Jevonian :8793 ─┐
kilo          ─► Jevonian :8795 ─┼─ Jev picks the tier (model + effort) ─► Alibaba Cloud Model Studio (Token Plan)
opencode      ─► Jevonian :8799 ─┘
claude        ─► Jevonian :8797 ─── Jev picks the tier (model + effort) ─► Anthropic, with your Claude Code login (OAuth)

jev-kilo      ─► jev-gateway :8785 (Jev picks the TOOL) ─► Jevonian :8795 (tier + effort) ─► Alibaba
jev-qwen      ─► jev-gateway :8787 (Jev picks the TOOL) ─► Jevonian :8793 (tier + effort) ─► Alibaba
jev-claude    ─► jev-gateway :8789 (Jev picks the TOOL) ─► Jevonian :8797 (tier + effort) ─► Anthropic
jev-opencode  ─► jev-gateway :8791 (Jev picks the TOOL) ─► Jevonian :8799 (tier + effort) ─► Alibaba
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
| Tools here | Qwen Code, Kilo, OpenCode, Claude Code | All four! Qwen Code, Kilo, OpenCode, Claude Code |
| Commands | `qwen`, `kilo`, `opencode`, `claude` | `jev-kilo`, `jev-qwen`, `jev-claude`, `jev-opencode` (they **also** go through Jevonian) |

**Use the plain commands (Jevonian) by default.** Tier routing is where the savings are.

Use `jev-kilo` / `jev-qwen` / `jev-claude` / `jev-opencode` to *add* Jev's tool routing on top. jev-gateway's own README says to expect better tool
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

- **Windows 10/11 and Shell Choice:**
  - The default setup uses **CMD** because CMD supports `doskey` macros and the `AutoRun` registry entry out of the box.
  - **If you use PowerShell, Windows Terminal (default profile), or VS Code integrated terminal:** `doskey` macros will **not** execute there. You must add the PowerShell profile functions provided in [PowerShell and VS Code Terminal Integration](#powershell-and-vs-code-terminal-integration) to use the shorthand commands (`qwen`, `kilo`, `claude`, etc.) in PowerShell.
- **No administrator rights needed:** `jev install` writes only to current-user registry (`HKCU`).
- **Node.js 22.15 or newer, with npm:** jev-gateway requires 22.15+, and Jevonian requires 22+.
- **The four client CLI tools on your `PATH`:**

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

| Tool | Version | Installed at | How to install if missing |
|---|---|---|---|
| Node.js / npm | v22.23.2 / 10.9.8 | `C:\nvm4w\nodejs` | via nvm-windows, winget, or nodejs.org |
| Claude Code | 2.1.282 | `%USERPROFILE%\.local\bin\claude.exe` | `npm install -g @anthropic-ai/claude-code` or native installer |
| OpenCode | 2.0.15 | `%USERPROFILE%\.bun\bin\opencode.exe` | `bun add -g opencode-ai` or `npm install -g opencode-ai` |
| Kilo (Kilo Code CLI) | 7.7.9 | `%USERPROFILE%\.bun\bin\kilo.exe` | `bun add -g @kilo/cli` or `npm install -g @kilo/cli` |
| Qwen Code | 0.24.4 | `C:\nvm4w\nodejs\qwen.cmd` | `npm install -g @qwen-code/cli` |
| jevonian | 0.1.7 (latest on npm and on GitHub `main`) | installed locally in step 3 | local project dependency |
| jev-gateway | 0.4.3 (latest on npm) | installed locally in step 4 | local project dependency |

### 3.1 Client Authentication & Setup Checklist

Before running routed commands, ensure each client tool is in a ready state:

1. **Claude Code OAuth Login:**
   - **Crucial:** Claude Code must be logged in beforehand with an active **Claude Pro or Max** subscription.
   - Run `claude`, execute `/login`, and complete the browser flow.
   - The Claude router relies on `%USERPROFILE%\.claude\.credentials.json` (OAuth access/refresh tokens); it has no API key of its own.
2. **OpenCode 2.x Requirements:**
   - OpenCode 2.x introduces a daemon/service model. The router commands pass `--standalone` and pin `PWD` to prevent background services from picking up unrelated directory configurations.
   - In automated scripts or piped commands, `opencode run` expects an EOF or closed stdin; redirect `< nul` in CMD or use null-piped input in PowerShell (see [Known behaviours](#11-known-behaviours-and-limits)).
3. **Kilo Code CLI:**
   - Ensure Kilo is initialized. The router injects `KILO_CONFIG_CONTENT` pointing to `jev-router-kilo\.kilo\kilo.json` and pins `-m jevonian/jevonian/auto`.
4. **Qwen Code:**
   - Ensure `qwen` is runnable from command line. The launcher injects `--auth-type openai --openai-base-url http://127.0.0.1:8793/v1` dynamically.

### 3.2 Enterprise Security & EDR Notice (Cylance, CrowdStrike, AppLocker)

In corporate environments with strict endpoint protection:
- **Registry AutoRun Restrictions:** Some EDR policies block or audit changes to `HKCU\Software\Microsoft\Command Processor\AutoRun`. If `jev install` fails or is reverted by group policy, you can run commands directly via `node path\to\jevonian\kilo.js` or use PowerShell functions.
- **PowerShell Script Blocking:** Security agents like Cylance Script Control may block chained PowerShell expressions joined with semicolons (`;`). Run commands individually, use CMD, or invoke scripts via `node path/to/script.js`.

### 3.3 Port Allocation & Conflicts

This setup reserves **10 local TCP ports** (listening strictly on `127.0.0.1`):

| Port | Service | Notes |
|---|---|---|
| `8785` | `jev-gateway` (Kilo) | Forwards to `:8795` |
| `8787` | `jev-gateway` (Qwen) | Forwards to `:8793` |
| `8789` | `jev-gateway` (Claude) | Forwards to `:8797` |
| `8791` | `jev-gateway` (OpenCode) | Forwards to `:8799` |
| `8793` / `8794` | `jev-router-qwen` | `:8793` router + `:8794` tunnel surface (requires key) |
| `8795` / `8796` | `jev-router-kilo` | `:8795` router + `:8796` tunnel surface (requires key) |
| `8797` / `8798` | `jev-router-claude` | `:8797` router + `:8798` tunnel surface (requires key) |
| `8799` / `8800` | `jev-router-opencode` | `:8799` router + `:8800` tunnel surface (requires key) |

Check before starting: `netstat -ano | findstr LISTENING | findstr ":878 :879 :880"`. It must return empty.

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

The core gateway architecture follows the official `jev-gateway` model, with documented environment settings and wrappers that connect each agent to its corresponding Jevonian router. In addition, this build incorporates targeted enhancements for multi-agent support and dashboard observability.

| Topic | Official jev-gateway 0.4.3 | This setup | Why |
|---|---|---|---|
| Install | `npm install -g jev-gateway` | `npm install` in `%JEVAI%\jev-gateway`, `"jev-gateway": "0.4.3"` | No global installs. |
| Supported Agents | Claude Code and OpenCode (with stubs for Gemini/Codex) | **Claude Code, OpenCode, Kilo Code, and Qwen Code** | Complete coverage of all 4 coding agents in both direct and gateway modes. |
| Launchers | `jev-claude`, `jev-opencode` on `PATH` | `jev-claude`, `jev-opencode`, `jev-kilo`, `jev-qwen` (CMD macros / PowerShell functions) | Native invocation from any folder with automatic router and gateway lifecycle management. |
| Jev key | asked on first run, saved to `~/.jev-gateway/.env` (`--setup`) | `TYPESAFE_API_KEY` from `jev-gateway\credentials\typesafe-ai-credential.txt` and `JEV_PROVIDER=typesafe`, as environment variables | Documented variables. No prompt, and the key stays in the folder. |
| Claude Code upstream | `https://api.anthropic.com/v1` | `JEV_CLAUDE_UPSTREAM_BASE_URL=http://127.0.0.1:8797/v1` (the Jevonian Claude router) | Tier and effort for jev-claude too. |
| OpenCode upstream / model | `https://api.openai.com/v1`, model `gpt-5` | `JEV_OPENCODE_UPSTREAM_BASE_URL=http://127.0.0.1:8799/v1`, `JEV_OPENCODE_MODEL=jevonian/auto` | The Jevonian OpenCode router picks the model. |
| Kilo upstream / model | *(not in official)* | `JEV_KILO_UPSTREAM_BASE_URL=http://127.0.0.1:8795/v1`, `JEV_KILO_MODEL=jevonian/auto` | Jevonian Kilo router picks the model. |
| Qwen upstream / model | *(not in official)* | `JEV_QWEN_UPSTREAM_BASE_URL=http://127.0.0.1:8793/v1`, `JEV_QWEN_MODEL=jevonian/auto` | Jevonian Qwen router picks the model. |
| `OPENAI_API_KEY` | your OpenAI key | `local-no-key` | Forwarded to Jevonian, which needs none on loopback. |
| Ports | claude 8789, opencode 8791 | kilo **8785**, qwen **8787**, claude **8789**, opencode **8791** | Dedicated port per agent. |
| Dashboard Model Column | Shows client-requested alias (always `jevonian/auto`) | Displays the **actual served model** (from PR #50) | Visibility into the concrete model chosen by Jevonian. |
| Logs, pid files | `%USERPROFILE%\.jev-gateway\` | the same | Official. |

### 4.4 Jev-Gateway Customizations & Enhancements (Over the Base Repository)

While the base [`vinilana/jev-gateway`](https://github.com/vinilana/jev-gateway) repository provides an excellent tool-steering foundation for Claude Code and OpenCode, this setup introduces **three major enhancements** over upstream to deliver full multi-agent support and end-to-end visibility:

#### 4.4.1 Served Model Column Display ([PR #50](https://github.com/vinilana/jev-gateway/pull/50))

- **The Upstream Problem:**
  In the official `vinilana/jev-gateway` dashboard (`/dashboard`), the "Model" column only displays `req.model`—the string sent by the client in the incoming request body.
  When `jev-gateway` is chained to an upstream model router like Jevonian, clients request a virtual tier identifier (such as `jevonian/auto` or `jev-gateway/jevonian/auto`). Consequently, the base gateway dashboard was **completely blind** to what model actually executed each turn. Every row in the base dashboard displayed `jevonian/auto`, hiding whether Jevonian chose `glm-5.3`, `qwen3.8-flash`, `deepseek-v4.1-flash`, or `claude-haiku-4-5-20251001`.

- **The Enhancement ([PR #50](https://github.com/vinilana/jev-gateway/pull/50) by `@piratchai`):**
  - **Pull Request:** [`fix(dashboard): show the model that actually served a request, not just the one asked for`](https://github.com/vinilana/jev-gateway/pull/50) (Branch: `fix/dashboard-served-model`).
  - **Response Stream Sniffing (`src/usage.ts`):** Added `readServedModel(req, res)` which inspects incoming Server-Sent Event (SSE) chunks (e.g. `data: {"model": "..."}`) or buffered JSON bodies returned by the upstream provider.
  - **Event Data Model (`src/events.ts`):** Extended `RouteEvent` with `servedModel?: string`.
  - **Completion Logging (`src/app.ts`):** Passes `servedModel: readServedModel(req, res)` into `logWhenDone()`.
  - **Dashboard Visualization (`src/dashboard.html`):** The "Model" table column now prominently displays the **actual concrete served model** that handled the turn, accompanied by a subtle tag or hover tooltip showing the client's requested alias (`jevonian/auto`).
  - **Developer Benefit:** Developers get simultaneous, unified visibility into both layers—the gateway's tool steering verdict (`hint`, `none`, `forced`, `passthrough`) and the router's concrete model selection in a single table row.

#### 4.4.2 Kilo Code CLI Gateway Integration (`jev-kilo` on Port 8785)

- **Why it works identically to OpenCode:**
  Analysis of Kilo Code CLI (`kilo --version` 7.7.9) reveals it is an exact downstream build of OpenCode (`service=default version=7.6.2 ... opencode`) powered by Vercel's `@ai-sdk/openai-compatible`.
- **The Upstream Gap:**
  The base `jev-gateway` repository only ships launchers for `claude` and `opencode` (with experimental stubs for `gemini` and `codex`). It had no support for Kilo Code CLI.
- **Implementation & Architecture:**
  - **Port Mapping:** Gateway listens on port `8785` and forwards upstream to `http://127.0.0.1:8795/v1` (the Jevonian Kilo router).
  - **Environment (`gateway-env.js`):** Configures `JEV_KILO_UPSTREAM_BASE_URL: "http://127.0.0.1:8795/v1"`, `JEV_KILO_MODEL: "jevonian/auto"`, and `OPENAI_API_KEY: "local-no-key"`.
  - **Launcher (`bin/jev-kilo.mjs`):** Spawns Kilo with an injected `KILO_CONFIG_CONTENT` pointing provider to `http://127.0.0.1:8785/v1` with model `jev-gateway/jevonian/auto`, `--standalone`, and `PWD`.
  - **CMD Macro Wrapper (`jev-kilo.js`):** Manages the status window, starts the router and gateway on demand, and executes Kilo in the local folder.
  - **Live Verification:** Evaluated 53 tool declarations through Jev, successfully executing tools while routing concrete turns to Alibaba Token Plan models.

#### 4.4.3 Qwen Code CLI Gateway Integration (`jev-qwen` on Port 8787)

- **Why it works identically to OpenCode:**
  Qwen Code CLI (`qwen` 0.24.4) is a TypeScript CLI (forked from Google Gemini CLI) that implements the standard OpenAI chat completions format with native `tools` and `tool_calls` schemas.
- **The Upstream Gap:**
  The base `jev-gateway` repository has zero integration for Qwen Code CLI.
- **Implementation & Architecture:**
  - **Port Mapping:** Gateway listens on port `8787` and forwards upstream to `http://127.0.0.1:8793/v1` (the Jevonian Qwen router).
  - **Environment (`gateway-env.js`):** Configures `JEV_QWEN_UPSTREAM_BASE_URL: "http://127.0.0.1:8793/v1"`, `JEV_QWEN_MODEL: "jevonian/auto"`, and `OPENAI_API_KEY: "local-no-key"`.
  - **Launcher (`bin/jev-qwen.mjs`):** Spawns Qwen with CLI parameters `--auth-type openai --openai-base-url http://127.0.0.1:8787/v1 --openai-api-key local-no-key -m jevonian/auto`.
  - **CMD Macro Wrapper (`jev-qwen.js`):** Manages the status window, starts the router and gateway on demand, and executes Qwen in the local folder.
  - **Live Verification:** Evaluated 5 tool declarations through Jev, steered tools in `mode: forced` with 94% confidence, and executed local commands cleanly.

### 4.5 The clients: nothing global is changed

| Client | How it's pointed at its router | What is *not* touched |
|---|---|---|
| Qwen Code | command-line flags on every run: `--auth-type openai --openai-base-url http://127.0.0.1:8793/v1 --openai-api-key local-no-key -m jevonian/auto` (or `:8787/v1` for `jev-qwen`) | `~/.qwen/settings.json` |
| Kilo | `KILO_CONFIG_CONTENT` = `jev-router-kilo\.kilo\kilo.json` (provider named `jevonian` → `:8795/v1` or `jev-gateway` → `:8785/v1`), plus `-m jevonian/jevonian/auto` and `PWD` | your Kilo config and login |
| OpenCode | `OPENCODE_CONFIG_CONTENT` = `jev-router-opencode\opencode.json` (provider `jevonian` → `:8799/v1` or `jev-gateway` → `:8791/v1`), plus `--standalone`, `PWD`, `-m jevonian/jevonian/auto` | `~/.config/opencode` |
| Claude Code | the official `jevonian launch claude` (environment for that one session only) | `~/.claude/settings.json` |

The `*-direct` commands (`claude-direct`, `kilo-direct`, `opencode-direct`, `qwen-direct`) run the real clients with no router, exactly as before.

### 4.6 What was added around the two projects

None of this is part of Jevonian or base jev-gateway. It's a clean management and integration layer, all Node, with no `.bat` files:

| File | What it does |
|---|---|
| `jevonian\jev-router-*\start.js` | Reads the keys, points Jevonian at the router folder, applies the patches, runs the official `serve`. |
| `jevonian\kilo.js`, `qwen.js`, `opencode.js`, `claude.js` | The CMD commands for Jevonian routers. They start the router if it's down, open the status window, and run the real client. |
| `jev-gateway\start-gateway.js` | Background daemon manager for all 4 gateways (ports 8785, 8787, 8789, 8791). |
| `jev-gateway\jev-kilo.js`, `jev-qwen.js`, `jev-claude.js`, `jev-opencode.js` | The gateway CMD commands: start the upstream Jevonian router and the gateway daemon, then run the client. |
| `jev-gateway\bin\jev-kilo.mjs`, `jev-qwen.mjs` | Custom launchers implementing Kilo and Qwen CLI wiring for jev-gateway. |
| `jevonian\lib\targets.js` | Shared targets registry for all **8 servers** (4 routers + 4 gateways). |
| `jevonian\lib\common.js` | Shared code: health checks, start/stop, status window management, and client execution. |
| `jevonian\jev.js` | `jev`: start, stop, restart, status, logs, windows, dashboards, `test`, and `install` / `uninstall` of the CMD commands. |
| `jevonian\jev.doskey` + CMD AutoRun registry | Make `kilo`, `qwen`, `opencode`, `claude`, `jev-kilo`, `jev-qwen`, `jev-claude`, `jev-opencode`, `jev` and `*-direct` exist in every new CMD window. |

---

## 5. Build it, step by step

Every command below is for **CMD**. You pick **one root folder** at the start; all commands use it through the variable
`%JEVAI%`, so nothing in this guide needs editing:
- **Paths:** the scripts find everything relative to themselves. The only absolute paths are the ones `jev install` writes
  into `jev.doskey`.
- **Siblings:** the two folders must stay siblings (`%JEVAI%\jevonian` and `%JEVAI%\jev-gateway`).

The original build used `D:\learn\JevAI`. Outputs quoted below come from it, so their paths show that folder.

### Step 0: choose the root folder, and stop any other copy

In the CMD window you use for steps 1 to 7 (set it again if you open another window before step 7):

```bat
set JEVAI=D:\learn\JevAI
```

**If another copy of this setup is already installed on this PC**, stop it first, **in a CMD window of the old copy**. Both
copies use the same ports and the same `%USERPROFILE%\.jev-gateway` files:

```bat
jev stop
jev windows close
```

If you skip this, `jev start` finds the old routers answering on the ports and doesn't start the new ones.

Then make sure nothing listens on the ports:

```bat
netstat -ano | findstr LISTENING | findstr ":878 :879 :880"
```

It must print nothing. Lines in states like `TIME_WAIT`, from connections that just closed, don't matter; that's why the
command filters on `LISTENING`.

### Step 1: create the folders

```bat
mkdir %JEVAI%\jevonian\credentials
mkdir %JEVAI%\jevonian\lib
mkdir %JEVAI%\jevonian\jev-router-qwen\config     %JEVAI%\jevonian\jev-router-qwen\.qwen
mkdir %JEVAI%\jevonian\jev-router-kilo\config     %JEVAI%\jevonian\jev-router-kilo\.kilo
mkdir %JEVAI%\jevonian\jev-router-claude\config
mkdir %JEVAI%\jevonian\jev-router-opencode\config
mkdir %JEVAI%\jev-gateway\credentials
```

### Step 2: the key files

Create them with Notepad, so the keys never land in your shell history:

```bat
notepad %JEVAI%\jevonian\credentials\qwen-alibaba-credential.txt
notepad %JEVAI%\jevonian\credentials\typesafe-ai-credential.txt
notepad %JEVAI%\jevonian\credentials\vercel-ai-gateway-credential.txt
notepad %JEVAI%\jev-gateway\credentials\typesafe-ai-credential.txt
```

| File | Content |
|---|---|
| `qwen-alibaba-credential.txt` | `API Key: sk-…` (your Token Plan key; other lines, such as console links, are ignored) |
| `typesafe-ai-credential.txt` (both copies) | the TypeSafe key alone |
| `vercel-ai-gateway-credential.txt` | the Vercel AI Gateway key alone |

If you already have these four files from an earlier copy of this setup, copy them instead. `OLD` is that copy's root:

```bat
set OLD=D:\path\to\the\old\root
copy "%OLD%\jevonian\credentials\*.txt" "%JEVAI%\jevonian\credentials\"
copy "%OLD%\jev-gateway\credentials\typesafe-ai-credential.txt" "%JEVAI%\jev-gateway\credentials\"
```

### Step 3: the four Jevonian routers

Each router folder is self-contained: its own `package.json`, its own copy of the official package, config, patches and data.
Every file shown in a box below goes to the path in its title, relative to `%JEVAI%`.

**3a. `package.json` in each router folder.** Each pins `"jevonian": "0.1.7"`, exact, with no `^`:

<details><summary><code>jevonian\jev-router-qwen\package.json</code></summary>

```json
{
  "name": "jev-router-qwen",
  "version": "1.0.0",
  "description": "Self-contained Jevonian router + Qwen Code config",
  "dependencies": {
    "jevonian": "0.1.7"
  }
}
```

</details>

<details><summary><code>jevonian\jev-router-kilo\package.json</code></summary>

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

</details>

<details><summary><code>jevonian\jev-router-claude\package.json</code></summary>

```json
{
  "name": "jev-router-claude",
  "version": "1.0.0",
  "description": "Self-contained Jevonian router for Claude Code (claude.ai subscription via OAuth)",
  "dependencies": {
    "jevonian": "0.1.7"
  }
}
```

</details>

<details><summary><code>jevonian\jev-router-opencode\package.json</code></summary>

```json
{
  "name": "jev-router-opencode",
  "version": "1.0.0",
  "description": "Self-contained Jevonian router + OpenCode config",
  "dependencies": {
    "jevonian": "0.1.7"
  }
}
```

</details>

Then install the official package in each (about 2 seconds each):

```bat
cd /d %JEVAI%\jevonian\jev-router-qwen      && npm install --no-audit --no-fund
cd /d %JEVAI%\jevonian\jev-router-kilo      && npm install --no-audit --no-fund
cd /d %JEVAI%\jevonian\jev-router-claude    && npm install --no-audit --no-fund
cd /d %JEVAI%\jevonian\jev-router-opencode  && npm install --no-audit --no-fund
```

Expected output, each time: `added 15 packages in …s`. `node_modules\jevonian\package.json` should say `"version": "0.1.7"`.

**3b. `config\config.json`: providers, tiers, effort, brains.** The three Alibaba routers differ only in `"port"`
(qwen 8793, kilo 8795, opencode 8799):

<details><summary><code>jevonian\jev-router-qwen\config\config.json</code></summary>

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

<details><summary><code>jevonian\jev-router-kilo\config\config.json</code></summary>

```json
{
  "listen": {
    "host": "127.0.0.1",
    "port": 8795
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

<details><summary><code>jevonian\jev-router-opencode\config\config.json</code></summary>

```json
{
  "listen": {
    "host": "127.0.0.1",
    "port": 8799
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
| `tunnel`, `modelSync` (Claude) | Tunnel off. Model auto-sync off, so the file is never rewritten. (Alibaba routers keep the official default: auto-sync on, but API-key providers are not synced.) |

**3c. The patches.** Save the two shared patches in `jev-router-qwen`, then copy them to the other three routers. Save the
Haiku patch in `jev-router-claude` only. You don't run them yourself: `start.js` does, on every start.

<details><summary><code>jevonian\jev-router-qwen\patch-jevonian-waf.mjs</code> (then copied to all routers)</summary>

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

<details><summary><code>jevonian\jev-router-qwen\patch-jevonian-effort.mjs</code> (then copied to all routers)</summary>

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

<details><summary><code>jevonian\jev-router-claude\patch-jevonian-haiku.mjs</code> (Claude router only)</summary>

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

**3d. `start.js`.** Save it in `jev-router-qwen`, then copy it (and the two shared patches) to the other routers:

<details><summary><code>jevonian\jev-router-qwen\start.js</code> (then copied to all routers)</summary>

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

```bat
cd /d %JEVAI%\jevonian
for %r in (kilo claude opencode) do copy /y jev-router-qwen\start.js jev-router-%r\ && copy /y jev-router-qwen\patch-jevonian-waf.mjs jev-router-%r\ && copy /y jev-router-qwen\patch-jevonian-effort.mjs jev-router-%r\
```

Expected output: CMD echoes each `copy …` line, followed by `1 file(s) copied.` three times, for kilo, claude and opencode
(nine copies in total).

**3e. The client configs.** They're read by the CMD commands (Kilo, OpenCode), or used when you run the real client inside
the router folder (Qwen's `/model` list):

<details><summary><code>jevonian\jev-router-kilo\.kilo\kilo.json</code> (injected as <code>KILO_CONFIG_CONTENT</code>)</summary>

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

<details><summary><code>jevonian\jev-router-opencode\opencode.json</code> (injected as <code>OPENCODE_CONFIG_CONTENT</code>)</summary>

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

<details><summary><code>jevonian\jev-router-qwen\.qwen\settings.json</code> (for <code>qwen-direct</code> run inside <code>jev-router-qwen</code>)</summary>

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
- **The `limit` values** (200,000 context, 65,536 output) are what the clients plan with. The router still sends each turn to
  the model the tier names.

### Step 4: jev-gateway (with Kilo & Qwen launchers and PR #50)

<details><summary><code>jev-gateway\package.json</code></summary>

```json
{
  "name": "jev-gateway-local",
  "private": true,
  "description": "jev-gateway installed locally with Kilo Code and Qwen Code integrations and PR #50 served-model display.",
  "dependencies": {
    "jev-gateway": "0.4.3"
  }
}
```

</details>

```bat
cd /d %JEVAI%\jev-gateway && npm install --no-audit --no-fund
```

Expected output: `added 3 packages in …s`. Add the configuration and launcher scripts below:

<details><summary><code>jev-gateway\gateway-env.js</code></summary>

```js
// Environment for jev-gateway launchers across all 4 coding agents.
//   jev-kilo     : Kilo Code   -> jev-gateway :8785 (Jev picks TOOL) -> Jevonian :8795 (tier + effort) -> Alibaba
//   jev-qwen     : Qwen Code   -> jev-gateway :8787 (Jev picks TOOL) -> Jevonian :8793 (tier + effort) -> Alibaba
//   jev-claude   : Claude Code -> jev-gateway :8789 (Jev picks TOOL) -> Jevonian :8797 (tier + effort) -> Anthropic
//   jev-opencode : OpenCode    -> jev-gateway :8791 (Jev picks TOOL) -> Jevonian :8799 (tier + effort) -> Alibaba
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
  ANTHROPIC_AUTH_TOKEN: "jevonian-local",
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
    Object.assign(env, { JEV_CLAUDE_UPSTREAM_BASE_URL: "http://127.0.0.1:8797/v1" }, CLAUDE_VIA_JEVONIAN);
  }
  if (client === "opencode") {
    Object.assign(env, {
      JEV_OPENCODE_UPSTREAM_BASE_URL: "http://127.0.0.1:8799/v1",
      JEV_OPENCODE_MODEL: "jevonian/auto",
      OPENAI_API_KEY: "local-no-key",
    });
  }
  if (client === "kilo") {
    Object.assign(env, {
      JEV_KILO_UPSTREAM_BASE_URL: "http://127.0.0.1:8795/v1",
      JEV_KILO_MODEL: "jevonian/auto",
      OPENAI_API_KEY: "local-no-key",
    });
  }
  if (client === "qwen") {
    Object.assign(env, {
      JEV_QWEN_UPSTREAM_BASE_URL: "http://127.0.0.1:8793/v1",
      JEV_QWEN_MODEL: "jevonian/auto",
      OPENAI_API_KEY: "local-no-key",
    });
  }
  return env;
}

function applyGatewayEnv(client) {
  Object.assign(process.env, gatewayEnv(client));
}

module.exports = { gatewayEnv, applyGatewayEnv };
```

</details>

<details><summary><code>jev-gateway\start-gateway.js</code> (gateway daemon launcher)</summary>

```js
"use strict";
const { join } = require("path");
const { homedir } = require("os");
const { mkdirSync } = require("fs");
const { applyGatewayEnv } = require("./gateway-env");

const client = process.argv[2] || "claude";
applyGatewayEnv(client);

const ports = { claude: 8789, opencode: 8791, kilo: 8785, qwen: 8787 };
const upstreams = {
  claude: "http://127.0.0.1:8797/v1",
  opencode: "http://127.0.0.1:8799/v1",
  kilo: "http://127.0.0.1:8795/v1",
  qwen: "http://127.0.0.1:8793/v1",
};

const port = ports[client] || 8791;
const upstream = upstreams[client] || "http://127.0.0.1:8795/v1";
const stateDir = join(homedir(), ".jev-gateway");
mkdirSync(stateDir, { recursive: true });
const logFile = join(stateDir, `${client}.log`);

process.env.PORT = String(port);
process.env.UPSTREAM_BASE_URL = upstream;
process.env.JEV_CLIENT = client;
process.env.JEV_LOG_FILE = logFile;

import("./node_modules/jev-gateway/dist/index.js");
```

</details>

<details><summary><code>jev-gateway\bin\jev-kilo.mjs</code></summary>

```js
#!/usr/bin/env node
import { runLauncher } from "../node_modules/jev-gateway/bin/launcher.mjs";

function kiloInlineConfig(origin) {
  const model = process.env.JEV_KILO_MODEL ?? "jevonian/auto";
  return {
    $schema: "https://kilo.ai/config.json",
    model: `jev-gateway/${model}`,
    small_model: `jev-gateway/${model}`,
    provider: {
      "jev-gateway": {
        npm: "@ai-sdk/openai-compatible",
        name: "Jev Gateway",
        options: { baseURL: `${origin}/v1`, apiKey: "{env:OPENAI_API_KEY}" },
        models: {
          [model]: {
            name: `Jev Gateway (${model})`,
            limit: { context: 200000, output: 65536 },
            tool_call: true,
          },
        },
      },
    },
  };
}

export const kilo = {
  name: "jev-kilo",
  client: "kilo",
  portEnv: "JEV_KILO_PORT",
  defaultPort: 8785,
  upstream: () => process.env.JEV_KILO_UPSTREAM_BASE_URL ?? "http://127.0.0.1:8795/v1",
  upstreamHelp: "JEV_KILO_UPSTREAM_BASE_URL where Kilo traffic goes (default http://127.0.0.1:8795/v1)",
  env: (origin) => ({
    KILO_CONFIG_CONTENT: JSON.stringify(kiloInlineConfig(origin)),
    PWD: process.cwd(),
  }),
  configHelp: (origin) => `Point Kilo at ${origin}/v1 using KILO_CONFIG_CONTENT`,
};

await runLauncher(kilo);
```

</details>

<details><summary><code>jev-gateway\bin\jev-qwen.mjs</code></summary>

```js
#!/usr/bin/env node
import { runLauncher } from "../node_modules/jev-gateway/bin/launcher.mjs";

export const qwen = {
  name: "jev-qwen",
  client: "qwen",
  portEnv: "JEV_QWEN_PORT",
  defaultPort: 8787,
  upstream: () => process.env.JEV_QWEN_UPSTREAM_BASE_URL ?? "http://127.0.0.1:8793/v1",
  upstreamHelp: "JEV_QWEN_UPSTREAM_BASE_URL where Qwen traffic goes (default http://127.0.0.1:8793/v1)",
  args: (origin) => [
    "--auth-type", "openai",
    "--openai-base-url", `${origin}/v1`,
    "--openai-api-key", "local-no-key",
    "-m", process.env.JEV_QWEN_MODEL ?? "jevonian/auto",
  ],
  configHelp: (origin) => `qwen --auth-type openai --openai-base-url ${origin}/v1 --openai-api-key local-no-key -m jevonian/auto`,
};

await runLauncher(qwen);
```

</details>

<details><summary><code>jev-gateway\jev-kilo.js</code></summary>

```js
#!/usr/bin/env node
"use strict";
const { join, resolve } = require("path");
const { applyGatewayEnv } = require("./gateway-env");

const JEVONIAN = process.env.JEVONIAN_DIR ? resolve(process.env.JEVONIAN_DIR) : resolve(__dirname, "..", "jevonian");
const { launch, withModel, hasModelFlag } = require(join(JEVONIAN, "lib", "launch"));
const { TARGETS } = require(join(JEVONIAN, "lib", "targets"));

const LAUNCHER_FLAGS = new Set(["--gateway-help", "--print-config", "--start", "--stop", "--status", "--logs", "--dashboard", "--routing", "--setup"]);

applyGatewayEnv("kilo");
process.env.PWD = process.cwd();
const t = TARGETS["gateway-kilo"];
launch({
  key: "gateway-kilo",
  exe: process.execPath,
  display: `jev-kilo (jev-gateway ${t.bin})`,
  wire: (argv) => ({
    args: [t.bin, ...(LAUNCHER_FLAGS.has(argv[0]) ? argv : withModel(argv, "jev-gateway/jevonian/auto"))],
    env: {},
    how: `jev-kilo, JEV_KILO_UPSTREAM_BASE_URL=http://127.0.0.1:8795/v1 (Jevonian), JEV_KILO_MODEL=jevonian/auto, -m ${hasModelFlag(argv) ? "(yours)" : "jev-gateway/jevonian/auto"}`,
  }),
});
```

</details>

<details><summary><code>jev-gateway\jev-qwen.js</code></summary>

```js
#!/usr/bin/env node
"use strict";
const { join, resolve } = require("path");
const { applyGatewayEnv } = require("./gateway-env");

const JEVONIAN = process.env.JEVONIAN_DIR ? resolve(process.env.JEVONIAN_DIR) : resolve(__dirname, "..", "jevonian");
const { launch, hasModelFlag } = require(join(JEVONIAN, "lib", "launch"));
const { TARGETS } = require(join(JEVONIAN, "lib", "targets"));

const LAUNCHER_FLAGS = new Set(["--gateway-help", "--print-config", "--start", "--stop", "--status", "--logs", "--dashboard", "--routing", "--setup"]);

applyGatewayEnv("qwen");
const t = TARGETS["gateway-qwen"];
const [first] = process.argv.slice(2);
const isLauncherFlag = LAUNCHER_FLAGS.has(first);

if (isLauncherFlag) {
  launch({
    key: "gateway-qwen",
    exe: process.execPath,
    display: `jev-qwen (jev-gateway ${t.bin})`,
    wire: (argv) => ({
      args: [t.bin, ...argv],
      env: {},
      how: "jev-qwen launcher action",
    }),
  });
} else {
  launch({
    key: "gateway-qwen",
    exe: "qwen",
    display: `jev-qwen (jev-gateway :${t.port} -> Jevonian :8793)`,
    wire: (argv) => ({
      args: [
        "--auth-type", "openai",
        "--openai-base-url", `http://127.0.0.1:${t.port}/v1`,
        "--openai-api-key", "local-no-key",
        ...(hasModelFlag(argv) ? [] : ["-m", "jevonian/auto"]),
        ...argv,
      ],
      env: {},
      how: `jev-qwen, JEV_QWEN_UPSTREAM_BASE_URL=http://127.0.0.1:8793/v1 (Jevonian), -m ${hasModelFlag(argv) ? "(yours)" : "jevonian/auto"}`,
    }),
  });
}
```

</details>

<details><summary><code>jev-gateway\jev-claude.js</code></summary>

```js
#!/usr/bin/env node
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
    args: [t.bin, ...(LAUNCHER_FLAGS.has(argv[0]) ? argv : withStandalone(argv))],
    env: {},
    how: "official jev-opencode, JEV_OPENCODE_UPSTREAM_BASE_URL=http://127.0.0.1:8799/v1 (Jevonian), JEV_OPENCODE_MODEL=jevonian/auto, --standalone",
  }),
});
```

</details>

### Step 5: the command layer

Save the nine files of [Appendix A](#appendix-a-the-command-layer-every-file-in-full) at the paths in their titles:
`jevonian\jev.js`, `jevonian\kilo.js`, `jevonian\qwen.js`, `jevonian\opencode.js`, `jevonian\claude.js`,
`jevonian\lib\targets.js`, `jevonian\lib\common.js`, `jevonian\lib\launch.js`, `jevonian\lib\monitor.js`. What they do:

- **`lib\targets.js`:** the eight servers: ports, URLs, folders, window titles.
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
- **`kilo.js` / `qwen.js` / `opencode.js` / `claude.js` / `jev-kilo.js` / `jev-qwen.js` / `jev-claude.js` / `jev-opencode.js`:** the client wiring.
- **`jev.js`:** `jev start | stop | restart | status | logs | windows | dashboards | test | install | uninstall`.

Check the syntax of all 16 scripts; each prints `ok` and its name:

```bat
cd /d %JEVAI%
for %f in (jevonian\*.js jevonian\lib\*.js jevonian\jev-router-qwen\start.js jevonian\jev-router-kilo\start.js jevonian\jev-router-claude\start.js jevonian\jev-router-opencode\start.js jev-gateway\*.js) do @node --check "%f" && echo ok %f
```

### Step 6 (optional): prove the installs are the official ones

Do this **before step 8** (the first start patches Jevonian):

```bat
mkdir %TEMP%\jevpack
cd /d %TEMP%\jevpack
npm pack jevonian@0.1.7 jev-gateway@0.4.3
mkdir jevonian jev-gateway
tar -xzf jevonian-0.1.7.tgz -C jevonian
tar -xzf jev-gateway-0.4.3.tgz -C jev-gateway
fc /b jevonian\package\dist\cli.mjs %JEVAI%\jevonian\jev-router-kilo\node_modules\jevonian\dist\cli.mjs
fc /b jev-gateway\package\dist\app.js %JEVAI%\jev-gateway\node_modules\jev-gateway\dist\app.js
```

- **Before the first start:** both comparisons print `FC: no differences encountered`. In the original build, all four
  Jevonian copies and jev-gateway were identical to the tarballs, file for file.
- **After the first start:** the Jevonian `cli.mjs` differs by exactly the patch edits ([Appendix B](#appendix-b-exact-diff-of-the-patched-jevonian-against-the-official-017)),
  and jev-gateway stays identical.

### Step 7: install the CMD commands

```bat
node %JEVAI%\jevonian\jev.js install
```

The output, from the original build. On a PC without an older copy, the first line reads
`[jev] AutoRun set; previous value saved to …`:

```text
[jev] AutoRun set (replaced 1 older jev.doskey entry); previous value saved to D:\learn\JevAI\jevonian\autorun.backup.json
[jev] macros written to D:\learn\JevAI\jevonian\jev.doskey
[jev] Open a NEW CMD window, then type:  kilo | qwen | opencode | claude   (Jevonian)
[jev]                                   jev-claude | jev-opencode     (jev-gateway -> Jevonian)   jev status
```

It writes two things:

1. **`%JEVAI%\jevonian\jev.doskey`:** plain-text CMD macros (`doskey`), with no `.bat` files. In the build (the
   `*-direct` lines point to wherever `where claude`, `where kilo`… finds the real programs on your PC):

   ```text
   kilo=node "D:\learn\JevAI\jevonian\kilo.js" $*
   qwen=node "D:\learn\JevAI\jevonian\qwen.js" $*
   opencode=node "D:\learn\JevAI\jevonian\opencode.js" $*
   claude=node "D:\learn\JevAI\jevonian\claude.js" $*
   jev-kilo=node "D:\learn\JevAI\jev-gateway\jev-kilo.js" $*
   jev-qwen=node "D:\learn\JevAI\jev-gateway\jev-qwen.js" $*
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
   - Entries that load a `jev.doskey` from another folder (an older copy) are replaced, so the commands now point to this copy.
   - `jev uninstall` removes exactly this fragment.

**Open a new CMD window now.** `doskey /macros` lists the commands. Everything from here on runs in such a window, and
doesn't need `%JEVAI%`. A doskey macro file has no comment syntax, so it has no header line; a `;` line would print
"Invalid macro definition." in every window.

**Limits of doskey macros:** they work at the **interactive CMD prompt** only. They don't work in `.bat` files, in PowerShell,
or with `cmd /d`. There, run the file directly, for example `node %JEVAI%\jevonian\kilo.js run "…"`. The commands behave the
same; the macro only saves typing.

#### PowerShell and VS Code Terminal Integration

Because modern Windows (Windows Terminal, VS Code, PowerShell 5.1 & 7+) defaults to PowerShell rather than CMD, `doskey` macros will not be available in those shells.

To make `qwen`, `kilo`, `opencode`, `claude`, `jev-kilo`, `jev-qwen`, `jev-claude`, `jev-opencode`, and `jev` work identically in any PowerShell terminal:

1. Open (or create) your PowerShell profile in Notepad:
   ```powershell
   if (!(Test-Path -Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
   notepad $PROFILE
   ```
2. Paste the following functions (adjust `$env:JEVAI` if your root folder is different):
   ```powershell
   # Jevonian & jev-gateway CLI integrations
   $env:JEVAI = "D:\learn\JevAI"
   function qwen         { & node "$env:JEVAI\jevonian\qwen.js" @args }
   function kilo         { & node "$env:JEVAI\jevonian\kilo.js" @args }
   function opencode     { & node "$env:JEVAI\jevonian\opencode.js" @args }
   function claude       { & node "$env:JEVAI\jevonian\claude.js" @args }
   function jev-kilo     { & node "$env:JEVAI\jev-gateway\jev-kilo.js" @args }
   function jev-qwen     { & node "$env:JEVAI\jev-gateway\jev-qwen.js" @args }
   function jev-claude   { & node "$env:JEVAI\jev-gateway\jev-claude.js" @args }
   function jev-opencode { & node "$env:JEVAI\jev-gateway\jev-opencode.js" @args }
   function jev          { & node "$env:JEVAI\jevonian\jev.js" @args }
   ```
3. Save and open a new PowerShell terminal. You can now use all commands natively!

### Step 8: start everything

```bat
jev start
```

Output from the build, starting all 8 servers:

```text
[jev] qwen         :8793  started
[jev] kilo         :8795  started
[jev] claude       :8797  started
[jev] opencode     :8799  started
[jev] jev-kilo     :8785  started
[jev] jev-qwen     :8787  started
[jev] jev-claude   :8789  started
[jev] jev-opencode :8791  started
```

If a line says `already running` on a fresh build, an older copy still owns that port: go back to step 0.

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

The Claude router also prints `Haiku patch: applied` and `models: auto-sync disabled`. Then check with `jev status`
(the status window column says `open` only once you've done step 10):

```text
  name           command        port   state  status window   web page
  qwen           qwen           8793   UP     open            http://127.0.0.1:8793/  (+ /logs)
  kilo           kilo           8795   UP     open            http://127.0.0.1:8795/  (+ /logs)
  claude         claude         8797   UP     open            http://127.0.0.1:8797/  (+ /logs)
  opencode       opencode       8799   UP     open            http://127.0.0.1:8799/  (+ /logs)
  jev-kilo       jev-kilo       8785   UP     open            http://127.0.0.1:8785/dashboard?peers=none
  jev-qwen       jev-qwen       8787   UP     open            http://127.0.0.1:8787/dashboard?peers=none
  jev-claude     jev-claude     8789   UP     open            http://127.0.0.1:8789/dashboard?peers=none
  jev-opencode   jev-opencode   8791   UP     open            http://127.0.0.1:8791/dashboard?peers=none

  Upstreams: qwen, kilo, opencode -> Alibaba Token Plan; claude -> Anthropic (your Claude Code login)
             jev-kilo -> Jevonian kilo :8795; jev-qwen -> Jevonian qwen :8793
             jev-claude -> Jevonian claude :8797; jev-opencode -> Jevonian opencode :8799
  CMD commands: installed (new CMD windows have them)
```

### Step 9: test everything

```bat
jev test
```

It takes about 5 minutes and runs three groups. The output is also saved to `jevonian\run\`:

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

   The full client output goes to `run\test-output\`. Run `set JEV_TEST_WINDOWS=1` first (after step 10) to watch it live in
   the status windows.

The real results of the original build. Times, costs, `JEV-NOTES-…` values and, for `jevonian/auto`, Jev's tier picks will
differ in yours; every line must say `PASS`:

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

### Step 10: open the eight status windows

```bat
jev windows
```

This opens eight console windows: `JEVONIAN - QWEN - :8793`, `JEVONIAN - KILO - :8795`, `JEVONIAN - CLAUDE - :8797`,
`JEVONIAN - OPENCODE - :8799`, `JEV-GATEWAY - KILO - :8785`, `JEV-GATEWAY - QWEN - :8787`, `JEV-GATEWAY - CLAUDE - :8789` and `JEV-GATEWAY - OPENCODE - :8791`.
- Each shows its header (section 7) and the requests so far.
- Closing a window never stops a router.
- The CMD commands also open their own window when you use them.

### Step 11: check every tool in the browser (nine tabs)

Open these **nine tabs**, one per web server plus the combined gateway page. Paste the URLs, or run `jev dashboards`.
`jev dashboards` opens tabs 5 to 8 as listed, and tabs 1 to 4 on the router's **Overview** page (`/`); click **Logs**
in its left menu. Open tab 9 by hand.

| Tab | Tool (command) | URL |
|---|---|---|
| 1 | Qwen Code (`qwen`) | http://127.0.0.1:8793/logs |
| 2 | Kilo (`kilo`) | http://127.0.0.1:8795/logs |
| 3 | Claude Code through Jevonian (`claude`), and the Jevonian half of `jev-claude` | http://127.0.0.1:8797/logs |
| 4 | OpenCode through Jevonian (`opencode`), and the Jevonian half of `jev-opencode` | http://127.0.0.1:8799/logs |
| 5 | Kilo Code through jev-gateway (`jev-kilo`) | http://127.0.0.1:8785/dashboard?peers=none |
| 6 | Qwen Code through jev-gateway (`jev-qwen`) | http://127.0.0.1:8787/dashboard?peers=none |
| 7 | Claude Code through jev-gateway (`jev-claude`) | http://127.0.0.1:8789/dashboard?peers=none |
| 8 | OpenCode through jev-gateway (`jev-opencode`) | http://127.0.0.1:8791/dashboard?peers=none |
| 9 | all gateways on one page | http://127.0.0.1:8789/dashboard |

Then send one request per tool: `jev test clients` does all eight. Or, from any project folder, one tool at a time:
`qwen -p "Reply with exactly: QWEN_OK"`, `kilo run "Reply with exactly: KILO_OK"`, `opencode run "Reply with exactly: OPENCODE_OK"`,
`claude -p "Reply with exactly: CLAUDE_OK"`, `jev-kilo run "Reply with exactly: JEV_KILO_OK"`,
`jev-qwen -p "Reply with exactly: JEV_QWEN_OK"`, `jev-claude -p "Reply with exactly: JEV_CLAUDE_OK"`,
`jev-opencode run "Reply with exactly: JEV_OPENCODE_OK"`. Refresh the tabs and check:

**Tabs 1 to 4 (Jevonian, the Logs page):**
- **A new row per request, at the time you sent it,** with columns TIME, MODEL, PROVIDER, PHASE, EFFORT, STATUS, COST,
  LATENCY:
  - PHASE is the tier Jev chose.
  - MODEL and EFFORT must be that tier's model and effort from [the tier tables](#the-tiers): for example `chat` →
    deepseek-v4.1-flash · low, or `utility` → claude-sonnet-5 · medium.
  - STATUS must be 200.
  - PROVIDER is `Alibaba Tokenplan` (tabs 1, 2, 4), or `Claude` / `Claude Subscription Haiku` (tab 3).
- **"details →" on a row** opens the decision:
  - provider · model, "you requested" (`jevonian/auto` or a pinned tier), phase, thinking effort, and the reason
    (e.g. `brain:chat`)
  - the Jev brain call with its verdict and the probability of every tier
  - the captured prompt
- **The same rows as the tool's status window** (section 7): same time, phase, model, effort, status, cost and latency.

**Tabs 5 to 8 (jev-gateway, one tool each):**
- **The card** (`kilo · :8785`, `qwen · :8787`, `claude · :8789`, or `opencode · :8791`):
  - a status of **Routing** or **Passthrough only**, never "Jev is failing" or "Offline"
  - the upstream line pointing to the corresponding Jevonian router
  - `Jev model jev-latest via typesafe`
- **"Recent requests":** a row per request, with Client, **Model** (with PR #50: displays the concrete served model, e.g. `qwen3.8-flash` or `glm-5.3`, alongside requested `jevonian/auto`), Tools, Mode (`hint`, `none`, `passthrough`, or `forced`), Tool, Conf., Reason and Status 200.
- **The same request on the Jevonian tab behind it:** appears in the router's logs 1 to 2 s later.

**Tab 9:** all cards side by side on the combined dashboard page.

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

jev-kilo                                 :: Kilo Code through jev-gateway -> Jevonian
jev-qwen                                 :: Qwen Code through jev-gateway -> Jevonian
jev-claude --dangerously-skip-permissions   :: Claude Code through jev-gateway -> Jevonian
jev-opencode                             :: OpenCode through jev-gateway -> Jevonian; one-shot: jev-opencode run "…"
```

**Pin a tier** (skip Jev's choice):

| Tool | Command |
|---|---|
| Qwen Code | `qwen -m jevonian/large` |
| Kilo | `kilo -m jevonian/jevonian/plan` |
| OpenCode | `opencode -m jevonian/jevonian/small` |
| Claude Code | `claude --model jevonian/plan` (plan, heavy, execute, utility, chat) |
| jev-kilo | `jev-kilo -m jevonian/large` |
| jev-qwen | `jev-qwen -m jevonian/plan` |
| jev-claude | `jev-claude --model jevonian/heavy` |
| jev-opencode | `jev-opencode -m jevonian/jevonian/plan` |

**Manage everything:**

| Command | Does |
|---|---|
| `jev status` | Up/down, status window and web page of all eight targets. Also whether the CMD commands are installed. |
| `jev start` / `jev stop` / `jev restart` `[name]` | All eight, or one (`qwen`, `kilo`, `claude`, `opencode`, `jev-kilo`, `jev-qwen`, `jev-claude`, `jev-opencode`). A gateway also starts the Jevonian router behind it. Stop does the gateways first. |
| `jev logs <name>` | The last 40 lines of that router's `logs\serve.log`, or the gateway's `%USERPROFILE%\.jev-gateway\<client>.log`. |
| `jev windows` / `jev windows close` | Open or close all eight status windows. |
| `jev dashboards` | Open all eight web pages in the browser. |
| `jev test [brains\|tiers\|clients] [name]` | The checks from step 9. |
| `jev-claude --status` / `--dashboard` / `--routing off` / `--routing on` / `--stop` | The jev-gateway launcher flags, passed through unchanged (supported on `jev-kilo`, `jev-qwen`, `jev-claude`, `jev-opencode`). |

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

Every command opens (or reuses) one console window per server, titled like `JEVONIAN - KILO - :8795` or `JEV-GATEWAY - KILO - :8785`. Real content of the
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
| jev-gateway lines | the gateway's `/dashboard/events`: the same data as its web page (mode, tool Jev picked, served model, confidence, tokens) |
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
- **Closing a window never stops its router.** `jev windows close` closes all eight.

---

## 8. Web pages: one per tool

Each of the eight servers is its own web server on 127.0.0.1. [Step 11](#step-11-check-every-tool-in-the-browser-seven-tabs)
lists the tabs to open and what to check on each.

| Tool | Server | Page | What it shows |
|---|---|---|---|
| `qwen` | Jevonian | http://127.0.0.1:8793/logs (Overview at `/`) | every request: time, model, provider, phase (tier), **effort**, status, cost, latency; "details" shows the prompt and Jev's decision |
| `kilo` | Jevonian | http://127.0.0.1:8795/logs | the same |
| `claude` | Jevonian | http://127.0.0.1:8797/logs | the same; also `jev-claude`'s turns, which pass through it |
| `opencode` | Jevonian | http://127.0.0.1:8799/logs | the same; also `jev-opencode`'s turns |
| `jev-kilo` | jev-gateway | http://127.0.0.1:8785/dashboard?peers=none | gateway status, tool decisions, tokens, and live requests showing served model (PR #50) |
| `jev-qwen` | jev-gateway | http://127.0.0.1:8787/dashboard?peers=none | gateway status, tool decisions, tokens, and live requests showing served model (PR #50) |
| `jev-claude` | jev-gateway | http://127.0.0.1:8789/dashboard?peers=none | gateway status, tool decisions, tokens, and live requests showing served model (PR #50) |
| `jev-opencode` | jev-gateway | http://127.0.0.1:8791/dashboard?peers=none | gateway status, tool decisions, tokens, and live requests showing served model (PR #50) |
| all gateways | jev-gateway | http://127.0.0.1:8789/dashboard | combined page with all active gateway cards side by side |

- **jev-gateway's page is only at `/dashboard`.** Its root `http://127.0.0.1:<port>/` answers **404 "Not found"** by design.
- **Without `?peers=none`**, the page also shows every other gateway it finds on the official ports. `?peers=none` is jev-gateway's own option for "only this gateway".
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

**12. Windows Job Objects & Background Process Lifecycles:**
- `jev start` and `start.js` launch background processes using Node's `spawn(..., { detached: true })`. This works smoothly in interactive CMD/PowerShell windows.
- However, in automated scripts, CI pipelines, subshell wrappers, or terminal panes that close immediately upon command completion, Windows Job Objects may automatically terminate any child processes attached to that session.
- To keep the routers and gateways running persistently:
  - Keep a status terminal open via `jev windows`, or
  - Start them inside a dedicated long-running console window, or
  - Manage them as background services / daemons (e.g. via PM2 or NSSM if running headless on a server).

**13. Non-interactive Automation & Stdin Redirection:**
- When running one-shot commands (`opencode run`, `claude -p`) from automated scripts or tools without an interactive TTY, some clients hang waiting for user input on `stdin`.
- Always close or redirect standard input when executing non-interactively:
  - **In CMD:** `node %JEVAI%\jevonian\opencode.js run "prompt" < nul`
  - **In PowerShell:** `$null | node $env:JEVAI\jevonian\opencode.js run "prompt"`
  - **In Node.js:** Use `spawnSync(..., { stdio: ["ignore", "inherit", "inherit"] })` or `{ input: "" }`.

---

## 12. Troubleshooting

First check `jev status` (what's up), `jev logs <name>` (why a router or gateway didn't start), and `jev test brains`
(the keys).

| Symptom | Cause | Fix |
|---|---|---|
| `kilo`, `claude`… open the plain client, or "is not recognized" | The CMD window was opened before `jev install`, or you're in PowerShell / VS Code terminal | In CMD: open a **new** window. In PowerShell: add functions to `$PROFILE` (see [PowerShell integration](#powershell-and-vs-code-terminal-integration)). |
| "Invalid macro definition." in every new CMD window | A line in `jev.doskey` that isn't `name=command` (e.g. a comment) | Run `jev install` again (it rewrites the file) |
| Routers exit immediately after `jev start` when script closes | Terminal runner / Windows Job Object terminates detached children | Run routers in a dedicated persistent console window or with `jev windows` (see known behaviour 12) |
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
| `opencode run` hangs with nothing in the ledger (scripts) | `run` reads stdin when it isn't a terminal | Close stdin: add `< nul` in CMD or pipe `$null \|` in PowerShell (see known behaviour 13) |
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

The commands below use `%JEVAI%`: run `set JEVAI=<your root>` first in that CMD window, as in [step 0](#step-0-choose-the-root-folder-and-stop-any-other-copy).

### Jevonian (one router at a time; each has its own copy)

1. **Read what changed:** `npm view jevonian version`, then https://github.com/xinyao27/jevonian/releases. Look for changes to
   routing or effort (`withEffort`, `clientEffortOf`, `parseRoutingEntry`), the brain state (WAF patch), and Claude
   Code / Haiku handling. **If a release adds per-route effort, a client-effort override, or Haiku handling officially,
   delete that patch file instead of updating it.**
2. **Stop the router:** `jev stop kilo`.
3. **Install, exact version, in that folder only:**
   `cd /d %JEVAI%\jevonian\jev-router-kilo && npm install jevonian@X.Y.Z --save-exact`.
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
cd /d %JEVAI%\jev-gateway && npm install jev-gateway@X.Y.Z --save-exact
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
  2. Delete `%JEVAI%\jevonian` and `%JEVAI%\jev-gateway`.
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
| jev-gateway | 0.4.3 (npm latest) | enhanced with Kilo Code and Qwen Code integrations and PR #50 served-model display |
| Claude Code / OpenCode / Kilo / Qwen Code | 2.1.282 / 2.0.15 / 7.7.9 / 0.24.4 | all 4 coding agents answered in both direct Jevonian router mode and jev-gateway mode |

| Test | Result |
|---|---|
| First start after fresh installs | all eight servers up; every patch `applied` (WAF, Effort 5 edits, Haiku on Claude) |
| `jev test brains` | **8/8**: TypeSafe (`jev-latest`, ≈0.7 s) and the Vercel AI Gateway fallback (`typesafe-ai/jev`, ≈0.65 s) on all four routers |
| `jev test tiers` | **23/23**: every Alibaba tier (3 routers × 6) and every Claude tier (5) served by its configured model at its configured effort |
| `jev test clients` | **16/16**: all 8 commands (`qwen`, `kilo`, `opencode`, `claude`, `jev-kilo`, `jev-qwen`, `jev-claude`, `jev-opencode`) answered a marker and read a file through their router or gateway |
| Gateway Tool Steering | **Verified live**: Kilo evaluated 53 tools; Qwen steered in `mode: forced` with 94% confidence; Claude steered in `mode: hint`; OpenCode routed tools |
| Served Model Display (PR #50) | **Verified live**: `/dashboard` displays actual concrete served models (e.g. `qwen3.8-flash`, `deepseek-v4.1-flash`, `glm-5.3`) alongside requested alias `(jevonian/auto)` |
| The three views | For every tool, the status window and the web page showed the same rows (section 9) |
| Restart | The `JEV-GATEWAY` windows follow gateway restarts and keep showing new requests (section 7) |
| Same folder | Both official packages installed and ran from one folder; one request went gateway → Jevonian → Alibaba (section 10) |
| Browser | All nine dashboard pages opened in tabs and matched the status windows (section 8) |

---

## 16. Jev-Router for Claude Code — Complete Setup, Customization & Standalone Architecture

This section documents the standalone **Jev-Router for Claude Code** implementation, detailing how Claude Code is routed turn-by-turn through TypeSafe AI Jev, how the 6-tier matrix rewrites models and reasoning effort, how the Web UI dashboard displays workspace telemetry with path locations, and how to execute `claude --dangerously-skip-permissions` directly from plain Windows CMD without invoking `jev-*.js` wrapper scripts manually.

---

### 16.1 Architecture & Direct Integration: Why Claude Code Differs

Unlike standard OpenAI-compatible coding agents, **Claude Code (v2.1.282+)** has unique characteristics:

1. **Native Anthropic OAuth Authentication:**  
   Claude Code logs in via browser OAuth (`claude login`), storing tokens in `%USERPROFILE%\.claude\.credentials.json`. It does not require a raw `ANTHROPIC_API_KEY` in `.env`. The router forwards the user's authentic credential upstream to `https://api.anthropic.com`.
2. **Dynamic Ephemeral Reverse Proxy:**  
   When a session starts, Jev-Router spawns a local loopback proxy on an ephemeral port (e.g. `http://127.0.0.1:51234`).
3. **Environment Injection:**  
   Claude Code is started with:
   - `ANTHROPIC_BASE_URL=http://127.0.0.1:<proxy_port>` (redirects Claude Code's network requests to the proxy).
   - `ANTHROPIC_CUSTOM_MODEL_OPTION=jev-router` (adds Jev-Router into Claude Code's model picker).
   - `ANTHROPIC_MODEL=jev-router` (sets Jev-Router as the active model for the session).
   - `CLAUDE_CODE_DISABLE_UNKNOWN_MODEL_WINDOW_ENFORCEMENT=1` (allows the virtual model name to pass client-side validation).
   - `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1` (enables model discovery from the proxy).
4. **Per-Turn Dynamic Rewriting:**  
   On every user turn, Claude Code sends `POST /v1/messages` with `model: "jev-router"`. The proxy intercepts the prompt and tool definitions, evaluates them using TypeSafe AI Jev (`@typesafe-ai/sdk`), rewrites the model to the optimal Claude model (Haiku 4.5, Sonnet 5, Opus 5.5), injects or strips reasoning effort, forwards the payload to Anthropic, and streams tokens back to the user's terminal while recording telemetry.

```text
[Windows CMD]
   │
   ▼
[claude.cmd] (Current Directory)
   │
   ▼
[jev-claude-router.js] ──► Reads workspace .env (TYPESAFE_API_KEY, JEV_API_KEY)
   │
   ├─► Starts local dynamic reverse proxy (127.0.0.1:ephemeral_port)
   ├─► Configures terminal statusline (bin/jev-statusline.mjs)
   │
   ▼
[claude.exe] (Official binary with ANTHROPIC_BASE_URL=proxy, MODEL=jev-router)
   │
   ▼ (Sends /v1/messages with model: "jev-router")
[Jev Reverse Proxy]
   │
   ├─► Calls TypeSafe Jev Brain (askJev / @typesafe-ai/sdk)
   │   (Scores: taskComplexity, reasoningRequired, toolComplexity, contextSize)
   │
   ├─► Rewrites model: "jev-router" ──► "claude-sonnet-5" / "claude-opus-5-5" / "claude-haiku-4-5"
   ├─► Sets / strips reasoning effort (none / low / medium / high / xhigh)
   │
   ├─► Forwards upstream to https://api.anthropic.com with client credentials
   ├─► Streams response tokens back to Claude Code terminal
   └─► Logs telemetry with cwd/project to %TEMP%\jev-claude\events.jsonl
         ▲
         │ (Polls events.jsonl every 1s)
[Web UI Dashboard] (http://127.0.0.1:8790 or 8792)
```

---

### 16.2 The 6-Tier Model & Reasoning Effort Routing Matrix

Configured in `src/config.mjs`, Jev-Router dynamically routes each turn to one of six specialized tiers:

| Tier Name | Target Model ID | Family | Thinking | Reasoning Effort | Task Classification & Typical Workloads |
|---|---|---|---|---|---|
| **`chat`** | `claude-haiku-4-5-20251001` | Haiku | `false` | `null` | **Conversations & Greetings:** Short chat replies, acknowledgments, general non-code questions. |
| **`small`** | `claude-sonnet-5` | Sonnet | `true` | `low` | **Small Tasks:** Well-scoped 1–5 line bug fixes, simple script runs, variable renames, typo fixes. |
| **`utility`** | `claude-sonnet-5` | Sonnet | `true` | `medium` | **Utilities & Lookups:** Summarizing files, doc lookups, listing APIs, mechanical checks, architecture walkthroughs. |
| **`medium`** | `claude-sonnet-5` | Sonnet | `true` | `high` | **Standard Implementation:** Multi-file feature engineering, multi-step debugging across related files, tool loops. |
| **`plan`** | `claude-opus-5-5` | Opus | `true` | `high` | **Architecture & Planning:** System architecture, database schema design, multi-module specifications, roadmaps. |
| **`heavy`** | `claude-opus-5-5` | Opus | `true` | `xhigh` | **Complex Refactoring & Deep Bugs:** Difficult concurrency/race conditions, distributed transactions, high blast-radius changes. |

#### Scoring Dimensions Evaluated by TypeSafe Jev:
- **`taskComplexity` (0.0 to 1.0):** Overall structural difficulty of the requested operation.
- **`reasoningRequired` (0.0 to 1.0):** Depth of logic, cross-file deduction, and planning needed.
- **`toolComplexity` (0.0 to 1.0):** Number and types of MCP / file tools required to execute the turn.
- **`contextSize` (0.0 to 1.0):** Approximate token length of conversation history and loaded files.
- **`minConfidence` (default: 0.3):** If classification confidence falls below 0.3, the router falls back to the baseline model without breaking.

---

### 16.3 Downloading & Setting Up Jev-Gateway & gargpratyush-jev-router

To set up the environment in any workspace folder (e.g. `D:\learn\gemini-mcp\gemini-blogdee-subdomain` or `D:\learn\graduated_project`):

#### 1. Download / Clone the Repositories
```bash
# Clone jev-gateway (tool steering gateway)
git clone https://github.com/vinilana/jev-gateway.git

# Clone gargpratyush-jev-router (Claude Code model router)
git clone https://github.com/gargpratyush/jev-router.git gargpratyush-jev-router
```

#### 2. Install Dependencies
```bash
cd gargpratyush-jev-router
npm install --no-audit --no-fund
cd ..

cd jev-gateway
npm install --no-audit --no-fund
cd ..
```

#### 3. Workspace File Layout
```text
<workspace_root>\
├── .claude\
│   └── settings.local.json     <-- Pre-approves workspace MCP servers
├── .env                        <-- JEV_API_KEY, TYPESAFE_API_KEY, JEV_DASHBOARD_PORT
├── .mcp.json                   <-- Workspace-scoped MCP servers
├── CLAUDE.md                   <-- Guidance for Claude Code sessions
├── claude.cmd                  <-- Local CMD redirect to jev-claude-router.js
├── package.json                <-- Workspace npm scripts
├── jev-claude-router.js        <-- Launcher: Claude Code with Jev-Router
├── jev-dashboard-router.js     <-- Launcher: Standalone Web Dashboard
├── jev-explain-router.js       <-- Inspector: Explain last routing decision
├── jev-history-router.js       <-- Inspector: Tail recent routing events
├── gargpratyush-jev-router\    <-- Router source and dashboard assets
└── memory\
    └── memory.jsonl            <-- Isolated workspace knowledge graph
```

---

### 16.4 Customizing Jev-Router for Claude Code

Several production fixes and enhancements were applied directly to the router codebase:

#### 1. Per-Turn Reasoning Effort Fix (`src/proxy.mjs`)
* **Problem:** Anthropic Sonnet 5 and Haiku 4.5 throw HTTP 400 if per-turn `effort` is passed in `output_config`. Only Opus models support adaptive thinking effort.
* **Fix:** In `src/proxy.mjs`, when rewriting to non-Opus models (`family !== "opus"`), `effort` is stripped from `output_config`, preventing HTTP 400 errors while preserving adaptive thinking.

#### 2. Workspace Location & Telemetry Tracking (`src/proxy.mjs` & `src/status.mjs`)
* **Problem:** Telemetry events only logged `path: "/v1/messages"` without tracking which local workspace directory triggered the request.
* **Fix:** Injected `cwd: process.cwd()` and `project: basename(process.cwd())` into `routeEventData` in `src/proxy.mjs` and `logRouteEvent()` in `src/status.mjs`.

#### 3. Dynamic Terminal Statusline (`bin/jev-statusline.mjs`)
* Hooked into Claude Code via `--settings` with a custom command statusline.
* Renders the actively routed model (e.g. `sonnet 5 [high]` or `opus 5.5 [high]`), routing latency (ms), and confidence in the terminal prompt.

#### 4. Radix UI Modern Web Dashboard (`src/dashboard.html` & `bin/jev-dashboard.mjs`)
* **Workspace Path Badge in Header:** Displays `📁 <workspace_path>` next to the `live` status badge.
* **Router Card Metadata:** Displays `📁 Path: <workspace_path>` in the router status card.
* **Location Column in Table:** Displays `📁 <project_name>` in the Recent Requests table with hover tooltip revealing full path and turn details.
* **Thailand Time (24h format):** Renders all event timestamps in ICT (UTC+7 / Asia/Bangkok, `HH:mm:ss`).
* **Automatic `.env` Loading:** Calls `process.loadEnvFile()` in `bin/jev-dashboard.mjs` to auto-detect `JEV_DASHBOARD_PORT`.

---

### 16.5 Enabling Direct Execution in Plain Regular Windows CMD

#### The Problem: Why `jev-claude` Was Previously Required
Previously, typing `claude` in regular Windows CMD invoked the global `claude.cmd` located in Node's directory (`C:\nvm4w\nodejs\claude.cmd`), which bypassed Jev-Router or had a hardcoded path to a single folder.

#### The Solution: Local & Dynamic CMD Dispatchers

1. **Local Project Dispatcher (`claude.cmd` in Project Root):**
   Create a [`claude.cmd`](file:///D:/learn/graduated_project/claude.cmd) in each project root:
   ```bat
   @ECHO off
   node "%~dp0jev-claude-router.js" %*
   ```
2. **Smart Global Dispatcher (`C:\nvm4w\nodejs\claude.cmd`):**
   Update the global `claude.cmd` to inspect the current working directory (`%CD%`):
   ```bat
   @ECHO off
   IF EXIST "%CD%\jev-claude-router.js" (
     node "%CD%\jev-claude-router.js" %*
   ) ELSE IF EXIST "%CD%\gargpratyush-jev-router\bin\jev-claude.mjs" (
     node "%CD%\gargpratyush-jev-router\bin\jev-claude.mjs" %*
   ) ELSE (
     node "D:\learn\gemini-mcp\gemini-blogdee-subdomain\gargpratyush-jev-router\bin\jev-claude.mjs" %*
   )
   ```

#### Why This Works in Windows CMD:
Under Windows Command Prompt command precedence rules, CMD searches the **current directory (`%CD%`)** before searching directories in `%PATH%`. Therefore:
```cmd
D:\learn\graduated_project>claude --dangerously-skip-permissions
```
1. CMD executes `D:\learn\graduated_project\claude.cmd`.
2. It forwards execution to `jev-claude-router.js` in that folder.
3. The local proxy starts with that project's `.env` and `.mcp.json`.
4. Claude Code runs completely routed, requiring **zero extra typing**.

---

### 16.6 Configuration, Credentials & Environment Isolation

#### Workspace `.env` File
Create `.env` in the root of the workspace:
```env
# TypeSafe Jev Router Configuration for Claude Code
JEV_API_KEY=apikey_2667b02557855514355b68885339c5481be_d88b96a997cdbe72e8d3e59d4111ccbcb7a01a8bf4c18f1622c76c81ecd2431b
TYPESAFE_API_KEY=apikey_2667b02557855514355b68885339c5481be_d88b96a997cdbe72e8d3e59d4111ccbcb7a01a8bf4c18f1622c76c81ecd2431b
JEV_DEBUG=1
JEV_DASHBOARD_PORT=8790   # Use 8790 for subdomain, 8792 for graduated_project
```

#### Authentication Rules:
* **TypeSafe AI Brain:** Uses `TYPESAFE_API_KEY` (or `JEV_API_KEY`) to query TypeSafe routing models.
* **Anthropic API:** Uses the user's existing Claude Code OAuth login (`claude login`). No Anthropic key is stored in `.env`.

---

### 16.7 Workspace-Scoped MCP Servers vs. Global Pollution

To ensure clean isolation and avoid global tool pollution:

1. **Define Tools in `.mcp.json` (`Project MCPs`):**
   Put all MCP servers in `.mcp.json` in the workspace root. They will only be loaded when working in this project.
2. **Pre-Approve in `.claude/settings.local.json`:**
   ```json
   {
     "enabledMcpjsonServers": [
       "playwright",
       "playwright-chrome",
       "read-website-fast",
       "open-websearch",
       "memory",
       "ftp-server",
       "mysql-database",
       "context7",
       "ref",
       "stitch",
       "autonews-web-hosting-mcp"
     ]
   }
   ```
3. **Isolate State Storage:**
   Configure stateful tools (e.g. `memory`) with workspace-specific paths:
   ```json
   "env": {
     "MEMORY_FILE_PATH": "D:\\learn\\graduated_project\\memory\\memory.jsonl"
   }
   ```
4. **Prevent `~/.claude.json` Pollution:**
   Ensure `projects[<path>].mcpServers` in `C:\Users\<user>\.claude.json` is set to `{}` so Claude Code does not classify project tools as "Local MCPs".

---

### 16.8 Running Dedicated Dashboards on Separate Ports

To monitor multiple workspaces simultaneously without port collisions (`EADDRINUSE`):

| Workspace | Dashboard Port | Web URL | Background Daemon Command |
|---|---|---|---|
| **`gemini-blogdee-subdomain`** | **`8790`** | `http://127.0.0.1:8790/` | `node jev-dashboard-router.js 8790` |
| **`graduated_project`** | **`8792`** | `http://127.0.0.1:8792/` | `node jev-dashboard-router.js 8792` |

Both dashboards can run 24/7 side-by-side in separate browser tabs.

---

### 16.9 Replicating to Any New Workspace (5-Minute Checklist)

To add Claude Code + Jev-Router to any new directory (e.g. `D:\learn\my-new-project`):

1. **Copy Router Source:**  
   Copy `gargpratyush-jev-router` into `D:\learn\my-new-project\gargpratyush-jev-router`.
2. **Copy Runner Scripts:**  
   Copy `jev-claude-router.js`, `jev-dashboard-router.js`, `jev-explain-router.js`, `jev-history-router.js`, and `claude.cmd` to the project root.
3. **Create `.env`:**  
   Set `TYPESAFE_API_KEY` and pick a unique `JEV_DASHBOARD_PORT` (e.g. `8794`).
4. **Create `.mcp.json` & `.claude/settings.local.json`:**  
   Define project tools and adjust any file paths (such as `memory.jsonl`).
5. **Launch:**  
   Open plain Windows CMD in the project folder and run:
   ```cmd
   D:\learn\my-new-project>claude --dangerously-skip-permissions
   ```
   Claude Code will automatically start, route through Jev, and display live telemetry on your designated dashboard port.

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
//                                jev-kilo, jev-qwen, jev-claude, jev-opencode -> jev-gateway launchers -> Jevonian
//                                jev                             -> this script
//                                claude-direct, opencode-direct, kilo-direct, qwen-direct -> the real client, no router
//   jev uninstall              undo it (restores any AutoRun value you had before)
//   jev status                 every router/gateway: port, up/down, status window, web page
//   jev start   [name]         start routers + gateways (default: all eight; a gateway also starts its Jevonian router)
//   jev stop    [name]         stop them (default: all eight)
//   jev restart [name]
//   jev logs    <name>         the last 40 lines of that router's / gateway's log
//   jev windows [close]        open (or close) every status window
//   jev dashboards             open every web page in the browser
//   jev test [brains|tiers|clients] [name]   check everything against the configs (default: all three)
// name = qwen | kilo | claude | opencode | jev-kilo | jev-qwen | jev-claude | jev-opencode
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
    `jev-kilo=${n(join(GATEWAY, "jev-kilo.js"))}`,
    `jev-qwen=${n(join(GATEWAY, "jev-qwen.js"))}`,
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
    "gateway-kilo": (p) => [join(GATEWAY, "jev-kilo.js"), "run", p],
    "gateway-qwen": (p) => [join(GATEWAY, "jev-qwen.js"), "-p", p],
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
  "gateway-kilo": gateway("kilo", 8785, `Jevonian Kilo router :8795 (tier + effort) -> ${ALIBABA}`),
  "gateway-qwen": gateway("qwen", 8787, `Jevonian Qwen router :8793 (tier + effort) -> ${ALIBABA}`),
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
