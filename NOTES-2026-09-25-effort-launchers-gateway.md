# Notes 2026-09-25: per-tier effort, port-aware launchers, jev-gateway 6-tier routing

Add-on to [readme.md](readme.md). Everything below was built and tested on 2026-09-25 on Windows 11,
Node v22.23.2, jevonian 0.1.7, jev-gateway 0.4.3, OpenCode 2.0.15, Kilo 7.7.9, Qwen Code 0.24.4
and Claude Code 2.1.282. The working copy lives in `D:\learn\gemini-mcp\agy-opencode-jev\`:
`jevnonian\` holds the three Jevonian routers and `jev-gateway\` holds the gateway.

---

## 1. Summary

| | Jevonian (OpenCode / Kilo / Qwen) | jev-gateway (Claude Code / OpenCode) |
|---|---|---|
| Ports | OpenCode **8791**, Qwen **8793**, Kilo **8795** (each also binds port+1) | Claude Code **8789**, OpenCode **8797** |
| Upstream | Alibaba Token Plan | Anthropic (your claude.ai login) / Alibaba Token Plan |
| Tier picked by | Jev brain (TypeSafe, Vercel fallback) | Jev (TypeSafe), one call per fresh user turn |
| Effort per tier | **Yes**, needs the patch in section 3 | **Yes**, local patch (section 5) |
| See the port | Launcher banner, `/model` provider name, `node jev.js status` | Launcher banner, `node gateway.js status`, dashboard |

OpenCode is wired to both. Jevonian uses `jevonian/jevonian/<tier>` on :8791 and jev-gateway uses
`jev-gateway/jev-gateway/<tier>` on :8797. The two can run at the same time because the gateway's
OpenCode instance moved off 8791 to 8797.

---

## 2. "When I run `kilo` in CMD, how do I know which port it uses?"

**Plain `kilo` doesn't tell you.** It silently reads `.kilo\kilo.json` from the folder it starts in, and
nothing else. Two changes fix that.

### 2a. Launchers that print a banner and work from *any* project folder

```bat
node D:\learn\gemini-mcp\agy-opencode-jev\jevnonian\kilo.js                 :: Kilo     -> :8795
node D:\learn\gemini-mcp\agy-opencode-jev\jevnonian\opencode.js             :: OpenCode -> :8791
node D:\learn\gemini-mcp\agy-opencode-jev\jevnonian\qwen.js                 :: Qwen     -> :8793
node D:\learn\gemini-mcp\agy-opencode-jev\jev-gateway\claude.js             :: Claude   -> :8789
node D:\learn\gemini-mcp\agy-opencode-jev\jev-gateway\opencode.js           :: OpenCode -> :8797
```

Every argument is passed through, for example `kilo.js run "fix the test"` or
`claude.js --dangerously-skip-permissions`. Each launcher does three things:

1. It starts its router or gateway in the background if it isn't running.
2. It prints a banner with the port, API URL, dashboard and Logs URL, the tier → model → effort table,
   the ledger/log path, the client binary and the working folder.
3. It runs the client **in the folder you ran it from**, pointed at that port:

| Client | How the launcher points it at the router | Gotcha found while testing |
|---|---|---|
| Kilo | `KILO_CONFIG_CONTENT` = the router's `.kilo\kilo.json`, plus `-m jevonian/jevonian/auto` | Kilo **ignores the config's default `model`** and falls back to its own cloud default (`minimax/...`, "Add credits to continue"). Always pass `-m`. |
| OpenCode | `OPENCODE_CONFIG_CONTENT` = the router's `opencode.json`, plus `--standalone`, plus `PWD` | OpenCode 2.x talks to a shared **background service** that may have been started elsewhere with another config ("Model unavailable: jevonian/jevonian/auto"). `--standalone` gives it a private server. This build also resolves config against `PWD`, so a spawn without a shell must set it. |
| Qwen Code | `--auth-type openai --openai-base-url http://127.0.0.1:8793/v1 --openai-api-key local-no-key -m jevonian/auto` | Qwen only reads `.qwen\settings.json` from the startup folder, so the CLI flags carry the router instead. |
| Claude Code | `ANTHROPIC_BASE_URL=http://127.0.0.1:8789` only | No key is set, so your claude.ai login keeps working. |

Managers:

```bat
node D:\learn\gemini-mcp\agy-opencode-jev\jevnonian\jev.js start|stop|status|test|logs [opencode|qwen|kilo]
node D:\learn\gemini-mcp\agy-opencode-jev\jev-gateway\gateway.js start|stop|status [claude|opencode]
```

### 2b. The port in the model picker

