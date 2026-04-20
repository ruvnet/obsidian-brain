# obsidian-brain

[![Release](https://img.shields.io/github/v/release/ruvnet/obsidian-brain?label=release)](https://github.com/ruvnet/obsidian-brain/releases)
[![Tests](https://img.shields.io/github/actions/workflow/status/ruvnet/obsidian-brain/test.yml?label=tests)](https://github.com/ruvnet/obsidian-brain/actions)
[![Obsidian ≥ 1.4.0](https://img.shields.io/badge/Obsidian-%E2%89%A5%201.4.0-7c3aed)](https://obsidian.md)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![BRAT](https://img.shields.io/badge/install-via%20BRAT-ff7a59)](https://github.com/TfTHacker/obsidian42-brat)

> **Give your vault a memory that actually understands what you wrote.**
> Type a question and get real answers grounded in your own notes.
> Ask your AI assistants questions and they'll draw from the same notes
> you just wrote. Share selectively with a community knowledge base.

---

## What this plugin does, in plain English

You write notes in Obsidian. This plugin does three things with them:

1. **It understands them.** Press <kbd>⌘⇧B</kbd> and type what you're
   looking for. Instead of matching keywords, it finds notes that mean
   something similar to your question — even if you used different
   words. Press <kbd>⌘⇧K</kbd> instead and you'll get a Q&A view that
   pulls together the relevant passages across your whole vault.

2. **It shares them with your AI tools.** Claude Code, Codex CLI,
   Gemini CLI and other MCP-speaking assistants can read (and write)
   the same notes you see in Obsidian. No more re-explaining context
   at the start of every session — your agent already knows what you
   decided last week.

3. **It connects you to a bigger brain.** Optionally pull research,
   patterns, and solutions from the shared [pi.ruv.io](https://pi.ruv.io)
   knowledge base — or publish your own notes back to it when you want
   to contribute. Everything is scanned for prompt-injection and
   personal data before it's stored.

It does *not* replace your vault. Your notes stay as Markdown files
you own. The plugin adds a fast, semantic *index* on top — one that
your other tools can share.

---

## Five-minute setup

### 1. Install the plugin (via BRAT — easy beta install)

1. Inside Obsidian, install the [BRAT](https://github.com/TfTHacker/obsidian42-brat)
   community plugin.
2. Open the command palette, run `BRAT: Add a beta plugin for testing`,
   and paste `ruvnet/obsidian-brain`.
3. Go to *Settings → Community plugins → Installed* and enable
   **RuVector Brain**.

(Alternatively, download `main.js`, `manifest.json`, and `styles.css`
from the [latest release](https://github.com/ruvnet/obsidian-brain/releases/latest)
and drop them into `<your-vault>/.obsidian/plugins/obsidian-brain/`.)

### 2. Start the brain and embedder

The plugin is intentionally thin — it talks to two small background
services that do the heavy lifting. One call gets you both:

```bash
git clone https://github.com/ruvnet/RuVector
cd RuVector
cargo build --release -p mcp-brain-server --features local --bin mcp-brain-server-local
./target/release/mcp-brain-server-local &          # the "brain"
ruvultra-embedder &                                 # the embedder
```

Both listen only on your own machine (`127.0.0.1`). Nothing talks to
the internet unless you explicitly turn on the pi.ruv.io integration.

Prefer a managed setup? Copy `systemd/*.service` into
`~/.config/systemd/user/` and run `systemctl --user enable --now
ruvector-brain ruvector-embedder`.

### 3. Point the plugin at them

*Settings → RuVector Brain → Test connection.* If it says "Brain OK"
and "Embedder OK", you're done. Open any note, then press
<kbd>⌘⇧B</kbd> and start typing — results update as you type.

### 4. Index what you already have

*Command palette → `Brain: Bulk-sync vault → brain`.* Every note is
scanned for prompt injection, embedded, and stored. Unchanged notes
are skipped on re-runs.

---

## What you can do with it

| Do this | With this |
| --- | --- |
| **Find notes by meaning, not keyword.** | <kbd>⌘⇧B</kbd> — type a question, results rank by semantic similarity. Add `category:project` to narrow by tag. |
| **Ask your vault a question.** | <kbd>⌘⇧K</kbd> — Q&A modal stitches together relevant passages from your notes *and* the shared pi.ruv.io brain (if you opt in). Insert or copy the bits you want. |
| **See related notes while you write.** | Right sidebar, brain icon. Updates every time you switch notes. Use ↑/↓ or j/k to navigate, Enter to open. |
| **Auto-save to the brain as you work.** | *Settings → Auto-index on save.* Debounced so it doesn't thrash. Fails *closed* — if the brain isn't reachable, the note stays in a queue and gets indexed later. Nothing is ever silently discarded. |
| **Colour your graph by category.** | *Command palette → `Graph overlay: apply category colors`.* Writes reversible colour groups to `.obsidian/graph.json`. Clear them anytime. |
| **Collect training data for a model.** | DPO preference pairs: mark a preferred note, then mark a weaker alternative. Export them as a Markdown table under `Brain/Exports/`. |
| **Pull research from pi.ruv.io.** | *Command palette → `pi.ruv.io: pull memories into local brain`.* 12,000+ shared memories, optionally filtered by category or search query. Each one becomes a Markdown stub under `Brain/Pi/`. |
| **Publish a note to pi.ruv.io.** | *Command palette → `pi.ruv.io: publish current note`.* The server scans for injection and PII, adds differential-privacy noise to the embedding, and returns a witness hash. Takes ~20 seconds. |
| **Remember what you wrote last year.** | *Daily recall* command generates a note summarizing everything you wrote on this day in prior years. |
| **See what the brain is doing.** | *Brain Ops* modal shows CPU/GPU load, training stats, storage size, and lets you checkpoint the database or export DPO pairs. |

---

## How it fits together

```
                   ┌──────────────────────────────────────┐
                   │           Obsidian vault             │
                   │  ┌─────────────────────────────┐     │
                   │  │  obsidian-brain plugin       │     │   you
                   │  └─────────────┬───────────────┘     │
                   └────────────────┼─────────────────────┘
                                    │  local HTTP
                                    ▼
                   ┌──────────────────────────────────────┐
                   │   RuVector brain (on your machine)    │
                   │  • Vector search (DiskANN)            │
                   │  • Prompt-injection + PII scanning    │
                   │  • SQLite store + MCP server          │◄── your
                   └────────────────┬─────────────────────┘    AI agents
                                    │  (optional, opt-in)
                                    ▼
                   ┌──────────────────────────────────────┐
                   │       pi.ruv.io shared brain          │
                   │  12,000+ memories across contributors │
                   └──────────────────────────────────────┘
```

The plugin is one client of many. Your editor, your command line, your
AI coding assistant — all talk to the same brain. Index a note in
Obsidian and Claude Code can cite it in the next message.

---

## Sharing the brain with your AI agents

The brain already speaks **MCP** (Model Context Protocol). Point any
MCP-capable assistant at it and they see the memories you indexed.

**Claude Code** — `~/Library/Application Support/Claude/claude_desktop_config.json`:

```jsonc
{
  "mcpServers": {
    "ruvector-brain": {
      "command": "http",
      "args": ["http://127.0.0.1:9876"]
    }
  }
}
```

**Codex CLI** — `.codex/mcp.json`:

```jsonc
{ "servers": [{ "name": "ruvector-brain", "endpoint": "http://127.0.0.1:9876" }] }
```

**Gemini CLI** — `.gemini/settings.json`:

```jsonc
{ "mcp": { "ruvector-brain": { "url": "http://127.0.0.1:9876" } } }
```

*Settings → RuVector Brain → Copy MCP endpoint* drops the right URL
onto your clipboard.

---

## Safety and privacy

- **Nothing leaves your machine by default.** Both the brain and the
  embedder bind to localhost. The shipped systemd units use
  `IPAddressDeny=any` + `IPAddressAllow=127.0.0.0/8`.
- **Prompt-injection and PII screening run before every write** —
  and you can't turn them off from the plugin side. If the brain isn't
  reachable, the plugin fails *closed*: your note is queued locally
  but not indexed, so nothing is ever silently bypassed.
- **pi.ruv.io is entirely opt-in.** It requires a bearer token you
  paste into settings; without it, the pull/publish commands simply
  say "not configured."
- **No telemetry, no third-party calls.** The plugin talks to
  localhost and (if you turn it on) pi.ruv.io over authenticated
  HTTPS.
- **A warning about Obsidian Sync.** Your pi.ruv.io token is stored
  in `.obsidian/plugins/obsidian-brain/data.json`. If you use
  Obsidian Sync across machines, that file travels with your vault.

---

## Settings cheat sheet

| Setting | Default | What it does |
| --- | --- | --- |
| Brain URL / Embedder URL | `127.0.0.1:9876` / `:9877` | Loopback addresses of the two services |
| Default category | `obsidian` | Used when a note has no `brain-category` frontmatter |
| Auto-index on save | off | Debounced save-to-brain; fails closed when offline |
| Index minimum characters | 40 | Skips very short notes |
| AIDefence scan | on | Keep on unless you really know what you're doing |
| Search / Related limit | 8 | Top-k results shown |
| Bulk-sync include / exclude | *(whole vault)* / `.obsidian,.trash` | Folder filters, comma-separated |
| DPO default direction | `quality` | Label stored on every preference pair |
| pi URL / token | `https://pi.ruv.io` / *(empty)* | Bearer token needed for memory endpoints |
| pi pull limit / query / category | 20 / *(empty)* / *(empty)* | Controls what gets pulled |

---

## Development & testing

```bash
npm install
npm run dev        # rebuild on change — reload with "Reload plugin" in Obsidian
npm run build      # full typecheck + production bundle
npm test           # real-service protocol tests (no mocks)
```

Tests spin up a real `mcp-brain-server-local` subprocess and assert
the exact response shapes the plugin parses. No mocking, no drift
between what tests check and what production does.

| Suite | How to run | Checks |
| --- | --- | --- |
| Local brain protocol | `npm test` | 9 — every endpoint the plugin uses |
| pi.ruv.io protocol | `BRAIN_API_KEY=… npm test` | 6 — including live write-through |
| Real-Obsidian end-to-end | `OBSIDIAN_E2E=1 npm test` | 11 — downloads the real Obsidian AppImage, launches under xvfb, runs a harness plugin inside the real app |

Current baseline: **15 protocol tests + 11 real-Obsidian harness checks,
all passing.**

---

## Troubleshooting

**Status bar says "Brain: offline."** Your `mcp-brain-server-local`
isn't answering. Run `curl http://127.0.0.1:9876/health`. Writes you
make while offline queue up and retry every 30 seconds.

**Search results look semantically random.** You're probably on the
fallback 16-dim stub embedder. Start `ruvultra-embedder` on `:9877`
*before* the brain, then wipe `.brain-data/` — a vector index can't
mix dimensions.

**Clicking a search result just shows a preview.** The plugin doesn't
know which vault file that memory came from. Run `Bulk-sync vault →
brain` once — idempotent and hash-deduped.

**"Blocked by AIDefence" on a note you trust.** It matched one of the
injection patterns (e.g. phrases like *"ignore all previous
instructions"*). Rephrase the content, or toggle AIDefence off in
settings if you accept the tradeoff.

**pi.ruv.io publish takes ~20 seconds.** That's expected — the server
runs AIDefence, differential-privacy Gaussian noising, and RVF
segmentation on ingest.

---

## How this compares

| | obsidian-brain | [obsidian-mind](https://github.com/breferrari/obsidian-mind) | Smart Connections |
| --- | --- | --- | --- |
| Primary audience | Humans + AI agents | AI coding agents | Humans |
| Embeddings | Real bge-small-en-v1.5 (GPU) | Optional, light | In-browser Transformers.js |
| Storage engine | External Rust brain, DiskANN | QMD SQLite | JSON in vault |
| Prompt-injection / PII screen | ✅ server-side, non-bypassable | grep patterns | ❌ |
| Shared knowledge base | ✅ pi.ruv.io pull + publish | ❌ | ❌ |
| Training-data export | ✅ DPO pairs | ❌ | ❌ |
| Graph view overlays | ✅ reversible colour groups | ❌ | ❌ |
| Agent access | ✅ MCP | ✅ MCP | ❌ |
| Tests against real Obsidian | ✅ xvfb harness, 11 checks | ? | ? |

obsidian-mind is more mature and has a richer "agent as co-pilot"
story (session hooks, specialized subagents). obsidian-brain is
stronger on embedding quality, federated knowledge, and security
posture. They can coexist on the same machine pointing at different
vaults.

---

## What's next

See [ADR-152 §Phase-5](https://github.com/ruvnet/RuVector/blob/main/docs/adr/ADR-152-obsidian-brain-plugin.md#phase-5--roadmap).
Tracked:

- Jump to the matching passage inside a note on result click
- Suggest `[[wikilinks]]` from semantically-related notes
- A memory explorer pane with category / date filters
- Diversity reranking for search
- Multi-brain federation (local + pi + team)
- Obsidian Canvas integration

Issues, feedback, and PRs are very welcome at
[github.com/ruvnet/obsidian-brain/issues](https://github.com/ruvnet/obsidian-brain/issues).

---

## Credits

Built on top of [**RuVector**](https://github.com/ruvnet/RuVector) —
the brain, the embedder, and the AIDefence regex set. The federated
[pi.ruv.io](https://pi.ruv.io) brain provides shared research memory.
Inspired in parts by [**obsidian-mind**](https://github.com/breferrari/obsidian-mind)'s
agent-first memory vision.

## License

[MIT](./LICENSE). Use it, fork it, ship it.
