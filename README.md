# tech-watch-agent

Data store and reports for automated tech watch runs.

## How it works

- **`sources.md`** — you maintain this. Each topic is a `##` header, followed by a `-` list of sources (newsletters, blogs, subreddits, X accounts, arXiv categories, etc.) to check for that topic.
- **`reports/<topic-slug>/<YYYY-MM-DD>.md`** — one compiled report per topic per run: an abstract, a curated list of the most cited/important/latest items found, each summarized, plus the sources consulted (including any newly discovered ones).
- When a run finds a good source that isn't in `sources.md` yet, it gets appended there automatically under the right topic, so future runs pick it up too.

## Running a tech watch

- **On demand**: in a Claude Code session, run `/tech-watch [topic] [period]`, e.g. `/tech-watch "LLM agents" this year`. Omit the topic to run all topics in `sources.md`. Default period is "this week".
- **Automatically**: a weekly cloud routine runs every Monday at 08:00 UTC across all topics in `sources.md`, covering the prior week, and commits/pushes the new reports here.

## Quality bar

Reports favor items that are well-cited/discussed, from credible primary sources, and clearly dated within the requested period. Low-signal reposts, unverifiable paywalled teasers, and duplicates of already-covered items are skipped. Each topic report is capped to keep signal high rather than listing everything found.

## Roadmap

- Discord delivery (one channel per topic, plus on-demand triggering from Discord) is planned but not yet wired up. Until then, results are delivered directly in chat for on-demand runs, and via commits to this repo for the weekly automated run.
