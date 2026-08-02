# tech-watch-agent

Data store and reports for automated tech watch runs.

## How it works

- **`sources.md`** — you maintain this. Each topic is a `##` header, followed by a `-` list of sources (newsletters, blogs, subreddits, X accounts, arXiv categories, etc.) to check for that topic.
- **`reports/<topic-slug>/<YYYY-MM-DD>.md`** — one compiled report per topic per run: an abstract, a curated list of the most cited/important/latest items found, each summarized, plus the sources consulted (including any newly discovered ones).
- When a run finds a good source that isn't in `sources.md` yet, it gets appended there automatically under the right topic, so future runs pick it up too.

## Running a tech watch

- **On demand**: in a Claude Code session, run `/tech-watch [topic] [period]`, e.g. `/tech-watch "LLM agents" this year`. Omit the topic to run all topics in `sources.md`. Default period is "this week".
- **Automatically**: a weekly cloud routine runs every Monday at 08:00 UTC across all topics in `sources.md`, covering the prior week, and posts each topic's report to its Discord thread (see below). Note: `reports/`, `sources.md`, and `discord-threads.json` are all `.gitignore`d, so the routine's `git add -A` step does not persist any of them to this repo — Discord is the only durable record of report content, and both the source list and the topic->thread-ID mapping live embedded in the `weekly-tech-watch` routine's own config instead, manually re-synced there when they change.

## Quality bar

Reports favor items that are well-cited/discussed, from credible primary sources, and clearly dated within the requested period. Low-signal reposts, unverifiable paywalled teasers, and duplicates of already-covered items are skipped. Each topic report is capped to keep signal high rather than listing everything found.

## Discord delivery

The weekly automated run posts to a Discord Forum channel: one thread per topic, created on first post and reused every week after via a topic->thread-ID mapping. The webhook URL and that mapping both live only in the `weekly-tech-watch` cloud routine's config (https://claude.ai/code/routines), not in this repo — `discord-threads.json` locally is a gitignored scratch copy, not the source of truth. On-demand `/tech-watch` runs still just present results in chat.

## Roadmap

- Triggering new on-demand runs from Discord is not yet wired up.
