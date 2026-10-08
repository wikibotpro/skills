# Wikibot solutions and best practices

Read this before building anything new for a client — a request like "the
bot should know our catalog" or "send the operator a summary" usually has one
mechanism that fits clearly better than the others, and picking the wrong one
is far more expensive to undo than an API call (tables, tools, scenarios and
jobs can only be removed in the dashboard).

Each recipe below says which parts this API can do and which stay in the
dashboard. When a step is dashboard-only, say so plainly and give the user
the exact place to click — don't try to approximate it through the prompt.

Sources: https://wikibot.pro/docs/solutions/* and https://wikibot.pro/docs/learning/*.
Every doc page is also served as Markdown by appending `.md` to its URL
(e.g. `https://wikibot.pro/docs/learning/memory.md`) — fetch that when you need
more detail than this file has.

## Pick the mechanism first

Ask what kind of information or behavior the client actually has, then match:

| What the client has / wants | Use | Why not the alternatives |
| --- | --- | --- |
| Structured records with fields: product catalog, price list, branches with addresses, tariffs, schedules | **Memory table** (datatable) | The bot queries it with SQL — exact filters ("under 5000 ₽", "in stock in Kazan"), counts, sorting. KB search is fuzzy and mixes rows from different chunks; a prompt can't hold hundreds of rows. |
| Data the bot should *collect* during dialogs: leads/contacts, complaints, quality ratings, client profiles | **Memory table** the bot writes to | Persists across conversations and can be reported on (`POST /datatables/query`). Shared data (below) dies with the conversation. |
| Live data that changes by the minute or lives in another system: stock, order status, balance, ticket history | **Custom REST function tool** | A table or KB copy goes stale. Call the source of truth. |
| Long explanatory text: manuals, policies, how-tos, FAQs in prose | **Knowledge base** (crawl, upload, or private article) | Retrieval works well for prose; tables don't. |
| An answer that must be word-for-word per regulation, or a sensitive topic that must always go to a human (refunds, data deletion) | **First line (L1)** — `/first-line` | A prompt rule gets paraphrased; L1 returns the fixed text or transfers instead of a composed answer. Fires only when the agent searches, and early in a conversation — see "First line" in SKILL.md. |
| A flood of identical simple questions (hours, delivery terms, contacts) in widget/Telegram/MAX | **Button menu** — dashboard only | Answers without an LLM call — zero credits. Not supported for helpdesk integrations yet. |
| Company-specific terms, abbreviations, slang | **Glossary** (`glossary` in `/config` if the schema exposes it, otherwise dashboard) | Improves KB/L1 matching at index time; a prompt can't fix retrieval. |
| Key facts the bot must never get wrong: contacts, requisites, which email to give | **Default agent prompt** | Anything not written down is a grey zone the model fills in by guessing. |
| A narrow multi-step process (booking, refund intake, delivery calc) | **Scenario** | Keeps the main prompt short; see "Scenarios" below for which mode. |
| Act after the client goes quiet (follow-up, "is your issue solved?") | **Job** | The only thing that runs without a client message. |

When a request fits two rows (a catalog that also has long product
descriptions), split it: structured fields go to a memory table, the prose to
the KB, and the prompt tells the agent which to use for what.

## Recipes

### Product catalog / price list → memory table

The default answer when a client says "the bot should know our catalog /
assortment / price list". Memory has a ready-made "Каталог товаров" template in
the dashboard; via the API you build the same thing yourself.

1. Ask for a sample of the data (a few rows of their spreadsheet) and design
   columns from it: identifying fields (`sku`, `name`, `category`), the fields
   clients filter by (`price` as `number`, `in_stock` as `boolean`, `city`,
   `size`), and one short `text` column for a description if needed. Keep
   long descriptions out — they bloat the size limit and belong in the KB.
2. `POST /api/bot/datatables` with a clear table `description` — the bot
   reads it to decide when to query the table, so write it as "Каталог
   товаров магазина: цены, наличие, характеристики. Используй для любых
   вопросов о товарах, ценах и наличии."
