---
name: wikibot-config
description: Read and edit a Wikibot bot's full configuration — bot-level settings, agents (prompts, on/off, options), scenarios, flow cloning, proactive jobs, datatables (schema and data), custom REST function tools, private knowledge base articles, and conversation history — via the Wikibot public API. Use when the user asks to change bot behavior, edit an agent's instruction, create or clone a flow, set up a proactive job, manage a datatable, wire up a REST function, write a KB article, or debug a conversation for their Wikibot integration.
---

# Wikibot bot configuration

Talk to a running Wikibot bot's public API to inspect and change its settings:
bot-level templates/filters, per-agent prompts and on/off switches, proactive
jobs, datatable schemas, and custom REST function tools. This is the same
surface the in-product Copilot uses, exposed over HTTP for scripted/agent use.

**This API can create and edit, but it can never delete anything** — no
endpoint removes an agent, job, datatable, or setting. If the user wants
something removed, tell them to do it from the dashboard.

Private (manually-written, not crawled) knowledge base articles can be
created and edited too — see `/api/bot/articles` below. `/api/bot/kb/*`
still only adds a crawled URL or uploads a file; you cannot edit the text
of an article that came from a crawl or upload, only a private one.

## How Wikibot works

Read this before touching anything — most "why doesn't this do anything"
problems trace back to one of these rules, not to a wrong API call. This is
the same mental model the in-product Copilot works from.

### Flow → agents → scenarios

A bot has one or more **flows** (`GET /api/bot/flows`).
Each flow has one **default agent** (the one that actually talks to the
client) plus:

- five fixed **service agents** — `spam`, `operator`, `editor`, `translator`,
  `summary` — each implementing one specific behavior (see below), on or off
  independently of everything else.
- any number of **scenario agents** (`scenario_<name>`) — a separate
  instruction for one narrow topic (refunds, booking, a specific product).
  This API can create a new one (`POST /flows/:flowId/agents`) as well as
  edit an existing one's prompt/options — see below. **Creating a scenario
  does nothing by itself**: the default agent only knows it exists once its
  own prompt names it (see the pipeline notes below) — always pair a
  scenario creation with a `PATCH` to the default agent's prompt in the same
  turn.

### The processing pipeline (why a setting can look like it "does nothing")

A client message goes through, in order — and several steps can answer and
stop *before the AI agent is ever called*:

1. **Preprocessing** — merges rapid-fire messages from the same client into
   one, truncates an overlong message, checks attachments
   (`disableMediaMiddleware`), anti-flood, "ignore" phrase list (matched →
   bot stays silent).
2. **Early template replies, checked in this order — first match wins, agent
   never runs:**
   1. Gratitude phrase → `templates.gratitude`.
   2. Greeting phrase (first message only) → `templates.greeting.initial`.
   3. `spam` agent (if enabled, first message only) — spam → stop.
   4. `operator` agent (if enabled, first message only) — decides bot vs.
      human.
   5. Outside bot's working hours → `templates.nonWorkingBotRedirect`.
   6. Consecutive-answer limit exceeded → transfer or skip.
3. **The AI agent** (default, or a scenario it forwards to) — this is where
   the agent's `prompt` and its custom function tools actually run.
4. **Postprocessing** — `editor` agent rewrites the answer, `translator`
   translates it to the client's language, `templates.success` /
   `noAnswer` / `operator` wrap it (each has a `working` / `nonWorking`
   variant depending on **operator** working hours — a different schedule
   from the bot's own hours in step 2.5), overlong answer gets truncated,
   `summary` agent writes a handoff note once enough client messages piled
   up.

Consequences worth knowing:
- A wrong-looking greeting/gratitude/"no answer" reply is a **bot-level
  template** (`/config`), not the agent's `prompt` — editing the prompt
  won't touch it.
- There are **four different "transfer to operator" templates**
  (`templates.operator.*` for the client explicitly asking for a human,
  `templates.l1Redirect.*` for the bot deciding to transfer on its own —
  the most common case, `templates.nonWorkingBotRedirect.*` for outside the
  bot's hours, `templates.noOperatorError` when transfer isn't possible at
  all), each split further into `working`/`nonWorking` by *operator* hours.
  A wording change usually has to be applied to several of these at once —
  check `GET /api/bot/config` for every template containing the old text.
- Attachment handling also depends on the default agent's `analyzeImages` /
  `analyzeAudio` options (and the model supporting it) — a media complaint
  may need both a bot-level flag and an agent option.

### Scenarios: two modes, and the one thing that actually turns one on

A scenario agent works in one of two modes, visible via its `options.workAsSkill`:

- **Dialog transfer** (`workAsSkill` false/absent, the default) — the
  default agent hands the whole conversation to the scenario, which then
  talks to the client directly with its own prompt/options/model until it
  hands the conversation back.
- **Attached to the default agent** (`workAsSkill: true`) — nothing is
  handed off; the default agent loads the scenario's instruction itself and
  keeps talking. The scenario's own options/model stop applying. At most
  three attached scenarios can be loaded at once.

**Critical: creating or editing a scenario does nothing by itself.** The
default agent only knows a scenario exists because its *own* `prompt`
mentions it by name — there's no separate "enabled" switch. So:
- Debugging "requests never reach my scenario" always starts with reading
  the **default agent's** prompt for a line naming that scenario — not the
  scenario's own prompt or its purpose.
- Flipping `workAsSkill` is **always a two-part change**: the option itself,
  plus rewriting that mention in the default agent's prompt (transfer mode
  names it as something to hand off to; attached mode names it as a skill to
  use inline). Send both in the same response — changing only one leaves the
  scenario broken or not working at all.

