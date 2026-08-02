---
name: tech-watch
description: Runs a tech watch — reads topics/sources from the tech-watch-agent repo's sources.md, researches each topic in parallel via the tech-watch-researcher sub-agent, compiles a markdown report with an abstract per topic, updates sources.md with newly discovered sources, commits/pushes the results, and presents the findings directly. Use for "/tech-watch", "do a tech watch on X", "run this week's tech watch", etc.
---

# Tech Watch

Orchestrates a tech watch: one sub-agent (`tech-watch-researcher`) per topic runs the actual research in parallel; you (the orchestrator) parse inputs, compile the final report, maintain `sources.md`, and present the result.

Data lives in `~/Developer/tech-watch-agent` (a dedicated git repo — cloned locally, and also used by the weekly automated cloud routine). If that directory doesn't exist, tell the user and stop.

## 1. Parse the invocation

Args may include a **topic** and/or a **period**. Examples: `/tech-watch`, `/tech-watch "LLM agents"`, `/tech-watch this year`, `/tech-watch "LLM agents" last month`.

- **Topic**: if given, match case-insensitively (substring OK) against the `##` headers in `sources.md`. If it matches none, tell the user and list the available topics instead of guessing. If no topic given, run **all** topics in `sources.md` (skip the "Example Topic (delete me)" placeholder if it's still there).
- **Period**: default is **this week**. Recognize "this week/month/year", "last week/month/year", or an explicit range. Resolve to a concrete date range with Bash — don't guess relative dates:
  ```bash
  today=$(date +%Y-%m-%d)
  # this week: today minus 7 days -> today
  since=$(date -d "7 days ago" +%Y-%m-%d)
  # this month: first of current month -> today
  # this year: Jan 1 of current year -> today
  # last week/month/year: shift the same window back by one period
  ```
  Always pass the resolved concrete range (e.g. "2026-07-26 to 2026-08-02") to sub-agents, never a relative phrase.

## 2. Sync and read sources.md

`sources.md` and `discord-threads.json` are gitignored in this repo — the actual source of truth is the private `thoutmose/tech-watch-config` repo, shared with the weekly cloud routine and the `tech-watch-cli-bot` Discord bot. Before reading, sync it:

```bash
if [ -d ~/Developer/tech-watch-config/.git ]; then
  git -C ~/Developer/tech-watch-config pull --ff-only
else
  git clone https://github.com/thoutmose/tech-watch-config ~/Developer/tech-watch-config
fi
cp ~/Developer/tech-watch-config/sources.md ~/Developer/tech-watch-config/discord-threads.json ~/Developer/tech-watch-agent/
```

Then read `~/Developer/tech-watch-agent/sources.md`. Parse `##` headers as topic names, and the `-` list beneath each (until the next `##`) as that topic's sources.

## 3. Research each topic in parallel

For each selected topic, call the `Agent` tool with `subagent_type: tech-watch-researcher`, `run_in_background: false` is not required — launch **all topics' Agent calls in the same message** so they run in parallel. Prompt each with: the topic name, its source list, the resolved date range, and today's date. See the sub-agent's own instructions for what it returns (per-item summaries + optional "new sources discovered" section).

## 4. Compile the report per topic

For each topic, write `~/Developer/tech-watch-agent/reports/<topic-slug>/<YYYY-MM-DD>.md` (slug = lowercase, spaces/punctuation → hyphens), structured as:

```markdown
# <Topic> — Tech Watch (<period label>)
_Generated <YYYY-MM-DD>, covering <date range>_

## Abstract
<3-6 sentences synthesizing the period's themes across what was found — the takeaway, not a list recap>

## Findings
<the sub-agent's per-item output, verbatim or lightly cleaned up>

## Sources consulted
- <each source from sources.md that was checked>
- <newly discovered sources, marked (NEW)>
```

If a sub-agent found nothing meeting the quality bar, say so in the abstract rather than omitting the topic silently.

## 5. Update sources.md

For every "New sources discovered" entry a sub-agent returned, append it to the right topic's list in `sources.md` (dedupe against what's already there — case-insensitive, match on URL or name). Don't touch topics that weren't part of this run.

If anything changed, propagate it back to the shared config repo so the weekly routine and the Discord bot see it too:

```bash
cp ~/Developer/tech-watch-agent/sources.md ~/Developer/tech-watch-config/
cd ~/Developer/tech-watch-config && git add sources.md && git commit -m "On-demand tech watch: new sources — <date>" && git push
```

## 6. Commit and push

```bash
cd ~/Developer/tech-watch-agent
git add -A
git commit -m "Tech watch: <topic(s)> — <date>"
git push
```
This repo is dedicated solely to tech-watch data, so committing/pushing is the normal, expected outcome of every run — do it without asking each time, but mention in your final message that it happened (and where).

## 7. Present the result directly

In your reply to the user, for each topic show the abstract, the findings (titles + 1-line summaries + links is fine if the report itself is long — full detail is in the file), and the sources consulted. Note the report file path(s) and that they were committed/pushed. Don't just say "done, check the file" — the user explicitly wants the result delivered directly in chat.

## Notes for the future

Discord delivery for the **automated weekly cloud routine** is implemented via a webhook (not an MCP connector, not this skill): one Forum channel, one persistent thread per topic. The topic->thread-ID mapping and the source list live in the private `thoutmose/tech-watch-config` repo (see step 2 above), which the routine clones at the start of every run and writes back to directly — see the `weekly-tech-watch` routine at https://claude.ai/code/routines. This on-demand skill path does not post to Discord; it only presents results in chat, per step 7.

Editing topics/sources from Discord (add/remove topic, add/remove source) is handled by the `tech-watch-cli-bot` Discord bot listening on a `#cli` channel — see `~/Developer/tech-watch-cli-bot`. It also reads/writes `tech-watch-config` directly.

Triggering new *research* runs from Discord (as opposed to editing config) is still not implemented.