The provider name now carries the port, for example `Jevonian :8795 (Jev router -> Alibaba)` in Kilo
and OpenCode, and `… — Jevonian :8793` on every Qwen model. Kilo's and OpenCode's `/model`, and
Qwen's `/model`, now show which router you're on.

---

## 3. Effort per tier on Jevonian (needs a small patch)

Requested table (6 tiers):

| Tier | Jevonian routing id | Model | Effort |
|---|---|---|---|
| chat | `chat` | deepseek-v4.1-flash | low |
| small task | `small` (custom) | qwen3.8-flash | low |
| medium task | `execute` (builtin, label "Medium task") | qwen3.8-flash | high |
| large / heavy task | `large` (custom) | qwen3.8-max | xhigh |
| utility | `utility` | qwen3.7-plus | medium |
| plan | `plan` | glm-5.3 | **high** (see 3b) |

### 3a. Why a patch is needed

In jevonian 0.1.7 a routing entry only carries `id`, `label`, `description`, `models` and `providers`.
The level it sends is **brain pick → `x-jevonian-effort` header → `routing.defaultEffort`**, clamped
to the model's supported levels (`routing.capacities.<model>.efforts`). A per-model setting can't
express "qwen3.8-flash at **low** for small, but at **high** for medium". Worse, **pinned requests
(`jevonian/<id>`) take an early-return path that sends no effort at all**, not even
`defaultEffort`. The Logs page shows those as `default`.

`patch-jevonian-effort.mjs` (below) adds an `"effort"` field to each routing and makes it win on
both paths. Put it next to `patch-jevonian-waf.mjs` and have `start.js` run both before
`serve`, so an `npm install` can't silently undo them:

```js
for (const patch of ["patch-jevonian-waf.mjs", "patch-jevonian-effort.mjs"]) {
  const result = spawnSync(process.execPath, [join(ROUTER_DIR, patch)], { stdio: "inherit" });
  if (result.status !== 0) { console.error(`${patch} failed — refusing to start`); process.exit(1); }
}
```

```js
// patch-jevonian-effort.mjs — idempotent, jevonian 0.1.7. Each edit has its own marker; every
// pattern is checked before anything is written (nothing is written if one is missing).
import { readFileSync, writeFileSync } from "node:fs";
import { fileURLToPath } from "node:url";
import { dirname, join } from "node:path";

const file = join(dirname(fileURLToPath(import.meta.url)), "node_modules", "jevonian", "dist", "cli.mjs");
let src = readFileSync(file, "utf8");
const edits = [
  { // 1) keep `effort` when a routing entry is parsed
    mark: "/* routing-effort-patch v1 */",
    from: "\t\tmodels,\n\t\t...providers ? { providers } : {}\n\t};\n}\n/**\n* Build the routings list",
    to: (m) => "\t\tmodels,\n\t\t...providers ? { providers } : {},\n\t\t" + m + "\n\t\t...typeof value.effort === \"string\" && isReasoningEffort(value.effort) ? { effort: value.effort } : {}\n\t};\n}\n/**\n* Build the routings list" },
  { // 2) Jev-routed turns (jevonian/auto): the chosen routing's effort wins
    mark: "/* routing-effort-patch v1b */",
    from: "\tconst appliedEffort = clampEffort(wanted ?? requestedEffort ?? defaultEffort, ",
    to: (m) => "\t" + m + "\n\tconst routingEffort = config.routing.routings.find((entry) => entry.id === phase)?.effort;\n\tconst appliedEffort = clampEffort(routingEffort ?? wanted ?? requestedEffort ?? defaultEffort, " },
  { // 3) pinned turns (jevonian/<id>): upstream returns with no effort at all
    mark: "/* routing-effort-patch v2 */",
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
  if (n !== 1) { console.error(`Effort patch: ${e.mark} pattern found ${n} times — refusing, nothing written`); process.exit(1); }
  src = src.replace(e.from, e.to(e.mark)); applied++;
}
if (!applied) { console.log("Effort patch: already applied"); process.exit(0); }
writeFileSync(file, src);
console.log(`Effort patch: applied (${applied} edits)`);
```

`config\config.json` (the same in all three routers; only `listen.port` differs):