**The mention line uses the BARE scenario name, never the stored agent
name.** A scenario created as `name: "refunds"` is stored as the agent
`scenario_refunds` — but the runtime hand-off tool re-adds the `scenario_`
prefix itself when it resolves the name the default agent's model wrote
down, so a mention line that includes the prefix (`"scenario_refunds"`)
makes it look for `scenario_scenario_refunds`, finds nothing, and the
hand-off silently fails ("agent is not found or disabled" — the client just
gets an ordinary reply from the default agent instead of reaching the
scenario, with no error visible anywhere but the request audit). Exact
format (either mode, one line per scenario):

- Dialog transfer — section `### Сценарные агенты ###`:
  `- "<description>" - вызывай агента с именем "<name>".`
- Attached/skill — section `### Навыки ###`:
  `- "<description>" - используй навык "<name>".`

`<name>` is `refunds`, not `scenario_refunds`. When you create a scenario and
then edit the default agent's prompt to reference it, always use the `name`
you passed to `POST /flows/:flowId/agents` (the value before prefixing), and
double check the mention line before sending the `PATCH` — this is the most
common way a freshly created scenario ends up silently unreachable.

### Bot-level settings vs. agent options

`/config` covers pipeline steps 1, 2, and 4 (bot-wide: templates, ignore/
greeting/gratitude phrase lists, media/operator-transfer flags, answer
delay, language) — the same setting regardless of which agent answers.
`PATCH /flows/:flowId/agents/:name` covers step 3 and the service agents
themselves (per-agent prompt, and options — including the five service
agents' on/off switches, `translator`'s language list, `summary`'s message
threshold). Getting a request "wrong bot behavior" from a user, figure out
which of the two layers it's actually in before proposing a fix.

### Jobs run outside this pipeline entirely

Everything above happens in response to a client message. A job is the one
exception: it fires on its own after a conversation has been silent for a
while, and the agent gets a system message instead of a client message —
see the Jobs section below for the mechanics.

## Setup

Requires two environment variables, provided by the user:

- `WIKIBOT_API_KEY` — an integration API key from the bot's Settings → API
  Keys page. **The key's scopes must include `manage`** for everything under
  `/config`, `/flows/:flowId/agents`, `/flows/:flowId/jobs`, `/datatables`,
  `/tools`, `/articles`, `/conversations`, `/journal`, and
  `GET /api/bot/flows` — a key with only `ask`/`anonymization` (what every key
  had before scopes existed) gets a 403 on those. A handful of older, simpler
  endpoints (`GET /api/bot/agents`, the deprecated alias of `GET /flows`,
  `GET /api/bot/kb`, `POST /api/bot/kb/create`,
  `POST /api/bot/kb/upload-file`, `GET /api/bot/search`) accept **either**
  `ask` or `manage` — see "Other endpoints" below.
- `WIKIBOT_API_URL` — base URL, defaults to `https://api.wikibot.pro` if unset.

All requests send the key as the `Authorization` header (no `Bearer` prefix):

```bash
curl -s "$WIKIBOT_API_URL/api/bot/config" -H "Authorization: $WIKIBOT_API_KEY"
```

If either variable is missing, ask the user for it before doing anything else.

**Rate limits (429).** Counted per bot, not per key, in fixed one-minute
windows: 120 requests/minute across these endpoints, plus a tighter
20/minute shared by the three whose cost is the scan behind them —
`GET /journal`, `GET /conversations/:chatId/history` and
`POST /datatables/query`. A 429 carries a `Retry-After` header with the
seconds left in the window; wait that long instead of retrying immediately.
`/ask`, `/search`, `/anonymize` and `/deanonymize` are not limited. Paging
through the journal or a long dialog is what actually hits this — narrow the
filters (`from`/`to`, `type`, `query`) rather than walking every page.

**Non-ASCII text (Cyrillic, etc.) — never put it inline in a `curl -d`/`-X`
argument.** On Windows, MSYS/Git Bash re-encodes a native program's argv to
the system ANSI codepage before the process ever sees it — Cyrillic (or any
non-ASCII) characters in an inline argument silently become `?` before curl
even sends the request; the API then faithfully stores exactly that. This is
a shell-encoding issue, not something the API does — bodies sent as raw file
content arrive correctly. Always write the JSON body to a file (UTF-8, no
BOM) and send it with `--data-binary @file`, never `-d '{"text":"..."}'` with
literal non-ASCII inside the quotes:

