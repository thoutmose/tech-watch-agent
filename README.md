# tech-watch-agent

Data store and reports for automated tech watch runs.

## How it works

- **`sources.md`** — each topic is a `##` header, followed by a `-` list of sources (newsletters, blogs, subreddits, X accounts, arXiv categories, etc.) to check for that topic. This file (and `discord-threads.json`, see below) is gitignored *here* — the actual source of truth is the private [`thoutmose/tech-watch-config`](https://github.com/thoutmose/tech-watch-config) repo, which the on-demand skill, the weekly cloud routine, and the `#cli` Discord bot all read from and write to directly. Edit topics/sources either straight in that repo, or live via the `#cli` Discord channel (see `~/Developer/tech-watch-cli-bot`).
- **`reports/<topic-slug>/<YYYY-MM-DD>.md`** — one compiled report per topic per run: an abstract, a curated list of the most cited/important/latest items found, each summarized, plus the sources consulted (including any newly discovered ones). Reports *are* committed to this repo.
- When a run finds a good source that isn't in `sources.md` yet, it gets appended there and pushed to `tech-watch-config` automatically, so future runs — on-demand or scheduled — pick it up too. Same for newly created Discord threads.

## Running a tech watch

- **On demand**: in a Claude Code session, run `/tech-watch [topic] [period]`, e.g. `/tech-watch "LLM agents" this year`. Omit the topic to run all topics in `sources.md`. Default period is "this week".
- **Automatically**: a weekly cloud routine runs every Monday at 08:00 UTC across all topics in `sources.md`, covering the prior week, and posts each topic's report to its Discord thread (see below). It clones both this repo and `tech-watch-config` at the start of each run, reading sources/thread-mapping from the latter and writing any discoveries straight back to it — no manual re-syncing required.
- **Editing config from Discord**: type commands like `topic add "<Name>"` or `source add "<Topic>" <url>` in the `#cli` Discord channel and the `tech-watch-cli-bot` bot applies them to `tech-watch-config` immediately. See that project's README for setup.

## Quality bar

Reports favor items that are well-cited/discussed, from credible primary sources, and clearly dated within the requested period. Low-signal reposts, unverifiable paywalled teasers, and duplicates of already-covered items are skipped. Each topic report is capped to keep signal high rather than listing everything found.

## Discord delivery

The weekly automated run posts to a Discord Forum channel: one thread per topic, created on first post and reused every week after via a topic->thread-ID mapping stored in `tech-watch-config`'s `discord-threads.json`. The webhook URL itself still lives only in the `weekly-tech-watch` cloud routine's config (https://claude.ai/code/routines) — it's a secret, not project data, so it isn't in any repo. On-demand `/tech-watch` runs still just present results in chat.

## Roadmap

- Triggering new *research* runs from Discord (as opposed to editing config, which the `#cli` bot now handles) is not yet wired up.