```json
"providers": [{ "name": "alibaba-tokenplan", "type": "openai",
  "baseUrl": "https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1",
  "apiKeyEnv": "ALIBABA_TOKENPLAN_API_KEY", "billing": "subscription",
  "models": ["deepseek-v4.1-flash", "qwen3.8-flash", "qwen3.7-plus", "glm-5.3", "qwen3.8-max"] }],
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
  "brainPicksEffort": false,
  "capacities": {
    "qwen3.8-flash":       { "contextWindow": 983616,  "maxOutput": 65536, "efforts": ["low","medium","high","xhigh"] },
    "qwen3.7-plus":        { "contextWindow": 983616,  "maxOutput": 65536, "efforts": ["low","medium","high","xhigh"] },
    "glm-5.3":             { "contextWindow": 200000,  "maxOutput": 65536, "efforts": ["low","high","max"] },
    "deepseek-v4.1-flash": { "contextWindow": 1000000, "maxOutput": 65536, "efforts": ["low","medium","high","xhigh"] },
    "qwen3.8-max":         { "contextWindow": 983616,  "maxOutput": 65536, "efforts": ["low","medium","high","xhigh"] }
  }
}
```

The builtins `plan`, `execute`, `utility` and `chat` always exist in Jevonian, so "medium task" is
the relabelled `execute`, and `small` and `large` are custom routings appended after them. Add
`jevonian/small` and `jevonian/large` to each client config so they can be pinned.

> Editing routings on the dashboard's **Routing** page may rewrite `config.json` without the
> `effort` fields, because the dashboard doesn't know them. Edit `config.json` by hand, then restart.

### 3b. What the Alibaba Token Plan accepts (probed live with `reasoning_effort`)

| Model | low | medium | high | xhigh |
|---|---|---|---|---|
| deepseek-v4.1-flash | ✅ | ✅ | ✅ | ✅ |
| qwen3.8-flash | ✅ | ✅ | ✅ | ✅ |
| qwen3.7-plus | ✅ | ✅ | ✅ | ✅ |
| qwen3.8-max | ✅ | ✅ | ✅ | ✅ |
| **glm-5.3** | ✅ | ❌ 400 | ✅ | ❌ 400: *"'reasoning_effort' must be one of: 'low', 'high', 'max'"* |

So "plan = high/xhigh" becomes **high** on glm-5.3 (its `efforts` list is `low, high, max`, so
Jevonian clamps anything else). Medium = **high** and large = **xhigh**. Each is a one-word change.

### 3c. Verified

- 18 pinned requests (6 tiers × 3 routers) plus 3 `jevonian/auto` requests: **all 21 on the right model with the right `x-jevonian-effort`**.
- Browser, **Logs → Effort column** on :8791, :8793 and :8795: chat `low`, small `low`, execute `high`, large `xhigh`, utility `medium`, plan `high`. Rows sent before the pinned-path fix show `default`.
- Real clients from an unrelated folder: `kilo.js run …` routed as `chat/low/jev` and `kilo.js run -m jevonian/jevonian/plan …` as `glm-5.3/high`. OpenCode and Qwen replied correctly through their launchers.

---

## 4. Windows gotchas found today

- **`credentials\vercel-credential.txt` was stale.** Vercel returned 401 for it. The key that works is
  `AI_GATEWAY_API_KEY` in the root `.env`, which is exactly where the guide's `start.js` reads it.
- **A locked `node_modules\...\dist` folder** (IDE file watcher) made `rm -rf` fail halfway, and the router
  then refused to start rather than run unpatched, as intended. Fix: `npm pack jevonian@0.1.7`, then
  unpack the tarball into the existing folder, then re-run both patches.
- OpenCode's `--standalone` and `PWD`, and Kilo's `-m`: see section 2a.

---

## 5. jev-gateway: upstream 0.4.3 has **no model tiers**, so this is a local patch

Section 16 of the readme describes jev-gateway routing Claude Code across Haiku, Sonnet and Opus, and
OpenCode across Alibaba models. **Upstream `vinilana/jev-gateway` 0.4.3 doesn't do that.** It only
asks Jev which *tool* to call next (modes `forced`/`hint`/`none`/`direct`/`passthrough`). The
customized copy the readme mentions wasn't published. The tier routing was therefore rebuilt as a
self-contained local addition. Upstream's tool routing is untouched and still runs alongside it.