```bash
cat > /tmp/body.json <<'EOF'
{"path": "templates.greeting.initial", "value": "Здравствуйте!"}
EOF
curl -s -X PATCH "$WIKIBOT_API_URL/api/bot/config" \
  -H "Authorization: $WIKIBOT_API_KEY" -H "Content-Type: application/json" \
  --data-binary @/tmp/body.json
```

If a response you just wrote comes back with `?` where Cyrillic should be,
this is almost certainly why — redo it via a file, don't assume the API
mangled it.

## Workflow

Always follow this order — never PATCH blind:

1. **`GET /api/bot/config/schema`** — the list of settings you're allowed to
   touch, with a human description and, for confusingly-named booleans, what
   `true`/`false` each actually does. Read this first in a new session; it is
   the full and only surface — there is no path outside this list.
2. **`GET /api/bot/config`** — the bot's current settings (templates, filters,
   working hours, language, answer delay, glossary).
3. Decide the change(s) and **`PATCH /api/bot/config`** with `"dryRun": true`
   first. It validates everything and returns the before/after diff without
   writing anything.
4. **Show the diff to the user and get explicit confirmation** before
   re-sending the same request with `dryRun` omitted (or `false`) to apply it.
   Never skip straight to a live write on the first attempt.

## Endpoints

**Scoping is mixed, and the path doesn't always show it.** Agents and jobs
are flow-scoped (`/flows/:flowId/agents`, `/flows/:flowId/jobs`) — a bot's
flows are genuinely separate agent sets. Tools, datatables, articles, the
journal, and conversation history are **bot-scoped** (`/tools`, `/datatables`,
`/articles`, `/journal`, `/conversations`), never nested under `/flows/:id/`,
even though a tool or datatable is ultimately assigned to specific agents
within specific flows — that assignment is a property of the row
(`assignedAgentIds`, `PATCH .../agents/:agentId`), not a path segment. Don't
guess `/flows/:flowId/tools` — it doesn't exist.

### `GET /api/bot/config`

Returns the current whitelisted config view:

```json
{
  "templates": { "greeting": { "initial": "..." }, "gratitude": "...", "...": "..." },
  "filters": { "ignore": { "patterns": ["..."] }, "...": "..." },
  "disableMediaMiddleware": false,
  "disabledOperatorRegexp": false,
  "answerDelayMs": null,
  "workingHours": null,
  "botWorkingHours": null,
  "language": "ru",
  "glossary": [{ "term": "...", "definition": "..." }]
}
```

### `GET /api/bot/config/schema`

```json
[
  {
    "path": "templates.greeting.initial",
    "kind": "text",
    "description": "Приветствие"
  },
  {
    "path": "filters.disableGreetingCheck",
    "kind": "boolean",
    "description": "Не проверять приветствие",
    "effect": {
      "onTrue": "приветствия обрабатывает сам агент, шаблон не подставляется",
      "onFalse": "чистое приветствие получает ответ шаблоном (значение по умолчанию)"
    }
  }
]
```

`kind` tells you what to send for that path:

| kind       | send in body as   | notes                                   |
| ---------- | ------------------ | ---------------------------------------- |
| `text`     | `value` (string)   | max 2000 chars                           |
| `boolean`  | `value` (boolean)  |                                          |
| `number`   | `value` (number)   | `answerDelayMs` is bounded 0..60000      |
| `patterns` | `values` (string[])| a list of phrases, not a single string   |

### `PATCH /api/bot/config`

```json
{
  "dryRun": true,
  "changes": [
    { "path": "templates.greeting.initial", "value": "Здравствуйте!" },
    { "path": "filters.ignore.patterns", "values": ["спам", "реклама"] }
  ]
}
```

Response (both for `dryRun: true` and after a real apply):

```json
{
  "applied": false,
  "changes": [
    {
      "path": "templates.greeting.initial",
      "oldValue": "Привет!",
      "newValue": "Здравствуйте!"
    }
  ]
}
```

- A change whose new value equals the current one is silently dropped from
  the diff — it never shows up as "applied".
- A `text` path that currently holds several greeting variants is rejected
  with a 400 rather than overwritten (writing a single string would delete
  the other variants) — surface that error to the user instead of retrying.
- All changes in one request are validated against the same snapshot and
  applied together; if any one is invalid, the whole request is rejected
  (400) and nothing is written.
