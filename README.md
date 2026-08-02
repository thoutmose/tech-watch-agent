# tech-watch-agent

*[Français](README.fr.md)*

An automated + on-demand tech watch: a daily cloud routine and an interactive Claude Code skill research a configurable list of topics, write structured reports, and deliver them to Discord — with a Discord-native command line for managing the whole thing.

> **Built entirely by agentic AI.** Every part of this project — the research pipeline, the cloud routine wiring, the Discord bot and its commands, the incident debugging, and this README (English and French) — was designed, written, and operated by Claude (Anthropic's agentic coding assistant), working autonomously and interactively with the project owner. No code or documentation here was hand-written by a human.

## Table of contents

- [Overview](#overview)
- [Repositories](#repositories)
- [Architecture](#architecture)
- [How a run works](#how-a-run-works)
- [Running a tech watch](#running-a-tech-watch)
  - [On-demand: the `/tech-watch` skill](#on-demand-the-tech-watch-skill)
  - [Automated: the `weekly-tech-watch` cloud routine](#automated-the-weekly-tech-watch-cloud-routine)
  - [On-demand from Discord: the `run` command](#on-demand-from-discord-the-run-command)
- [Discord interface](#discord-interface)
- [Command reference](#command-reference)
- [Configuration & data flow](#configuration--data-flow)
- [Known limitations & operational notes](#known-limitations--operational-notes)
- [Setup](#setup)
- [Roadmap](#roadmap)

## Overview

Twelve (or however many are currently configured) topics — things like `dbt`, `Apache Airflow`, `Cybersecurity`, `Linux` — each have a curated list of sources (blogs, subreddits, GitHub release feeds, newsletters). A run researches each topic over a time window, filters findings against a quality bar, writes one Markdown report per topic, and delivers it either straight into chat (on-demand) or into a persistent Discord thread (automated).

There are three moving pieces, in three repos, plus one Claude Code cloud routine:

## Repositories

| Repo | Visibility | Role |
| --- | --- | --- |
| **[tech-watch-agent](https://github.com/thoutmose/tech-watch-agent)** (this repo) | Public | The on-demand skill, the research sub-agent, and committed reports. |
| **[tech-watch-config](https://github.com/thoutmose/tech-watch-config)** | Private | Source of truth for the topic list, sources, and the Discord thread-ID mapping (`sources.md`, `discord-threads.json`). |
| **[tech-watch-cli-bot](https://github.com/thoutmose/tech-watch-cli-bot)** | — | A Discord bot that turns a `#cli` channel into a command line for editing `tech-watch-config` and for firing on-demand runs. |
| **`weekly-tech-watch`** | — | Not a repo — a [Claude Code cloud routine](https://code.claude.com/docs/en/routines) (claude.ai/code/routines) that runs the automated daily pipeline and posts to Discord. |

## Architecture

```mermaid
flowchart LR
    subgraph Config["tech-watch-config (private repo)"]
        SRC["sources.md"]
        THR["discord-threads.json"]
    end

    subgraph Agent["tech-watch-agent (this repo)"]
        SKILL["/tech-watch skill"]
        SUB["tech-watch-researcher\nsub-agent (parallel, per topic)"]
        REP["reports/&lt;topic&gt;/&lt;date&gt;.md"]
    end

    subgraph Routine["weekly-tech-watch (Claude Code cloud routine)"]
        PROMPT["Stored prompt\n(embeds a snapshot of\nsources.md + discord-threads.json)"]
    end

    subgraph Bot["tech-watch-cli-bot"]
        CLI["#cli Discord commands"]
    end

    subgraph Discord["Discord"]
        FORUM["Forum channel\none persistent thread per topic"]
    end

    User1(("You, in a\nClaude Code session")) -->|"/tech-watch [topic] [period]"| SKILL
    SKILL --> SUB --> REP
    SKILL <-.synced before each run.-> Config

    User2(("You, in Discord")) -->|"topic/source commands"| CLI
    CLI <-->|"git read/write"| Config
    User2 -->|"run \"Topic\" [--notify]"| CLI
    CLI -->|"routine-fire API\n(optional Topic: hint)"| PROMPT

    Cron(("Daily cron\n05:00 UTC")) --> PROMPT
    PROMPT -->|"webhook POST"| FORUM
    CLI -.completion ping.-> User2
```

The important asymmetry in this diagram: the on-demand skill and the bot both talk to `tech-watch-config` **live**, but the cloud routine does not — see [Configuration & data flow](#configuration--data-flow).

## How a run works

For each topic in scope, a run:

1. Checks every listed source for items published in the time window (WebSearch/WebFetch).
2. Runs broader searches to catch what the listed sources missed, and to spot new sources worth adding permanently.
3. Filters by a quality bar: primary sources over aggregator rewrites; a real citation/discussion signal (HN/Reddit points, GitHub stars, arXiv citations, independent corroboration); a confirmed in-window publish date; no unverifiable paywalled teasers; deduplicated.
4. Writes `reports/<topic-slug>/<YYYY-MM-DD>.md`: a title, a 3–6 sentence abstract synthesizing the period's themes, a findings section (title, source, date, link, why it matters, summary — per item), and a sources-consulted list with newly discovered sources marked `(NEW)`.
5. If nothing met the bar, the report says so plainly instead of being padded or skipped.

## Running a tech watch

### On-demand: the `/tech-watch` skill

Run inside a Claude Code session:

| Invocation | Effect |
| --- | --- |
| `/tech-watch` | Every topic in `sources.md`, past week |
| `/tech-watch "LLM agents"` | Just the topic matching "LLM agents" (substring match) |
| `/tech-watch this month` / `last week` / `last year` | Change the period |
| `/tech-watch "LLM agents" last month` | Combine topic + period |

One `tech-watch-researcher` sub-agent (currently **Sonnet**) runs per topic, in parallel. Results are compiled and presented directly in chat, and `sources.md` is updated with any newly discovered sources. **This path does not post to Discord** — see the next two sections for that.

### Automated: the `weekly-tech-watch` cloud routine

Runs on its own, daily, no interaction needed:

- **Schedule**: `0 5 * * *` (05:00 UTC — 07:00 Europe/Paris in summer). The schedule has no timezone field, so it drifts by an hour across DST transitions.
- **Window**: the trailing 1 day (matched to the daily cadence — a longer window would re-report the same stories every day).
- **Delivery**: posts each topic's report — or a short "nothing new today" note if the window was quiet — into that topic's Discord thread via a webhook, splitting into multiple messages if a report exceeds Discord's 2000-character cap.
- **Model / schedule changes**: only reliably changeable from the routine's web UI editor at [claude.ai/code/routines](https://claude.ai/code/routines) — see [Known limitations](#known-limitations--operational-notes).

### On-demand from Discord: the `run` command

Typing `run "<Topic>"` in `#cli` fires the *same* routine immediately, via Anthropic's [routine-fire API](https://platform.claude.com/docs/en/api/claude-code/routines-fire), instead of waiting for the daily schedule. Topic scoping works through a small opt-in convention: the fire request's optional `text` field carries a `Topic: <name>` hint, which arrives wrapped in an untrusted `<routine-fire-payload>` block (Anthropic's API never lets fire-time input act as an instruction on its own); the routine's stored prompt explicitly reads that hint as *data* and restricts itself to the named topic. `run all` (or bare `run`) omits the hint and processes everything, same as the daily run.

## Discord interface

One Forum channel, one persistent thread per topic, created automatically the first time a topic posts and reused every time after (the mapping lives in `tech-watch-config`'s `discord-threads.json`). A separate `#cli` text channel is the bot's command line — see below.

## Command reference

All of these are typed in the `#cli` Discord channel and handled by `tech-watch-cli-bot` (see [that repo](https://github.com/thoutmose/tech-watch-cli-bot) for the source and full setup instructions):

| Command | Effect |
| --- | --- |
| `topic add "<Name>"` | Add a new topic |
| `topic rm "<Name>"` | Remove a topic and its sources |
| `topic list` | List all topics |
| `source add "<Topic>" <url>` | Add a source to a topic |
| `source rm "<Topic>" <url-or-index>` | Remove a source (by URL/text or by its number from `source list`) |
| `source list "<Topic>"` | List a topic's sources |
| `run "<Topic>"` | Fire an on-demand run scoped to one topic |
| `run all` | Fire an on-demand run for every topic |
| `run "<Topic>" --notify` / `run all --notify` | Same, and pings you in `#cli` once it actually posts (or times out) |
| `help` | Show this list |

Topic names match case-insensitively; unambiguous substrings work (`topic rm "airflow"` matches "Apache Airflow"). Quote names containing spaces.

There is intentionally **no `schedule` command** — see [Known limitations](#known-limitations--operational-notes) for why.

## Configuration & data flow

`tech-watch-config` (private) is the source of truth for topics, sources, and Discord thread IDs. Two of the three consumers stay live-synced with it:

- The **on-demand skill** syncs from it before each run.
- The **`#cli` bot** reads and writes it directly, on every command, via git.

**The daily cloud routine is the exception.** It does not clone or live-read `tech-watch-config` at runtime — its stored prompt embeds a *static snapshot* of `sources.md` and `discord-threads.json` written into it the last time someone (a maintainer, via the routine's own settings or Claude Code's internal tooling) refreshed it. A topic or source added via the bot shows up immediately in `tech-watch-config` and in the on-demand skill's next run, but **not** in the daily routine until that snapshot is manually refreshed. This is a known gap, not a design choice — see the roadmap.

## Known limitations & operational notes

These cost real debugging time to discover; recorded here so they don't have to be rediscovered:

- **Network access.** The routine's cloud environment must allow `discord.com` — either **Full** network access or **Custom** with `discord.com` in the allowlist. The default **Trusted** tier does *not* include it (it only covers package registries, GitHub, cloud provider APIs, and similar). Under **Trusted**, every webhook POST fails silently at the proxy with `CONNECT tunnel failed, 403` — the run otherwise completes normally (reports get written, git commits happen), so nothing *looks* broken except an empty Discord thread. Changeable only from the environment's settings at [claude.ai/code](https://claude.ai/code) (no API for this either).
- **No public API for schedule or model changes.** Anthropic's [routine-fire API](https://platform.claude.com/docs/en/api/claude-code/routines-fire) can only *start a run* of the routine's already-saved prompt — it cannot read or rewrite that prompt, the cron schedule, or the model, from outside claude.ai. This is deliberate (a routine-scoped bearer token that could rewrite its own config would be far more dangerous if leaked). Those changes require the web UI or an authenticated interactive Claude Code session.
- **Never let the routine delete Discord messages.** Its prompt explicitly forbids any `DELETE` request against Discord, added after an incident where a thread's starter/title message disappeared following two on-demand runs. The thread kept working fine without it (Discord doesn't require a title post to accept new messages), but it's a good reminder that an autonomous agent with webhook credentials should be told, explicitly, never to delete anything.
- **Use `curl`, not a scripting language's HTTP client, for the webhook posts.** Discord's edge (Cloudflare) has been observed to fingerprint and block non-browser-like clients (e.g. Python's `urllib`) with a 403, even against a perfectly valid webhook.

## Setup

- **This repo**: nothing to install — it's data (`reports/`, gitignored `sources.md`) plus the skill/agent definitions under `.claude/`.
- **`tech-watch-cli-bot`**: full setup (Discord bot creation, GitHub token, the routine-fire token) is documented in [that repo's README](https://github.com/thoutmose/tech-watch-cli-bot#one-time-setup).
- **The `weekly-tech-watch` routine**: created and configured at [claude.ai/code/routines](https://claude.ai/code/routines); its prompt, schedule, and environment are managed there (see limitations above for what can and can't be done from outside that UI).

## Roadmap

- [x] Automated daily delivery to Discord
- [x] On-demand run triggering from Discord (`run` command, with topic scoping and `--notify`)
- [ ] Automatic sync of the routine's embedded config snapshot with `tech-watch-config` (currently a manual step)
- [ ] A reliable, API-driven way to change the routine's schedule/model without the web UI