**Files** (in `jev-gateway\`): `src/tiers.ts` (new), plus small hooks in `src/app.ts`,
`src/upstream.ts`, `src/events.ts` and `src/dashboard.html`. Tests are in `test/tiers.test.ts`
(35 tests). The full suite gives **199/201**; the 2 failures are upstream Windows-only tests (POSIX
file mode `0600`, and a `printf` shell test) and fail on a clean checkout too.

**Tables**

| Tier | Claude Code (guide 16.3) | OpenCode (this note's table) |
|---|---|---|
| chat | claude-haiku-4-5-20251001 | deepseek-v4.1-flash, low |
| utility | claude-haiku-4-5-20251001 | qwen3.7-plus, medium |
| small | claude-haiku-4-5-20251001 | qwen3.8-flash, low |
| medium | claude-sonnet-5, effort medium | qwen3.8-flash, high |
| plan | claude-opus-5-5, effort high | glm-5.3, high |
| large | claude-opus-5-5, effort max | qwen3.8-max, xhigh |

Override the model with `JEV_ROUTING_<TIER>` and the effort with `JEV_EFFORT_<TIER>`. OpenCode's
effort goes out as `reasoning_effort` unless the client set its own.

**Behaviour**

- **One Jev call per fresh user turn.** A tool result as the last turn means "same turn", and the tier
  picked at the turn's start is kept for the whole tool loop. That avoids thinking-block and prompt-cache
  trouble when switching models mid-loop.
- Tier threshold `JEV_TIER_MIN_CONFIDENCE`, default **0.6**. This is separate from the tool threshold of 0.7.
- **Claude below the threshold:** Claude Code's own model is kept. **OpenCode `jev-gateway/auto` below the threshold:**
  it falls back to `JEV_OPENCODE_FALLBACK_TIER` (default `medium`), because `jev-gateway/auto` is not a model
  Alibaba knows.
- `jev-gateway/<tier>` pins a tier with no Jev call. Any other model name is left alone.
- The guide's 16.4 **`stripEffort`** fix is applied: per-message `output_config` is stripped for non-Opus targets.
- **Haiku:** `thinking`, `output_config` and `context_management` are removed, `max_tokens` is clamped to 8192,
  and the `context-1m-2025-08-07` beta header is dropped. Also **mid-conversation `role:"system"` messages are
  folded into the user turn as `<system-reminder>` blocks** (see below).
- **If the upstream rejects a tier rewrite (400/422), the client's original request is replayed unchanged**
  (logged as `upstream_rejected_tier`). Same rule as upstream's own rewrites.
- `/router/decide` also returns `taskPhase`, `effort`, `routedModel`, `tierConfidence` and `tierSource`.
  The dashboard shows `task=… effort=…` under each row's model, and probes only 8789 and 8797, since
  8791–8796 are Jevonian routers.

**Two real bugs found by live testing with Claude Code 2.1.282, both fixed**

1. **No tier was ever applied.** Claude Code 2.1.x appends a **mid-conversation `role:"system"` message**
   (environment context) *after* your prompt. The last turn was therefore `system`, and the "fresh user
   turn?" check said no. Fix: trailing system/developer messages don't count as a turn.
2. **Haiku answered 400 "role 'system' is not supported on this model"** to that same message. Claude Code
   quietly retried without the mid-conversation-system beta, so answers still arrived, but each 400 cost a
   round trip. Fix: fold those messages into the user turn for Haiku targets, plus the replay safety net above.

**Verified live**

- Claude Code → :8789 → Anthropic (claude.ai login, no key). Covered: "reply exactly", a chat turn, a
  tool-use turn (Read), an edit turn (Edit, which changed the file) and a plan-size question. **All 9
  requests 200, 0 `upstream_rejected_*`.** Haiku served chat, utility and tool-loop turns. Earlier runs
  showed plan → `claude-opus-5-5` at effort high and medium → `claude-sonnet-5` at effort medium.
- OpenCode → :8797 → Alibaba. Covered: "reply exactly" (chat → deepseek, low), pinned plan
  (glm-5.3, high) and a tool-use turn (utility → qwen3.7-plus, medium; the continuation reused the cached
  tier). OpenCode's title calls went through `small_model` → qwen3.8-flash, low. **All 200.** One
  `upstream_rejected_forced`: Alibaba refused upstream's *forced* `tool_choice`, the request was
  replayed with the tier kept, and it returned 200.
- Dry run of `/router/decide` with `jev-gateway/auto`: all 6 sample prompts landed on the intended tier.

---

## 6. What Qwen recommended (shared chat, 2026-09-23)

> **"Jevonian is better for Kilo/Qwen. jev-gateway is better for Claude Code. They're complementary,
> not competing."** OpenCode: either.

That's the layout above. Two of its claims were wrong:
- *"Jevonian doesn't support Claude Code"*: readme section 11 sets exactly that up.
- *"`npm install -g jev-gateway`"*: this setup keeps everything **local** (per-folder `node_modules`), as
  the readme itself insists for Jevonian.

Its point that jev-gateway's tool routing "saves more tokens" is only partly backed by the gateway's own
benchmark. Debugging tasks improved for every model, but feature work got **worse** on Opus 5 and
Sonnet 5. Use the dashboard's baseline switch (`routing off`) to measure your own work.