- Errors are plain `{ "error": "..." }` bodies with the matching HTTP
  status (400 for validation, 403 if the key lacks the `manage` scope, 404 if
  the bot can't be resolved). Every endpoint in this file uses that same
  shape.

### `GET /api/bot/flows` and `GET /api/bot/flows/:flowId/agents`

A bot has one or more **flows** (`GET /api/bot/flows`: `{ id, name, enabled }`;
needs `manage`). `GET /api/bot/agents` is a deprecated alias of the same
endpoint that still accepts `ask` — despite its name it has always returned
flows, not agents; use `GET /flows`, the name collision with the actual agent
list below is not worth the confusion. Each flow has its own set of **agents**: the
built-in service agents (`default`, `spam`, `operator`, `editor`,
`translator`, `summary`) plus any custom scenarios (`scenario_something`).
Everything below is scoped to one flow — get its id from `GET /api/bot/flows`
first.

`GET /api/bot/flows/:flowId/agents` returns every agent of that flow,
including service agents that were never turned on (shown with their default
prompt and `enabled: false`):

```json
[
  { "id": 12, "name": "default", "type": "default", "enabled": true, "prompt": "...", "options": { "analyzeImages": true } },
  { "id": 0, "name": "spam", "type": "spam", "enabled": false, "prompt": "### Правила...", "options": {} },
  { "id": 34, "name": "scenario_refunds", "type": "scenario", "enabled": true, "prompt": "...", "options": { "workAsSkill": false } }
]
```

`id: 0` means the row doesn't exist yet — the API creates it on first write,
exactly like the dashboard does on first toggle.

### `POST /api/bot/flows/:flowId/agents` (create a scenario)

Only creates **scenario** agents — service agents (`spam`, `operator`, ...)
already exist virtually (see `id: 0` above) and are created on first
`PATCH`, not through this endpoint. Body:

```json
{
  "dryRun": true,
  "name": "refunds",
  "description": "Клиент просит вернуть оплату за заказ.",
  "prompt": "Полный текст инструкции сценария..."
}
```

- `name` — latin letters, digits and underscore, starting with a letter, max
  64 characters. 400 if: empty, already taken on this flow, already starts
  with `scenario_` (the prefix is added automatically — don't include it),
  or matches a built-in agent name (`default`, `spam`, `operator`, `editor`,
  `translator`, `summary`) — confusing side-by-side in
  `GET /flows/:flowId/agents`, so this endpoint rejects it outright. The
  stored agent name becomes `scenario_<name>`.
- `description`, `prompt` — required, non-empty. `description` is read by the
  default agent's own instruction/skill logic to decide when to hand off to
  this scenario, **not shown to the client** — one sentence is enough. It's
  editable later too, see `PATCH` below. `prompt` is the scenario's full
  instruction text.
- `dryRun: true` — validates everything (including the name-taken check) and
  returns a preview **without creating anything**: `{ "applied": false,
  "scenario": {...} }`. Use it first, same rule as everywhere else in this
  API; repeat without `dryRun` to actually create.
- Real create returns `{ "applied": true, "scenario": { "id", "name",
  "type", "enabled", "prompt", "options" } }`.

**The new scenario does nothing until you also edit the default agent's
prompt** to name it (see "Scenarios: two modes..." above for the exact line
format per mode) — always send both changes in the same turn and tell the
user both happened, not just the creation.

### `PATCH /api/bot/flows/:flowId/agents/:name`

Addressed by `name`, not `id` — the one write endpoint in this API that
works this way. A service agent can be `id: 0` (no row yet, see above), so
`id` isn't a stable handle for it the way it is for a tool or a datatable;
`name` always exists and never changes, even before the first write creates
the row. `:name` is the `name` field from the list above (e.g. `spam`,
`scenario_refunds`). Body — any subset of:

```json
{
  "dryRun": true,
  "enabled": true,
  "prompt": "New instruction text",
  "options": { "analyzeImages": true, "workAsSkill": false }
}
```

Rules the API enforces (same as the dashboard/Copilot):

- `enabled` only works on `spam`, `operator`, `editor`, `translator`,
  `summary` — `default` always runs and a scenario's activation isn't a
  switch this API exposes; both are rejected with 400.
- `prompt` only works on `default`, `spam`, `operator`, `editor`, `summary`,
  and any scenario — `translator` has no instruction (400 if attempted).
- `options.workAsSkill` only applies to a scenario. `options.languages`
  (array of language codes) and `options.minClientMessages` only apply to
  `translator` / `summary` respectively. `options.description` only applies
  to a scenario — this is how you edit the `description` set at creation
  (must stay non-empty); it is the same field the default agent's own
  routing logic reads, so update the mention line if the meaning changed
  enough to matter. Every other option key applies to whichever agent you
  addressed.
- Response shape matches `/config`: `{ "applied": bool, "changes": [{ "field", "oldValue", "newValue" }] }`.
  Use `dryRun: true` first, same rule as bot-level config: show the diff,
  get confirmation, then repeat without `dryRun`.

### `POST /api/bot/flows/:flowId/clone` (duplicate a flow)

```json
{ "dryRun": true, "name": "New flow name" }
```

Duplicates the flow: every agent (service and scenario, with prompt/options/
enabled state), their tool and datatable access, and every job.

- `name` — required, non-empty.
- `dryRun: true` — validates (source flow exists, flow-count cap not
  exceeded, name non-empty) and returns `{ "applied": false, "flow": {
  "name", "enabled": false } }` **without cloning anything**. Use it first;
  repeat without `dryRun` to actually clone.
- Real clone returns `{ "applied": true, "flow": { "id", "name", "enabled":
  false } }`.
- The clone **always starts disabled** — enable it explicitly afterward via
  `PATCH /flows/:flowId/agents/default` if that's what the user wants (a
  flow itself has no `enabled` field exposed by this API to flip directly;
  ask the dashboard-side "which flow is active" question if it matters).
- Cloned jobs are also created disabled, regardless of the source job's
  state.
- Tools and datatables are **not duplicated** — the new flow's agents keep
  access to the exact same tool/table rows as the source flow. Editing one of
  those (e.g. via `PATCH /tools/:id`) affects both flows.
- 400 if the bot is already at the flow limit (10 per bot) or the source
  flow doesn't exist.

### Jobs (proactive triggers)

- `GET /api/bot/flows/:flowId/jobs` — list, including `enabledAt` and run
  counts:
  ```json
  [
    {
      "id": 5,
      "name": "Follow up after silence",
      "enabled": true,
      "enabledAt": "2026-01-01T00:00:00.000Z",
      "instructions": "Ask the client if they still need help.",
      "inactivityMinutes": 60,
      "conversationFilter": "ALL",
      "maxRunsPerConversation": 1,
      "runs": { "completed": 12, "skipped": 3, "failed": 0 }
    }
  ]
  ```
  `runs.skipped` climbing without config changes usually means conversations
  are failing the `conversationFilter`, not that the job is broken.
- `POST /api/bot/flows/:flowId/jobs` — create:
  ```json
  {
    "name": "Follow up after silence",
    "instructions": "Ask the client if they still need help.",
    "inactivityMinutes": 60,
    "conversationFilter": "all",
    "maxRunsPerConversation": 1,
    "enabled": true
  }
  ```
  `conversationFilter` is one of `ALL`, `WITH_OPERATOR`, `NO_CLIENT_REPLY`
  (uppercase — matches the stored enum, not a display string).
- `PATCH /api/bot/flows/:flowId/jobs/:jobId` — same fields, all optional
  (only send what changes; the rest is kept from the existing job).

No `dryRun` here — jobs are cheap to create and easy to re-edit, so just show
the user what you're about to send before calling it.

How a job actually fires, and where it stalls:
1. It only considers conversations that went quiet **after** the job was
   last switched on — turning a job back on resets that clock, so
   conversations that already went quiet while it was off are never picked
   up. Warn the user about this before they re-enable one.
2. It then checks `inactivityMinutes` of silence, then `conversationFilter`.
   A conversation that fails the filter once is skipped **permanently** for
   that job — tightening the filter later won't make it reconsider
   conversations it already passed on. If the user is loosening a filter,
   the fix only affects future conversations.
3. `maxRunsPerConversation` caps how many times it can still fire in one
   conversation.
4. On a hit, the agent gets a system message ("job fired: <name>" plus the
   job's `instructions`) and writes the actual client-facing message itself
   — `instructions` is guidance to the agent, not a ready-made text to send.

Read `GET .../jobs` before proposing a new one (don't duplicate an existing
trigger) and when the user reports "the job never fires" — walk through the
four points above against what's there before guessing.

### Datatables

- `GET /api/bot/datatables` — list (id, name, description, columns, row count).
- `POST /api/bot/datatables` — create:
  ```json
  {
    "name": "orders",
    "description": "Customer orders the agent can look up",
    "columns": [
      { "name": "order_id", "type": "text", "required": true },
      { "name": "status", "type": "text" }
    ],
    "assignedAgentIds": [34]
  }
  ```
  `type` is one of `text`, `number`, `boolean`, `date`. `assignedAgentIds` is
  optional — omit it and every enabled `default`/scenario agent of the bot
  gets access, same as the dashboard's default. Don't define your own id or
  created/updated-at column — every table automatically gets `_id`,
  `_created_at`, `_updated_at` (queryable, e.g. `ORDER BY _created_at DESC`;
  not part of `columns`, read-only); a name starting with `_` is reserved and
  the create/edit call rejects it.
- `PATCH /api/bot/datatables/:id` — `{ "description"?, "columns"? }`. Every
  `:id` on the datatable endpoints accepts either the table's UUID or its
  `name`, so a name from `GET /datatables` can be used directly. Sending
  `columns` replaces the schema (not the data) — validation errors (bad name,
  too many columns, table not found) come back as 400 with the message.
- `POST /api/bot/datatables/query` — read the actual rows, same read-only SQL
  the in-product Copilot's memory-mode assistant runs:
  ```json
  { "tables": ["orders"], "sql": "SELECT status, count(*) FROM orders GROUP BY status" }
  ```
  Runs against an isolated in-memory copy of just the named tables, never the
  real database — any read-only `SELECT` is safe here, while anything that
  writes or reaches outside the query is rejected with a 400. `tables` must
  name tables this bot actually has (`GET /datatables` for the list); `sql`
  may reference any of them. Returns `{ columns, rows, truncated }`;
  `truncated: true` means more rows exist than were returned (row and
  total-size limits apply, same as the dashboard's assistant) — narrow the
  query rather than assuming you saw everything. 400 with a clear message for
  an unknown table name, invalid SQL, or a query joining more data than the
  size limit allows.

### Datatable access (which agents can query which table)

Only `default` and scenario agents can ever be granted access — `spam`,
`operator`, `translator`, `editor`, `summary` never see datatables, by
design, not something this API can override.

- `GET /api/bot/agents/assignable` — every agent across the bot's flows
  eligible to be granted a tool or a datatable: `{ id, name, flowId,
  flowName, flowEnabled }`. Not datatable-specific despite living next to
  these endpoints — this is also where `assignedAgentIds` for `POST /tools`
  (below) comes from. Use this to find an agent's numeric `id` for the calls
  below (this `id` is the same one `GET /api/bot/flows/:flowId/agents`
  returns).
- `GET /api/bot/datatables/:id/agents` — current assignments for one table:
  `[{ agentId, enabled }]`.
- `PATCH /api/bot/datatables/:id/agents/:agentId` — `{ "enabled": true|false }`.
  Grants or revokes that one agent's access to that one table. The agent must
  come from `/agents/assignable` for this bot — anything else is a 400.

### Custom REST function tools

A "tool" is a function the agent's model can call during a conversation to
hit an external REST API (look up an order, check a balance, create a
ticket). One tool call can fan out into several HTTP requests ("actions").

**Security — always tell the user this before creating or editing one:**

- Every `url` is checked to be a **public** http(s) address — not
  `localhost`, a private IP range (`192.168.*`, `10.*`, `127.*`), or cloud
  metadata. A private address is rejected with a 400. This bot calls the URL
  unattended, from the platform's own servers.
- **Never put a real secret** (an API token, password, key) the user pasted
  into the chat into `headers`/`url`/`payload` without flagging it first —
  a tool's header values are readable back through `GET /tools/:id` by any
  holder of a `manage` key for this bot, so tell the user where the
  credential is going and who can read it.

Endpoints:

- `GET /api/bot/tools` — list: `{ id, name, description, assignedTo: [{agentId, agentName, enabled}] }`.
  An agent only appears in `assignedTo` while connected — `enabled` is
  currently always `true` there; disconnecting removes the entry rather than
  flipping it to `false` (see `PATCH .../agents/:agentId` below).
- `GET /api/bot/tools/:id` — full definition including `parameters` (JSON
  Schema) and `actions` (with real header values and URLs).
- `POST /api/bot/tools` — create:
  ```json
  {
    "name": "get_order_status",
    "description": "Look up an order's shipping status by its order id",
    "parameters": {
      "type": "object",
      "properties": { "orderId": { "type": "string" } },
      "required": ["orderId"]
    },
    "actions": [
      {
        "method": "GET",
        "url": "https://api.example.com/orders/{orderId}",
        "headers": [{ "key": "Authorization", "value": "Bearer ..." }]
      }
    ],
    "assignedAgentIds": [34]
  }
  ```
  `parameters` is a JSON Schema object (OpenAI function-calling format) — not
  a string. `{orderId}` in `url`/`payload` is replaced with the matching
  argument at call time; only `properties` names are valid placeholders.
  `payload` (body template) is only used for POST/PUT/PATCH. Up to 5 actions.
  `assignedAgentIds` is optional — omit it to create the tool unassigned and
  grant access afterwards. Valid ids come from `GET /api/bot/agents/assignable`
  (same list datatables use — it isn't tool- or datatable-specific).
- `PATCH /api/bot/tools/:id` — any subset of `{name, description, parameters, actions}`.
  Sending `actions` replaces the FULL list — read `GET /tools/:id` first if
  you're only changing one action among several, and resend the others
  unchanged.
- `GET /api/bot/tools/:id/agents` / `PATCH /api/bot/tools/:id/agents/:agentId`
  (`{ "enabled": true|false }`) — grant/revoke, but unlike datatable access
  this isn't a stored flag: `enabled: true` connects the tool to the agent
  (creating the link if it didn't exist), `enabled: false` disconnects it
  outright (deletes the link — the same thing the dashboard's own
  "Disconnect" action does). Fully reversible by PATCHing `enabled: true`
  again, so it isn't gated behind a delete-only-in-the-dashboard rule. Only
  `default` and scenario agents are eligible.

### Knowledge base articles

Only **private** articles (manually written, not from a crawl or upload) can
have their content created or edited here — same restriction the in-product
Copilot has. A crawled/uploaded article's text comes from its source and
would just be overwritten by the next crawl; only its search-index inclusion
can be toggled.

- `GET /api/bot/articles/:id` — `{ id, title, content, indexed, editable }`.
  `editable: false` means it's a crawled/uploaded article — you can still
  toggle `indexed` on it via `PATCH`, but not `title`/`content`.
- `POST /api/bot/articles` — create a new private article:
  ```json
  { "title": "Refund policy", "content": "Full article text...", "indexed": true }
  ```
  `title`/`content` required and non-empty. `indexed` is optional, defaults
  to `true` — set it `false` up front instead of a separate `PATCH` right
  after if the article shouldn't be searchable yet. Returns `{ id }`.
  Triggers re-indexing automatically — no separate publish step.
- `PATCH /api/bot/articles/:id` — any of:
  ```json
  { "title": "New title", "content": "New full text" }
  ```
  or
  ```json
  { "indexed": false }
  ```
  `title` and `content` must be sent together (read `GET /articles/:id`
  first if you're only changing one) and 400 if the article isn't editable.
  `indexed` can be sent alone and works on any article regardless of
  `editable`. Sending both a content edit and `indexed` in the same call
  works too.
- Articles are subject to a per-article length cap, and the account's
  knowledge-base size limit applies to the bot as a whole. Keep articles
  reasonably sized, and when a user wants to bulk-import a lot of content,
  tell them to check their plan's knowledge-base limit in the dashboard
  first.

### `GET /api/bot/conversations/:chatId/history` (dialog history + audit)

Read-only — the same lookup the in-product Copilot uses to answer "why did
the bot reply that way" / "why didn't it answer". `:chatId` is the external
identifier of the conversation, and its shape depends on the channel — a
chat-center id (`wb_85`), a plain number (messenger), a helpdesk ticket
number, a sandbox id (`P-...`). **Don't validate or reject its format** —
any string the user calls an identifier is worth trying as-is; a 404 tells
you it didn't resolve, guessing "that doesn't look like an id" up front does
not.

Query param `?page=0` (default 0, 30 messages per page).

```json
{
  "mainConversationId": 123,
  "page": 0,
  "pageSize": 30,
  "hasMore": false,
  "logs": [
    {
      "requestId": "...",
      "request": "...",
      "response": "...",
      "type": "answer",
      "isSubAgent": false,
      "steps": ["..."]
    }
  ],
  "items": [
    { "id": 1, "conversationId": 123, "isSubAgent": false, "author": "user", "message": "..." }
  ]
}
```

- `items` — every message of the dialog, including sub-conversations opened
  by a scenario (`isSubAgent: true`). `message` is plain text, not the raw
  stored provider JSON — envelope fields (role, tool_call_id, content-parts)
  are stripped. Internal system-injected messages (bracketed
  `[AGENT SYSTEM MESSAGE]` notes the pipeline sends itself, e.g. retry nudges)
  are left out entirely, not just unmarked.
- `logs` — the parsed audit of each request (same steps shown in the
  dashboard's "Analysis" panel): what template/agent/skill handled it and
  why. Only returned on `page=0` — later pages return `logsNote` instead
  (the audit covers the whole dialog, not one page of it).
- 404 if `chatId` doesn't resolve to anything for this bot — at that point,
  and only then, ask the user whether it might be a different kind of id or
  belong to a different bot.
- If the request never reached an agent conversation at all (blocked by a
  filter, redirected to an operator, matched a template) the shape is
  different: `{ "chatId", "type": "preprocessed", "note", "request",
  "response", "explain", "responseType", "logs" }` — a single request/
  response pair instead of `items`/pagination.

### `GET /api/bot/journal` (search/browse the request log)

Same underlying log the dashboard's own Journal page lists — one row per
request/response pair, not per dialog. Use this to *find*
which requests match a date range, an outcome type, or a text search;
once you have a `chatId` from a row, `GET /conversations/:chatId/history`
gives you the full dialog (and audit) around it. Every filter is a query
parameter, page size is fixed (20), always ascending/descending by one
field at a time:

```
GET /api/bot/journal?page=0&query=refund&from=2026-01-01T00:00:00Z&to=2026-01-31T23:59:59Z&type=OPERATOR,ERROR&sortField=createdAt&sortDesc=true
```

- `page` — 0-based, default 0. Page size is fixed at 20 (`pageSize` in the
  response), not configurable.
- `query` — optional. Matches request text (substring, case-insensitive)
  OR an exact `chatId`/`requestId`/row `id` (numeric, with or without the
  `A` prefix shown in the dashboard, e.g. `A1234` or `1234`).
- `from`/`to` — optional ISO 8601 timestamps, **must be given together**
  (400 if only one is set). Inclusive on both ends. Unlike the dashboard's
  own date picker (which silently widens the end date to cover the whole
  day in local time), this endpoint does no day-boundary widening — pass
  the exact bounds you mean, e.g. `to` should already be end-of-day if
  that's the intent.
- `type` — optional, comma-separated outcome types: `SUCCESS`, `FALLBACK`,
  `NO_ANSWER`, `ERROR`, `GREETING`, `OPERATOR`, `SKIP`, `GRATITUDE`,
  `RATING_ACCEPTED`, `NO_OPERATOR_ERROR`.
- `operatorReason` — optional, comma-separated reasons a request was handed
  to an operator, meaningful when `type` includes `OPERATOR`: `REQUEST`,
  `RULE`, `NO_ANSWER`, `PREPROCESSING`.
- `sortField` — one of `id`, `chatId`, `type`, `createdAt` (default
  `createdAt`). `sortDesc` — `true` (default) or `false`.
- `includeAudit` — `false` by default. Set `true` to also get each row's
  parsed request-log steps (`steps`) — the same step-by-step analysis
  (what template/agent/skill handled it and why) shown in the dashboard's
  "Analysis" panel and in `GET /conversations/:chatId/history`'s `logs`.
  This costs a second (batched, not per-row) query, so leave it off while
  you're just searching/browsing for the right row, and turn it on only
  once you actually want to read the analysis inline instead of following
  up with a second request.

Response (`includeAudit=true` shown; `steps` is absent from each item
otherwise):

```json
{
  "items": [
    {
      "id": 4521,
      "chatId": "wb_85",
      "type": "SUCCESS",
      "createdAt": "2026-01-15T10:30:00.000Z",
      "operatorReason": null,
      "request": "Как оформить возврат?",
      "response": "Чтобы оформить возврат...",
      "explain": "...",
      "agentConversationId": 123,
      "mainConversationId": 123,
      "steps": ["..."]
    }
  ],
  "page": 0,
  "pageSize": 20,
  "total": 137,
  "hasMore": true
}
```

`mainConversationId` is the id to hand to `GET /conversations/:chatId/history`
for the full dialog this row belongs to (it's the same value that endpoint's
own `mainConversationId` field would show).

- `GET /api/bot/journal/:id` — one row by its numeric `id` (the same `id`
  field from the list above), same shape as one item. Also takes
  `?includeAudit=true` for the same `steps`. 404 if it doesn't belong to
  this bot.
- The journal returns the bot's own client traffic. A row count that differs
  from what a Wikibot staff member sees in the dashboard is expected and not
  a bug.

## Other endpoints (accept `ask` or `manage`)

These predate the config API and aren't config-shaped, but are useful in the
same conversations and work with an `ask`-only key too (no need for `manage`
if that's all the user's key has):

- `GET /api/bot/agents` — flow list, see above; the deprecated alias of
  `GET /flows` and the only one of the two an `ask`-only key can call.
- `GET /api/bot/ask` (params in the query string) / `POST /api/bot/ask`
  (same fields as JSON body) — send a message to the bot as if from a real
  client, the way a messenger integration would. Fields: `query` (string,
  required — the message text; **not** `question`/`message`/`text`),
  `chatId` (string, required — the conversation id; reuse the same value to
  continue one dialog), `format` (`"links"` default or `"raw"`), `msgId`
  (optional), `attachments` (optional string array), `agentId` (optional
  number). If the integration has a `webhookUrl` configured the response is
  delivered async to that webhook instead of in the HTTP response — for a
  synchronous reply while testing, use a key without one configured. Useful
  to generate a real conversation to then read back with
  `GET /conversations/:chatId/history`.
- `GET /api/bot/kb` — list existing knowledge base sources: `{ url, sources: [{ id, url, createdAt, indexedAt }] }`.
  `url` is kept for backward compatibility (the earliest source only) — use
  `sources` for the full list.
- `POST /api/bot/kb/create` — add a URL to be crawled into the KB. Only
  checked for being non-empty, **not** for being a well-formed URL (some
  integrations send a bare hostname with no scheme) — double check with the
  user that what you're sending is really reachable if the crawl later
  turns up nothing. This adds a *source*, it does not let you edit article
  content — that's dashboard-only (see the top of this file).
- `POST /api/bot/kb/upload-file` — upload a file as a KB source.
- `GET /api/bot/search` — run the bot's own retrieval search against its KB
  (useful to check whether an answer is actually groundable before editing a
  prompt to fix a wrong answer). Each result includes `id` — the same article
  id `GET/PATCH /api/bot/articles/:id` use, and currently the only way to
  find the id of an *existing* article through this API (`POST /articles`
  only tells you the id of the one it just created).

## Rules

- Never call a write endpoint with `dryRun` omitted (where the endpoint
  supports it — `/config`, `/flows/:flowId/agents`,
  `/flows/:flowId/agents/:name`, and `/flows/:flowId/clone`) unless the user
  has already seen and approved the diff from a prior `dryRun: true` call in
  this same conversation.
- Only use `path` values that came back from `/config/schema` in this
  session — do not guess a path or reuse one from memory of a different bot.
- Treat `effect.onTrue`/`effect.onFalse` as authoritative for what a boolean
  actually does; several of these flags are named as a negation ("disable…")
  where `true` and `false` read backwards from the name.
- Never invent an agent `name`, job `id`, or datatable `id` — always resolve
  it from a `GET` call in this session first.
- There is no delete endpoint anywhere in this API. If the user asks to
  remove an agent's scenario, a job, or a datatable, tell them to do it from
  the Wikibot dashboard instead of trying to fake it via this API.
- **The creating POST endpoints are not idempotent** (`/tools`,
  `/datatables`, `/articles`, `/flows/:flowId/agents`, `/flows/:flowId/clone`)
  — there is no `Idempotency-Key`. A retried POST after a timeout or a
  dropped connection can create a duplicate, and — see above — there's no
  delete endpoint to clean it up afterwards; only the dashboard can. Don't
  blindly retry a POST that may have already succeeded server-side; if a call
  times out, check the resource's list endpoint first to see whether it
  actually landed before deciding to resend.
