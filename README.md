# Wikibot agent skills

Configure your Wikibot bot by talking to your coding agent — no dashboard
clicking for the routine changes.

## Install

**Any agent** (Claude Code, Codex, Cursor, Copilot, Windsurf, Gemini, Zed and
others) — via [skills.sh](https://www.skills.sh):

```
npx skills add wikibotpro/skills
```

**Claude Code**, as a plugin, if you want it managed with the rest of your
plugins:

```
/plugin marketplace add wikibotpro/skills
/plugin install wikibot-config@wikibot
```

Either way you end up with the same skill — pick whichever fits how you
already install things.

## Setup

The skill needs two environment variables:

| Variable          | What it is                                                           |
| ----------------- | -------------------------------------------------------------------- |
| `WIKIBOT_API_KEY` | An integration API key from your bot's **Settings → API Keys** page. |
| `WIKIBOT_API_URL` | Optional. Defaults to `https://api.wikibot.pro`.                     |

**The key must have the `manage` scope** — tick it when creating the key, or
add it to an existing key on the same page. A key that only has `ask` can read
the flow list and the knowledge base sources, but everything else returns 403.

Set them in your shell before starting the agent:

```bash
export WIKIBOT_API_KEY="..."
```

## What it can do

- **Bot settings** — greeting/gratitude/no-answer templates, operator transfer
  wording, ignore phrases, working hours, answer delay, glossary.
- **Agents** — edit the default agent's instruction, turn the `spam`,
  `operator`, `editor`, `translator` and `summary` agents on or off, change
  their options.
- **Scenarios** — create a scenario agent and wire it into the default agent's
  instruction, in either dialog-transfer or attached-skill mode.
- **Flows** — clone a whole flow with its agents, jobs and access grants.
- **Jobs** — proactive follow-ups that fire after a conversation goes quiet.
- **Datatables** — create tables, change their schema, query the rows with
  read-only SQL, and control which agents may read them.
- **REST function tools** — define a function the bot's model can call against
  your own API, and connect it to specific agents.
- **Knowledge base** — write and edit private articles, add crawl sources,
  upload files, and run the bot's own retrieval search.
- **Debugging** — read a conversation's full history with the step-by-step
  analysis of what handled each request, and search the request journal by
  date, outcome or text.

## Notes

- **Nothing can be deleted through this skill.** It creates and edits only;
  removing an agent, job, datatable or tool stays a dashboard action. The one
  exception is disconnecting a tool from an agent, which is reversible.
- Changes to settings and agents are previewed first: the agent shows you the
  before/after diff and applies it only after you confirm.
- Requests are rate limited per bot (120/minute, and 20/minute for the
  journal, dialog history and datatable queries). The agent backs off on its
  own when it hits the limit.
