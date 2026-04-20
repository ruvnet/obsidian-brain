# obsidian-brain

Obsidian plugin that bridges your vault with the **RuVector Brain**.
Semantic search (DiskANN, 384-dim `bge-small-en-v1.5`), AIDefence
scanning before every index, live Related side-panel, DPO preference
pairs, graph colour overlay, **pi.ruv.io** federated shared-brain
integration (pull + publish), Q&A modal, offline queue. No mocks in
tests — 15 protocol tests + 11 real-Obsidian harness checks pass.

The same memory store is accessible to AI coding agents (Claude Code,
Codex CLI, Gemini CLI) via `mcp-brain-server` / `ruvbrain-sse` — the
plugin is the human UI, the MCP server is the agent UI, both read/write
the same AIDefence-gated store.

Implements [ADR-152 / ADR-SYS-0025](https://github.com/ruvnet/RuVector/blob/main/docs/adr/ADR-152-obsidian-brain-plugin.md).

## Install

### Option A — BRAT (community beta plugins)

1. Install the [BRAT](https://github.com/TfTHacker/obsidian42-brat)
   community plugin.
2. `BRAT → Add beta plugin` → `ruvnet/obsidian-brain` (this repo).
3. Enable **RuVector Brain** under *Community plugins → Installed*.

### Option B — manual

Download the three assets from the latest release
(`main.js`, `manifest.json`, `styles.css`) and drop them into
`<your-vault>/.obsidian/plugins/obsidian-brain/`. Reload Obsidian.

### Option C — source

```bash
git clone https://github.com/ruvnet/obsidian-brain
cd obsidian-brain
npm install
npm run build
./scripts/setup.sh /path/to/your-vault
```

## Prerequisites

The plugin is a **client** — it expects the RuVector brain + embedder
to be running on loopback:

| Service | Default | What it does |
| --- | --- | --- |
| `mcp-brain-server-local` | `127.0.0.1:9876` | DiskANN index, AIDefence, SQLite store, MCP server |
| `ruvultra-embedder` | `127.0.0.1:9877` | 384-dim bge-small-en-v1.5 embeddings (cuda) |

Build from the RuVector workspace:

```bash
git clone https://github.com/ruvnet/RuVector
cd RuVector
cargo build --release -p mcp-brain-server --features local --bin mcp-brain-server-local
```

systemd user units (loopback-only, `IPAddressDeny=any`) are under
[`systemd/`](./systemd).

## Commands

| Command | Hotkey | What it does |
| --- | --- | --- |
| Semantic search | `Cmd+Shift+B` | DiskANN search; `category:<x>` prefix filters; fuzzy fallback when brain offline |
| Semantic search on current selection | — | Seeded with the current editor selection |
| Ask the brain (Q&A modal) | `Cmd+Shift+K` | Retrieval-grounded — blends local + pi top-k, renders markdown cards |
| Index current note | — | Force-reindex the active note (AIDefence-scanned, hash-deduped) |
| Find related memories for current note | — | Re-fires the Related side-panel |
| Toggle related panel | — | Shows/reveals the right-sidebar Related view |
| Bulk-sync vault → brain | — | Progress modal, include/exclude filters |
| DPO: mark current note as chosen | — | First step of preference-pair workflow |
| DPO: create pair with current note (rejected) | — | Second step; prompts for direction label |
| DPO: status / clear / export | — | Export pairs to `Brain/Exports/preference-pairs.md` |
| Graph overlay: apply category colors | — | Writes `tag:#brain/<category>` color groups to `.obsidian/graph.json` |
| Graph overlay: clear category colors | — | Removes only the `#brain/*` groups |
| Brain ops (workload, training, export, checkpoint) | — | Read-only dashboard + DPO export + WAL checkpoint |
| pi.ruv.io: pull memories into local brain | — | Mirrors pi memories into `Brain/Pi/<title>.md` |
| pi.ruv.io: search shared brain directly | — | Queries pi's `/v1/memories/search` |
| pi.ruv.io: publish current note | — | POSTs to pi's `/v1/memories` (AIDefence on the server, ~20s) |
| pi.ruv.io: status | — | Global pi stats |
| Daily recall — memories from this day | — | Generates `Brain/Recall/Recall-YYYY-MM-DD.md` |
| Offline queue: retry pending now | — | Manual drain; queue auto-retries every 30s |
| Brain info / health | — | Health + version + engine mode |

## Settings

Open **Settings → RuVector Brain**. Highlights:

- **Brain URL / Embedder URL** — loopback endpoints
- **Auto-index on save** — debounced; fails *closed* when brain offline
- **AIDefence scan before indexing** — on by default
- **Bulk-sync include/exclude folders**
- **pi.ruv.io** — URL, bearer token, pull limit, pull query, pull category
- **Agent access (MCP)** — "Copy MCP endpoint" button for
  `claude_desktop_config.json` / `.codex/mcp.json` / `.gemini/settings.json`
- **DPO** — default direction label

## Live dev session

`./scripts/run-dev.sh` boots a scratch vault, the brain, the embedder
(prefers real `ruvultra-embedder`, falls back to a 16-dim stub), seeds
demo notes, optionally pulls pi.ruv.io memories (when `BRAIN_API_KEY`
is set), writes `.obsidian/graph.json` colour groups, and launches the
real Obsidian AppImage under an isolated HOME.

```bash
./scripts/run-dev.sh                           # offline
PI_LIMIT=30 ./scripts/run-dev.sh               # plus 30 pi memories
PI_QUERY="hnsw diskann" ./scripts/run-dev.sh   # pull by semantic query
```

## Tests

No mocks. `npm test` runs:

1. **Protocol** — spins up a real `mcp-brain-server-local` subprocess,
   validates every endpoint shape the plugin parses. 9 tests.
2. **pi.ruv.io protocol** — gated on `BRAIN_API_KEY`. Asserts pi's
   response shapes incl. write-through. 6 tests.
3. **Real Obsidian E2E** — gated on `OBSIDIAN_E2E=1`. Downloads the
   real Obsidian AppImage, extracts it, launches under `xvfb-run` with
   a companion harness plugin that exercises 11 checks inside the real
   Obsidian runtime.

```bash
npm test                                      # protocol only
BRAIN_API_KEY=… npm test                      # + live pi
OBSIDIAN_E2E=1 BRAIN_API_KEY=… npm test       # full, requires xvfb+libfuse2
```

## Security

- AIDefence regex screens content **server-side** before it's stored.
  When the brain is unreachable, auto-index fails *closed*.
- Bearer tokens for pi.ruv.io live in the vault's `.obsidian/plugins/
  obsidian-brain/data.json`. Don't sync that file across devices
  via Obsidian Sync unless you're OK with the token travelling with it.
- Brain + embedder bind loopback only; shipped systemd units set
  `IPAddressDeny=any` + `IPAddressAllow=127.0.0.0/8`.

## License

MIT.
