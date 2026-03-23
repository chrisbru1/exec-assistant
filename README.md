# exec-assistant

An AI executive assistant built on [Claude Code](https://claude.ai/code) scheduled triggers + MCP servers. No custom backend. No infrastructure to maintain. Just a git repo, markdown files, and three cron jobs.

## What it does

Three scheduled agents run on weekdays:

**Morning Briefing (6:57am)** — Pulls your Google Calendar, scans priority Slack channels, and reads Gmail. Delivers a scannable Slack DM with your day's schedule, what needs attention, due commitments, and overnight signal. Read-only — doesn't modify anything.

**End-of-Day Capture (5:03pm)** — Scans the day's meetings, emails, and Slack conversations. Extracts decisions, commitments, and project updates. Updates the persistent context store and commits to git. Sends a summary of what it captured so you can confirm or correct.

**Weekly Narrative (Friday 3:57pm)** — Synthesizes the week into executive-level outputs: a week-in-review, a CEO update draft (written in your voice), and board narrative building blocks. The kind of writing you'd normally spend Friday afternoon doing from scratch — now you're editing a strong draft instead.

## Why this architecture

The insight: **you don't need a custom backend if Claude Code can already talk to your tools via MCP.**

Claude Code's first-party MCP servers provide direct access to Slack, Google Calendar, and Gmail. Scheduled triggers (RemoteTriggers) run Claude Code sessions on a cron. The "database" is markdown files in a git repo — human-readable, version-controlled, and editable by both you and the agent.

| | Custom Agent | This Approach |
|---|---|---|
| **Time to v1** | Weeks | Hours |
| **Maintenance** | Ongoing | Near zero |
| **Infrastructure** | Server + DB + auth | Git repo |
| **Cost** | Hosting + API | API usage only |

If you outgrow this, the context store and prompts carry directly into a custom agent — nothing is wasted.

## Architecture

```
exec-assistant/
├── CLAUDE.md                    # System prompt (loaded every run)
├── context/                     # [gitignored] Your personal context
│   ├── brubaker.md              # Who you are, your role, communication style
│   ├── postscript.md            # Company context
│   ├── org_chart.md             # Key people and relationships
│   ├── channel_config.yaml      # Slack channel priorities
│   └── writing_samples.md       # Examples of your writing voice
├── state/                       # [gitignored] Persistent memory
│   ├── projects.md              # Active project tracker
│   ├── commitments.md           # Open commitments (who, what, when)
│   ├── relationships.md         # Stakeholder map
│   ├── narrative.md             # Accumulated themes, wins, blockers
│   └── meetings/
│       └── _index.md            # Rolling 2-week meeting log
├── output/                      # [gitignored] Generated artifacts
│   ├── briefings/               # Morning briefing archives
│   └── weeklies/                # Weekly narrative archives
├── prompts/                     # Prompt templates for each trigger
│   ├── morning_briefing.md
│   ├── eod_capture.md
│   └── weekly_narrative.md
└── examples/                    # Example templates (safe to share)
    ├── channel_config.example.yaml
    ├── projects.example.md
    └── commitments.example.md
```

The `context/`, `state/`, and `output/` directories are gitignored — they contain your actual data. The `examples/` directory shows the structure so you can set up your own.

## The context store

The key design decision: **markdown files as the persistent "memory" across agent runs.**

Each scheduled run reads the current state, does its work (pull calendar, scan Slack, etc.), and writes updates back. The EOD agent commits changes to git, so every state change is versioned and auditable.

**Why markdown over a database:**
- Human-readable and human-editable (you can fix mistakes directly)
- Version-controlled (git blame tells you exactly when something changed)
- No schema migrations, no ORM, no connection strings
- Claude is exceptionally good at reading and surgically updating markdown
- Works as both the "database" and the "UI" — open it in any editor

## Channel priority system

With 50+ Slack channels, the agent needs to know what matters. The `channel_config.yaml` defines four tiers:

- **P0 (always read):** Leadership, finance, exec channels — 5-10 channels max
- **P1 (daily, decisions only):** Cross-functional channels — scan for decisions, asks, escalations
- **P2 (weekly scan):** Broader channels — only surface if you're mentioned
- **Ignore:** Social channels

Plus: an `always_surface_from` list (your CEO, direct reports, key stakeholders) and `escalation_keywords` that elevate any message regardless of channel.

## Commitment tracking

The EOD agent extracts commitments aggressively from Slack and email:

- "I'll send that over" → commitment
- "Let me circle back" → commitment
- "Can you review by Friday?" → commitment (others → you)

Uncertain extractions are marked with `[?]` so you can confirm or dismiss in the nightly summary. Over time, the prompts get tuned based on your signal.

## Voice matching

The weekly narrative agent writes in your voice, not generic AI. It reads:
- Your communication style doc (`context/brubaker.md`)
- Real writing samples (`context/writing_samples.md`)
- Accumulated narrative context (`state/narrative.md`)

The more writing samples you provide, the better the voice match. First few weeks, you'll redline the drafts. After that, you're mostly editing, not rewriting.

## Setup

### Prerequisites
- [Claude Code](https://claude.ai/code) with a Team or Pro plan
- MCP servers: Slack, Google Calendar, Gmail (first-party Anthropic connectors)
- A git repo (this one)

### Bootstrap (interactive, ~1.5 hours)

1. **Clone and create your private files:**
   ```bash
   cp examples/channel_config.example.yaml context/channel_config.yaml
   # Create context/your_name.md with your role, team, style
   # Create context/company.md with company context
   ```

2. **Discover Slack channels:**
   Run Claude Code in this directory and ask it to list your channels. Assign P0/P1/P2/Ignore priorities. It writes `channel_config.yaml` with real IDs.

3. **Seed the context store:**
   Ask Claude Code to scan your last 2 weeks of calendar, email, and Slack. It drafts `state/projects.md` and `state/commitments.md`. You review and correct.

4. **Test run:**
   Run the morning briefing prompt manually. Review output. Tune.

5. **Activate triggers:**
   ```
   /schedule create morning-briefing --cron "57 6 * * 1-5" --prompt prompts/morning_briefing.md
   /schedule create eod-capture --cron "3 17 * * 1-5" --prompt prompts/eod_capture.md
   /schedule create weekly-narrative --cron "57 15 * * 5" --prompt prompts/weekly_narrative.md
   ```

### Expected ramp

- **Week 1:** Noisy. Over-extracts some commitments, misses others. You correct directly in the markdown files.
- **Week 2:** Context is rich enough for useful briefings. First weekly narrative.
- **Week 3+:** Net positive. You're editing strong drafts, not building from scratch.

## Key design decisions

**Read-only morning, read-write evening.** The morning briefing never modifies state files — it just reads and delivers. The EOD capture is the only agent that writes. This prevents conflicts and makes debugging easy.

**Aggressive extraction with [?] markers.** Better to capture a false positive and let you dismiss it than to miss a real commitment. The `[?]` convention makes it easy to scan what the agent was uncertain about.

**Git as audit trail.** Every EOD and weekly run commits to git. If the agent writes something wrong, `git log` and `git diff` show exactly what changed and when.

**Channel priority over channel count.** Reading 50+ channels per run would blow through context windows and rate limits. The tiered priority system means P0 channels get deep reads, P1 gets keyword scans, and P2 is weekly only.

## Who this is for

This was built for a finance executive at a ~230-person tech company, but the architecture works for anyone who:
- Drowns in Slack, email, and calendar
- Needs to track commitments across many conversations
- Writes executive updates, board narratives, or team summaries regularly
- Wants an AI that learns their context over time, not one that starts fresh every conversation

The prompts and context structure are designed for a senior leader, but you can adapt them for any role where information overload is the bottleneck.

## Limitations

- **Not real-time.** Runs on a schedule, not event-driven. If you need instant alerts, this isn't it.
- **Can't see private conversations.** Phone calls, in-person meetings, and DMs to other people won't be captured. The EOD summary asks "anything I missed?" to fill gaps.
- **Context window limits.** 50+ P0 channels would overwhelm a single run. The priority system is the defense.
- **Voice matching takes iteration.** The first few weekly narratives will need heavy edits. It gets better as you add writing samples and the agent accumulates your actual context.

## License

MIT