3. Load the rows: if the user gives you the file, convert it and
   `POST /datatables/:id/rows` in batches of up to 1000 (numbers as numbers,
   booleans as `true`/`false`; show a sample and the total count before
   sending). If they'd rather do it themselves, the dashboard imports CSV/XLSX
   directly: Обучение → Память → the table. Afterwards, verify with
   `POST /datatables/query` (`SELECT count(*) FROM <table>`, plus a sample
   `SELECT ... LIMIT 5`). To refresh prices later, `PATCH` the changed rows
   by `_id` rather than re-inserting — a second insert duplicates the
   catalog.
4. Add a short section to the default agent's prompt (dry-run first) on how to
   use it — a specific instruction in the prompt overrides the table's
   generic description:
   ```
   ### Каталог ###
   На вопросы о товарах, ценах и наличии отвечай только по таблице "catalog".
   Если товара нет в таблице — так и скажи, не придумывай. Не показывай больше 5 позиций за раз.
   ```
5. **Check the plan limit before committing to this design** —
   `GET /api/bot/billing` → `datatables` gives this bot's actual
   `tablesLimit`, `sizeLimit`/`sizeUsed` (bytes) and `maxRowsPerTable`. For
   reference: Free 300 KB (and 1000 rows/table, 3 tables), Start 1 MB,
   Growth 5 MB, Business 10 MB; 50,000 rows per table at most on any plan. A
   catalog of tens of thousands of
   rows with descriptions won't fit — then either trim columns, or use a REST
   tool against the client's own catalog API, or (prose-heavy catalogs) put
   one KB document per category and use the "specific document" recipe below.
6. If prices/stock change daily and the client has an API, prefer a REST tool
   for those fields — a table imported once a week answers with stale prices.

### Collecting leads, complaints, ratings → memory table the bot fills

Same `POST /datatables`, but the columns describe what to capture (`name`,
`phone`, `reason`, `rating`...) and the description says *when* to write:
"Записывай сюда каждую жалобу клиента: причину и номер заказа, если назван."
Before finishing each dialog the bot checks every assigned table on its own,
guided by its description and column descriptions. Report on it later with
`POST /datatables/query` (`GROUP BY reason`, `ORDER BY _created_at DESC`).

Don't confuse with **shared data** (agent setting "Совместные данные") — a
key/value store that lives only until the end of one conversation, used to
carry e.g. an order number across scenario hand-offs.

### Conversation history from the helpdesk → REST tool

(docs/solutions/history — HelpDeskEddy, Usedesk, Omnidesk.) Lets the bot read
the earlier client↔operator exchange in the ticket instead of re-asking.

- `POST /api/bot/tools`, e.g. for HelpDeskEddy:
  ```json
  {
    "name": "get_ticket",
    "description": "Получение истории переписки по заявке",
    "parameters": {
      "type": "object",
      "properties": {
        "ticketId": { "type": "number", "description": "_ticketId из текущего общего контекста" },
        "page": { "type": "number", "description": "Страница с ответами, по умолчанию 1" }
      },
      "required": ["ticketId", "page"]
    },
    "actions": [{
      "method": "GET",
      "url": "https://helpdeskeddy.ru/api/v2/tickets/{ticketId}/posts?page={page}",
      "headers": [
        { "key": "Authorization", "value": "<helpdesk API key>" },
        { "key": "Content-Type", "value": "application/json" }
      ]
    }]
  }
  ```
  Replace the host with the client's own helpdesk domain if they have one.
  `_ticketId` is put into shared data by the integration automatically. Flag
  the helpdesk API key before sending it (see the security note in SKILL.md).
- Without a prompt instruction the tool won't be used reliably. Add:
  ```
  При ответе на запрос следуй по алгоритму:
  1. Запроси заявку, чтобы получить историю переписки клиента и оператора и понять суть проблемы. Проанализируй время сообщений, чтобы понять текущую проблему.
  2. Дай ответ с учетом истории переписки.
  ```

### Summary for the operator on transfer

First choice: the built-in **`summary` agent** — `PATCH .../agents/summary`
with `enabled: true` (and `options.minClientMessages`, default 2). It costs
0.1 credit per summary and is delivered natively to: API (`summary` field),
chat-center, Usedesk, Omnidesk, Carrot Quest, Freshchat, Chat2Desk, Bitrix24.

