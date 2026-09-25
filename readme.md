# Jevonian + jev-gateway: Complete Guide (Windows CMD)

**OpenCode, Kilo, Qwen Code and Claude Code through [Jevonian](https://github.com/xinyao27/jevonian), plus
Claude Code and OpenCode through [jev-gateway](https://github.com/vinilana/jev-gateway). You type plain
`kilo` / `qwen` / `opencode` / `claude` in CMD. No `.bat` files.**

Both projects are used **as officially released** (jevonian **0.1.7**, jev-gateway **0.4.3**), each installed locally
with an exact version pin:
- **jev-gateway is not modified at all.**
- Jevonian gets three small, idempotent patches ([section 4](#4-the-patches-small-idempotent-each-checks-before-it-writes)),
  re-applied automatically on every start.

Anything bigger would break on the next official upgrade, and that's the rule this guide follows.

> **Last verified 2026-09-25** on Windows 11, Node v22.23.2, OpenCode 2.0.15, Kilo 7.7.9, Qwen Code 0.24.4 and
> Claude Code 2.1.282. The results are in [Proof of Working](#proof-of-working).

---

## Quick start

| You type in CMD | Goes through | Port | Upstream |
|---|---|---|---|
| `kilo` | Jevonian | 8795 | Alibaba Token Plan |
| `qwen` | Jevonian | 8793 | Alibaba Token Plan |
| `opencode` | Jevonian | 8799 | Alibaba Token Plan |
| `claude` | Jevonian (its official `jevonian launch claude`) | 8797 | Anthropic, your Claude login (no API key) |
| `jev-claude` | jev-gateway (official launcher) → Jevonian | 8789 → 8797 | Anthropic, your Claude login |
| `jev-opencode` | jev-gateway (official launcher) → Jevonian | 8791 → 8799 | Alibaba Token Plan |
| `jev status` / `jev start` / `jev stop` / `jev windows` / `jev dashboards` | everything above | | |
| `claude-direct`, `opencode-direct`, `kilo-direct`, `qwen-direct` | the real client, no router | | |

```bat
:: one-time: make the commands exist in every NEW CMD window (no .bat files; see the CMD commands section)
node D:\learn\gemini-mcp\agy-opencode-jev\cli\jev.js install

:: then, in any project folder, in a NEW CMD window:
claude --dangerously-skip-permissions
kilo
jev status
```

When you run a command:
1. It **starts its router with `node` if it's down.**
2. The **first time** it starts one, it **opens that router's dashboard in your browser**.
3. It opens (or reuses) a **status window** showing the port, URL, provider, the tier → model → effort table, and a live
   line per request.
4. The client runs in *your* window and folder.

---

## Jevonian or jev-gateway: which one?

They answer **different questions**, and they can be stacked:

| | **Jevonian** | **jev-gateway** |
|---|---|---|
| Jev decides | **which model and effort** serves this turn (tier routing) | **which tool** the agent should call next (tool routing) |
| Saves money by | sending easy turns to cheap models (Haiku, DeepSeek, Qwen Flash) | steering tool choice; the model stays the same |
| Clients here | OpenCode, Kilo, Qwen Code, Claude Code | Claude Code, OpenCode |
| Commands | `opencode`, `kilo`, `qwen`, `claude` | `jev-claude`, `jev-opencode` (these **also** go through Jevonian) |

**Use the plain commands (Jevonian) by default.** Tier routing is where the savings are.
Use **`jev-claude` / `jev-opencode`** to *add* Jev's tool routing on top. That helps most with very large tool lists and
debugging-style work. jev-gateway's own benchmark shows gains on debugging, but **worse** results for Opus 5 / Sonnet 5 on
feature work, so measure your own tasks (`jev-claude --routing off` / `on`).

Qwen Code's comparison (shared chat, 2026-09-23) reached the same conclusion: *"Jevonian is better for Kilo/Qwen, jev-gateway
for Claude Code… complementary, not competing."* It also said Jevonian can't do Claude Code. That's wrong:
Jevonian has an official `jevonian launch claude`.

---

## FAQ: starting, stopping, dashboards

**Do I start the routers by hand?** No. Every command checks its router's health URL. If nothing answers, it starts it
with `node` in the background (Jevonian: `jevnonian\jev.js start <name>`; jev-gateway: its official `--start`), and
`jev-claude` / `jev-opencode` start their Jevonian router too. Starting takes about 2 seconds, once. After that, every command
reuses the running router.

**Why didn't a browser open before?** The routers were running, but the guide's `start.js` sets `JEVONIAN_NO_OPEN=1` so that
`jev start` doesn't open four tabs, and jev-gateway only opens its dashboard with `--dashboard`. Now, when a command
**starts** a router, it opens that dashboard once. It prints
`[jev] started JEVONIAN CLAUDE -> opened its dashboard http://127.0.0.1:8797/`. Set `JEV_OPEN_DASHBOARD=0` to turn this off.
`jev dashboards` opens all six at any time.

**After a reboot?** Nothing runs until your first command, which starts what it needs. There's no Windows service or
autostart. Pre-start everything with `jev start`, and stop everything with `jev stop`.

**Is anything exposed?** No. Every router and gateway listens on **127.0.0.1 only** (loopback). Jevonian's tunnel is
off (`"tunnel": {"enabled": false}`). Check with `jev status` or `netstat -ano | findstr ":879"`. Keys stay in
`credentials\` and `.env`, and only the router processes read them.

---

## Table of Contents

- [What is Jevonian?](#what-is-jevonian)
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Folder Structure](#folder-structure)
- [Router Setup (per tool)](#router-setup-per-tool)
- [Client Configuration](#client-configuration)
- [Testing Each Tool](#testing-each-tool)
- [Verification & Debugging](#verification--debugging)
- [Port Map and the Port Spacing Gotcha](#port-map-and-the-port-spacing-gotcha)
- [Troubleshooting](#troubleshooting)
- [Claude Code: tiers, effort and pricing](#claude-code-tiers-effort-and-pricing)
- [Upgrading](#upgrading)
- [Plain CMD commands without .bat, and status windows](#plain-cmd-commands-without-bat-and-status-windows)
- [jev-gateway (official): Claude Code and OpenCode](#jev-gateway-official-claude-code-and-opencode)
- [Summary](#summary) · [Proof of Working](#proof-of-working) · [Quick Reference Card](#quick-reference-card) · [Final Checklist](#final-checklist)

---

## What is Jevonian?

**Jevonian** is a local AI routing proxy that sits between your coding agent and your model providers. Instead of you
switching models by hand, Jevonian asks **Jev** (TypeSafe's fast decision model) to route each turn to the most
cost-effective capable model, based on:
- **Phase:** planning, execution, utility, chat, and so on
- **Context:** conversation size and complexity
- **Quota:** provider health and remaining budget
- **Cache:** recent hit ratios and switch penalties

Every request is logged with the model actually used, the reason, token counts and estimated cost.

**Documentation:** https://github.com/xinyao27/jevonian

---
## Prerequisites

### Required APIs

| Service | Purpose | File in `credentials\` (or `.env`) |
|---|---|---|
| **Alibaba Cloud Model Studio, Token Plan** | Models for OpenCode, Kilo and Qwen (DeepSeek, Qwen, GLM) | `qwen-alibaba-credential.txt`, line `API Key: sk-sp-…` |
| **TypeSafe** | The Jev brain (routing decisions), for Jevonian and jev-gateway | `typesafe-ai-credential.txt`, one line |
| **Vercel AI Gateway** | Jevonian's *fallback* brain | `AI_GATEWAY_API_KEY=…` in the root `.env` |
| **Claude Pro/Max login** | Claude Code through Jevonian or jev-gateway; OAuth, **no API key** | Claude Code's own `~/.claude/.credentials.json` |

> **Check your keys before debugging anything else.** On 2026-09-25 `credentials\vercel-credential.txt` turned out to be
> **stale** (Vercel answered 401) while `AI_GATEWAY_API_KEY` in `.env` worked, which is the one `start.js` reads.
> One-liner to test the Alibaba key and list its models:
> `node -e "…fetch('https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1/models',{headers:{authorization:'Bearer '+KEY}})…"`

### Required Tools

- **Node.js 22.15+** (tested 22.23.2). jev-gateway needs ≥ 22.15.
- **OpenCode**, **Kilo**, **Qwen Code**, **Claude Code** installed and on `PATH` (tested versions at the top).
- **Nothing is installed globally for the routers.** Every router folder has its own `node_modules\jevonian`, and
  `jev-gateway\` has its own `node_modules\jev-gateway`. Upgrade one without touching the others (see
  [Upgrading](#upgrading)).

---

## Architecture Overview

```
 CMD: kilo | qwen | opencode | claude          CMD: jev-claude | jev-opencode
        │ (doskey macros → node launchers,              │ (doskey macros → official jev-gateway launchers)
        │  status window per router)                    │
        ▼                                               ▼
 ┌──────────── Jevonian routers (official 0.1.7) ─┐    ┌── jev-gateway (official 0.4.3, unmodified) ──┐
 │ qwen      :8793  ─┐                            │    │ jev-claude   :8789 → api.anthropic.com        │
 │ kilo      :8795  ─┼─ Jev picks TIER + EFFORT ─▶ Alibaba Token Plan                                  │
 │ opencode  :8799  ─┘   each turn                │◀───┤ jev-opencode :8791 → Jevonian :8799           │
 │ claude    :8797  ── Jev picks TIER + EFFORT ──▶ Anthropic (your Claude login, OAuth)                 │
 └────────────────────────────────────────────────┘◀───┤ jev-claude   :8789 → Jevonian :8797           │
                                                       │   Jev picks the next TOOL; model untouched   │
                                                       └───────────────────────────────────────────────┘
 Brain: TypeSafe Jev (Vercel AI Gateway as Jevonian's fallback). Keys: credentials\ + .env, never copied.
```

- **Jevonian** is a model router. Each turn, Jev picks a *routing* (tier), and the routing names the model. The effort
  level comes from the routing's `effort` (patch, [section 4b](#4b-patch-jevonian-effortmjs-per-tier-effort)).
- **jev-gateway** is a tool router. Each turn, Jev picks which *tool* the agent should call (`forced` / `hint` / `none` /
  `direct` / `passthrough`). It does **not** change the model. The official version has no model tiers.
- `jev-claude` and `jev-opencode` send the agent through jev-gateway, then through the matching Jevonian router,
  using only jev-gateway's official `JEV_CLAUDE_UPSTREAM_BASE_URL` / `JEV_OPENCODE_UPSTREAM_BASE_URL` settings. The result is
  tool routing *and* tier + effort routing, with no code changes to either project.

---

## Folder Structure

```
agy-opencode-jev\
├── credentials\                     keys (never committed)
├── .env                             AI_GATEWAY_API_KEY (Vercel fallback brain)
├── cli\                             the plain CMD commands
│   ├── jev.js                       install/uninstall + `jev status|start|stop|windows`
│   ├── jev.doskey                   the macros (generated by install)
│   ├── launch.js, common.js         shared launcher: start router, status window, run client
│   ├── monitor.js                   the status-window program
│   ├── targets.js                   every router/gateway: port, URL, provider
│   ├── jev-claude.js, jev-opencode.js, gateway-env.js    wrappers around the OFFICIAL jev-gateway launchers
│   └── run\                         status-window pid files
├── jevnonian\                       Jevonian routers
│   ├── jev.js                       start|stop|status|test|logs [opencode|qwen|kilo|claude]
│   ├── kilo.js, qwen.js, opencode.js, claude.js       what the CMD commands run
│   ├── jev-router-qwen\      :8793  config\config.json, .qwen\settings.json, start.js, patches, data\, logs\
│   ├── jev-router-kilo\      :8795  config\config.json, .kilo\kilo.json, start.js, patches, data\, logs\
│   ├── jev-router-claude\    :8797  config\config.json, start.js, patches (+haiku), data\, logs\
│   └── jev-router-opencode\  :8799  config\config.json, opencode.json, start.js, patches, data\, logs\
└── jev-gateway\                     package.json → "jev-gateway": "0.4.3" (official, unmodified, from npm), README.md
```

Each router folder is self-contained: its own port, config, ledger (`data\ledger.jsonl`), dashboard and local
`node_modules\jevonian`. No `.bat`/`.sh` files are used anywhere.

---

## Router Setup (per tool)

The three Alibaba routers (Qwen, Kilo, OpenCode) are identical except for `listen.port` and their client config.
The Claude router is described in [its own section](#claude-code-tiers-effort-and-pricing).

### 1. package.json — exact pin, local install

```json
{ "name": "jev-router-kilo", "version": "1.0.0", "dependencies": { "jevonian": "0.1.7" } }
```

```bat
cd jevnonian\jev-router-kilo
npm install --no-audit --no-fund
```

**Never** `npm install -g`. The pin is exact (`0.1.7`, not `^0.1.7`) because the patches match the bundle's text.

### 2. config/config.json — six tiers, each with a model and an effort

```json
{
  "listen": { "host": "127.0.0.1", "port": 8795 },
  "defaultProvider": "alibaba-tokenplan",
  "providers": [
    { "name": "alibaba-tokenplan", "type": "openai",
      "baseUrl": "https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1",
      "apiKeyEnv": "ALIBABA_TOKENPLAN_API_KEY", "billing": "subscription",
      "models": ["deepseek-v4.1-flash", "qwen3.8-flash", "qwen3.7-plus", "glm-5.3", "qwen3.8-max"] }
  ],
  "routing": {
    "mode": "auto",
    "routings": [
      { "id": "plan",    "label": "Plan",        "description": "architecture, design, multi-file planning, hard reasoning before code", "models": ["glm-5.3"],             "effort": "high" },
      { "id": "execute", "label": "Medium task", "description": "typical implementation or debugging across a few files, tool loops",  "models": ["qwen3.8-flash"],       "effort": "high" },
      { "id": "utility", "label": "Utility",     "description": "summaries, lookups, small mechanical edits",                           "models": ["qwen3.7-plus"],        "effort": "medium" },
      { "id": "chat",    "label": "Chat",        "description": "short conversational replies, acknowledgements",                       "models": ["deepseek-v4.1-flash"], "effort": "low" },
      { "id": "small",   "label": "Small task",  "description": "a small, well-scoped change: one file or a few lines, a quick fix or single command", "models": ["qwen3.8-flash"], "effort": "low" },
      { "id": "large",   "label": "Large task",  "description": "large or heavy work: big multi-file changes, hard debugging, maximum reasoning", "models": ["qwen3.8-max"], "effort": "xhigh" }
    ],
    "sessionTtlMinutes": 720,
    "baselineModel": "glm-5.3",
    "brainPicksEffort": false,
    "capacities": {
      "qwen3.8-flash":       { "contextWindow": 983616,  "maxOutput": 65536, "efforts": ["low", "medium", "high", "xhigh"] },
      "qwen3.7-plus":        { "contextWindow": 983616,  "maxOutput": 65536, "efforts": ["low", "medium", "high", "xhigh"] },
      "glm-5.3":             { "contextWindow": 200000,  "maxOutput": 65536, "efforts": ["low", "high", "max"] },
      "deepseek-v4.1-flash": { "contextWindow": 1000000, "maxOutput": 65536, "efforts": ["low", "medium", "high", "xhigh"] },
      "qwen3.8-max":         { "contextWindow": 983616,  "maxOutput": 65536, "efforts": ["low", "medium", "high", "xhigh"] }
    },
    "brains": [
      { "channel": "typesafe", "apiKeyEnv": "TYPESAFE_API_KEY",   "minConfidence": 0.6, "timeoutMs": 8000 },
      { "channel": "vercel",   "apiKeyEnv": "AI_GATEWAY_API_KEY", "minConfidence": 0.6, "timeoutMs": 20000 }
    ]
  }
}
```

| Tier (virtual model) | Model | Effort |
|---|---|---|
| `jevonian/chat` | deepseek-v4.1-flash | low |
| `jevonian/small` | qwen3.8-flash | low |
| `jevonian/execute` ("Medium task") | qwen3.8-flash | high |
| `jevonian/large` | qwen3.8-max | xhigh |
| `jevonian/utility` | qwen3.7-plus | medium |
| `jevonian/plan` | glm-5.3 | high |
| `jevonian/auto` | Jev picks one of the six each turn | |

- `plan`, `execute`, `utility` and `chat` are Jevonian's **built-in** routings (always present). `small` and `large`
  are custom routings appended after them. "Medium task" is the relabelled built-in `execute`.
- **`effort` on a routing is not an official 0.1.7 field.** It needs the small patch in [4b](#4b-patch-jevonian-effortmjs-per-tier-effort).
  The official knobs are only `brainPicksEffort`, `defaultEffort` and per-model `capacities.efforts`. Those can't give
  the same model (`qwen3.8-flash`) `low` for small and `high` for medium.
- **What the Alibaba Token Plan accepts (probed live with `reasoning_effort`):**

  | Model | low | medium | high | xhigh |
  |---|---|---|---|---|
  | deepseek-v4.1-flash, qwen3.8-flash, qwen3.7-plus, qwen3.8-max | ✅ | ✅ | ✅ | ✅ |
  | **glm-5.3** | ✅ | ❌ 400 | ✅ | ❌ 400: *"'reasoning_effort' must be one of: 'low', 'high', 'max'"* |

  So plan uses **high** on glm-5.3, and `capacities."glm-5.3".efforts` is `["low","high","max"]`, so Jevonian clamps
  anything else correctly.
- **Change a tier by editing `config.json`, then restart** (`node jevnonian\jev.js stop kilo` and `… start kilo`). The
  dashboard's Routing page doesn't know the `effort` field and may drop it when it saves.

### 3. start.js — keys, paths, patches, serve

Identical in every router. It loads the keys from the repo root, points Jevonian at *this* folder, and re-applies *this
folder's* patches on every start. If a patch can't apply, it refuses to start, so a bundle replaced by `npm install`
can never run unpatched.

```javascript
// start.js — Windows-friendly router startup
const { existsSync, readFileSync } = require("fs");
const { join, resolve } = require("path");
const { spawnSync, spawn } = require("child_process");

const ROUTER_DIR = __dirname;
const PROJECT_DIR = resolve(ROUTER_DIR, "..", "..");
function readText(rel) { try { return readFileSync(join(PROJECT_DIR, rel), "utf8"); } catch { return ""; } }
function readEnv(rel) {
  const out = {};
  for (const line of readText(rel).split(/\r?\n/)) { const m = line.match(/^([^#=]+)=(.*)$/); if (m) out[m[1].trim()] = m[2].trim(); }
  return out;
}

const alibabaKey = (readText("credentials/qwen-alibaba-credential.txt").match(/^API Key:\s*(.+)$/m) || [, ""])[1].replace(/\s+/g, "");
const typesafeKey = readText("credentials/typesafe-ai-credential.txt").replace(/\s+/g, "");
const aiGatewayKey = readEnv(".env").AI_GATEWAY_API_KEY || "";
for (const [k, v] of [["ALIBABA_TOKENPLAN_API_KEY", alibabaKey], ["TYPESAFE_API_KEY", typesafeKey], ["AI_GATEWAY_API_KEY", aiGatewayKey]]) {
  if (!v) console.error(`WARNING: ${k} is empty`);
  process.env[k] = v;
}

process.env.JEVONIAN_CONFIG = join(ROUTER_DIR, "config", "config.json");
process.env.JEVONIAN_CREDENTIALS = join(ROUTER_DIR, "config", "credentials.json");
process.env.JEVONIAN_DATA_DIR = join(ROUTER_DIR, "data");
process.env.JEVONIAN_LEDGER = join(ROUTER_DIR, "data", "ledger.jsonl");
process.env.JEVONIAN_UPDATE_STATE = join(ROUTER_DIR, "data", "update.json");
process.env.JEVONIAN_NO_OPEN = "1";

// Each router carries only the patches it needs (the Claude router adds the Haiku patch).
const PATCHES = ["patch-jevonian-waf.mjs", "patch-jevonian-effort.mjs", "patch-jevonian-haiku.mjs"].filter((p) => existsSync(join(ROUTER_DIR, p)));
for (const patch of PATCHES) {
  const result = spawnSync(process.execPath, [join(ROUTER_DIR, patch)], { stdio: "inherit" });
  if (result.status !== 0) { console.error(`${patch} failed — refusing to start with an unpatched bundle.`); process.exit(1); }
}

const cli = join(ROUTER_DIR, "node_modules", "jevonian", "dist", "cli.mjs");
const child = spawn("node", [cli, "serve", "--foreground"], { stdio: "inherit" });
child.on("exit", (code) => process.exit(code));
```

### 4. The patches (small, idempotent, each checks before it writes)

| Patch | Why | Routers |
|---|---|---|
| `patch-jevonian-waf.mjs` (v2) | Cloudflare's WAF in front of TypeSafe can reject brain calls whose state contains `\|`, `/etc/`, `<script` or `../`. 0.1.7 already redacts shell commands; this defangs the rest of `brainState`. | all |
| `patch-jevonian-effort.mjs` | Per-tier `effort` (section 2), plus opt-in `forceEffort` so a tier's level wins over the client's (used for Claude Code). | all |
| `patch-jevonian-haiku.mjs` | Haiku 4.5 rejects fields Claude Code always sends (still needed on 0.1.7; see [Claude section](#claude-code-tiers-effort-and-pricing)). | Claude only |

#### 4a. patch-jevonian-waf.mjs (v2, for 0.1.7)

```javascript
// Idempotent patch for jevonian 0.1.7+ — defangs attack-looking text in the brain-state payload.
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

#### 4b. patch-jevonian-effort.mjs (per-tier effort)

Five edits, each with its own marker. Nothing is written if any pattern is missing.
1. Keep `effort` when a routing is parsed.
2. On Jev-routed turns (`jevonian/auto`), the chosen routing's effort wins.
3. On **pinned** turns (`jevonian/<id>`), which in 0.1.7 send **no effort at all**, the routing's effort is also sent.
4. Keep the opt-in `forceEffort: true` flag.
5. When the serving routing has `forceEffort`, ignore the **client's** own level for that turn. That's needed for Claude Code,
   which always sends one. OpenCode, Kilo and Qwen send none, so they don't need the flag.

Levels are still clamped to `capacities.<model>.efforts`. Without `forceEffort`, a level the client sets is never overridden,
which is the official rule.

```javascript
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

### 5. Start and check

```bat
node jevnonian\jev.js start          :: all four routers in the background (logs\serve.log in each folder)
node jevnonian\jev.js status         :: health + ledger line count per port
curl http://127.0.0.1:8795/healthz   :: {"ok":true,"sessions":0,"routing":"auto"}
curl http://127.0.0.1:8795/v1/models :: jevonian/auto, plan, execute, utility, chat, small, large
```

You don't need to start routers by hand: every CMD command ([CMD commands](#plain-cmd-commands-without-bat-and-status-windows))
starts its own router when it's down, and opens its dashboard the first time.

---

## Client Configuration

Each router folder also holds the client config for *plain* use inside that folder. The CMD commands inject the same
config from any folder (`KILO_CONFIG_CONTENT`, `OPENCODE_CONFIG_CONTENT`, or Qwen's CLI flags). The provider name
includes the **port**, so `/model` shows which router you're on.

### Kilo: `jev-router-kilo\.kilo\kilo.json`

```json
{
  "$schema": "https://kilo.ai/config.json",
  "provider": {
    "jevonian": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Jevonian :8795 (Jev router -> Alibaba)",
      "options": { "baseURL": "http://127.0.0.1:8795/v1", "apiKey": "local-no-key" },
      "models": {
        "jevonian/auto":    { "name": "Jev Auto (Jev picks tier + effort each turn)",       "limit": { "context": 200000, "output": 65536 }, "tool_call": true },
        "jevonian/chat":    { "name": "Jev Chat = deepseek-v4.1-flash, effort low (pinned)", "limit": { "context": 200000, "output": 65536 }, "tool_call": true },
        "jevonian/small":   { "name": "Jev Small = qwen3.8-flash, effort low (pinned)",      "limit": { "context": 200000, "output": 65536 }, "tool_call": true },
        "jevonian/execute": { "name": "Jev Medium = qwen3.8-flash, effort high (pinned)",    "limit": { "context": 200000, "output": 65536 }, "tool_call": true },
        "jevonian/large":   { "name": "Jev Large = qwen3.8-max, effort xhigh (pinned)",      "limit": { "context": 200000, "output": 65536 }, "tool_call": true },
        "jevonian/utility": { "name": "Jev Utility = qwen3.7-plus, effort medium (pinned)",  "limit": { "context": 200000, "output": 65536 }, "tool_call": true },
        "jevonian/plan":    { "name": "Jev Plan = glm-5.3, effort high (pinned)",            "limit": { "context": 200000, "output": 65536 }, "tool_call": true }
      }
    }
  },
  "model": "jevonian/jevonian/auto"
}
```

- It lives in `.kilo\kilo.json` (dot-directory), not `kilo.json`. Kilo merges configs from the working folder up to
  the git root, and the dot-directory copy wins over a parent's `kilo.json`.
- **Kilo ignores the config's `model` default** and uses its own cloud default instead (`minimax/…`, which answers
  *"Add credits to continue"*). Always pass `-m jevonian/jevonian/auto`; the `kilo` command does it for you.

### OpenCode: `jev-router-opencode\opencode.json`

Same shape as Kilo's, but with `$schema: https://opencode.ai/config.json`, `baseURL: http://127.0.0.1:8799/v1`
and the name `Jevonian :8799 (…)`. OpenCode 2.x needs three things from any launcher:
- **`--standalone`**, because otherwise it talks to a shared background service that may have been started from another
  folder with another config (*"Model unavailable: jevonian/jevonian/auto"*)
- **`PWD`** set to the working folder, because this build resolves config against it
- **`-m jevonian/jevonian/auto`**

The `opencode` command adds all three.

### Qwen Code: `jev-router-qwen\.qwen\settings.json`

```json
{
  "modelProviders": {
    "openai": [
      { "id": "jevonian/auto",    "name": "Jev Auto (Jev picks tier + effort each turn) — Jevonian :8793", "baseUrl": "http://127.0.0.1:8793/v1", "envKey": "JEV_ROUTER_API_KEY",
        "description": "Jev router :8793 -> Alibaba Token Plan (chat/small/medium/large/utility/plan)" },
      { "id": "jevonian/chat",    "name": "Jev Chat = deepseek-v4.1-flash, effort low (pinned) — Jevonian :8793", "baseUrl": "http://127.0.0.1:8793/v1", "envKey": "JEV_ROUTER_API_KEY" },
      { "id": "jevonian/small",   "name": "Jev Small = qwen3.8-flash, effort low (pinned) — Jevonian :8793",      "baseUrl": "http://127.0.0.1:8793/v1", "envKey": "JEV_ROUTER_API_KEY" },
      { "id": "jevonian/execute", "name": "Jev Medium = qwen3.8-flash, effort high (pinned) — Jevonian :8793",    "baseUrl": "http://127.0.0.1:8793/v1", "envKey": "JEV_ROUTER_API_KEY" },
      { "id": "jevonian/large",   "name": "Jev Large = qwen3.8-max, effort xhigh (pinned) — Jevonian :8793",      "baseUrl": "http://127.0.0.1:8793/v1", "envKey": "JEV_ROUTER_API_KEY" },
      { "id": "jevonian/utility", "name": "Jev Utility = qwen3.7-plus, effort medium (pinned) — Jevonian :8793",  "baseUrl": "http://127.0.0.1:8793/v1", "envKey": "JEV_ROUTER_API_KEY" },
      { "id": "jevonian/plan",    "name": "Jev Plan = glm-5.3, effort high (pinned) — Jevonian :8793",            "baseUrl": "http://127.0.0.1:8793/v1", "envKey": "JEV_ROUTER_API_KEY" }
    ]
  },
  "env": { "JEV_ROUTER_API_KEY": "local-no-key" },
  "security": { "auth": { "selectedType": "openai" } },
  "model": { "name": "jevonian/auto" },
  "$version": 4
}
```

Qwen reads `.qwen\settings.json` only from the folder it starts in. From any other folder, the `qwen` command passes
the router on the command line instead:
`--auth-type openai --openai-base-url http://127.0.0.1:8793/v1 --openai-api-key local-no-key -m jevonian/auto`.

---

## Testing Each Tool

From **any** folder, in a CMD window opened after `cli\jev.js install`:

```bat
kilo run "Reply with exactly: KILO_OK"
qwen --output-format text -p "Reply with exactly: QWEN_OK"
opencode run "Reply with exactly: OPENCODE_OK"
claude -p "Reply with exactly: CLAUDE_OK" --dangerously-skip-permissions --output-format text
jev-claude -p "Reply with exactly: JEV_CLAUDE_OK" --dangerously-skip-permissions --output-format text
jev-opencode run "Reply with exactly: JEV_OPENCODE_OK"
```

Pin a tier with the model flag: `kilo run -m jevonian/jevonian/plan "…"`, `qwen -m jevonian/large -p "…"`,
`claude --model jevonian/plan`. The built-in smoke test is `node jevnonian\jev.js test [opencode|qwen|kilo|claude]`.

Interactive is just the command: `kilo`, `qwen`, `opencode`, `claude --dangerously-skip-permissions`. In `/model`,
Kilo and OpenCode list the seven `Jev …` models under `Jevonian :<port>`, and Qwen lists them with `— Jevonian :8793`.

> Scripting `opencode run`? Add `< nul`. `run` otherwise waits for a piped stdin to close.

---

## Verification & Debugging

**Dashboards.** Every router has its own:
- http://127.0.0.1:8793/ (Qwen), :8795/ (Kilo), :8797/ (Claude), :8799/ (OpenCode)
- Pages: **Overview**, **Providers**, **Routing**, **Activity**, **Clients**, **Logs**
- **Logs** has an **Effort** column: the level the model was *actually sent*, read back from the outgoing body

The jev-gateway dashboard is http://127.0.0.1:8789/dashboard, and it shows both gateways on one page.

**The ledger** (`data\ledger.jsonl`, one JSON line per request):

```bat
:: last request, readable
node -e "const l=require('fs').readFileSync(process.argv[1],'utf8').trim().split('\n').map(JSON.parse).filter(e=>e.kind!=='brain').pop();console.log(l.phase,l.model,l.effort,l.status,l.brain)" jevnonian\jev-router-kilo\data\ledger.jsonl
```

Key fields: `ts`, `requestedModel` (e.g. `jevonian/auto`), `phase` (the tier), `model`, `effort` (sent), `status`,
`brain` (`jev` or `jev-low-confidence`), `reason`, `session`. Brain calls are their own lines (`"kind":"brain"`).

**Response headers** on every routed request: `x-jevonian-model`, `x-jevonian-phase`, `x-jevonian-effort`, and
`x-jevonian-effort-note` when a level was clamped.

**Router logs:** `node jevnonian\jev.js logs kilo`, or open `jevnonian\jev-router-kilo\logs\serve.log`.

**Don't trust a reply alone.** "hi" looks the same with or without the router. The ledger line count, or the status
window's live feed, must go up.

---

## Port Map and the Port Spacing Gotcha

**Jevonian binds two ports**: `listen.port` and `listen.port + 1` (its public/tunnel surface). Routers must be at least 2 apart.
**jev-gateway binds one port per client**, and its dashboard looks for the others on its **official default ports**
(8787–8791), so this repo leaves those to jev-gateway.

| Port | Owner | Notes |
|---|---|---|
| 8789 | jev-gateway `jev-claude` | official default |
| 8791 | jev-gateway `jev-opencode` | official default |
| 8793 (+8794) | Jevonian Qwen | |
| 8795 (+8796) | Jevonian Kilo | |
| 8797 (+8798) | Jevonian Claude Code | upstream of `jev-claude` |
| 8799 (+8800) | Jevonian OpenCode | upstream of `jev-opencode` |

On 2026-09-25 none of 8787–8800 was in a Windows (WinNAT/Hyper-V) excluded range. To check:
`netsh interface ipv4 show excludedportrange protocol=tcp`. `EADDRINUSE` usually means port **+1** is taken.

---

## Troubleshooting

### "Jev brain unavailable: all 2 configured brain(s) failed"

Both brains (TypeSafe, then Vercel) timed out or errored. Check the keys (see [Prerequisites](#prerequisites)), check
network access to `api.typesafe.ai`, and retry, since transient failures happen. The router log is
`jevnonian\jev-router-<name>\logs\serve.log`.

### Kilo answers "Add credits to continue, or switch to a free model"

Kilo ignored the config's default model and used its own cloud default (`minimax/…`). Pass `-m jevonian/jevonian/auto`.
The `kilo` command always does, unless you give your own `-m`.

### Kilo uses the wrong router (parent config)

Kilo merges configs from the working folder up to the git root. Keep router configs in `.kilo\kilo.json`, not
`kilo.json`. The `kilo` command avoids the question entirely by injecting `KILO_CONFIG_CONTENT`.

### OpenCode: "Model unavailable: jevonian/jevonian/auto", although `opencode models` lists it

OpenCode 2.x runs requests through a shared **background service**. One started earlier from another folder
(`opencode serve --service`) holds *that* folder's config. Use `--standalone` and set `PWD` to the working folder; the `opencode`
and `jev-opencode` commands do both.

### `opencode run` hangs with nothing in the ledger

`opencode run` also reads stdin when it isn't a terminal, and waits for the pipe to close. In scripts, add `< nul`.

### Qwen shows "Persisted model.baseUrl no longer matches"

Harmless. Qwen cached an earlier selection. Pick `jevonian/auto` once with `/model`.

### Effort shows `default` (or the client's level) instead of the tier's

- **Alibaba routers:** the effort patch is missing or old. Restart the router (`start.js` re-applies it). Check with
  `findstr /c:"routing-effort-patch v3b" node_modules\jevonian\dist\cli.mjs`.
- **Claude router:** the routing needs `"forceEffort": true`. Claude Code always sends its own level, and without that flag
  Jevonian keeps it.

### HTTP 400 on chat turns: `context_management: Extra inputs are not permitted` (or `role 'system' is not supported`)

The chat tier sent Claude Code's request to Haiku 4.5 without the Haiku patch. Make sure `patch-jevonian-haiku.mjs` is in
`jev-router-claude\`, and restart the router. Also check that the chat routing's `providers` is a **map** (`{"claude-haiku-4-5-20251001":
["claude-subscription-haiku"]}`), not an array.

### A patch prints "pattern … found 0 times"

The Jevonian version changed the code that patch targets. Nothing was written, and the router refuses to start rather than
run unpatched. **Don't** force it. See [Upgrading](#upgrading).

### `rm` / `npm install` fails with "resource busy" inside `node_modules\jevonian`

A file watcher (often the IDE) holds the folder. `npm pack jevonian@0.1.7`, unpack the tarball's `package\` contents over
`node_modules\jevonian\`, then restart the router.

### "No Jevonian API key exists yet"

That only concerns the tunnel surface (port+1), which is off. Ignore it for local use.

### Harmless Claude Code messages

- `[claude-code:unrecognized_model] {"model":"jevonian/auto"}` and *"jevonian/auto isn't described by this version's model
  catalog … auto-compact keeps this session within 200k tokens"*: Claude Code doesn't know the virtual model. It then
  assumes 200k, which suits the Haiku tier.
- *"claude.ai connectors are disabled because ANTHROPIC_API_KEY or another auth source is set"*: the router's auth token is set.
  claude.ai's cloud connectors (Gmail, Drive…) don't load in routed sessions. Your project MCP servers (`.mcp.json`) still do.

### `claude` in CMD opens plain Claude Code

- The window was opened before `cli\jev.js install`. Open a new one.
- You're in PowerShell or a `.bat`, where macros don't apply. Run `node …\jevnonian\claude.js …`.
- Check with `jev status`: the Jevonian Claude router and a status window should show.

### Tests look fine, but did the router really get the request?

Don't trust the reply text. The ledger line count (or the status window's live feed, or the dashboard) must go up.

---

## Claude Code: tiers, effort and pricing

Claude Code runs through the **Jevonian Claude router (:8797)** on both paths:

| You type | Path | Who picks what |
|---|---|---|
| `claude` | Claude Code → **Jevonian :8797** → Anthropic | Jev picks the **tier** (model + effort) each turn |
| `jev-claude` | Claude Code → **jev-gateway :8789** → Jevonian :8797 → Anthropic | jev-gateway: Jev picks the next **tool**. Jevonian: Jev picks the **tier** |

Both use your **Claude Pro/Max login** (OAuth from `~/.claude/.credentials.json`, used by Jevonian's official
`claude-subscription` provider). There's **no API key**, and nothing is written to `~/.claude/settings.json`.

### The tiers

| Tier (virtual model) | Model | Effort | Typical turn |
|---|---|---|---|
| `jevonian/plan` | Claude Opus 5.5 | **xhigh** | architecture, design, multi-file planning, hard reasoning |
| `jevonian/heavy` | Claude Opus 5.5 | **high** | large tasks, big multi-file work |
| `jevonian/execute` | Claude Sonnet 5 | **high** | implementation, debugging, tool loops |
| `jevonian/utility` | Claude Sonnet 5 | **medium** | summaries, lookups, small mechanical edits |
| `jevonian/chat` | Claude Haiku 4.5 | **low** (a 2,048-token thinking budget) | short replies, acknowledgements |
| `jevonian/auto` | Jev picks one of the five each turn | | the default |

What the Anthropic docs say about these models
([pricing](https://platform.claude.com/docs/en/about-claude/pricing),
[effort](https://platform.claude.com/docs/en/build-with-claude/effort)), with prices per million tokens:

| Model | Input | Cache hit | Output | Effort levels | Default effort |
|---|---|---|---|---|---|
| Opus 5.5 | $4 | $0.20 (0.05×) | $20 | low, medium, high, xhigh, max | **medium**; adaptive thinking always on |
| Sonnet 5 | $2 | $0.20 | $10 | low, medium, high, xhigh, max | high |
| Haiku 4.5 | $1 | $0.10 | $5 | **none**: not an effort model | |

- **Haiku has no effort parameter.** Jevonian expresses `low` as extended thinking with `budget_tokens: 2048`. That's
  its own mapping for legacy-thinking models (medium 8192, high 16384).
- The router records every turn's **"subscription value"** in the dashboard, meaning what it would have cost at these API
  prices. On a Claude subscription you aren't billed per token; turns count against your plan's usage limits.
- **xhigh on Opus 5.5 uses the most tokens.** It's reserved for `plan`, which Jev picks only for planning-type turns.
- All three models are 1M-context except Haiku (200k). Claude Code doesn't know `jevonian/auto`, so it assumes a 200k
  window and auto-compacts before that. That keeps every turn small enough for the Haiku tier.

### How effort is enforced: `forceEffort` (small patch)

Claude Code **always** sends its own `output_config.effort`. This was tested: even with no effort configured, it sends
`"high"`. Official Jevonian never overrides a level the client set, so without help every tier would run at Claude Code's one level.
Each Claude routing therefore sets `"forceEffort": true`. The effort patch ([section 4b](#4b-patch-jevonian-effortmjs-per-tier-effort))
then writes the routing's level instead of the client's. The consequence: **Claude Code's `/effort` doesn't change these tiers.** The router decides.

### The Claude router: `jevnonian\jev-router-claude\config\config.json`

```json
{
  "listen": { "host": "127.0.0.1", "port": 8797 },
  "defaultProvider": "claude-subscription",
  "providers": [
    { "name": "claude-subscription", "type": "anthropic", "baseUrl": "https://api.anthropic.com/v1",
      "auth": "oauth", "oauthSource": "claude-code", "billing": "subscription",
      "models": ["claude-opus-5-5", "claude-sonnet-5", "claude-fable-5-1", "claude-haiku-4-5-20251001"],
      "injectStreamUsage": true,
      "headers": { "anthropic-beta": "claude-code-20250219,context-1m-2025-08-07,interleaved-thinking-2025-05-14,thinking-token-count-2026-05-13,context-management-2025-06-27,prompt-caching-scope-2026-01-05,mid-conversation-system-2026-04-07,mid-conversation-tool-changes-2026-07-01,advisor-tool-2026-03-01,effort-2025-11-24" } },
    { "name": "claude-subscription-haiku", "type": "anthropic", "baseUrl": "https://api.anthropic.com/v1",
      "auth": "oauth", "oauthSource": "claude-code", "billing": "subscription",
      "models": ["claude-haiku-4-5-20251001"], "injectStreamUsage": true,
      "headers": { "anthropic-beta": "claude-code-20250219,interleaved-thinking-2025-05-14,thinking-token-count-2026-05-13" } }
  ],
  "tunnel": { "enabled": false, "provider": "cloudflare" },
  "routing": {
    "mode": "auto",
    "routings": [
      { "id": "plan",    "label": "Plan",    "description": "architecture, design, multi-file planning, hard reasoning", "models": ["claude-opus-5-5"],  "effort": "xhigh",  "forceEffort": true },
      { "id": "execute", "label": "Execute", "description": "implementation, debugging, tool loops",                     "models": ["claude-sonnet-5"],  "effort": "high",   "forceEffort": true },
      { "id": "utility", "label": "Utility", "description": "summaries, lookups, small mechanical edits",                "models": ["claude-sonnet-5"],  "effort": "medium", "forceEffort": true },
      { "id": "chat",    "label": "Chat",    "description": "short conversational replies, acknowledgements",           "models": ["claude-haiku-4-5-20251001"],
        "providers": { "claude-haiku-4-5-20251001": ["claude-subscription-haiku"] },                                                           "effort": "low",    "forceEffort": true },
      { "id": "heavy",   "label": "Heavy",   "description": "large tasks, big multi-file work, maximum reasoning",       "models": ["claude-opus-5-5"],  "effort": "high",   "forceEffort": true }
    ],
    "sessionTtlMinutes": 720,
    "baselineModel": "claude-sonnet-5",
    "capacities": {
      "claude-opus-5-5":           { "contextWindow": 1000000, "maxOutput": 128000, "efforts": ["low", "medium", "high", "xhigh", "max"] },
      "claude-sonnet-5":           { "contextWindow": 1000000, "maxOutput": 128000, "efforts": ["low", "medium", "high", "xhigh", "max"] },
      "claude-haiku-4-5-20251001": { "contextWindow": 200000,  "maxOutput": 65536,  "efforts": ["low", "medium", "high"] }
    },
    "quotaGuard": { "enabled": true, "lowPercent": 10 },
    "brains": [
      { "channel": "typesafe", "apiKeyEnv": "TYPESAFE_API_KEY",   "timeoutMs": 8000,  "minConfidence": 0.6 },
      { "channel": "vercel",   "apiKeyEnv": "AI_GATEWAY_API_KEY", "timeoutMs": 20000, "minConfidence": 0.6 }
    ],
    "brainPicksEffort": false
  },
  "modelSync": { "enabled": false, "intervalMinutes": 720 }
}
```

- `auth: "oauth"` with `oauthSource: "claude-code"` is Jevonian's documented Claude Pro/Max provider
  ([docs/providers.md](https://github.com/xinyao27/jevonian/blob/main/docs/providers.md)).
- **Haiku gets its own provider**, because Haiku 4.5 rejects the 1M-context beta. The chat routing's `providers` must be a
  **map** keyed by model id; an array is silently ignored.
- `modelSync` is off, so the Haiku provider's model list can't grow.

### Launching: Jevonian's official `jevonian launch claude`

`claude` in CMD runs Jevonian's own launcher, pointed at this router:

```bat
node jevnonian\jev-router-claude\node_modules\jevonian\dist\cli.mjs launch claude --model jevonian/auto -- <your args>
```

The launcher reads `JEVONIAN_CONFIG` (this router's `config.json`) for the port. It sets `ANTHROPIC_BASE_URL`,
`ANTHROPIC_AUTH_TOKEN=jevonian-local` and the Opus/Sonnet/Haiku → `jevonian/*` mapping, the same approach as `ollama launch claude`.
**Your arguments must come after `--`**, because `launch claude` drops flags placed before it; the `claude` command handles that.
`claude --model jevonian/plan` pins a tier. `jev-claude` sets the same Claude Code environment (see [jev-gateway](#jev-gateway-official-claude-code-and-opencode)).

### The Haiku patch is still needed on 0.1.7

This was verified on 2026-09-25. Without `patch-jevonian-haiku.mjs`, every chat-tier turn failed with
`400 context_management: Extra inputs are not permitted`. Claude Code always sends `context_management`,
`output_config.effort`, adaptive `thinking` and (since 2.1.x) mid-conversation `role:"system"` messages, and Haiku 4.5 rejects
each of them. The patch sanitizes the request body for legacy-thinking models only. It lives only in `jev-router-claude\`, and
`start.js` applies it. Its full source is in that folder: it hooks `withEffort()` and appends `haikuSafeBody()`, which
removes `output_config` and `context_management`, turns adaptive thinking into a budget, and drops mid-conversation system messages.

### Verified (2026-09-25), with the same results through both paths

| Request | Jevonian :8797 ledger (`claude …`) | Through jev-gateway (`jev-claude …`) |
|---|---|---|
| `--model jevonian/plan` | plan → claude-opus-5-5, effort **xhigh**, 200 | same, and the gateway logs `jevonian/plan` |
| `--model jevonian/heavy` | heavy → claude-opus-5-5, **high**, 200 | same |
| `--model jevonian/execute` | execute → claude-sonnet-5, **high**, 200 | same |
| `--model jevonian/utility` | utility → claude-sonnet-5, **medium**, 200 | same |
| `--model jevonian/chat` | chat → claude-haiku-4-5, **low**, 200 | same |
| `jevonian/auto`, "thanks, that is all!" | chat → claude-haiku-4-5, **low**, 200 | same |

The effort column is read back from the request actually sent to Anthropic.

---

## Upgrading

### Jevonian (one router at a time; each has its own `node_modules\jevonian`)

1. **Check the version and read what changed:** `npm view jevonian version`, then the release notes at
   https://github.com/xinyao27/jevonian/releases. Look for changes to routing, providers, the brain state (the WAF patch), or
   `withEffort` / Claude Code handling (the effort and Haiku patches).
   **If a release ships per-routing effort, a client-effort override, or Haiku handling officially, delete that patch file
   instead of updating it.**
2. **Stop the router** (`node jevnonian\jev.js stop kilo`). Don't reinstall while it has the package loaded.
3. **Install, exact pin, in that folder only:** `cd jevnonian\jev-router-kilo`, then `npm install jevonian@X.Y.Z --save-exact`.
4. **Start it** (`node jevnonian\jev.js start kilo`). `start.js` re-applies every `patch-jevonian-*.mjs` in the folder. Read
   the first lines of `logs\serve.log`: each patch prints `applied` or `already applied`.
5. **If a patch prints "pattern … found 0 times",** the router refuses to start and nothing was written. Find the new code shape
   with `findstr /n "withEffort brainState clientEffortOf parseRoutingEntry" node_modules\jevonian\dist\cli.mjs`, then update
   the patch's search strings, or roll back.
6. **Test the paths the patches protect:** pinned tiers (effort per tier), a Claude chat turn (Haiku), and a long session
   (WAF). Check `node jevnonian\jev.js test kilo` and the dashboard's **Logs → Effort** column.
7. **Roll back:** `npm install jevonian@<previous> --save-exact`, then restart.

### jev-gateway

```bat
cd jev-gateway
npm install jev-gateway@X.Y.Z --save-exact
node node_modules\jev-gateway\bin\jev-claude.mjs --stop
node node_modules\jev-gateway\bin\jev-opencode.mjs --stop
```

Nothing in it is patched. Check its release notes for renamed env variables (`JEV_CLAUDE_UPSTREAM_BASE_URL`,
`JEV_OPENCODE_*`) or changed default ports. The next `jev-claude` / `jev-opencode` starts the new version.

---

## Plain CMD commands without .bat, and status windows

### Is it possible? Yes, with doskey macros and CMD AutoRun

The goal: type plain `kilo`, `qwen`, `opencode` or `claude` in CMD, have it call the router's server files with `node`,
show the **port, URL and provider**, and use **no `.bat` files**.

| Approach | Works? | Why |
|---|---|---|
| A Node script `kilo.js` on `PATH` | ❌ | `.JS` is in `PATHEXT`, but Windows runs it with **Windows Script Host (JScript)**, not Node. A `claude.js` on the lookup path would also shadow `claude.exe`. |
| `kilo.cmd` / `kilo.bat` shims | ❌ (by choice) | These are batch files. |
| **doskey macros + CMD AutoRun** | ✅ | CMD's built-in alias feature. A plain-text macro file (`cli\jev.doskey`) maps `kilo` to `node "…\jevnonian\kilo.js" $*`, and CMD loads it in every new window through `HKCU\Software\Microsoft\Command Processor\AutoRun`. |

**Limits:**
- Macros work at the **interactive CMD prompt** only. They don't apply inside `.bat`/`.cmd` scripts, in **PowerShell**,
  in VS Code's Claude Code extension, or with `cmd /d`. There, run the script directly: `node …\jevnonian\kilo.js …`.
- Only **new** CMD windows get them.
- `claude-direct`, `kilo-direct`, `opencode-direct` and `qwen-direct` run the real clients with no router.
- AutoRun runs for every CMD window. The fragment is one silent `if exist … doskey /macrofile=…`. Any AutoRun value you
  already had is kept, and backed up to `cli\autorun.backup.json`.

### Install / remove

```bat
node D:\learn\gemini-mcp\agy-opencode-jev\cli\jev.js install     :: writes cli\jev.doskey + adds the AutoRun fragment
node D:\learn\gemini-mcp\agy-opencode-jev\cli\jev.js uninstall   :: removes exactly that fragment
```

Check it in a **new** CMD window: `doskey /macros` lists
`kilo`, `qwen`, `opencode`, `claude`, `jev-claude`, `jev-opencode`, `jev` and the four `*-direct` commands.

> doskey macro files have no comment syntax. A `;` line prints *"Invalid macro definition."* in every new CMD window, so
> the generated file has none.

### What happens when you type `claude` (or any command)

1. `node jevnonian\claude.js` checks `http://127.0.0.1:8797/healthz`. If the router is down, it starts it in the background
   with `node jevnonian\jev.js start claude` (`start.js` → patches → `jevonian serve`) and **opens its dashboard** in your browser.
2. It opens a **status window** titled `JEVONIAN - CLAUDE - :8797` (`start "…" cmd /k node cli\monitor.js jevonian-claude`), or
   reuses it if it's still open. Closing it does **not** stop the router.
3. It prints the same facts in your window, then runs Claude Code **in your current folder**, here through Jevonian's
   official `jevonian launch claude -- <your args>`.

| Command | Status window | Wiring |
|---|---|---|
| `kilo` | `JEVONIAN - KILO - :8795` | `KILO_CONFIG_CONTENT` + `-m jevonian/jevonian/auto` |
| `qwen` | `JEVONIAN - QWEN - :8793` | `--auth-type openai --openai-base-url http://127.0.0.1:8793/v1 -m jevonian/auto` |
| `opencode` | `JEVONIAN - OPENCODE - :8799` | `OPENCODE_CONFIG_CONTENT` + `--standalone` + `PWD` + `-m jevonian/jevonian/auto` |
| `claude` | `JEVONIAN - CLAUDE - :8797` | official `jevonian launch claude --model jevonian/auto -- …` |
| `jev-claude` | `JEV-GATEWAY - CLAUDE - :8789` | official jev-gateway `bin\jev-claude.mjs` + env (see the jev-gateway section) + `--model jevonian/auto` |
| `jev-opencode` | `JEV-GATEWAY - OPENCODE - :8791` | official `bin\jev-opencode.mjs` + env + `--standalone` |

### What a status window shows

Real output of `JEVONIAN - KILO - :8795` (colours removed):

```
==============================================================================
  JEVONIAN - KILO - :8795   Jevonian router for kilo
==============================================================================
  Status      : ONLINE   {"ok":true,"sessions":6,"routing":"auto"}
  Port        : 8795   (Jevonian also binds 8796 for its tunnel surface)
  URL         : http://127.0.0.1:8795/v1
  Dashboard   : http://127.0.0.1:8795/    Logs: http://127.0.0.1:8795/logs
  Provider    : Alibaba Cloud Model Studio - Token Plan  (https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1)
  Jev brain   : TypeSafe Jev (fallback: Vercel AI Gateway) picks the tier each turn
  Tiers (model, effort):
      jevonian/plan          glm-5.3                      high
      jevonian/execute       qwen3.8-flash                high
      jevonian/utility       qwen3.7-plus                 medium
      jevonian/chat          deepseek-v4.1-flash          low
      jevonian/small         qwen3.8-flash                low
      jevonian/large         qwen3.8-max                  xhigh
  Ledger      : D:\learn\gemini-mcp\agy-opencode-jev\jevnonian\jev-router-kilo\data\ledger.jsonl
==============================================================================
  Live requests (this window only watches; closing it does not stop the router)
  time      tier/phase   model                       effort  status  detail
  10:38:13  small        qwen3.8-flash               low     200      jevonian/small 2098ms
  10:38:15  large        qwen3.8-max                 xhigh   200      jevonian/large 1518ms
```

A jev-gateway window shows the gateway's own data instead: for each request, the **mode** (`forced` / `hint` / `none` /
`passthrough`), the tool Jev picked, and the reason. It also lists the Jevonian tiers behind it.

`jev status` shows everything at once, including which status windows are open. `jev windows` opens every status
window, and `jev dashboards` opens every dashboard.

> The one-shot forms of every command were tested from a script. Typing into the interactive TUIs from a script (SendKeys)
> was blocked by this machine's endpoint protection (Cylance Script Control), and it wasn't worked around. The interactive
> sessions were started for real in CMD windows, for manual use.

---

## jev-gateway (official): Claude Code and OpenCode

### What the official jev-gateway does

[vinilana/jev-gateway](https://github.com/vinilana/jev-gateway) **0.4.3** (latest on npm and GitHub, 2026-09-25) asks Jev
**which tool** the agent should call next, then steers the LLM:
- `forced`: sets `tool_choice`
- `hint`: for Claude Code with thinking on, a one-line suggestion
- `none`
- `direct`: no LLM call at all
- `passthrough`: everything else goes through untouched

**It does not choose models, and it has no tiers or effort settings.** In this setup it's used **only with Claude Code and
OpenCode**, and **unmodified**. Tiers and effort come from Jevonian, which it forwards to.

### Install (official package, local, unmodified)

```bat
cd D:\learn\gemini-mcp\agy-opencode-jev\jev-gateway
:: package.json: { "dependencies": { "jev-gateway": "0.4.3" } }
npm install --no-audit --no-fund
```

The official launchers are `node_modules\jev-gateway\bin\jev-claude.mjs` and `jev-opencode.mjs`. The `jev-claude` /
`jev-opencode` CMD commands (`cli\jev-claude.js`, `cli\jev-opencode.js`, `cli\gateway-env.js`) only set **documented**
environment variables and arguments, open the status window, and run those launchers.

### Wiring (documented settings only)

| Command | Port | Upstream | Settings |
|---|---|---|---|
| `jev-claude` | **8789** (default) | **Jevonian Claude router** `http://127.0.0.1:8797/v1` | jev-gateway: `TYPESAFE_API_KEY`, `JEV_PROVIDER=typesafe`, `JEV_CLAUDE_UPSTREAM_BASE_URL=http://127.0.0.1:8797/v1`. Claude Code: the same env `jevonian launch claude` sets (`ANTHROPIC_AUTH_TOKEN=jevonian-local`, Opus/Sonnet/Haiku → `jevonian/*`), plus `--model jevonian/auto` |
| `jev-opencode` | **8791** (default) | **Jevonian OpenCode router** `http://127.0.0.1:8799/v1` | `JEV_OPENCODE_UPSTREAM_BASE_URL=http://127.0.0.1:8799/v1`, `JEV_OPENCODE_MODEL=jevonian/auto`, `OPENAI_API_KEY=local-no-key`, plus `--standalone` and `PWD` for OpenCode 2.x |

- The Jev key is read from `credentials\typesafe-ai-credential.txt` at launch and **not** copied into `~/.jev-gateway/.env`.
  Real environment variables win over that file.
- **Claude Code** turns get a tool hint from jev-gateway, then a tier and effort from Jevonian (same table as `claude`). Jevonian
  talks to Anthropic with your Claude login.
- **OpenCode** turns get tool routing from jev-gateway, then a tier and effort from Jevonian's OpenCode router.
- Want jev-gateway **without** Jevonian? Point `JEV_CLAUDE_UPSTREAM_BASE_URL` at `https://api.anthropic.com/v1` and drop the
  `ANTHROPIC_*` remaps. Claude Code's own model and login then go straight through, with no tiers.
- **OpenCode 2.x** is outside jev-gateway's tested scope (v1). `--standalone` and `PWD` are the arguments it needs.

### Official launcher commands

Arguments pass through, so everything official works:

```bat
jev-claude --dangerously-skip-permissions        :: Claude Code through the gateway (→ Jevonian)
jev-claude --model jevonian/plan -p "…"          :: pin a tier
jev-claude --status                              :: is it running, where does it forward, which key
jev-claude --dashboard                           :: open http://localhost:8789/dashboard
jev-claude --routing off                         :: baseline: stop asking Jev about tools, keep metering
jev-claude --stop
jev-opencode        |  jev-opencode run "…"  |  jev-opencode --status  |  jev-opencode --stop
```

Logs are in `%USERPROFILE%\.jev-gateway\claude.log` and `opencode.log` (the official location).

### The dashboard

`http://127.0.0.1:8789/dashboard` shows **both** gateways, because they're on the official default ports it looks for:
- the **claude · :8789** card, with upstream `http://127.0.0.1:8797/v1`
- the **opencode · :8791** card, with upstream `http://127.0.0.1:8799/v1`
- Jev's calls, latency and confidence; LLM tokens; *Why requests were not routed*; and a live table (mode, tool, confidence, status)

Console `ERR_CONNECTION_REFUSED` for 8787/8788/8790 is expected: the page also probes the standalone server, Gemini and Codex
ports. "**Passthrough only**" isn't a failure: Jev said no tool was needed, or its confidence was below 0.7, or there
were no tools. Model and effort for the same turns are on the **Jevonian** dashboard (:8797 or :8799, Logs page).

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "router on :8789 forwards to …, expected …" | A gateway started with other settings still owns the port | `jev-claude --stop` (or `jev-opencode --stop`), then run the command again |
| `jev-opencode`: "Model unavailable" | OpenCode 2.x background service | The command adds `--standalone`; add it yourself when running the official launcher by hand |
| Every call fails / connection refused upstream | The Jevonian router behind it is down | Run the command again (it starts Jevonian first) or `jev start` |
| `EADDRINUSE` on 8789/8791 | Another process owns the port | `netstat -ano \| findstr ":8791"` |

### Verified (2026-09-25)

- **`jev-claude`, five pinned tiers plus auto:** the gateway logged `jevonian/plan … jevonian/auto`, all 200, and the same turns
  in Jevonian :8797 matched the table (plan Opus 5.5 xhigh … chat Haiku low).
- **Tool use through the chain** (`jev-claude -p "List the files…"`): answered correctly. The gateway logged Jev's tool check.
- **`jev-opencode`:** the gateway logged mode `none`, and the same turn in Jevonian :8799 showed chat → deepseek-v4.1-flash, low.
- **Dashboards:** starting `jev-opencode` opened `http://127.0.0.1:8791/dashboard`. The :8789 dashboard shows both gateways.

---

## Summary

| Tool | Command | Router | Port | Picks | Upstream |
|---|---|---|---|---|---|
| Kilo | `kilo` | Jevonian | 8795 | tier + effort | Alibaba Token Plan |
| Qwen Code | `qwen` | Jevonian | 8793 | tier + effort | Alibaba Token Plan |
| OpenCode | `opencode` | Jevonian | 8799 | tier + effort | Alibaba Token Plan |
| Claude Code | `claude` | Jevonian (`jevonian launch claude`) | 8797 | tier + effort | Anthropic, OAuth |
| Claude Code | `jev-claude` | jev-gateway → Jevonian | 8789 → 8797 | tool, then tier + effort | Anthropic, OAuth |
| OpenCode | `jev-opencode` | jev-gateway → Jevonian | 8791 → 8799 | tool, then tier + effort | Alibaba Token Plan |

| Tier table | Model | Effort |
|---|---|---|
| **Alibaba** (kilo, qwen, opencode) | chat deepseek-v4.1-flash · small qwen3.8-flash · execute qwen3.8-flash · large qwen3.8-max · utility qwen3.7-plus · plan glm-5.3 | low · low · high · xhigh · medium · high |
| **Claude** (claude, jev-claude) | plan Opus 5.5 · heavy Opus 5.5 · execute Sonnet 5 · utility Sonnet 5 · chat Haiku 4.5 | xhigh · high · high · medium · low |

Local changes to the official projects are small and all re-applied automatically:
- `patch-jevonian-waf.mjs` (from this guide)
- `patch-jevonian-effort.mjs` (per-tier `effort`, and `forceEffort`)
- `patch-jevonian-haiku.mjs` (from this guide; Claude router only)

jev-gateway: **none**.

---

## Proof of Working

Verified **2026-09-25**, Windows 11, Node v22.23.2.

| Component | Version |
|---|---|
| jevonian | 0.1.7 (npm latest; GitHub `main` = 0.1.7) |
| jev-gateway | 0.4.3 (npm latest), unmodified |
| OpenCode / Kilo / Qwen Code / Claude Code | 2.0.15 / 7.7.9 / 0.24.4 / 2.1.282 |

**All six commands, one-shot, from an unrelated project folder:** `kilo`, `qwen`, `opencode`, `claude`, `jev-claude` and
`jev-opencode` each answered correctly and opened their status window. `jev status` showed all six UP.

**Alibaba tiers, 18/18:** 6 tiers × Qwen :8793, Kilo :8795, OpenCode :8799. Checked with the
`x-jevonian-model` / `x-jevonian-effort` headers and in each dashboard's **Logs → Effort** column:
`chat=deepseek-v4.1-flash/low  small=qwen3.8-flash/low  execute=qwen3.8-flash/high  large=qwen3.8-max/xhigh  utility=qwen3.7-plus/medium  plan=glm-5.3/high`.

**Claude tiers, the same through `claude` and `jev-claude`:**
`plan=opus-5-5/xhigh  heavy=opus-5-5/high  execute=sonnet-5/high  utility=sonnet-5/medium  chat=haiku-4-5/low`. All
returned 200, and `jevonian/auto` on "thanks" routed to chat/low. The gateway logged the same turns on its way to Jevonian.

**From the project root**, `claude --dangerously-skip-permissions -p "Which folder are you in?"` answered
`D:\learn\gemini-mcp\agy-opencode-jev`, routed to utility → Sonnet 5 / medium, with the project's 9 MCP servers loaded.
An interactive session was started in a CMD window there.

**Chains:** `jev-opencode` → gateway :8791 (`none`) → Jevonian :8799 (chat, deepseek, low). `jev-claude` tool use →
gateway :8789 → Jevonian :8797, which listed the files.

**Dashboards:** starting a router through a command opened its dashboard, e.g. `http://127.0.0.1:8791/dashboard`.

---

## Quick Reference Card

```bat
:: one-time
node D:\learn\gemini-mcp\agy-opencode-jev\cli\jev.js install

:: everyday (new CMD window, any folder)
kilo            qwen            opencode            claude --dangerously-skip-permissions
jev-claude --dangerously-skip-permissions           jev-opencode
jev status      jev start       jev stop            jev windows       jev dashboards

:: pin a tier
kilo -m jevonian/jevonian/plan     qwen -m jevonian/large     opencode -m jevonian/jevonian/small
claude --model jevonian/plan       jev-claude --model jevonian/heavy

:: dashboards
::   Jevonian     http://127.0.0.1:8793/  :8795/  :8797/  :8799/     (Logs -> Effort column)
::   jev-gateway  http://127.0.0.1:8789/dashboard                    (both gateways)

:: without any router
claude-direct   kilo-direct   qwen-direct   opencode-direct
```

---

## Final Checklist

- [ ] Node 22.15+; OpenCode, Kilo, Qwen Code and Claude Code on PATH; Claude Code logged in (`claude-direct`, then `/login`)
- [ ] `credentials\qwen-alibaba-credential.txt` and `typesafe-ai-credential.txt` valid; `AI_GATEWAY_API_KEY` in `.env` valid
- [ ] `npm install` done in each `jevnonian\jev-router-*` folder and in `jev-gateway\` (local, exact versions)
- [ ] Router logs show `WAF patch` / `Effort patch` (and `Haiku patch` for claude) as applied or already applied
- [ ] Ports free: 8789/8791 (jev-gateway) and 8793/8795/8797/8799 (Jevonian, +1 each)
- [ ] `node cli\jev.js install`, then a **new** CMD window, then `doskey /macros` lists the commands
- [ ] `jev status` shows all six UP once used, and each dashboard's Logs → Effort column matches the tier tables
- [ ] Nothing global: no `npm -g`, and nothing written to `~/.claude/settings.json` or `~/.config/opencode`
