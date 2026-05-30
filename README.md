```
██╗  ██╗██╗██╗      ██████╗
██║ ██╔╝██║██║     ██╔═══██╗
█████╔╝ ██║██║     ██║   ██║
██╔═██╗ ██║██║     ██║   ██║
██║  ██╗██║███████╗╚██████╔╝
╚═╝  ╚═╝╚═╝╚══════╝ ╚═════╝
```

# Kilo

A persistent, on-demand personal AI agent — **Codex as the engine, Telegram as the interface, a git-backed memory layer as the spine.** It runs unattended on a Linux VM, does real work, remembers across sessions, and improves over time — without behaving like an amnesiac or a runaway.

This repo is the architecture writeup. The runtime lives in a private repo; everything here is the design and the lessons.

## The shape

Every trigger only **enqueues**. A single `flock`'d dispatcher runs exactly one `codex exec` at a time — the load-bearing guarantee that two agent turns never corrupt the same session.

```
 triggers ──────────────┐
   alarm     (daily)     │
   heartbeat (interval)  ├─▶ enqueue ─▶ [ single-runner dispatcher · flock ]
   telegram  (you)       │                       │  one codex exec at a time
   sentinels (silent)    │                       ▼
                         │             codex exec [resume <thread>]
                         │                       │
                         │      ┌────────────────┼─────────────────┐
                         ▼      ▼                ▼                 ▼
                    reply → TG  memory layer   subagents        git commit
                               (git · index)  (delegated work)
```

## Engine — per-task `codex exec`

Not a long-lived daemon: each job spawns a fresh non-interactive `codex exec`, with a JSON-schema-constrained structured result the wrapper persists deterministically (journal entry, working state, one git commit). Stateless and crash-isolated; continuity is layered on top, not baked into a fragile long process.

## Memory — git-backed, index-and-fetch

Durable state is plaintext in a git repo, not the engine's runtime cache (which is experimental and version-coupled). The pattern is a compact always-loaded **index** + read-the-full-file-on-demand:

- `AGENTS.md` — the operating contract, auto-loaded every run.
- `MEMORY.md` — one line per memory; the agent opens a file only when relevant.
- `semantic/` durable facts · `episodic/journal.jsonl` events · `decisions/` ADRs · `STATE.md` working state · `BACKLOG.md` queue.

At a few hundred memories the index costs ~2% of the context window — embeddings/RAG are deferred as an escape hatch, not the backbone.

## Conversation & context lifecycle

The thing that stops it feeling dumb:

- **One rolling session per day.** Each Telegram message `resume`s the same thread, so it remembers the conversation (resolve "that", "the one we discussed" from context — don't take a message literally when the recent chat says otherwise).
- **Delegate heavy work to subagents.** Anything that reads a lot or runs many steps goes to a fresh subagent in its own context; only a tight summary returns. The main thread stays lean.
- **The daily diary consolidates, then rotates.** At day's end the agent writes a retrospective into durable memory and the conversation thread resets. Within a day → conversation memory; across days → the memory layer.

## Triggers

- **alarm** — a daily routine (project health-check) that pushes its summary to you.
- **heartbeat** — backlog drainer + liveness; silent when idle, one nudge per idle streak.
- **sentinels** — e.g. a site/TLS check; stay quiet unless something is actually wrong.
- **telegram** — you. A message is a task.

## Self-extension

Capabilities are exposed uniformly so the agent picks what to call:

- **Skills** (`SKILL.md` procedures, auto-discovered) — playbooks like the daily routine, the diary, and *authoring new jobs*.
- **MCP tools** — `recall` / `remember`, `add_backlog` / `complete_backlog`, `notify_loki`, `delegate`, and `schedule_task` — so it can **write its own skills and schedule its own recurring jobs**, version-controlled, on request.

## Talking to it

A private, single-user Telegram bot:

- **Long polling** — only outbound HTTPS, so it works behind NAT with no inbound port or public URL.
- **Single-user allowlist** — the bot token is not access control; an explicit user-id check is.
- **Markdown → Telegram HTML** so `code`, **bold**, and bullets render, with a plain-text fallback and chunking past the length cap.
- **Live progress streaming** — a single message edits in place to show each command / file edit / tool call as it happens, so you can watch (and catch a mistake) instead of only seeing the final claim.

## Safety

- **Single-runner `flock`** — concurrent-write corruption is structurally impossible.
- **Sleep window** — quiet overnight; no idle runs, no budget burn.
- **Deterministic guards, not prompt-trust** — "ping once per idle streak" and "scheduled reports must reach you" are enforced in the wrapper, because the model won't reliably self-limit.
- **Decisions recorded at the source** — intentional choices live in the project repo and reviews diff against an *accepted baseline*, so "intentional" never reads as a "finding".

## Hard-won lessons

1. **Prompt-based behavior is unreliable.** If an invariant matters (don't spam, always deliver a report, only run one agent), enforce it in code around the model, not in the prompt.
2. **`exec` is stateless about the conversation.** A chat agent needs an explicit rolling thread, or every message is a fresh amnesiac.
3. **Bound context deliberately** — delegate to subagents and rotate the session daily; don't let one thread grow forever.
4. **Record accepted decisions where the thing lives** (the repo), not in one agent's private memory — or other agents, and future-you, never see it.
5. **A journal only improves you if it's re-read and promoted** into the always-loaded contract/skills. Written-and-forgotten is just logging.
6. **Make the agent observable before making it more autonomous.** Visibility + a hard gate beat a smarter prompt.

## Stack

OpenAI Codex CLI (gpt-5.5) · Python (`python-telegram-bot`, `mcp`) · systemd timers · git. Runtime in a private repo.

## License

[MIT](LICENSE.md) — take it, build your own Kilo, just don't let it `using (true)` your prod.