Only if the integration isn't on that list, or the client needs a specific
format/destination, build the custom variant (docs/solutions/summary): a REST
tool `send_summary` (`ticketId` + `text`) posting a private comment into the
helpdesk, plus this prompt rule:
```
!Важно!: перед любым переводом на оператора, в том числе по инструкциям сценариев, всегда делай резюме беседы с помощью функции "send_summary". Перевод на оператора состоит из двух вызовов:
1. отправка резюме;
2. перевод на оператора (redirect_to_operator).
```
Don't enable both — the operator gets two summaries.

### Transfer to operator outside working hours

Two layers, pick by how simple the schedule is:

- **Fixed schedule** → bot-level settings: `botWorkingHours` /
  `workingHours` and the `templates.nonWorkingBotRedirect.*` text in
  `/config`. No prompt work, applied before the agent runs (cheaper).
- **Conditional logic** (only some topics, a lunch break, different text per
  day) → agent options "Дата и время" + "Сообщение при переводе" (check the
  agent's `options` keys in `GET .../agents`; if they aren't there, it's a
  dashboard toggle on the agent's settings gear) and a prompt rule. The agent
  receives time **in UTC** — convert the client's hours:
  ```
  Если запрос приходит с 15:00 до 06:00 UTC или в субботу-воскресенье, переведи на оператора с сообщением:
  "Ваш запрос принят, рассмотрим его в рабочее время — пн–пт, 9:00–18:00 МСК."
  ```

### Attach a link to the source document

(docs/solutions/links.) Enable the agent option "Ссылки на статьи" (dashboard
gear next to Scenarios, unless it shows up in the agent's `options`), then
add to the prompt:
```
Если у документа, на основе которого сделан ответ, в метаданных есть ссылка, прикрепи ее к своему ответу. Не используй markdown для ссылки.
```
Links don't exist for private sources (private articles, uploaded-only
content without a URL) — tell the user if their KB is mostly those.

### Answer from one specific document

(docs/solutions/select-doc.) Point the agent at the exact KB document title —
an exact title in the search query boosts that document's weight:
```
Если вопрос клиента о возврате, ищи в базе знаний по запросу "Возвраты.pdf".
Если вопрос о видеокамерах, ищи по запросу "Каталог видеокамер — <модель камеры из запроса клиента>".
```
Verify the title first with `GET /api/bot/search` — it must match what's
actually in the KB. Related agent settings (dashboard): "Раздельные документы"
(answer from one found document only — when mixing two instructions is
dangerous) and "Фильтровать по тегам БЗ" (tag documents by region/product
line and tell the agent in the prompt which tag to use when).

### Plain text without Markdown

(docs/solutions/markdown.) For channels that show `*` and `[]()` literally:
```
Для ответов используй обычный текст без форматирования. Не используй markdown для ссылок, выделений, заголовков.
```

### Answer general questions from the model's own knowledge

(docs/solutions/knowledges.) By default the bot only answers from the KB,
L1, prompt and dialog. To allow general knowledge for a topic:
```
Если пользователь спрашивает про <тема>, ты можешь использовать свои общие знания. Для таких вопросов не ищи в первой линии и базе знаний — сразу отвечай.
```
Scope it to named topics — a blanket permission invites made-up answers about
the client's own business, and an agent that answers without searching also
never reaches the first line.

### Fixed answers and forced transfers → first line

1. `GET /first-line` first — a question that already exists anywhere is
   rejected as a duplicate; extend that record instead.
2. One record per intent: a narrow main question plus up to 9 real client
   phrasings as `similarQuestions` (take them from the journal:
   `GET /journal?query=...`). For "always to a human" topics use
   `redirect: true` instead of an answer.
3. Test in a **new** conversation (`/ask` with a fresh `chatId`): L1 only
   fires while no earlier search in that conversation has found KB
   documents. If it doesn't fire, read the history audit for the search
   query the agent used and add it as a similar question.
4. A question that only makes sense mid-conversation ("а сколько это
   стоит?" after a product was discussed) is a poor L1 candidate — put the
   rule in the prompt instead.

### Create a deal/lead in a CRM

amoCRM has a built-in function (dashboard: Агенты → Функции → Добавить →
"Создание сделки в amoCRM"). Other CRMs → a REST tool. Either way the prompt
must say *when* to call it and *what* goes into each field, e.g. "Когда узнал
адрес и срок аренды — создай сделку с названием «Клиент хочет арендовать …»,
затем переведи на оператора."

### Follow up after silence

A job (`POST .../jobs`), e.g. `inactivityMinutes: 60`,
`conversationFilter: "NO_CLIENT_REPLY"`, `maxRunsPerConversation: 1`,
instructions "Уточни, решён ли вопрос клиента". See SKILL.md for the firing
rules — especially that it only picks up conversations that started after it
was enabled.

## Scenarios: which mode

| Situation | Mode |
| --- | --- |
| Client hops between topics in one conversation; main agent needs the scenario's tools/tables; full context matters | Attached (`workAsSkill: true`, `### Навыки ###`) |
| Long isolated process with its own rules, its own (cheaper or stronger) model or settings | Dialog transfer (default, `### Сценарные агенты ###`) |

In transfer mode the scenario gets a *summary* of the dialog, not the dialog
itself — anything that must survive the hand-off (order number, phone) should
be saved to shared data: "При получении номера заказа сохрани его по ключу
'заказ' в совместные данные и перейди к сценарию 'order_info'."

Split into scenarios when the default prompt grows a second unrelated
procedure — not for every topic. A handful of one-line rules stays in the
main prompt.

## Writing the agent prompt

- Give a clear role ("Ты менеджер по продажам туристического агентства"),
  then structured rules and step lists.
- For every condition, say what to do both when it holds and when it doesn't
  ("Если тур не найден — не говори, что туров нет, переведи на оператора").
- Put key company facts (contacts, requisites, which email to give and when)
  directly in the prompt — the most common hallucination is an invented
  contact.
- When two KB articles cover similar cases for different audiences (физлица
  vs юрлица), tell the agent to ask which one applies before answering.
- Include a short example dialog when the expected format matters.
- The agent sees roughly the last 25–30 messages and cannot open URLs that
  aren't in the KB.

## Fixing wrong answers — where to look

Read the request in `GET /conversations/:chatId/history` (the `logs` audit)
to see what the answer was based on, then:

| Symptom | Fix |
| --- | --- |
| Invented a fact (email, price, policy) | State the true fact and when to use it in the prompt; tighten the "general knowledge" permission if any |
| Expected L1 didn't fire | Did the agent search at all, and was it the conversation's first search that found anything? (L1 switches off after that.) If it searched in time, the audit shows the agent's search query — add it to that record's `similarQuestions` via `PATCH /first-line/:id` |
| Didn't find an article that has the answer | Make the article state the question explicitly (a heading phrased like the question); add the term to the glossary; for an uneditable crawled page — add "Дополнительный контент" in the dashboard |
| Found the right articles but answered from the wrong one | Clarify the articles' scope in their titles/extra content; add a prompt rule to ask the distinguishing question |
| Correct but not the wording the client wants | Fixed answer → L1 |
| Template text instead of an agent answer | It's a bot-level template (`/config`), see the pipeline in SKILL.md |

## Cost levers

Mention these when the client complains about credit spend. Start from
`GET /api/bot/billing` — it shows the remaining credits, day/month spend,
and any credit limits; a "bot went silent" complaint is often a hard limit
already reached, not a configuration problem.

- Pre-agent steps cost nothing extra: ignore phrases, greeting/gratitude
  phrase matching, the button menu. Handling more traffic there is the
  cheapest saving.
- Each agent action that calls the model is 1 credit. The L1 lookup inside
  the agent's search isn't billed on its own; a match ends the turn with the
  fixed answer, so the agent doesn't spend further steps on composing one.
- `editor` adds 0.1 credit to *every* text answer; `spam`, `operator` and
  `summary` cost 0.1 per hit. Turn on only what the client needs.
- A scenario in transfer mode can run on a cheaper model than the default
  agent.
