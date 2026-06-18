# skill-search

**Semantic, on-demand skill retrieval for Claude Code.** Claude Code injects a
short blurb for every installed skill into context on *every* turn so it can
decide which to use. As your skill count grows, that listing becomes a large
recurring token tax — and because the match is essentially name/description
keyword overlap, a skill whose name doesn't echo the user's words quietly never
fires.

skill-search replaces that with a vector retriever over the **full** skill
descriptions. Skills are set to `name-only` (name stays visible and invocable,
the description leaves the budget), and an MCP tool returns just the few skills
that semantically match the task at hand.

> **Where it shines:** this pays off once you have a lot of skills installed —
> roughly hundreds. With only a handful, the native listing is already cheap and
> you don't need the extra round-trip.

---

## Proof of value

All numbers below are **measured**, not estimated by vibes — on a real setup of
**117 active skills**. You can reproduce them (see [Reproduce](#reproduce-the-numbers)).

### 1. It reclaims a measurable chunk of every turn

The native skill listing injects name + description for all skills, every turn.
At `name-only`, only the names remain and the retriever supplies descriptions
on demand.

| | Tokens injected per turn | % of a 200K window |
|---|---:|---:|
| Native full listing (name + description) | ~7,487 | 3.74% |
| `name-only` + skill-search | ~617 | 0.31% |
| **Saved, every turn** | **~6,870** | **~3.4%** |

That's ~59 tokens/skill of description you stop paying for on turns that don't
need them. The worst offenders are generic-named skills whose value lives
entirely in the description (`context-mode:context-mode` alone is ~264 tokens) —
exactly the skills `name-only` + retrieval handles best.

### 2. It fixes the name-bias miss (real, unedited output)

`search_skills` ranks by meaning, so skills match even when they share no words
with the query:

```
$ query: "debug a failing test"
   0.672  gsd-debug
   0.659  superpowers:systematic-debugging
   0.658  gsd-forensics

$ query: "review my UI for accessibility"
   0.689  chrome-devtools-mcp:a11y-debugging     ← "a11y" never appears in the query
   0.667  frontend-design:frontend-design
   0.653  web-design-guidelines                  ← no shared keywords at all

$ query: "set up a supabase database with auth"
   0.717  supabase:supabase
   0.575  supabase:supabase-postgres-best-practices
   0.542  superpowers:executing-plans
```

### 3. It stays fast as the index grows

Reindex is incremental: each point stores a content hash, so only changed skills
re-embed and deleted skills are dropped.

| Operation (117 skills) | Time |
|---|---:|
| Full rebuild (`--rebuild`) | ~18.8s¹ |
| Incremental reindex, nothing changed | **~0.07s** |

¹ includes the one-time embedding-model download on first run.

### 4. It fails loudly, not silently

Because the retriever becomes the *only* discovery path, a stale or broken index
would otherwise hide skills with no symptom. Guards:

- `search_skills` appends a `warning` when skills changed on disk since the last index.
- `health` reports embedder/store reachability and lists any **dark** (on-disk
  but unindexed) or **stale** (indexed but deleted) skills, and exits non-zero
  when degraded (cron/CI-safe).

---

## How it works (two pieces, useless apart)

1. **`generate_overrides.py`** → sets ~all skills to `name-only` in
   `.claude/settings.local.json` → frees the budget. A tiny allowlist
   (the router skill) stays fully `"on"`.
2. **`server.py`** (MCP) → embeds full descriptions into a vector store; returns
   the top-k relevant skills → Claude invokes them by name (works at `name-only`).

Skip step 1 and you pay the native tax **and** the retriever. Do both.

| File | Role |
|---|---|
| `server.py` | MCP server: `search_skills`, `get_skill`, `reindex`, `health` |
| `skills_discovery.py` | Shared skill discovery — one source of truth for both halves |
| `generate_overrides.py` | Frees the budget by setting skills to `name-only` |

---

## Install

The **default tier is service-free** — embedded on-disk Qdrant + local ONNX
embeddings ([fastembed](https://github.com/qdrant/fastembed)). No Docker, no
Ollama, no manual model pull (the model downloads once, then runs offline).

```bash
pipx install .          # isolated install
# or run without installing:
uvx --from . skill-search --health

# 1. build the index once (incremental afterwards; --force for a full rebuild)
skill-search --reindex

# 2. free the budget (project scope; --global targets ~/.claude)
skill-search-overrides

# 3. register the MCP with Claude Code (no-arg console script = stdio server)
claude mcp add --transport stdio skill-search -- skill-search

# 4. (optional) confirm inside a Claude Code session
#    /mcp     and     /doctor
```

**Opt into the faster tier** (only if you already run them):

```bash
docker run -p 6333:6333 qdrant/qdrant
export SKILL_QDRANT_URL=http://localhost:6333          # Qdrant server
ollama pull embeddinggemma
export SKILL_EMBED_BACKEND=ollama                      # Ollama embeddings
```

Pin these as `--env` flags on `claude mcp add` to keep them for the registered server.

---

## The router skill (keep this one `"on"`)

Save to `~/.claude/skills/skill-search/SKILL.md`. It's the always-visible entry
point that tells Claude to retrieve before guessing.

```markdown
---
name: skill-search
description: Find the right skills for a task before acting. Use at the start of any multi-step or unfamiliar request to retrieve relevant skills by meaning, not name. Triggers when the user asks to build, set up, design, deploy, fix, or automate something and the right skill isn't obvious.
---

Before tackling this task, call the `search_skills` MCP tool with a short query
describing the user's goal. It returns ranked skills by semantic relevance.

Then:
1. Read the returned names + descriptions.
2. Invoke the genuinely relevant ones by name (e.g. /frontend-design).
3. Ignore low-score results — do not load skills that aren't relevant.
4. If a result looks promising but the description is thin, call `get_skill`
   on it before deciding.

Prefer 2-4 high-relevance skills over loading many. Precision keeps context lean.
```

---

## Configuration

All config is env-var overridable (`SKILL_*` prefix). Selection: set
`SKILL_QDRANT_URL` for a Qdrant server (else embedded); `SKILL_EMBED_BACKEND`
defaults to `fastembed`.

| Concern | Default (service-free) | Opt-in (faster) |
|---|---|---|
| Vector store | embedded on-disk Qdrant at `~/.cache/skill-search/qdrant` (`SKILL_QDRANT_PATH`) | `SKILL_QDRANT_URL` → Qdrant server |
| Embedder | fastembed `BAAI/bge-small-en-v1.5` (384-dim) | `SKILL_EMBED_BACKEND=ollama`, `SKILL_EMBED_MODEL` (`embeddinggemma`, 768-dim) |
| Results | `SKILL_TOP_K=6` | — |

Switching embedders changes the vector dimension; an existing collection can't
take it. This is guarded both ways — `reindex` raises a clear "run `--rebuild`"
error, and `health` flags the mismatch.

---

## Tests

```bash
pip install -e ".[dev]"
pytest -m "not integration"     # fast, offline (no network/model) — 13 tests
pytest -m integration           # end-to-end: real embed → search → incremental skip
```

Unit tests pin the highest-risk logic: skill discovery (parsing, plugin
namespacing, dedup precedence — the shared source of truth both halves depend on),
point-ID validity, content-hash determinism, and the staleness/manifest guards.
The `integration` marker gates the one test that loads the embedder.

> If a broken third-party pytest plugin in your env fails collection, run with
> `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1`.

---

## Reproduce the numbers

```bash
# token savings on YOUR skill set (chars/3.7 proxy; descriptions-only, so a floor)
python3 -c "
import skills_discovery as d
s = d.discover_skills()
tok = lambda x: round(len(x)/3.7)
desc = sum(tok(x['description']) for x in s)
print(f'{len(s)} skills | ~{desc} tokens saved/turn | {desc/200000*100:.2f}% of 200K')
"

# semantic ranking + incremental-reindex timing
skill-search --reindex          # full build, then run again to see the incremental skip
skill-search --health           # indexed vs on-disk, dark/stale skills, dims
```

---

## Caveats

- **Retriever is the sole discovery path.** At `name-only`, Claude can't
  auto-match on description. If `search_skills` misses, the skill goes dark.
  Tune `SKILL_TOP_K` up if recall feels low; keep critical skills `"on"`.
- **Re-index on change.** New/edited skills aren't searchable until `reindex`
  runs — but it's incremental and cheap, and drift is surfaced by `search_skills`
  warnings + `health`, so the failure mode is visible, not silent.
- **Embedded Qdrant locks its dir to one process.** Don't run a CLI `--reindex`
  while the MCP server is up in that mode — use the `reindex()` tool, or the
  Qdrant-server tier.
- **Tail-scale.** The payoff scales with how many skills you have installed —
  worth it at hundreds, overkill at a handful.

## License

MIT — see [LICENSE](LICENSE).
