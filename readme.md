# Jevonian + jev-gateway: Complete Guide (Windows CMD)

**OpenCode, Kilo, Qwen Code and Claude Code through [Jevonian](https://github.com/xinyao27/jevonian), plus
OpenCode and Claude Code through [jev-gateway](https://github.com/vinilana/jev-gateway). You type plain
`kilo` / `qwen` / `opencode` / `claude` in CMD. No `.bat` files.**

Both projects are used **as officially released** (jevonian **0.1.7**, jev-gateway **0.4.3**), each installed
locally with an exact version pin. Only three small, idempotent patches are applied to Jevonian (sections 4 and 11,
all re-applied automatically on every start), and **jev-gateway is not modified at all**. Anything bigger would break on
the next official upgrade. That's the rule this guide follows.

> **Last verified 2026-09-25** on Windows 11, Node v22.23.2, OpenCode 2.0.15, Kilo 7.7.9, Qwen Code 0.24.4 and
> Claude Code 2.1.282. The results are in [Proof of Working](#proof-of-working).

---

## Quick start

| You type in CMD | Goes through | Port | Upstream |
|---|---|---|---|
| `kilo` | Jevonian | 8795 | Alibaba Token Plan |
| `qwen` | Jevonian | 8793 | Alibaba Token Plan |
| `opencode` | Jevonian | 8799 | Alibaba Token Plan |
| `claude` | Jevonian (its official `jevonian launch claude`) | 8797 | Anthropic, with your Claude Code login (no API key) |
| `jev-claude` | jev-gateway (official launcher) | 8789 | Anthropic, with your Claude Code login forwarded as-is |
| `jev-opencode` | jev-gateway (official launcher) → Jevonian | 8791 → 8799 | Alibaba Token Plan |
| `jev status` / `jev start` / `jev stop` / `jev windows` | everything above | | |
| `claude-direct`, `opencode-direct`, `kilo-direct`, `qwen-direct` | the real client, no router | | |

```bat
:: one-time: make the commands exist in every NEW CMD window (no .bat files; see section 14)
node D:\learn\gemini-mcp\agy-opencode-jev\cli\jev.js install

:: then, in any project folder, in a NEW CMD window:
kilo
claude --dangerously-skip-permissions
jev status
```

Every command starts its router with `node` if it's down. It also opens (or reuses) a **status window** that shows the
**port, URL, dashboard and provider**, the tier → model → effort table, and a live line per request. The client itself
then runs in *your* window and folder.

**Which one when?** Qwen Code's own comparison (shared chat, 2026-09-23) matches how this repo is laid out:
*"Jevonian is better for Kilo/Qwen. jev-gateway is better for Claude Code. They're complementary, not competing."*
They answer different questions. **Jevonian** asks Jev *which model and effort* should serve this turn. **jev-gateway**
asks Jev *which tool* the agent should call next. `jev-opencode` chains both.

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
- [Claude Code through Jevonian (official `jevonian launch claude`)](#claude-code-through-jevonian-official-jevonian-launch-claude)
- [Claude Code: manual setup on any machine (Method A / Method B reference)](#claude-code-manual-setup-on-any-machine-method-a--method-b-reference)
- [Upgrading Jevonian to the Latest Version](#upgrading-jevonian-to-the-latest-version)
- [Claude Code: Problems You May Hit and How to Fix Them](#claude-code-problems-you-may-hit-and-how-to-fix-them)
- [14. Plain CMD commands without .bat, and status windows](#14-plain-cmd-commands-without-bat-and-status-windows)
- [15. Superseded: gargpratyush/jev-router and the .bat launchers](#15-superseded-gargpratyushjev-router-and-the-bat-launchers)
- [16. jev-gateway (official): Claude Code and OpenCode](#16-jev-gateway-official-claude-code-and-opencode)
- [Summary](#summary) · [Proof of Working](#proof-of-working) · [Quick Reference Card](#quick-reference-card) · [Final Checklist](#final-checklist)

---
## What is Jevonian?

**Jevonian** is a local AI routing proxy that sits between your coding agent and your model providers. Instead of manually switching models for different tasks, Jevonian uses **Jev** (a fast decision model from TypeSafe) to automatically route each turn to the most cost-effective capable model based on:

- **Phase**: planning, execution, utility, or chat
- **Context**: conversation size and complexity
- **Quota**: provider health and remaining budget
- **Cache**: recent hit ratios and switch penalties

Every request is logged with the actual model used, the reason, token counts, and estimated cost.

**Key features:**
- Per-turn routing decisions (not per-session)
- Multi-provider support (Alibaba Token Plan, OpenRouter, etc.)
- Local ledger with full audit trail
- Dashboard for monitoring and configuration
- Works with any OpenAI-compatible client

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
  [Upgrading](#upgrading-jevonian-to-the-latest-version)).

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
 │ claude    :8797  ── Jev picks TIER ──────────▶ Anthropic (your Claude login, OAuth)                  │
 └────────────────────────────────────────────────┘    │   Jev picks the next TOOL; the model is kept  │
                                                       └───────────────────────────────────────────────┘
 Brain: TypeSafe Jev (Vercel AI Gateway as Jevonian's fallback). Keys: credentials\ + .env, never copied.
```

- **Jevonian** is a model router. Each turn, Jev picks a *routing* (tier), and the routing names the model. The effort
  level comes from the routing's `effort` (patch, [section 4b](#4b-patch-jevonian-effortmjs-per-tier-effort)).
- **jev-gateway** is a tool router. Each turn, Jev picks which *tool* the agent should call (`forced` / `hint` / `none` /
  `direct` / `passthrough`). It does **not** change the model. The official version has no model tiers.
- `jev-opencode` sends OpenCode through jev-gateway, then through Jevonian's OpenCode router, using only
  jev-gateway's official `JEV_OPENCODE_UPSTREAM_BASE_URL` setting. The result is tool routing *and* tier + effort routing,
  with no code changes to either project.

---

## Folder Structure

```
agy-opencode-jev\
├── credentials\                     keys (never committed)
├── .env                             AI_GATEWAY_API_KEY (Vercel fallback brain)
├── cli\                             the plain CMD commands (section 14)
│   ├── jev.js                       install/uninstall + `jev status|start|stop|windows`
│   ├── jev.doskey                   the macros (generated by install)
│   ├── launch.js, common.js         shared launcher: start router, status window, run client
│   ├── monitor.js                   the status-window program
│   ├── targets.js                   every router/gateway: port, URL, provider
│   ├── jev-claude.js, jev-opencode.js, gateway-env.js    wrappers around the OFFICIAL jev-gateway launchers
├── jevnonian\                       Jevonian routers
│   ├── jev.js                       start|stop|status|test|logs [opencode|qwen|kilo|claude]
│   ├── kilo.js, qwen.js, opencode.js, claude.js       what the CMD commands run
│   ├── jev-router-qwen\      :8793  config\config.json, .qwen\settings.json, start.js, patches, data\, logs\
│   ├── jev-router-kilo\      :8795  config\config.json, .kilo\kilo.json, start.js, patches, data\, logs\
│   ├── jev-router-claude\    :8797  config\config.json, claude-jev-settings.json, start.js, patches (+haiku)
│   └── jev-router-opencode\  :8799  config\config.json, opencode.json, start.js, patches, data\, logs\
└── jev-gateway\                     package.json → "jev-gateway": "0.4.3" (official, unmodified, from npm)
```

Each router folder is self-contained: its own port, config, ledger (`data\ledger.jsonl`), dashboard and local
`node_modules\jevonian`. Nothing depends on the old `jev-router-guides\…` layout or on `.bat`/`.sh` files.

---

## Router Setup (per tool)

The three Alibaba routers (Qwen, Kilo, OpenCode) are identical except for `listen.port` and their client config.
The Claude router is described in [its own section](#claude-code-through-jevonian-official-jevonian-launch-claude).

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
| `patch-jevonian-effort.mjs` | Per-tier `effort` (section 2). | all |
| `patch-jevonian-haiku.mjs` | Haiku 4.5 rejects fields Claude Code always sends (still needed on 0.1.7; see [Claude section](#claude-code-through-jevonian-official-jevonian-launch-claude)). | Claude only |

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

Three edits, each with its own marker:
1. Keep `effort` when a routing is parsed.
2. On Jev-routed turns (`jevonian/auto`), the chosen routing's effort wins.
3. On **pinned** turns (`jevonian/<id>`), which in 0.1.7 take an early-return path that sends **no effort at all**,
   not even `defaultEffort`, the routing's effort is also sent.

It is still clamped to `capacities.<model>.efforts`. A level the **client** sets itself (`reasoning_effort`, `thinking`)
is still never overridden, which is the official rule.

```javascript
// patch-jevonian-effort.mjs — idempotent, jevonian 0.1.7. Nothing is written if any pattern is missing.
import { readFileSync, writeFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const file = join(dirname(fileURLToPath(import.meta.url)), "node_modules", "jevonian", "dist", "cli.mjs");
let src = readFileSync(file, "utf8");
const edits = [
  { mark: "/* routing-effort-patch v1 */",
    from: "\t\tmodels,\n\t\t...providers ? { providers } : {}\n\t};\n}\n/**\n* Build the routings list",
    to: (m) => "\t\tmodels,\n\t\t...providers ? { providers } : {},\n\t\t" + m + "\n\t\t...typeof value.effort === \"string\" && isReasoningEffort(value.effort) ? { effort: value.effort } : {}\n\t};\n}\n/**\n* Build the routings list" },
  { mark: "/* routing-effort-patch v1b */",
    from: "\tconst appliedEffort = clampEffort(wanted ?? requestedEffort ?? defaultEffort, ",
    to: (m) => "\t" + m + "\n\tconst routingEffort = config.routing.routings.find((entry) => entry.id === phase)?.effort;\n\tconst appliedEffort = clampEffort(routingEffort ?? wanted ?? requestedEffort ?? defaultEffort, " },
  { mark: "/* routing-effort-patch v2 */",
    from: "\t\t\tvirtual: true,\n\t\t\trouted: true,\n\t\t\treason,\n\t\t\tsession\n\t\t};\n\t}\n\tconst brains = config.routing.brains;",
    to: (m) => "\t\t\tvirtual: true,\n\t\t\trouted: true,\n\t\t\treason,\n\t\t\tsession,\n\t\t\t" + m + "\n" +
      "\t\t\t...(() => {\n" +
      "\t\t\t\tconst pinned = clampEffort(config.routing.routings.find((entry) => entry.id === phase)?.effort ?? headerEffort(headers) ?? brainEffort(config.routing.defaultEffort), effectiveCapabilities(picked.model, config.routing.capacities?.[picked.model]).efforts);\n" +
      "\t\t\t\treturn pinned ? { effort: pinned } : {};\n" +
      "\t\t\t})()\n" +
      "\t\t};\n\t}\n\tconst brains = config.routing.brains;" },
];
let applied = 0;
for (const e of edits) {
  if (src.includes(e.mark)) continue;
  const n = src.split(e.from).length - 1;
  if (n !== 1) { console.error(`Effort patch: pattern for ${e.mark} found ${n} times (expected 1) — refusing, nothing written`); process.exit(1); }
  src = src.replace(e.from, e.to(e.mark)); applied++;
}
if (!applied) { console.log("Effort patch: already applied"); process.exit(0); }
writeFileSync(file, src);
console.log(`Effort patch: applied (${applied} edits)`);
```

### 5. Start and check

```bat
node jevnonian\jev.js start          :: all four routers in the background (logs\serve.log in each folder)
node jevnonian\jev.js status         :: health + ledger line count per port
curl http://127.0.0.1:8795/healthz   :: {"ok":true,"sessions":0,"routing":"auto"}
curl http://127.0.0.1:8795/v1/models :: jevonian/auto, plan, execute, utility, chat, small, large
```

You don't need to start routers by hand: every CMD command ([section 14](#14-plain-cmd-commands-without-bat-and-status-windows))
starts its own router when it's down.

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
| 8791 | jev-gateway `jev-opencode` | official default. **Moved the Jevonian OpenCode router off 8791** (older versions of this guide used 8791 for it) |
| 8793 (+8794) | Jevonian Qwen | |
| 8795 (+8796) | Jevonian Kilo | |
| 8797 (+8798) | Jevonian Claude Code | |
| 8799 (+8800) | Jevonian OpenCode | upstream of `jev-opencode` |

On 2026-09-25 none of 8787–8800 was in a Windows (WinNAT/Hyper-V) excluded range. To check:
`netsh interface ipv4 show excludedportrange protocol=tcp`. `EADDRINUSE` usually means port **+1** is taken.

---

## Troubleshooting

### "Jev brain unavailable: all 2 configured brain(s) failed"

**Cause:** Both TypeSafe and Vercel brain channels timed out or errored.

**Fix:**
1. Check credentials in `credentials/` directory
2. Verify network connectivity to `api.typesafe.ai` and Vercel
3. Retry the request — transient failures are common
4. Check router logs: `cat logs/serve.log | grep brain`

### Kilo uses wrong router (parent config)

**Cause:** Kilo merges configs from cwd up to git root. A parent `kilo.json` overrides the subfolder's `kilo.json`.

**Fix:** Use `.kilo/kilo.json` (dot-directory) instead of `kilo.json` at the subfolder root. The dot-directory config is read from cwd and takes precedence.

### Qwen shows "Persisted model.baseUrl no longer matches"

**Cause:** Qwen cached a previous model selection with a different baseUrl.

**Fix:** Re-select the model once with `/model` → `jevonian/auto`. The warning is harmless and self-corrects.

### Router starts but no ledger entries

**Cause:** Requests aren't reaching the router (wrong port in client config).

**Fix:**
1. Verify the client config points to the right port (Qwen 8793, Kilo 8795, Claude 8797, OpenCode 8799). The CMD commands always do
2. Watch the router's status window, or run `jev status`: the live feed shows every request that reaches it
3. Ensure the router is running: `curl http://127.0.0.1:<port>/healthz`

### WAF patch fails

**Cause:** The Jevonian version changed. This repo's patches (WAF v2, effort, Haiku) target **0.1.7**.

**Fix:**
1. Check the version: `node -e "console.log(require('./node_modules/jevonian/package.json').version)"`
2. If it isn't 0.1.7, reinstall the pinned version (`npm install jevonian@0.1.7 --save-exact`) or follow [Upgrading](#upgrading-jevonian-to-the-latest-version). Never force a patch whose pattern isn't found.

### "No Jevonian API key exists yet"

**Cause:** The public surface (tunnel listener) requires a key, but none has been generated.

**Fix:** This only affects the tunnel (port+1), not the main listener. For local use, ignore it. If you need the tunnel, generate a key via the dashboard at `/keys`.

### Kilo answers "Add credits to continue, or switch to a free model"

Kilo ignored your config's default model and used its own cloud default (`minimax/…`). Pass
`-m jevonian/jevonian/auto`. The `kilo` command always does unless you give your own `-m`.

### OpenCode: "Model unavailable: jevonian/jevonian/auto", although `opencode models` lists it

OpenCode 2.x runs requests through a shared **background service**. If one was started earlier from another folder
(`opencode serve --service` in Task Manager), it holds *that* folder's config. Use `--standalone` (a private server
per run) and make sure `PWD` is the working folder. The `opencode` and `jev-opencode` commands do both. Old background
services can be stopped with `opencode service` (see `opencode --help`).

### `opencode run` hangs with no request in the ledger

`opencode run` also reads stdin when it isn't a terminal, and waits for the pipe to close. In scripts, add `< nul`.
An interactive CMD window is a terminal, so this never happens there.

### Effort shows `default` (or nothing) on pinned `jevonian/<tier>` requests

The effort patch is missing or old. In 0.1.7, pinned routings take a path that sends no effort at all. Restart the
router (`start.js` re-applies `patch-jevonian-effort.mjs`) and check with `findstr /c:"routing-effort-patch v2" node_modules\jevonian\dist\cli.mjs`.

### `rm`/`npm install` fails with "resource busy" inside `node_modules\jevonian`

A file watcher (often the IDE) holds the folder. The router then refuses to start, which is intended, rather than run
unpatched. Fix it without waiting for the lock: `npm pack jevonian@0.1.7`, unpack the tarball's `package\` contents over
`node_modules\jevonian\`, then restart the router (the patches re-apply).

---

## Claude Code through Jevonian (official `jevonian launch claude`)

Jevonian 0.1.7 ships its **own** Claude Code launcher, `jevonian launch claude [--model M] [--] [claude args…]`
([docs/cli.md](https://github.com/xinyao27/jevonian/blob/main/docs/cli.md)). It works like `ollama launch claude`:
- it points Claude Code at the local Anthropic-compatible endpoint (`ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`)
- it remaps Opus/Sonnet/Haiku onto `jevonian/*`
- it **never touches `~/.claude/settings.json`**

This repo uses it as-is. Typing `claude` in CMD runs:

```bat
node jevnonian\jev-router-claude\node_modules\jevonian\dist\cli.mjs launch claude --model jevonian/auto -- --settings jevnonian\jev-router-claude\claude-jev-settings.json <your args>
```

with `JEVONIAN_CONFIG` pointing at `jev-router-claude\config\config.json`. That's how `launch claude` finds port **8797**.
Your arguments go after `--`, so `claude --dangerously-skip-permissions` and `claude -p "…"` work as usual.
`claude --model jevonian/plan` pins a tier.

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
      { "id": "plan",    "label": "Plan",    "description": "architecture, design, multi-file planning, hard reasoning", "models": ["claude-opus-5-5"] },
      { "id": "execute", "label": "Execute", "description": "implementation, debugging, tool loops",                     "models": ["claude-sonnet-5"] },
      { "id": "utility", "label": "Utility", "description": "summaries, lookups, small mechanical edits",                "models": ["claude-sonnet-5"] },
      { "id": "chat",    "label": "Chat",    "description": "short conversational replies, acknowledgements",           "models": ["claude-haiku-4-5-20251001"],
        "providers": { "claude-haiku-4-5-20251001": ["claude-subscription-haiku"] } },
      { "id": "heavy",   "label": "Heavy",   "description": "large tasks, big multi-file work, maximum reasoning",       "models": ["claude-sonnet-5"] }
    ],
    "sessionTtlMinutes": 720,
    "baselineModel": "claude-sonnet-5",
    "capacities": {
      "claude-opus-5-5":           { "contextWindow": 1000000, "maxOutput": 128000 },
      "claude-sonnet-5":           { "contextWindow": 1000000, "maxOutput": 128000 },
      "claude-haiku-4-5-20251001": { "contextWindow": 200000,  "maxOutput": 65536 }
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

The provider fields `auth: "oauth"` and `oauthSource: "claude-code"` are the documented Claude Pro/Max provider
([docs/providers.md](https://github.com/xinyao27/jevonian/blob/main/docs/providers.md)). Haiku gets its own provider,
because Haiku rejects the 1M-context beta. The chat routing's `providers` must be a **map** keyed by model id.
`modelSync` is off, so the Haiku provider's list can't grow.

### Effort for Claude Code

Claude Code **always sends its own effort level**, and Jevonian never overrides a client's level (official rule). So the
router's `effort` field doesn't apply to Claude. The level is set **inside Claude Code**, per virtual model, with
`jev-router-claude\claude-jev-settings.json`, which the `claude` command passes as `--settings`:

```json
{
  "model": "jevonian/auto",
  "modelSettings": {
    "jevonian/auto":    { "effortLevel": "high" },
    "jevonian/plan":    { "effortLevel": "high" },
    "jevonian/execute": { "effortLevel": "high" },
    "jevonian/utility": { "effortLevel": "low" },
    "jevonian/heavy":   { "effortLevel": "xhigh" }
  }
}
```

`jevonian/auto` runs at one level (`high`) for the whole session. Pin `--model jevonian/heavy` to get `xhigh`.

### The Haiku patch is still needed on 0.1.7

This was verified on 2026-09-25. Without it, every chat-tier turn failed:
`400 context_management: Extra inputs are not permitted`. Claude Code always sends `context_management`,
`output_config.effort`, adaptive `thinking` and, since 2.1.x, **mid-conversation `role:"system"` messages**, and Haiku 4.5
rejects each of those. The guide's `patch-jevonian-haiku.mjs` ([Step 7a](#7a-patch-jevonian-haikumjs--makes-the-haiku-chat-tier-work)
below, used verbatim) sanitizes the body for legacy-thinking models only. With it, chat → Haiku returns 200. It lives
**only** in `jev-router-claude\`, and `start.js` applies it there.

### Verified (2026-09-25)

`claude -p …` through :8797 returned 200 on every request:
- "Reply exactly" and "thanks!" routed to chat → `claude-haiku-4-5-20251001` (provider `claude-subscription-haiku`)
- a tool-use turn routed to execute → `claude-sonnet-5`, then utility → `claude-sonnet-5`

The Logs page shows effort `high` (Claude Code's own, from `claude-jev-settings.json`).

> The official alternative, **Clients → Connect Claude** on the dashboard, writes `~/.claude/settings.json`, so *every*
> `claude` on the machine then depends on this router. The `claude` command keeps it per-invocation instead. Method A/B
> below are the manual equivalents, for machines without this repo's `cli\`.

---

## Claude Code: manual setup on any machine (Method A / Method B reference)

> **In this repo, just type `claude`** (previous section). It runs Jevonian's official `jevonian launch claude`
> against `jevnonian\jev-router-claude` (port 8797). What follows is the original manual procedure, still valid, for a
> machine or project without this repo's `cli\` folder. The example port `8744` is arbitrary. Keep any router at least
> 2 away from 8789/8791 (jev-gateway) and 8793–8800 (this repo's Jevonian routers).

This section sets up **Claude Code → local Jevonian router → your Claude subscription** on any Windows machine, in any project folder. Every command is for **CMD** (Command Prompt). Nothing depends on a particular folder layout: you set four variables once, and every command after that uses them.

When you finish, typing this in your project folder will route through Jev:

```bat
claude --dangerously-skip-permissions
```

### How it fits together

```
 claude (Claude Code)
    │   ANTHROPIC_BASE_URL=http://127.0.0.1:<PORT>   ← from .claude\settings.local.json (or the launcher)
    ▼
 Jevonian router  (%PROJECT%\.claude\jev-router, node start.js)
    │   Jev brain (TypeSafe) picks a tier per turn: plan / execute / utility / chat / heavy
    ▼
 https://api.anthropic.com  using your Claude Code login (OAuth, subscription billing)
```

How this differs from the OpenCode/Kilo/Qwen setup above:

- **No model API key.** The router forwards to Anthropic using the OAuth login Claude Code already has (`"auth": "oauth", "oauthSource": "claude-code"`). The only key you need is the **TypeSafe** key for the Jev routing brain.
- **Claude Code doesn't read `.env` files.** It picks up the router in one of two ways: a launcher script that sets the environment (**Method A**), or an `env` block written into `.claude\settings.local.json` (**Method B**). Method B is what makes a plain `claude --dangerously-skip-permissions` work.
- **Haiku needs a patch.** Claude Code always sends fields that Haiku 4.5 rejects with HTTP 400 (`output_config.effort`, adaptive `thinking`, `context_management`, mid-conversation `system` messages). If the chat tier routes to Haiku, you need `patch-jevonian-haiku.mjs`.

### Files you will create

```
%PROJECT%\
├── .claude\
│   ├── jev.env                     ← ON/OFF switch (USE_JEV) + router port
│   ├── claude-jev-settings.json    ← effort level per tier
│   ├── run-claude.js               ← Method A launcher
│   ├── jev-apply.js                ← Method B: writes/removes router settings in settings.local.json
│   ├── settings.local.json         ← (existing or new) Claude Code reads this; jev-apply.js edits it
│   └── jev-router\                 ← the router itself
│       ├── package.json
│       ├── start.js
│       ├── patch-jevonian-haiku.mjs
│       ├── patch-jevonian-waf.mjs
│       ├── .env                    ← optional: AI_GATEWAY_API_KEY (fallback brain)
│       ├── credentials\
│       │   └── typesafe-ai-credential.txt   ← your TypeSafe key, one line
│       ├── config\config.json
│       └── data\                   ← ledger.jsonl etc. (created at runtime)
└── scripts\
    └── claude-jev.bat              ← (optional) Method A shortcut; not needed
```

---

### Step 1 — Check prerequisites

Open **CMD** and run:

```bat
node --version
npm --version
where claude
curl --version
```

- `node` must be **v22 or newer**.
- `where claude` must print the path to `claude.exe`. If it prints nothing, install Claude Code first.
- `curl` comes with Windows 10 and newer.

**Log in to Claude Code with your Claude subscription** if you haven't already. The router reuses this login:

```bat
claude
```

Inside Claude Code, run `/login` if it asks, then exit with `/exit`.

**Get a TypeSafe API key** (https://typesafe.ai). This is the Jev routing brain. A Vercel AI Gateway key is an optional fallback brain.

---

### Step 2 — Set your variables (once per CMD window)

Change the path to your own project folder. Pick any free port; 8744 is only an example.

```bat
set PROJECT=C:\path\to\your\project
set ROUTER=%PROJECT%\.claude\jev-router
set PORT=8744
```

Check that the port **and the port after it** are both free (Jevonian always binds `PORT` and `PORT+1`):

```bat
netstat -ano | findstr ":8744 :8745"
```

No output means both are free. If anything prints, pick another port that is at least 2 away from every other router, then `set PORT=...` again.

> `set` only lasts for the current CMD window. If you open a new window, run Step 2 again.

---

### Step 3 — Create the router folder and install Jevonian (locally)

```bat
mkdir "%ROUTER%\config" "%ROUTER%\data" "%ROUTER%\credentials"
cd /d "%ROUTER%"
npm init -y
npm install jevonian@0.1.7 --save-exact
```

- Always install into this folder. **Never use `npm install -g`.** Each router keeps its own copy, so you can upgrade one without touching the others.
- `--save-exact` pins the exact version. Patches match the bundle's text, so an unexpected version bump can break them. To check for a newer version first, see [Upgrading Jevonian](#upgrading-jevonian-to-the-latest-version).

---

### Step 4 — Add your keys

```bat
notepad "%ROUTER%\credentials\typesafe-ai-credential.txt"
```

Paste **only** your TypeSafe key on one line, save, and close.

Optional fallback brain:

```bat
notepad "%ROUTER%\.env"
```

```ini
AI_GATEWAY_API_KEY=your-vercel-ai-gateway-key
```

Keep these files out of git. Add these lines to `%PROJECT%\.gitignore`:

```gitignore
.claude/jev-router/node_modules/
.claude/jev-router/data/
.claude/jev-router/credentials/
.claude/jev-router/.env
.claude/settings.local.json
```

> **Sharing one set of keys between several routers:** set `JEV_ROUTER_CREDENTIALS_DIR` to a folder that contains `credentials\typesafe-ai-credential.txt` (and optionally `.env`) before starting the router. `start.js` reads from there instead of its own folder, so you don't have to copy the secrets around.

---

### Step 5 — `config\config.json`

```bat
notepad "%ROUTER%\config\config.json"
```

Paste the following and **change `8744` to your port**:

```json
{
  "listen": { "host": "127.0.0.1", "port": 8744 },
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
        "claude-opus-5",
        "claude-fable-5",
        "claude-sonnet-4-6",
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
      "models": ["claude-haiku-4-5-20251001"],
      "injectStreamUsage": true,
      "headers": {
        "anthropic-beta": "claude-code-20250219,interleaved-thinking-2025-05-14,thinking-token-count-2026-05-13"
      }
    }
  ],
  "tunnel": { "enabled": false, "provider": "cloudflare" },
  "routing": {
    "mode": "auto",
    "routings": [
      { "id": "plan",    "label": "Plan",    "description": "architecture, design, multi-file planning, hard reasoning", "models": ["claude-opus-5-5"] },
      { "id": "execute", "label": "Execute", "description": "implementation, debugging, tool loops",                     "models": ["claude-sonnet-5"] },
      { "id": "utility", "label": "Utility", "description": "summaries, lookups, small mechanical edits",                "models": ["claude-sonnet-5"] },
      { "id": "chat",    "label": "Chat",    "description": "short conversational replies, acknowledgements",           "models": ["claude-haiku-4-5-20251001"],
        "providers": { "claude-haiku-4-5-20251001": ["claude-subscription-haiku"] } },
      { "id": "heavy",   "label": "Heavy",   "description": "large tasks, big multi-file work, maximum reasoning",       "models": ["claude-sonnet-5"] }
    ],
    "sessionTtlMinutes": 720,
    "baselineModel": "claude-sonnet-5",
    "capacities": {
      "claude-opus-5-5":           { "contextWindow": 1000000, "maxOutput": 128000 },
      "claude-sonnet-5":           { "contextWindow": 1000000, "maxOutput": 128000 },
      "claude-haiku-4-5-20251001": { "contextWindow": 200000,  "maxOutput": 65536 }
    },
    "quotaGuard": { "enabled": true, "lowPercent": 10 },
    "brains": [
      { "channel": "typesafe", "apiKeyEnv": "TYPESAFE_API_KEY",   "timeoutMs": 8000,  "minConfidence": 0.6 },
      { "channel": "vercel",   "apiKeyEnv": "AI_GATEWAY_API_KEY", "timeoutMs": 20000, "minConfidence": 0.6 }
    ],
    "brainPicksEffort": false
  },
  "modelSync": { "enabled": true, "intervalMinutes": 720 }
}
```

Why it's shaped this way:

- **Two providers, same login.** The main provider sends the full `anthropic-beta` list, which 1M-context models need. Haiku 4.5 rejects `context-1m-2025-08-07` outright, so Haiku gets its own provider with a shorter list. The provider's header **replaces** Claude Code's beta list rather than merging with it, so each provider has to declare every beta its models need.
- **`"providers"` inside the chat routing must be a map keyed by model id**, as shown. An array is silently ignored, and Haiku then goes through whichever provider lists it first (the main one), which gives you a 400.
- **Model ids change over time.** If Anthropic renames or adds models, edit the `models` lists. The router's dashboard (Providers page) shows what it can see.

---

### Step 6 — `start.js`

```bat
notepad "%ROUTER%\start.js"
```

```js
// start.js — start this folder's Jevonian router (Windows-friendly, no bash needed).
// Loads the Jev brain keys, points Jevonian at this folder's config/data, runs `jevonian serve`.
const { readFileSync } = require("fs");
const { join } = require("path");
const { spawn } = require("child_process");

const ROUTER_DIR = __dirname;
// Folder holding credentials\typesafe-ai-credential.txt and .env. Defaults to this
// router folder; set JEV_ROUTER_CREDENTIALS_DIR to share one folder between routers.
const CREDS_DIR = process.env.JEV_ROUTER_CREDENTIALS_DIR || ROUTER_DIR;

function readText(rel) { try { return readFileSync(join(CREDS_DIR, rel), "utf8"); } catch { return ""; } }
function readEnv(rel) {
  const out = {};
  for (const line of readText(rel).split(/\r?\n/)) {
    const m = line.match(/^([^#=]+)=(.*)$/);
    if (m) out[m[1].trim()] = m[2].trim();
  }
  return out;
}

// A key already set in the environment wins over the files.
const typesafeKey = process.env.TYPESAFE_API_KEY || readText("credentials/typesafe-ai-credential.txt").replace(/\s+/g, "");
const aiGatewayKey = process.env.AI_GATEWAY_API_KEY || readEnv(".env").AI_GATEWAY_API_KEY || "";

for (const [k, v] of [["TYPESAFE_API_KEY", typesafeKey], ["AI_GATEWAY_API_KEY", aiGatewayKey]]) {
  if (!v) console.error(`WARNING: ${k} is empty (looked in ${CREDS_DIR})`);
  process.env[k] = v;
}

process.env.JEVONIAN_CONFIG = join(ROUTER_DIR, "config", "config.json");
process.env.JEVONIAN_CREDENTIALS = join(ROUTER_DIR, "config", "credentials.json");
process.env.JEVONIAN_DATA_DIR = join(ROUTER_DIR, "data");
process.env.JEVONIAN_LEDGER = join(ROUTER_DIR, "data", "ledger.jsonl");
process.env.JEVONIAN_UPDATE_STATE = join(ROUTER_DIR, "data", "update.json");
process.env.JEVONIAN_NO_OPEN = "1";

const cli = join(ROUTER_DIR, "node_modules", "jevonian", "dist", "cli.mjs");
const child = spawn("node", [cli, "serve", "--foreground"], { stdio: "inherit" });
child.on("exit", (code) => process.exit(code));
```

---

### Step 7 — The two patches

Both patches edit `node_modules\jevonian\dist\cli.mjs` in place. Both are **idempotent**: each checks for its marker and exits if it's already applied. Both **check every pattern before writing anything**, so a failed patch leaves the file untouched. `npm install` replaces the patched file, so **re-run both after every install or upgrade**.

#### 7a. `patch-jevonian-haiku.mjs` — makes the Haiku chat tier work

```bat
notepad "%ROUTER%\patch-jevonian-haiku.mjs"
```

```js
// Idempotent patch — makes legacy-thinking models (Haiku 4.5 and older, i.e. every model
// anthropicThinkingSupport() classifies as `adaptive: false`) able to serve Claude Code
// requests. Claude Code always sends `output_config.effort`, adaptive `thinking`,
// `context_management`, and mid-conversation `role:"system"` messages; Haiku 4.5 rejects
// each with HTTP 400. This sanitizes the outgoing body for exactly those models at
// withEffort(), which every outgoing Anthropic body passes through. Sonnet/Opus untouched.
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

#### 7b. `patch-jevonian-waf.mjs` — keeps Cloudflare's WAF from rejecting brain calls

The routing brain (TypeSafe) sits behind Cloudflare. If a turn's text contains something that *looks* like an attack (`| python -c`, `/etc/passwd`, `<script`, `../`), the WAF can reject the brain call mid-session. **Jevonian 0.1.7 and newer already redact shell commands on their own** (`formatToolCallForBrain` → `<command redacted>`). This v2 patch covers the rest: it defangs the raw text inside `brainState`.

```bat
notepad "%ROUTER%\patch-jevonian-waf.mjs"
```

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

> **Staying on jevonian 0.1.6?** The v2 WAF patch still applies there, but 0.1.6 doesn't redact shell commands on its own, so that half goes unprotected. Moving to 0.1.7+ is simpler.

#### 7c. Apply and validate

```bat
cd /d "%ROUTER%"
node patch-jevonian-haiku.mjs
node patch-jevonian-waf.mjs
node --check node_modules\jevonian\dist\cli.mjs && echo OK: bundle syntax valid
find /c "haiku-safe-patch v1" node_modules\jevonian\dist\cli.mjs
find /c "waf-safe-patch v2" node_modules\jevonian\dist\cli.mjs
```

Expected: both patches print `applied` (or `already applied`), the syntax check prints `OK`, and each `find /c` reports a count of **2** or more. If a patch prints `pattern not found`, **don't force it or skip it.** See [Problems you may hit](#claude-code-problems-you-may-hit-and-how-to-fix-them).

---

### Step 8 — Start the router and check it

Start it in its own window so it keeps running:

```bat
start "jev-router" cmd /k node "%ROUTER%\start.js"
```

In the new window you should see `jevonian listening on http://127.0.0.1:<PORT>/`, both providers listed, and the virtual models (`jevonian/auto`, `jevonian/plan`, `jevonian/execute`, `jevonian/utility`, `jevonian/chat`, `jevonian/heavy`). If you see `WARNING: TYPESAFE_API_KEY is empty`, the key file is missing or in the wrong folder (Step 4).

Back in your original window:

```bat
curl -s -o nul -w "status %%{http_code}\n" http://127.0.0.1:%PORT%/v1/models -H "Authorization: Bearer jevonian-local"
```

Expected: `status 200`. (Typed directly at the CMD prompt, use a single `%` in `%{http_code}`. Double it to `%%` only inside a `.bat` file.)

Open the router's dashboard:

```bat
start http://127.0.0.1:%PORT%/
```

It shows the running version, both providers as `live`, and the tier table. Keep it open: after each test below, the **Overview** request count should go up.

---

### Step 9 — Project files

Create the `.claude` folder if it doesn't exist:

```bat
mkdir "%PROJECT%\.claude" "%PROJECT%\scripts" 2>nul
```

#### 9a. `.claude\jev.env` — the switch

```bat
notepad "%PROJECT%\.claude\jev.env"
```

```ini
# Jev switch for Claude Code in this project.
#   USE_JEV=true   -> Claude Code routes through the local Jevonian router
#   USE_JEV=false  -> plain Claude Code (default)
#
# This file is read by .claude\run-claude.js (Method A) and .claude\jev-apply.js (Method B).
# A bare `claude` never reads it directly. After editing, run:  node .claude\jev-apply.js
USE_JEV=true

# Must match "listen.port" in .claude\jev-router\config\config.json
JEV_ROUTER_PORT=8744
```

Set `JEV_ROUTER_PORT` to your port.

#### 9b. `.claude\claude-jev-settings.json` — effort per tier

Claude Code sets the **effort level itself, per model id**, and the router never overrides it. This file is where each virtual model's effort is set:

```bat
notepad "%PROJECT%\.claude\claude-jev-settings.json"
```

```json
{
  "model": "jevonian/auto",
  "modelSettings": {
    "jevonian/auto":    { "effortLevel": "high" },
    "jevonian/plan":    { "effortLevel": "high" },
    "jevonian/utility": { "effortLevel": "low" },
    "jevonian/execute": { "effortLevel": "high" },
    "jevonian/heavy":   { "effortLevel": "xhigh" }
  }
}
```

> Consequence: `jevonian/auto` runs at **one** effort level (here `high`) for the whole session, even on turns where Jev routes to the heavy tier. To actually get `xhigh`, start with `--model jevonian/heavy`.

#### 9c. `.claude\run-claude.js` — Method A launcher

```bat
notepad "%PROJECT%\.claude\run-claude.js"
```

```js
// run-claude.js — launch Claude Code for this project, through the local Jevonian router
// when USE_JEV=true in .claude\jev.env; otherwise launch plain Claude Code unchanged.
//
// Do NOT rename this file to claude.js: Windows PATHEXT includes .JS, so a claude.js on the
// lookup path shadows the real claude.exe and the launcher ends up spawning itself (EFTYPE).
//
// Usage:  node .claude\run-claude.js [any claude args]
//         node .claude\run-claude.js --dangerously-skip-permissions
//         set USE_JEV=true && node .claude\run-claude.js -p "hi"     (one-off override)
const { spawnSync } = require("child_process");
const { existsSync, readFileSync } = require("fs");
const { join, resolve } = require("path");
const { homedir } = require("os");

const HERE = __dirname; // .claude\

function loadEnvFile(path) {
  const out = {};
  if (!existsSync(path)) return out;
  for (const line of readFileSync(path, "utf8").split(/\r?\n/)) {
    const t = line.trim();
    if (!t || t.startsWith("#")) continue;
    const eq = t.indexOf("=");
    if (eq !== -1) out[t.slice(0, eq).trim()] = t.slice(eq + 1).trim();
  }
  return out;
}
const fileEnv = loadEnvFile(join(HERE, "jev.env"));
// The process environment wins over the file, so a one-off `set USE_JEV=...` works.
function flag(name, fallback) {
  const v = process.env[name] ?? fileEnv[name];
  return v === undefined ? fallback : v;
}
const USE_JEV = /^(true|1|on|yes)$/i.test(flag("USE_JEV", "false"));
const PORT = Number(flag("JEV_ROUTER_PORT", "8744"));
const SETTINGS_FILE = flag("JEV_CLAUDE_SETTINGS", join(HERE, "claude-jev-settings.json"));

// Find the real claude.exe, never anything inside .claude\ and never a script file.
function findClaude() {
  const isWin = process.platform === "win32";
  const badExt = /\.(js|sh|cmd|bat|mjs|cjs)$/i;
  const ok = (p) => p && existsSync(p) && !badExt.test(p)
    && !resolve(p).toLowerCase().startsWith(resolve(HERE).toLowerCase());
  const probe = spawnSync(isWin ? "where" : "which", isWin ? ["claude.exe"] : ["claude"], { encoding: "utf8" });
  for (const c of (probe.stdout || "").trim().split(/\r?\n/).filter(Boolean)) if (ok(c)) return c;
  const name = isWin ? "claude.exe" : "claude";
  for (const c of [join(homedir(), ".local", "bin", name), join(homedir(), ".claude", "local", name)]) if (ok(c)) return c;
  throw new Error("Claude Code binary not found — install Claude Code and make sure `where claude` finds claude.exe.");
}

const argv = process.argv.slice(2);
const dd = argv.indexOf("--");
const claudeArgs = dd === -1 ? argv : argv.slice(dd + 1);
let env = { ...process.env };

if (USE_JEV) {
  if (!claudeArgs.some((a) => a === "--settings" || a.startsWith("--settings="))) {
    if (existsSync(SETTINGS_FILE)) claudeArgs.unshift("--settings", SETTINGS_FILE);
    else console.error(`[jev] warning: ${SETTINGS_FILE} not found — launching without per-tier effort`);
  }
  env = {
    ...env,
    ANTHROPIC_BASE_URL: `http://127.0.0.1:${PORT}`,   // no /v1 — Claude Code appends /v1/messages
    ANTHROPIC_AUTH_TOKEN: "jevonian-local",            // loopback sentinel the router accepts from 127.0.0.1
    ANTHROPIC_API_KEY: "",                              // never send a real key through the router
    ANTHROPIC_DEFAULT_OPUS_MODEL: "jevonian/plan",
    ANTHROPIC_DEFAULT_SONNET_MODEL: "jevonian/auto",
    ANTHROPIC_DEFAULT_HAIKU_MODEL: "jevonian/utility",
    CLAUDE_CODE_SUBAGENT_MODEL: "jevonian/utility",
    ANTHROPIC_SMALL_FAST_MODEL: "jevonian/utility",     // keeps background calls off the Haiku chat tier
    ANTHROPIC_CUSTOM_MODEL_OPTION: "jevonian/heavy",
    CLAUDE_CODE_MAX_CONTEXT_TOKENS: "1000000",
    CLAUDE_CODE_DISABLE_UNKNOWN_MODEL_WINDOW_ENFORCEMENT: "1",
    CLAUDE_CODE_ATTRIBUTION_HEADER: "0",
    DISABLE_ERROR_REPORTING: "1",
    DISABLE_FEEDBACK_COMMAND: "1",
    CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY: "1",
  };
}

let claude;
try { claude = findClaude(); } catch (e) { console.error(e.message); process.exit(1); }

console.error(`[jev] USE_JEV=${USE_JEV}`);
console.error(`[jev] claude -> ${claude}`);
console.error(USE_JEV ? `[jev] base   -> ${env.ANTHROPIC_BASE_URL}  model -> jevonian/auto` : `[jev] base   -> (unmodified — plain Claude Code)`);

const result = spawnSync(claude, claudeArgs, { stdio: "inherit", env, windowsHide: false });
if (result.error) { console.error(result.error.message); process.exit(1); }
process.exit(result.status === null ? 1 : result.status);
```

#### 9d. (Optional) `scripts\claude-jev.bat` — Method A shortcut

> Not needed, and not used in this repo (no `.bat` files): run `node .claude\run-claude.js …` directly, or map it to a
> command with a doskey macro ([section 14](#14-plain-cmd-commands-without-bat-and-status-windows)). The shortcut is only for people who want one.

```bat
notepad "%PROJECT%\scripts\claude-jev.bat"
```

```bat
@echo off
rem Launch Claude Code for this project via .claude\run-claude.js (reads .claude\jev.env).
rem   scripts\claude-jev.bat --dangerously-skip-permissions
node "%~dp0..\.claude\run-claude.js" %*
```

#### 9e. `.claude\jev-apply.js` — Method B (makes a bare `claude` work)

Claude Code reads an `env` block, `model`, and `modelSettings` from `.claude\settings.local.json` in the folder you launch it from. This script copies `jev.env` into those keys, or removes exactly those keys again. Everything else in the file is left alone.

```bat
notepad "%PROJECT%\.claude\jev-apply.js"
```

```js
// jev-apply.js — make a PLAIN `claude` (no launcher) follow USE_JEV in .claude\jev.env.
//   USE_JEV=true  -> writes router env + model + per-tier effort into .claude\settings.local.json
//   USE_JEV=false -> removes exactly those keys again (everything else is left alone)
//
// Run after every change to jev.env:   node .claude\jev-apply.js
// Then start a NEW session:            claude --dangerously-skip-permissions
//
// Only keys Claude Code already knows are written (no marker keys): a settings file that
// fails validation is silently ignored in -p mode.
const { existsSync, readFileSync, writeFileSync } = require("fs");
const { join } = require("path");

const DIR = __dirname;
const SETTINGS = join(DIR, "settings.local.json");

function loadEnvFile(path) {
  const out = {};
  if (!existsSync(path)) return out;
  for (const line of readFileSync(path, "utf8").split(/\r?\n/)) {
    const t = line.trim();
    if (!t || t.startsWith("#")) continue;
    const eq = t.indexOf("=");
    if (eq !== -1) out[t.slice(0, eq).trim()] = t.slice(eq + 1).trim();
  }
  return out;
}

const fileEnv = loadEnvFile(join(DIR, "jev.env"));
const USE_JEV = /^(true|1|on|yes)$/i.test(fileEnv.USE_JEV || "false");
const PORT = Number(fileEnv.JEV_ROUTER_PORT || 8744);

// Same variables run-claude.js sets, minus ANTHROPIC_API_KEY: an empty key in settings still
// counts as an auth source, and the sentinel token is all the router needs.
const JEV_ENV = {
  ANTHROPIC_BASE_URL: `http://127.0.0.1:${PORT}`,
  ANTHROPIC_AUTH_TOKEN: "jevonian-local",
  ANTHROPIC_DEFAULT_OPUS_MODEL: "jevonian/plan",
  ANTHROPIC_DEFAULT_SONNET_MODEL: "jevonian/auto",
  ANTHROPIC_DEFAULT_HAIKU_MODEL: "jevonian/utility",
  CLAUDE_CODE_SUBAGENT_MODEL: "jevonian/utility",
  ANTHROPIC_SMALL_FAST_MODEL: "jevonian/utility",
  ANTHROPIC_CUSTOM_MODEL_OPTION: "jevonian/heavy",
  CLAUDE_CODE_MAX_CONTEXT_TOKENS: "1000000",
  CLAUDE_CODE_DISABLE_UNKNOWN_MODEL_WINDOW_ENFORCEMENT: "1",
  CLAUDE_CODE_ATTRIBUTION_HEADER: "0",
  DISABLE_ERROR_REPORTING: "1",
  DISABLE_FEEDBACK_COMMAND: "1",
  CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY: "1",
};
const JEV_MODEL = "jevonian/auto";
const JEV_MODEL_SETTINGS = JSON.parse(readFileSync(join(DIR, "claude-jev-settings.json"), "utf8")).modelSettings;

const settings = existsSync(SETTINGS) ? JSON.parse(readFileSync(SETTINGS, "utf8")) : {};

if (USE_JEV) {
  settings.env = { ...(settings.env || {}), ...JEV_ENV };
  settings.model = JEV_MODEL;
  settings.modelSettings = { ...(settings.modelSettings || {}), ...JEV_MODEL_SETTINGS };
} else {
  if (settings.env) {
    for (const k of Object.keys(JEV_ENV)) delete settings.env[k];
    if (!Object.keys(settings.env).length) delete settings.env;
  }
  if (settings.model === JEV_MODEL) delete settings.model;
  if (settings.modelSettings) {
    for (const k of Object.keys(JEV_MODEL_SETTINGS)) delete settings.modelSettings[k];
    if (!Object.keys(settings.modelSettings).length) delete settings.modelSettings;
  }
}

writeFileSync(SETTINGS, JSON.stringify(settings, null, 2) + "\n");
console.log(`[jev-apply] USE_JEV=${USE_JEV} -> ${SETTINGS} ${USE_JEV ? `routes plain claude to 127.0.0.1:${PORT}` : "restored to plain Claude Code"}`);
```

---

### Step 10 — Run `claude --dangerously-skip-permissions` through Jev

**Use a fresh CMD window** (see the note at the end of this step), then:

```bat
set PROJECT=C:\path\to\your\project
cd /d "%PROJECT%"
node .claude\jev-apply.js
claude --dangerously-skip-permissions
```

`jev-apply.js` should print `USE_JEV=true -> ... routes plain claude to 127.0.0.1:<PORT>`.

To check you really are on Jev, look for these inside the session:

- Send any short message, for example `hi`. The router dashboard's **Overview → Requests** count goes up, and **Activity** shows the model it went to. This is the most reliable check.
- `/model` reports the current model as **`jevonian/auto`**.
- **Don't** expect the numbered list in the `/model` picker to say "Jevonian". It keeps Claude Code's stock names (Opus / Sonnet / Haiku / Fable). There are env vars named `ANTHROPIC_DEFAULT_*_MODEL_NAME`, but they don't rename those entries. The signal that counts is `jevonian/auto` as the current model, plus requests showing up on the dashboard.
- One benign line on startup: `[claude-code:unrecognized_model] {"model":"jevonian/auto",...}`. It only means `jevonian/auto` isn't in Claude Code's built-in model list.
- If you also have a claude.ai login, this banner is benign: `claude.ai connectors are disabled because ANTHROPIC_API_KEY or another auth source is set`.

**Method A (launcher) instead:**

```bat
cd /d "%PROJECT%"
node .claude\run-claude.js --dangerously-skip-permissions
```

It prints `[jev] USE_JEV=true` and `[jev] base -> http://127.0.0.1:<PORT>` before Claude Code starts. With Method A you don't have to run `jev-apply.js`, because it reads `jev.env` on every launch.

| | Method A — `node .claude\run-claude.js` | Method B — `jev-apply.js` + plain `claude` |
|---|---|---|
| What you type | `node .claude\run-claude.js --dangerously-skip-permissions` | `claude --dangerously-skip-permissions` |
| When a `jev.env` change takes effect | next launch, automatically | after `node .claude\jev-apply.js` **and** a new session |
| If you forget | you get plain Claude Code, **with no error** | not applicable: plain `claude` follows the last applied state |
| Good for | scripts, one-off `-p` runs | your everyday terminal |

You can set up both; they don't conflict.

> **Fresh CMD window:** `ANTHROPIC_*` variables in the window's environment override the project settings file. If this window was opened from another tool (or from inside another Claude Code session) that sets them, `claude` may go somewhere else. Check with `set ANTHROPIC`. The right answer is `Environment variable ANTHROPIC not defined`. If anything is listed, clear it for this window:
>
> ```bat
> set ANTHROPIC_BASE_URL=
> set ANTHROPIC_AUTH_TOKEN=
> set ANTHROPIC_API_KEY=
> ```

---

### Step 11 — Turn it off (and back on)

**Off:**

```bat
cd /d "%PROJECT%"
notepad .claude\jev.env
```

Set `USE_JEV=false` and save, then run:

```bat
node .claude\jev-apply.js
```

It prints `restored to plain Claude Code`. **Start a new session.** A session that's already running keeps the settings it started with.

**On again:** set `USE_JEV=true`, run `node .claude\jev-apply.js`, and start a new session.

> While `USE_JEV=true` is applied, a plain `claude` **depends on the router running**. If the router window is closed, every request fails until you start the router again (Step 8) or switch Jev off.

---

### Step 12 — Test checklist (CMD)

Run these in a fresh CMD window from `%PROJECT%`, with the router running. Use this helper to see the last ledger lines:

```bat
node -e "const l=require('fs').readFileSync(process.argv[1],'utf8').trim().split('\n');console.log(l.length+' lines');console.log(l.slice(-2).join('\n'))" "%ROUTER%\data\ledger.jsonl"
```

1. **Router health:** the `curl` from Step 8 prints `status 200`.
2. **Jev on, bare `claude`:**
   ```bat
   node .claude\jev-apply.js
   claude --dangerously-skip-permissions -p "Reply with exactly: hi" --output-format text
   ```
   Expect `hi`. The ledger line count goes up, the last line has `"requestedModel":"jevonian/auto"` and `"status":200`.
3. **Chat tier (tests the Haiku patch):**
   ```bat
   claude --dangerously-skip-permissions -p "thanks!" --output-format text
   ```
   The last ledger line shows `"provider":"claude-subscription-haiku"`, `"model":"claude-haiku-4-5-20251001"`, `"status":200`. Jev decides the tier, so a different short message may land on another tier. Any `400` here means the Haiku patch isn't active.
4. **Launcher:** `node .claude\run-claude.js -p "Reply with exactly: hi" --output-format text`. The `[jev]` banner appears and the ledger count goes up.
5. **Jev off:** set `USE_JEV=false`, run `node .claude\jev-apply.js`, then repeat test 2. Expect `hi` and **no** new ledger line. That proves the router was not contacted.
6. **Back on:** set `USE_JEV=true` and run `node .claude\jev-apply.js`.
7. **Tools/MCP still work through Jev (optional):**
   ```bat
   claude --dangerously-skip-permissions -p "List the files in this folder." --output-format text
   ```

> In `-p` (headless) runs, pass `--dangerously-skip-permissions` (or `--permission-mode bypassPermissions`) **on the command line**. A `defaultMode` set in `.claude\settings.json` doesn't reliably apply to headless runs; tool calls such as MCP tools get blocked.

---

### Stopping, restarting, uninstalling

- **Stop the router:** press `Ctrl+C` in the `jev-router` window. If you've lost the window:
  ```bat
  netstat -ano | findstr ":%PORT%"
  taskkill /PID <pid-from-last-column> /F
  ```
- **Restart it:** `start "jev-router" cmd /k node "%ROUTER%\start.js"`.
- **Uninstall:** set `USE_JEV=false` and run `node .claude\jev-apply.js`, stop the router, then delete `.claude\jev-router\`, `.claude\jev.env`, `.claude\claude-jev-settings.json`, `.claude\run-claude.js`, `.claude\jev-apply.js` and `scripts\claude-jev.bat`. Nothing outside the project was changed.

---

## Upgrading Jevonian to the Latest Version

This applies to any router folder in this guide. Each router has its **own** local `node_modules\jevonian`, so upgrade them one at a time. It's fine, and sometimes deliberate, for different routers to run different versions.

### 1. Find the latest version and read what changed

```bat
cd /d "%ROUTER%"
npm view jevonian version
node -e "console.log(require('./node_modules/jevonian/package.json').version)"
```

The first command prints the newest published version; the second prints the one you have. Then read the release notes:

```
https://github.com/xinyao27/jevonian/releases
```

Look for:
- changes to `config.json` fields you use (`routing.routings[].providers`, provider fields, `brains`)
- changes to how tool calls or brain state are built (this is what the WAF patch edits)
- changes to `serve` or the dashboard

A release with only bug fixes is low risk. Anything that touches routing or providers needs a full re-test.

### 2. Stop the router

Press `Ctrl+C` in its window, or use `taskkill` as above. Don't reinstall packages while a running process has them loaded.

### 3. Install the new version, exact pin, in this folder only

```bat
npm install jevonian@X.Y.Z --save-exact
node -e "console.log(require('./node_modules/jevonian/package.json').version)"
```

Replace `X.Y.Z` with the version from step 1. Use `--save-exact` so a later `npm install` can't slip in a version you haven't tested.

### 4. Re-apply the patches and validate

```bat
node patch-jevonian-haiku.mjs
node patch-jevonian-waf.mjs
node --check node_modules\jevonian\dist\cli.mjs && echo OK: bundle syntax valid
find /c "haiku-safe-patch v1" node_modules\jevonian\dist\cli.mjs
find /c "waf-safe-patch v2" node_modules\jevonian\dist\cli.mjs
```

**If a patch says `pattern not found`**, the upgrade changed the code that patch edits. Nothing was written, because the patches check before they write. Find the new code shape:

```bat
findstr /n "withEffort formatToolCallForBrain brainState calls.push" node_modules\jevonian\dist\cli.mjs
```

Then update the patch's search strings to match. Real example: going from 0.1.6 to 0.1.7, the old WAF patch looked for `` calls.push(`${name}(${args})`) ``, but 0.1.7 had replaced that with its own `formatToolCallForBrain()` helper, which already redacts shell commands. The fix was the v2 patch in Step 7b: it drops the edits upstream now handles and keeps only the `brainState` wrap. Check the release notes to confirm this kind of thing ("tool arguments are now redacted before reaching the brain").

### 5. Restart and test the paths the patches protect

```bat
start "jev-router" cmd /k node "%ROUTER%\start.js"
curl -s -o nul -w "status %{http_code}\n" http://127.0.0.1:%PORT%/v1/models -H "Authorization: Bearer jevonian-local"
cd /d "%PROJECT%"
claude --dangerously-skip-permissions -p "Reply with exactly: hi" --output-format text
claude --dangerously-skip-permissions -p "thanks!" --output-format text
```

Check the ledger (Step 12 helper): `status 200`, and for the chat test `claude-subscription-haiku` with no 400s. The dashboard's update card should show the new version as `current`. If it shows a version-mismatch notice, the package is new but the process is old, so restart the router.

### 6. Roll back if needed

```bat
npm install jevonian@<previous-version> --save-exact
node patch-jevonian-haiku.mjs
node patch-jevonian-waf.mjs
```

Then restart the router. Versions older than 0.1.7 need the older WAF patch v1, which isn't reproduced here any more. Staying on 0.1.7 is simpler.

### 7. In this repo: patches re-apply themselves

In `jevnonian\jev-router-*`, `start.js` re-applies every `patch-jevonian-*.mjs` in its folder on each start, and
refuses to start if one can't apply. After `npm install jevonian@X.Y.Z --save-exact`:
1. Restart the router: `node jevnonian\jev.js stop kilo`, then `… start kilo`.
2. Read the first lines of `logs\serve.log`: each patch prints `applied` or `already applied`.
3. If one prints `pattern … found 0 times`, the new bundle changed that code. Diagnose it as in step 4 above, or roll back.
   **Check the release notes first.** If Jevonian ever ships per-routing effort or Haiku handling officially, delete
   that patch file instead of updating it.

jev-gateway upgrades are plain: `cd jev-gateway && npm install jev-gateway@X.Y.Z --save-exact`, then
`jev-claude --stop` and `jev-opencode --stop`. Nothing in it is patched. Check the release notes for changed env var
names or default ports.

---

## Claude Code: Problems You May Hit and How to Fix Them

These are real problems from setting this up, in the order they tend to appear.

### "I set `USE_JEV=true` but `claude` is still plain Claude Code"

**Signs:** the startup banner shows a stock model (for example `Sonnet 5 with high effort · Claude Max`), `/model` doesn't report `jevonian/auto`, and the dashboard request count doesn't move.

**Causes, most common first:**
1. You edited `jev.env` but **didn't run `node .claude\jev-apply.js`**. A bare `claude` never reads `jev.env`; it only reads `settings.local.json`.
2. You ran `jev-apply.js` but you're still in a **session started before that**. Exit and start a new one.
3. You launched `claude` from a **different folder**. Claude Code reads `.claude\settings.local.json` from the folder you start it in. `cd /d "%PROJECT%"` first.
4. The CMD window has its own `ANTHROPIC_BASE_URL` set (see the Step 10 note). `set ANTHROPIC` shows it; clear it.

**Don't** just retry. Check which of the four it is. First confirm the router itself is fine with the Step 8 `curl`, so you know the problem is on the launch side.

### "`/model` doesn't show Jev / Jevonian"

**Expected.** The picker's list always shows Claude Code's stock model names. What tells you Jev is active is `/model` reporting the current model as `jevonian/auto`, plus requests appearing on the router dashboard. The `ANTHROPIC_DEFAULT_*_MODEL_NAME` env vars exist but don't relabel those entries, so don't spend time on them.

**Watch out:** `/model` is a slash command, and **slash commands don't run in `-p` mode**. `claude -p "/model"` just sends the text "/model" as a prompt. To check the model, use an interactive session, or use the dashboard.

### Every request fails / "connection refused"

The router isn't running, or it's on a different port than `JEV_ROUTER_PORT`. Start it (Step 8), and make sure `listen.port` in `config.json` equals `JEV_ROUTER_PORT` in `jev.env`. If you need Claude now and can't start the router, switch Jev off (Step 11).

### Router won't start: `EADDRINUSE`

`PORT` **or `PORT+1`** is already taken, often by another Jevonian router. Check `netstat -ano | findstr ":%PORT%"`, pick a port at least 2 away from other routers, and update **both** `config.json` and `jev.env`. Then run `jev-apply.js` again.

### Router log: `WARNING: TYPESAFE_API_KEY is empty`

The key file isn't where `start.js` looks. By default that's `%ROUTER%\credentials\typesafe-ai-credential.txt`, or under `JEV_ROUTER_CREDENTIALS_DIR` if you set it. Routing still works, but without the Jev brain it falls back to heuristics. Fix the file location, or `set TYPESAFE_API_KEY=...` before `node start.js`.

### HTTP 400 on short/chatty turns (the Haiku chat tier)

The Haiku patch isn't active. Usually an `npm install` or upgrade overwrote it. Run `node patch-jevonian-haiku.mjs` and restart the router. Also check that the chat routing's `"providers"` is a **map**, `{"claude-haiku-4-5-20251001": ["claude-subscription-haiku"]}`, and not an array. An array is silently ignored, so Haiku gets the 1M-context beta header and rejects it.

**Watch out:** on startup, `modelSync` can add more model ids to the `claude-subscription-haiku` provider's list in `config.json`. That's harmless as long as `claude-subscription` is listed first in `providers`. If the ledger ever shows Sonnet or Opus requests going through `claude-subscription-haiku`, set `"modelSync": { "enabled": false }` and remove the extra ids from the Haiku provider.

### A patch prints `pattern not found`

The Jevonian version changed the code the patch targets. Nothing was written. **Don't** force it, delete the check, or skip the patch without understanding why. Diagnose with `findstr` and read the release notes (see [Upgrading](#upgrading-jevonian-to-the-latest-version), step 4).

### Brain calls fail mid-session with WAF / 403-style errors

The WAF patch is missing, or it's the wrong version for your Jevonian. Use v2 on 0.1.7+ and check `find /c "waf-safe-patch v2" ...`. Restart the router afterwards.

### Tests look like they pass, but you're not sure the router was really used

Don't trust the reply text. `hi` looks the same from plain Claude Code and from Jev. Use the **ledger line count** or the **dashboard request count**: they must go up when Jev is on and stay the same when it's off. If you test from a window that inherited `ANTHROPIC_*` variables (for example one opened from inside another Claude Code session), the result is meaningless. Use a fresh CMD window and check `set ANTHROPIC`.

### Headless (`-p`) runs block tool calls or MCP tools

Pass `--dangerously-skip-permissions` or `--permission-mode bypassPermissions` on the command line. `defaultMode` in settings doesn't reliably apply to `-p` runs.

### `robocopy` / Windows paths misbehave when copying a router folder

If you copy an existing, working router folder instead of building a new one, do it from **CMD or PowerShell**, not a POSIX shell (Git Bash can misread `/E` as a path). Leave out the old ledger:

```bat
robocopy "C:\path\to\existing\jev-router" "%ROUTER%" /E /XD data .git
mkdir "%ROUTER%\data"
```

Then check every path in the copy: `start.js`'s credentials folder, `listen.port` in `config.json`, and `JEV_ROUTER_PORT` in `jev.env`. A copy keeps the original's assumptions.

### Intermittent 401s when two Claude routers run at once

Every router using `"oauthSource": "claude-code"` shares the **same** Claude login on the machine. Refresh tokens are effectively single-use, so two routers refreshing at about the same time can invalidate each other. Prefer **one** Claude router per machine and point several projects at it (same `JEV_ROUTER_PORT` in each project's `jev.env`). Only run a second one when you really need them isolated, and if 401s appear, stop one of them.

### Things not to do

- **Don't** name the launcher `claude.js`: Windows `PATHEXT` makes it shadow `claude.exe` and spawn itself.
- **Don't** put the router settings into your **global** `%USERPROFILE%\.claude\settings.json`. Every project on the machine would then depend on this router. Keep them in the project's `.claude\settings.local.json` (Method B) or in the launcher (Method A).
- **Don't** put the router `env` block into the project's shared `.claude\settings.json` either, for the same reason within a team. Teammates without the router would be broken. `settings.local.json` is per-machine and git-ignored.
- **Don't** install Jevonian globally (`npm install -g`) or without `--save-exact`.
- **Don't** commit `credentials\`, `.env`, `data\` or `settings.local.json`.
- **Don't** trust a UI label or a "looks fine" reply as proof. Check the ledger or dashboard numbers.

### `400 context_management: Extra inputs are not permitted` (or `role 'system' is not supported on this model`)

The chat tier sent Claude Code's request to **Haiku 4.5** without the Haiku patch. Seen on jevonian 0.1.7 on 2026-09-25.
Claude Code 2.1.x also sends a mid-conversation `role:"system"` message (environment context) after your prompt, which
Haiku rejects too. `patch-jevonian-haiku.mjs` handles all of these. Make sure it's in `jev-router-claude\` and restart
the router.

### `[claude-code:unrecognized_model] {"model":"jevonian/auto",…}` on stderr

Benign. `jevonian/auto` isn't in Claude Code's built-in model list. The request still goes through.

### `claude` in CMD opens plain Claude Code, not Jevonian

- The window was opened **before** `cli\jev.js install`. Open a new one.
- You're in PowerShell, or inside a `.bat`. Macros are CMD-prompt only: run `node …\jevnonian\claude.js …`.
- Your environment already has `ANTHROPIC_BASE_URL` (check with `set ANTHROPIC`). The launcher's values are set per process, but
  settings files can still win. Start from a clean window.

---

## 14. Plain CMD commands without .bat, and status windows

### 14.1 Is it possible? Yes, with doskey macros and CMD AutoRun

The goal: type plain `kilo`, `qwen`, `opencode` or `claude` in CMD, have it call the router's server files with
`node`, show which **port, URL and provider** it uses, and use **no `.bat` files**.

| Approach | Works? | Why |
|---|---|---|
| Name a Node script `kilo.js` and put it on `PATH` | ❌ | `.JS` is in `PATHEXT`, but Windows runs it with **Windows Script Host (JScript)**, not Node. Also, a `claude.js` on the lookup path would shadow `claude.exe`. |
| `kilo.cmd` / `kilo.bat` shims | ❌ (by choice) | These are batch files. |
| **doskey macros + CMD AutoRun** | ✅ | CMD has a built-in alias feature, **doskey**. A macro file (`cli\jev.doskey`, plain text) maps `kilo` to `node "…\jevnonian\kilo.js" $*`. CMD loads it in every new window through the per-user registry value `HKCU\Software\Microsoft\Command Processor\AutoRun`. |

**Limits. Please read before relying on it:**
- Macros work only at the **interactive CMD prompt**. They don't apply inside `.bat`/`.cmd` scripts, in
  **PowerShell** or Windows Terminal's PowerShell profile, in VS Code's Claude Code extension, or in `cmd /d` (AutoRun off).
  There, `kilo` is still the plain Kilo. Use `node …\jevnonian\kilo.js …` explicitly.
- Only **new** CMD windows pick them up. Macros shadow the real binaries at the prompt, so
  `claude-direct`, `kilo-direct`, `opencode-direct` and `qwen-direct` run the real clients with no router.
- AutoRun runs for every CMD window, including `cmd /c` started by other programs. The fragment is one
  `if exist … doskey /macrofile=…` and prints nothing. Any AutoRun value you already had is kept, and backed up.

### 14.2 Install / remove

```bat
node D:\learn\gemini-mcp\agy-opencode-jev\cli\jev.js install     :: writes cli\jev.doskey + adds the AutoRun fragment
node D:\learn\gemini-mcp\agy-opencode-jev\cli\jev.js uninstall   :: removes exactly that fragment (restores the rest)
```

Check it in a **new** CMD window: `doskey /macros` should list:

```
kilo=node "D:\…\jevnonian\kilo.js" $*          qwen=node "D:\…\jevnonian\qwen.js" $*
opencode=node "D:\…\jevnonian\opencode.js" $*  claude=node "D:\…\jevnonian\claude.js" $*
jev-claude=node "D:\…\cli\jev-claude.js" $*    jev-opencode=node "D:\…\cli\jev-opencode.js" $*
jev=node "D:\…\cli\jev.js" $*                  claude-direct / opencode-direct / kilo-direct / qwen-direct
```

> The macro file must not contain comment lines: doskey has no comment syntax, and a `;` line prints
> *"Invalid macro definition."* in every new CMD window.

### 14.3 What happens when you type `kilo`

1. `node jevnonian\kilo.js` checks `http://127.0.0.1:8795/healthz`. If the router is down, it starts it in the background
   with `node jevnonian\jev.js start kilo`, which runs `start.js` and so the patches and `jevonian serve`.
2. It opens a **status window** titled `JEVONIAN - KILO - :8795` (`start "…" cmd /k node cli\monitor.js jevonian-kilo`),
   or reuses it if it's still open. Closing that window does **not** stop the router.
3. It prints the same facts in your window, then runs the real `kilo.exe` **in your current folder**, with
   `KILO_CONFIG_CONTENT` = the router's `.kilo\kilo.json` and `-m jevonian/jevonian/auto` (unless you gave `-m`).
   Every argument is passed through: `kilo run "…"`, `kilo -m jevonian/jevonian/plan`, and so on.

The other commands are the same with their own wiring:

| Command | Status window | Wiring |
|---|---|---|
| `qwen` | `JEVONIAN - QWEN - :8793` | `--auth-type openai --openai-base-url http://127.0.0.1:8793/v1 -m jevonian/auto` |
| `opencode` | `JEVONIAN - OPENCODE - :8799` | `OPENCODE_CONFIG_CONTENT`, `--standalone`, `PWD`, `-m jevonian/jevonian/auto` |
| `claude` | `JEVONIAN - CLAUDE - :8797` | official `jevonian launch claude -- --settings claude-jev-settings.json …` |
| `jev-claude` | `JEV-GATEWAY - CLAUDE - :8789` | official jev-gateway launcher `bin\jev-claude.mjs` |
| `jev-opencode` | `JEV-GATEWAY - OPENCODE - :8791` | official `bin\jev-opencode.mjs` + env (section 16) + `--standalone` |

### 14.4 What a status window shows

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

A jev-gateway window (`JEV-GATEWAY - OPENCODE - :8791`) shows the gateway's own data instead: for each request, its
**mode** (`forced` / `hint` / `none` / `passthrough`), the tool Jev picked, and the reason. For `jev-opencode` it also
lists the Jevonian tiers behind it.

`jev status` prints the same summary for everything at once, including which status windows are open.
`jev windows` opens all of them.

### 14.5 Why the interactive TUIs weren't driven by a script here

The one-shot forms of every command were tested from a CMD session (see [Proof of Working](#proof-of-working)), and the
status windows were opened for real. A script that *types* into a CMD window (SendKeys) was **blocked by the machine's
endpoint protection (Cylance Script Control)**, and it wasn't worked around. To see the TUIs, open a new CMD window and
type `claude --dangerously-skip-permissions`, `kilo`, `qwen`, `opencode`, `jev-claude` or `jev-opencode`.

---

## 15. Superseded: gargpratyush/jev-router and the .bat launchers

Earlier versions of this guide had:
- `.bat`/`.sh` launchers with `jev-launcher.js`, dynamic port fallback and client-config rewriting
- a recommendation to move Claude Code to `gargpratyush/jev-router` (`jev-claude`)

Both are **superseded**:
- The launchers are replaced by the plain CMD commands (section 14). Fixed ports and no config rewriting mean nothing
  can drift.
- Claude Code now has two **official** paths: Jevonian's own `jevonian launch claude` (model/tier routing, the `claude`
  command) and jev-gateway's official `jev-claude` (tool routing). A third router adds nothing.
- The old `plugins\jev-model-router\` function-hook plugin and its `pluginConfigs` in `.claude\settings.json` were
  removed. Leaving it in would double-route alongside the above.

---

## 16. jev-gateway (official): Claude Code and OpenCode

### 16.1 What the official jev-gateway does, and doesn't do

[vinilana/jev-gateway](https://github.com/vinilana/jev-gateway) **0.4.3** (latest on npm and GitHub, 2026-09-25) asks
Jev **which tool** the agent should call next, then steers the LLM (`forced` via `tool_choice`, `hint` for Claude Code
with thinking on, `none`, or `direct` with no LLM call). Everything else passes through untouched.
**It does not choose models, and it has no tiers or effort settings.**

> **Correction to earlier versions of this section.** They described jev-gateway routing Claude Code across
> Haiku/Sonnet/Opus and OpenCode across Alibaba models ("6-tier routing", `JEV_ROUTING_*`, a `stripEffort` fix in
> `src/adapters/messages.ts`). **Official jev-gateway has none of that.** It was a local modification of about 2,600
> diff lines in the gateway's core, which would break on every official upgrade. It has been **removed**. The patch is
> kept only as an archive file (`jev-gateway-local-tier-patch-2026-09-25.diff`). Model tiers and effort come from
> **Jevonian** instead, which is built for that.

In this setup, jev-gateway is used **only with Claude Code and OpenCode**, its two officially supported agents here.
Kilo and Qwen Code use Jevonian.

### 16.2 Install (official package, local, unmodified)

```bat
cd D:\learn\gemini-mcp\agy-opencode-jev\jev-gateway
:: package.json: { "dependencies": { "jev-gateway": "0.4.3" } }
npm install --no-audit --no-fund
```

The official launchers are then `node_modules\jev-gateway\bin\jev-claude.mjs` and `jev-opencode.mjs`. This repo never
edits them. The `jev-claude` and `jev-opencode` CMD commands only set **documented** environment variables and
arguments, open the status window, and then run those launchers.

### 16.3 Ports and wiring (all official settings)

| Command | Official launcher | Port | Upstream | Settings used (all documented in jev-gateway's README) |
|---|---|---|---|---|
| `jev-claude` | `jev-claude.mjs` | **8789** (default) | `https://api.anthropic.com/v1` (default) | `TYPESAFE_API_KEY`, `JEV_PROVIDER=typesafe` |
| `jev-opencode` | `jev-opencode.mjs` | **8791** (default) | **the Jevonian OpenCode router** `http://127.0.0.1:8799/v1` | `JEV_OPENCODE_UPSTREAM_BASE_URL=http://127.0.0.1:8799/v1`, `JEV_OPENCODE_MODEL=jevonian/auto`, `OPENAI_API_KEY=local-no-key` |

- The Jev key comes from `credentials\typesafe-ai-credential.txt` at launch and is **not** copied into `~/.jev-gateway/.env`.
  The official launcher lets real environment variables win over that file.
- **Claude Code** keeps its own claude.ai login. The official launcher sets only `ANTHROPIC_BASE_URL`.
- **OpenCode:** the official launcher injects a `jev-gateway` provider through `OPENCODE_CONFIG_CONTENT` (your files
  are never written) with the model `jev-gateway/jevonian/auto`. The gateway forwards it to Jevonian, which picks the
  tier and effort. So **OpenCode → jev-gateway (tool) → Jevonian (model + effort) → Alibaba**, with no code changes.
  Want OpenCode straight to Alibaba through the gateway instead? Set `JEV_OPENCODE_UPSTREAM_BASE_URL` to the Alibaba URL,
  `JEV_OPENCODE_MODEL` to one model (e.g. `qwen3.8-flash`) and `OPENAI_API_KEY` to the Alibaba key. Then there are no tiers.
- **OpenCode 2.x** is outside jev-gateway's tested scope (v1). The wrapper adds OpenCode's own `--standalone` and `PWD`
  (see [Client Configuration](#opencode-jev-router-opencodeopencodejson)). Those are arguments only.

### 16.4 Official launcher commands

Everything official works through the CMD commands, because arguments are passed through:

```bat
jev-claude --dangerously-skip-permissions        :: start the gateway if needed, then Claude Code through it
jev-claude --status                              :: is it running, where does it forward, which key
jev-claude --dashboard                           :: open http://localhost:8789/dashboard
jev-claude --routing off                         :: baseline mode: stop asking Jev, keep metering tokens
jev-claude --stop
jev-opencode                                     :: OpenCode TUI through the gateway (→ Jevonian)
jev-opencode run "fix the failing test"
jev-opencode --status | --stop
```

Logs are in `%USERPROFILE%\.jev-gateway\claude.log` and `opencode.log` (official location).

### 16.5 The dashboard

`http://127.0.0.1:8789/dashboard` shows **both** gateways on one page, because they're on the official default ports
that the page looks for:
- **claude · :8789** card: *Routing* / *Passthrough only* / *Idle*
- **opencode · :8791** card, showing its upstream `http://127.0.0.1:8799/v1`
- Jev's calls, latency and confidence; LLM tokens; *Why requests were not routed*; and a live table of requests
  (mode, tool, confidence, status)

The browser console shows `ERR_CONNECTION_REFUSED` for 8787/8788/8790: the page also probes the default ports of the
standalone server, Gemini and Codex, which aren't running. That's expected, per jev-gateway's own README.

"**Passthrough only**" is not a failure. For Claude Code, Jev often says no tool is needed (`no_tool_needed`), or the
request has no tools (`no_tools`). jev-gateway's own benchmark says the gain is mainly on large tool lists and debugging
tasks. Measure your own work with `--routing off` / `on`.

### 16.6 Verified (2026-09-25)

- `jev-claude -p "Reply with exactly: …"`: answered, and the dashboard logged `claude-sonnet-5` (Claude Code's own model)
  as `passthrough / no_tool_needed`, status 200.
- `jev-opencode run "Reply with exactly: …"`: answered. The gateway :8791 logged `jevonian/auto` with mode `none`,
  status 200. **The same turn** appears in Jevonian :8799's ledger as `chat → deepseek-v4.1-flash, effort low, brain jev`.
- Both gateways were started by their official launchers (`--start`) and show on one dashboard.

### 16.7 Claude Code + `AGENTS.md`: the context-mode gotcha

If Claude Code starts throwing:

```
Error: No such tool available: mcp__context_mode_ctx_search
```

…when the user asks it to "search your memory" or similar, the cause is a **tooling mismatch**
between Kilo and Claude Code in this repo:

- `AGENTS.md` at the project root is written for **Kilo**. It instructs the model to call
  `context-mode_ctx_*` tools (`ctx_search`, `ctx_execute`, `ctx_batch_execute`, …). Those tools are
  provided by Kilo's runtime; they are **not** installed in Claude Code.
- Claude Code's memory-file precedence is: `CLAUDE.md` first, then `AGENTS.md` as a fallback.
  When no `CLAUDE.md` exists, Claude Code reads `AGENTS.md`, sees the context-mode instructions,
  and the model invents a tool call (`mcp__context_mode_ctx_search`) — which fails because the tool
  is not in Claude Code's real tool list.

**Fix.** Create a project `CLAUDE.md` that tells Claude Code the truth about its tooling. With
`CLAUDE.md` present, Claude Code loads *it* instead of `AGENTS.md`, and the phantom-tool error
disappears. The Kilo `AGENTS.md` is left untouched — Kilo continues to load its own context-mode
instructions.

A minimal `CLAUDE.md` for this project:

```markdown
# CLAUDE.md — Claude Code guidance for this repo

This file is for **Claude Code**. (Kilo reads `AGENTS.md` instead.)

## Do NOT use context-mode tools here

The repo's `AGENTS.md` is written for Kilo, which ships the *context-mode* MCP tools
(`ctx_execute`, `ctx_search`, `ctx_batch_execute`, …). **Those tools are not installed in Claude
Code.** When you see the context-mode "Think-in-Code" / routing instructions, **ignore them in
Claude Code.** Use Claude Code's native tools instead:

| Instead of (context-mode) | Use (Claude Code native) |
|---|---|
| `ctx_execute` | `Bash` (write a short `node -e` / script) |
| `ctx_execute_file` | `Read` + `Bash`, or `Grep` |
| `ctx_search` / `ctx_batch_execute` | `Grep` and `Glob` |
| `ctx_fetch_and_index` | `WebFetch` |

In short: **prefer native Read/Grep/Glob/Bash/WebFetch.** There is no sandbox MCP server in this
Claude Code session, so do not reference one.
```

> **Why not install context-mode into Claude Code?** The `context-mode` npm package (v1.0.169)
> does support Claude Code, and `~/.claude/settings.json` already has
> `"enabledPlugins": { "context-mode@context-mode": true }` — but the marketplace that hosts the
> plugin is not registered, so the plugin never loads. Completing that install is a separate task
> (it needs the marketplace source URL, which the package doesn't document for Claude Code). The
> `CLAUDE.md` workaround is immediate, safe, and does not touch global settings.

### 16.8 jev-gateway troubleshooting quick list

| Symptom | Cause | Fix |
|---|---|---|
| `jev-opencode`: "router on :8791 forwards to …, expected …" | A gateway started with other settings (e.g. upstream OpenAI) still owns 8791 | `jev-opencode --stop`, then run it again |
| `jev-opencode`: "Model unavailable" | OpenCode 2.x background service | The command adds `--standalone`. If you run the official launcher by hand, add it yourself |
| Every Claude call 401s through `jev-claude` | Claude Code's own login is invalid (the gateway forwards it unchanged) | `claude-direct`, then `/login` |
| `EADDRINUSE` on 8789/8791 | Another process owns the port | `netstat -ano \| findstr ":8791"`. Older versions of this guide put the Jevonian OpenCode router there; it's on 8799 now |
| Dashboard card "Passthrough only" | Jev said no tool was needed, or there were no tools | Expected (16.5) |

---

## Summary

| Tool | Router | Port | Picks | Upstream |
|---|---|---|---|---|
| Kilo (`kilo`) | Jevonian | 8795 | tier + effort | Alibaba Token Plan |
| Qwen Code (`qwen`) | Jevonian | 8793 | tier + effort | Alibaba Token Plan |
| OpenCode (`opencode`) | Jevonian | 8799 | tier + effort | Alibaba Token Plan |
| Claude Code (`claude`) | Jevonian (`jevonian launch claude`) | 8797 | tier (effort: Claude Code's own) | Anthropic, OAuth |
| Claude Code (`jev-claude`) | jev-gateway (official) | 8789 | tool | Anthropic, OAuth |
| OpenCode (`jev-opencode`) | jev-gateway (official) → Jevonian | 8791 → 8799 | tool, then tier + effort | Alibaba Token Plan |

Local changes to the official projects, all small, all re-applied automatically:
- `patch-jevonian-waf.mjs` (from this guide)
- `patch-jevonian-effort.mjs` (per-tier effort)
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

**All six commands, one-shot, from an unrelated project folder:**

```
kilo run "Reply with exactly: KILO_OK"                         -> KILO_OK
qwen --output-format text -p "Reply with exactly: QWEN_OK"     -> QWEN_OK
opencode run "Reply with exactly: OPENCODE_OK"                 -> OPENCODE_OK
claude -p "Reply with exactly: CLAUDE_JEV_OK" …                -> CLAUDE_JEV_OK
jev-claude -p "Reply with exactly: JEV_CLAUDE_GW_OK" …         -> JEV_CLAUDE_GW_OK
jev-opencode run "Reply with exactly: JEV_OPENCODE_GW_OK"      -> JEV_OPENCODE_GW_OK
```

Each opened its status window (`JEVONIAN - KILO - :8795`, …, `JEV-GATEWAY - OPENCODE - :8791`). `jev status`
showed all six UP, each with its window open.

**Effort per tier: 18/18 match** (6 tiers × Qwen :8793, Kilo :8795, OpenCode :8799), read from the
`x-jevonian-model` / `x-jevonian-effort` headers and confirmed in each dashboard's **Logs → Effort** column:

```
chat=deepseek-v4.1-flash/low  small=qwen3.8-flash/low  execute=qwen3.8-flash/high
large=qwen3.8-max/xhigh      utility=qwen3.7-plus/medium  plan=glm-5.3/high
```

`jevonian/auto` on "thanks, that is all!" routed to chat → deepseek-v4.1-flash / low (`brain: jev`) on all three.

**Claude Code through Jevonian :8797 (official launcher):** chat → Haiku (after the Haiku patch; without it, 400), and
execute/utility → Sonnet 5, all 200.

**jev-gateway:** `jev-claude` → :8789, 200. `jev-opencode` → :8791 (mode `none`) → the same turn at Jevonian :8799
(chat → deepseek-v4.1-flash, low), 200. Both gateways are on one dashboard.

---

## Quick Reference Card

```bat
:: one-time
node D:\learn\gemini-mcp\agy-opencode-jev\cli\jev.js install

:: everyday (new CMD window, any folder)
kilo            qwen            opencode            claude --dangerously-skip-permissions
jev-claude --dangerously-skip-permissions           jev-opencode
jev status      jev start       jev stop            jev windows

:: pin a tier
kilo -m jevonian/jevonian/plan     qwen -m jevonian/large     opencode -m jevonian/jevonian/small     claude --model jevonian/plan

:: dashboards
::   Jevonian     http://127.0.0.1:8793/  :8795/  :8797/  :8799/     (Logs -> Effort column)
::   jev-gateway  http://127.0.0.1:8789/dashboard                    (both gateways)

:: without the router
claude-direct   kilo-direct   qwen-direct   opencode-direct
```

---

## Final Checklist

- [ ] Node 22.15+; OpenCode, Kilo, Qwen Code and Claude Code on PATH; Claude Code logged in (`claude-direct`, then `/login`)
- [ ] `credentials\qwen-alibaba-credential.txt` and `typesafe-ai-credential.txt` valid; `AI_GATEWAY_API_KEY` in `.env` valid (test them, as in Prerequisites)
- [ ] `npm install` done in each `jevnonian\jev-router-*` folder and in `jev-gateway\` (local, exact versions)
- [ ] `node jevnonian\jev.js start` prints `WAF patch` / `Effort patch` (and `Haiku patch` for claude) as applied or already applied
- [ ] Ports: 8789/8791 (jev-gateway), 8793/8795/8797/8799 (Jevonian, +1 each) are free
- [ ] `node cli\jev.js install`, then a **new** CMD window, then `doskey /macros` lists the commands
- [ ] `jev status` shows all six UP
- [ ] Each dashboard's Logs → Effort column matches the tier table
- [ ] Nothing global: no `npm -g`, and nothing written to `~/.claude/settings.json` or `~/.config/opencode`
