---
name: Datadog Docs Knowledge Agent — Build Plan
description: "Build plan and resume point for the Datadog Docs Knowledge Agent — citation-backed Q&A over all public Datadog docs. One monolithic corpus, suite→product→feature taxonomy plus suite-less platform topics, query-time site resolution (no 8× corpus copies), tab/selector-aware answers, cross-suite retrieval fan-out, per-citation verification."
id: project:tooling:documentation-agent
tags:
  - type:project
  - topic:tooling
  - topic:apm
  - topic:opentelemetry
  - status:complete
  - privacy:public
links:
  - idea:tooling:apm-the agent
  - index:workspace:efforts
  - knowledge-base:code-repositories:datadog-public-source-code
updated: 2026-06-19T00:55:00Z
---

# Datadog Docs Knowledge Agent — Build Plan

## 0. RESUME HERE (handoff state)

**Status: D0 + P0 + P1 + P2a/b/c + P3 + P4 + P5 + P7 DONE (2026-06-18). Eval 98/98, citation-integrity 100%.** Only **P6 (Slack auto-responder)** remains — deliberately deferred to the very end. The whole-docs corpus + curated taxonomy are committed via LFS (`df2e224`), and `/refresh-docs-agent` rebuilds it in one command. **The predecessor `apm-docs-knowledge-agent` (module + subagent face) was deprecated and deleted 2026-06-19** — this whole-docs agent is now the sole docs agent (APM is a slice via `suite=apm`).

- **P2a rerank done + validated:** gateway-backed `Reranker` (retrieve a wide pool → Claude scores relevance → keep best k; returns `[]` for off-topic → drives a refusal). Wired through service/eval/CLI with a `-rerank` toggle; safe baseline fallback on any gateway/parse error. **Whole-docs eval went 10/12 → 12/12.**
  - Recall fixed: `python` answer surfaces `ddtrace` correctly.
  - Precision fixed: out-of-scope questions rerank to nothing → 0 citations (clean refusal).
  - **Eval maintenance (whole-docs reality, not agent fixes):** `python` assertion `dd-trace-py`→`ddtrace` (repo name vs SDK the answer actually uses); replaced `capital-of-France` out-of-scope case (the LLM-Obs eval docs contain that exact Q as example data, so the agent correctly cited a real doc) with a genuinely-absent question.
- **Hook fix (workspace):** memory-gate matcher `*/agent/*` → `*/agent/*.md` so Go code under the `agent/` package no longer trips the canonical-knowledge gate.

- **Eval rigor — deterministic, two-gate, ~100 cases (P3 expanded):**
  - **No LLM judge at eval time** (deliberate — determinism wins). The LLM only *builds* the suite; assertions are frozen, version-controlled, deterministic string/behavioral checks.
  - **Golden-citation assertions:** `cite_url_contains` = the specific canonical doc path (e.g. `database_monitoring/setup_postgres`), not a generic word. Only discriminating facts kept as `must_contain` (env vars, site endpoints, "not available"); single-word matches dropped. Plus always-on citation-integrity.
  - **Two gates, clean separation of concerns:**
    - `ka eval -retrieval` (OFFLINE, ~7s, no gateway) **owns golden-doc precision** — is the canonical doc retrievable (in the top-N pool)? Fully deterministic, never flakes. The always-run dev-loop gate. **Green: 85/0/13.**
    - `ka eval` (full pipeline) owns **citation-integrity + site facts + refusals** — checks that do NOT depend on *which* valid doc the LLM cites. It deliberately does **not** assert golden-cited (that rotates run-to-run with rerank/generation variance, which proved flaky — different 3 cases failed across two runs). Final-validation gate.
  - Stratified ~100 cases: every suite, platform topics, cross-suite, selector/tab, all 8 sites, refusals.
  - **Reranker hardened:** only returns `[]` (→ refusal) when the question is clearly unrelated to Datadog, preventing spurious empty results on in-scope questions.
  - **Precision findings → P2b:** specific goldens exposed APM-instrument questions top-retrieving AppSec pages, "install Agent" retrieving tracing tutorials, etc. **Taxonomy-filtered retrieval (P2b)** is the fix, now measurable by this eval.

- **Full whole-docs corpus built (live):** all `content/en` → **5,367 files, 29,538 chunks**, embedded in **5m54s** (ollama `nomic-embed-text`, ~71 chunks/s, 13% content-hash dedup). Artifacts: `chunks.jsonl` 34M + `embcache.bin` 76M (committed set ≈ **110M**) + `vectors.bin` 87M (gitignored, regenerable). One corpus vs the APM agent's 8 — the single-corpus win.
- **Eval at scale:** **10/12, citation-integrity 100%** (default `k`, now raised 8→16 for the 29.5k-chunk corpus). All site-resolution cases pass from the one corpus. The 2 residuals are retrieval-at-scale, not architecture:
  - `python/dd-trace-py` — recall: term is in-corpus but its chunk doesn't rank in top-k → **needs rerank (P2)**.
  - `out-of-scope-france` — precision: one loosely-related chunk cited for an off-topic question (still verified) → **needs relevance threshold/rerank (P2)**.
- **Default `k` raised 8→16** across CLI/service/eval (was the cheap recall win: runtime-metrics recovered at k=16).

- **P1 core done + validated (single corpus, query-time site resolution):**
  - `ResolveText` collapses both site mechanisms into one transform: region-param → `⟦rp:KEY⟧` placeholders, site-region → `⟦site:LIST⟧…⟦/site⟧` spans. Unit-tested (per-site resolve, unsupported flagging, span keep/drop, template round-trip).
  - Renderer `template` mode emits the site-agnostic corpus; `BuildTemplateCorpus` writes ONE `corpus/chunks.jsonl`.
  - **Index-canonical / serve-resolved:** `OpenCanonical` + `BuildCorpusVectors` index over `us`-resolved text (literal endpoints match); query/service/eval serve-resolve each retrieved chunk to the asker's site before answering, so citations stay verifiable against site-correct text.
  - CLI `build` now produces one template corpus; `query`/`search` take `-site` and serve-resolve.
  - **Live sample proof (no embeddings, no gateway):** one corpus, same chunk → `intake.profile.datadoghq.com` (us) / `…us3.datadoghq.com` (us3) / `…ddog-gov.com` (gov), doc URL stays site-agnostic. Site-correctness from a single corpus confirmed.
- **P1 remaining:** (a) full whole-docs build + embed (~30–60 min ollama, LFS budget — needs sign-off); (b) live eval gate via gateway; (c) **P1b** structural-`site-region` retrieval-precision edge (a chunk wholly inside one site's branch embeds under canonical `us`; rare — note + revisit if eval shows misses).

- **D0 done:** parallel module `ddocs` scaffolded by copying the validated `apmdocs` engine; renamed module path; `go build/vet/test ./...` all green (frozen baseline before generalizing).
- **P0 done (taxonomy foundation):**
  - `taxonomy/` package — `Class`/`Rule`/`Taxonomy`, `Load`, longest-prefix-wins `Classify`, `Inventory` coverage report. Unit tests pass.
  - `taxonomy.json` — curated seed: **80 areas, 100% file coverage** (5518/5518). Confidence H|M|L is the review queue.
  - Generalized `resolve.Chunk` — added `Suite/Product/Feature/PlatformTopic/Kind` + `Selectors` map (axis-typed tabs: language/deployment/provider/os/config), `Lang`/`Infra` kept as derived back-compat views.
  - Resolver threads taxonomy (`ResolveFile`/`BuildCorpus`/`BuildAllSites` take `*taxonomy.Taxonomy`, stamp every chunk).
  - CLI: `ka taxonomy -docs <repo> [-json]` coverage report; `build`/`resolve` gained `-taxonomy`.
  - **Review queue — DONE (2026-06-19):** all suite-boundary calls curated to H confidence. Operator decisions: OTel = own suite; containers = infra; error_tracking = cross-cutting (no suite); llm_observability = ai; agent = platform/no-suite; IDP = software-delivery; feature_flags + experiments + continuous_testing = digital-experience; change_tracking = platform (cross-cutting troubleshooting); dd_e2e = reference (internal docs tooling); cloudcraft = standalone/no-suite vs datadog_cloudcraft = infrastructure (integrated). Only 4 root Hugo nav pages remain at default-L (no rule warranted).

**Decisions locked with the user (2026-06-18):**
- **Monolith, not agent-teams.** One corpus + one generation agent. Suite routing is *retrieval metadata*, not separate agents. Rationale + the two-axes reframe → §1.
- **Cross-suite questions are the design center, not an edge case.** Handled by retrieval-side fan-out + a single generation call → §5. Agent-teams is rejected precisely because merging multiple cited answers breaks the verification guarantee.
- **Multidimensional taxonomy:** `suite` (optional) → `product` → `feature` (optional), **plus suite-less platform topics** (tagging, dashboards, monitors, notebooks, administration, RBAC, billing, API/SDK, …). Docs nav ≠ logical taxonomy (e.g. "Administrator's Guide" is not under an "Administration" section), so taxonomy is a **curated overlay**, not derived from file paths → §3.
- **Tabs are first-class.** Humans read the wrong instructions because they have the wrong tab selected. The agent flattens *all* tabs, answers for the implied branch, and **names the other branches**. Generalize today's `lang`/`infra` into a `selectors` map → §4.
- **Query-time site resolution.** Stop baking 8 per-site corpora; store one site-agnostic corpus and resolve region values at answer time. Collapses embedding + LFS cost ~8× → §6.
- **Migration shape (D0): PARALLEL MODULE** (user decision, 2026-06-18). Built `documentation-agent` (new module `ddocs`) **alongside** the then-frozen `apm-docs-knowledge-agent`; copied the engine, generalized, validated, then **deprecated and deleted the APM-only agent (2026-06-19)** once the whole-docs agent passed its eval gate and APM-as-a-slice matched. → §9 D0.

---

## 1. The core architecture decision (and why)

The "monolith vs. agent-teams" question conflates **two independent axes**:

| Axis | Question | Needs multiple agents? |
|---|---|---|
| **Corpus / index** | One index over all docs, or N per-suite indexes? | No |
| **Orchestration** | One agent, or a router + N suite sub-agents? | This is the only "agent-teams" part |

**RAG retrieval already does suite routing implicitly** — a vector search over the whole corpus surfaces the relevant suite's chunks on its own. Per-suite *agents* only earn their cost if they buy something a single well-built index doesn't. For this system they don't:

- **Cross-suite questions are the Datadog norm** ("correlate traces with logs", "RUM → APM", "trace-to-pod with infra tags"). A monolith retrieves across boundaries in one pass → one generation call → citations stay verifiable across suites. Agent-teams must fan out to N agents then **merge N cited answers** — the exact step that degrades the per-citation verification guarantee (this system's whole value).
- **Near-zero new architecture.** The pipeline is already suite-agnostic. A monolith is mostly "point ingest at the whole docs repo + add taxonomy metadata."
- **The grounding prompt encodes no suite expertise today** ("answer strictly from sources"), so a split would preserve nothing real.

**Decision: one corpus + one agent, with taxonomy as retrieval metadata.** Scale the *index*, not the *agent count*. You still get "agent-teams" UX for free — each Claude Code subagent (`apm-expert`, `security-expert`) is a thin `.md` wrapper calling the same backend with a taxonomy filter. **One backend, many faces.**

**Revisit agent-teams only on a measured trigger:** (a) eval shows retrieval precision collapses at full-docs scale *even with* taxonomy filtering, or (b) the the owning team org wants per-suite *ownership* (an org reason, not a technical one).

---

## 2. Multi-suite question handling (the thing to get right)

A cross-suite question is the case that **breaks agent-teams and suits a monolith**. Mechanism:

1. **Taxonomy filter is always optional.** Default = unfiltered = cross-cutting retrieval. Filter only when the asker scopes it (or a face pre-scopes it).
2. **Facet-aware retrieval fan-out** for multi-facet questions: detect candidate facets (cheap classifier or just always do balanced retrieval), gather top-k *per facet*, then **merge candidates** into one set. This fixes the only real risk — top-k skewing to one suite.
3. **Single generation call** over the merged set. Fan out where it's cheap (retrieval); stay single-threaded where it's load-bearing (generation + citation verification).

This is "query fan-out at retrieval, one cited answer out" — *not* agent-teams.

---

## 3. Taxonomy: suite → product → feature + platform topics

### Principle
Mirror the **workspace's own tag namespaces** (`suite:` → `product:` → `feature:`) so the agent's taxonomy and the workspace vocabulary stay aligned. Key realities the user flagged:
- **Not everything has a suite.** Platform-level topics (tagging, dashboards, monitors, notebooks, administration, RBAC/org settings, billing/usage, API & SDKs, service accounts, audit trail, …) are products/topics with **no parent suite**. Model `suite` as **optional/nullable**.
- **Docs nav ≠ logical taxonomy.** Sections are organized for browsing, not classification (e.g. the "Administrator's Guide" doesn't live under an "Administration" section). **Do not derive taxonomy from file paths alone.**

### Mechanism — curated taxonomy overlay (`taxonomy.json`)
Same pattern as `regions.json`: a committed, human-curated map that the Go tool consumes (no guessing at runtime).

- **Path-glob rules** map `content/en/**` paths → `{ suite?, product, feature?, platform_topic?, kind }`.
- **Default-derive** from the path where it's honest (most product dirs are clean), then **override** for the mismatches (Administrator's Guide → `platform_topic: administration`; cross-listed pages; guides that live far from their product).
- `kind` ∈ `{ suite, product, feature, platform, guide, getting_started, integration, api, reference }` — lets retrieval/eval treat platform + guide content as first-class even when suite is null.
- **Curation is the work, and it's tractable:** build a one-time path-inventory report, auto-propose tags from path + front-matter, then hand-correct the long tail. The report is a deliverable; the overrides are small and reviewed.

### Chunk metadata (generalized)
Extend `resolve.Chunk` (today: `Site, SourcePath, DocTitle, URL, SectionPath, Lang, Infra, ContentHash, Text`) with:
```
Suite        string            `json:"suite,omitempty"`         // optional — null for platform topics
Product      string            `json:"product"`
Feature      string            `json:"feature,omitempty"`
PlatformTopic string           `json:"platform_topic,omitempty"`
Kind         string            `json:"kind"`                    // suite|product|feature|platform|guide|...
Selectors    map[string]string `json:"selectors,omitempty"`     // generalized tabs (see §4); supersedes Lang/Infra
NavSection   string            `json:"nav_section,omitempty"`   // raw docs nav location (for debugging mismatches)
```
`Lang`/`Infra` become derived views over `Selectors["lang"]` / `Selectors["deployment"]` for back-compat with the APM slice.

---

## 4. Tabs / selectors (the wrong-tab problem)

**User insight:** most "I can't find / it doesn't work" confusion is a human reading the wrong tab. The agent's structural edge: the resolver **flattens every tab into a labeled section**, so retrieval sees *all* branches simultaneously — something the docs UI cannot do.

### What changes from the APM build
Today the chunker hard-codes two selector kinds (`lang`, `infra`) via `knownLangs`. At full-docs scale tabs span many axes:
- **deployment** (Kubernetes / ECS / Fargate / Host / Docker)
- **language** (Java / Python / Go / Node / .NET / …)
- **cloud provider** (AWS / Azure / GCP)
- **OS** (Linux / Windows / macOS)
- **config method** (UI / API / Terraform / env var / YAML)
- **install type**, **package manager**, **version**, etc.

So: **generalize tabs into a `selectors` map** keyed by axis. Infer the axis from the tab-group label where the docs provide one; fall back to a value→axis dictionary (extends today's `knownLangs`) for bare `tabs`. Tag each chunk with the full selector set in scope.

### What the agent does with it (the payoff)
1. **Answer for the branch the question implies** ("in Fargate", "for .NET") even when a human would've been on the wrong tab.
2. **When the question doesn't specify a branch the docs split on, name the options** rather than silently picking one — pre-empting the wrong-tab error.
3. **Surface the covered branch in the output** as an explicit field, e.g. `branches: { deployment: "Fargate", language: "Java" }`, and warn when other branches materially differ. Mirrors the existing `site_name`/`truncated` output discipline.

Grounding rule (extends current rule 2): *"When the documents branch by a selector (`[deployment: X]`, `[language: Y]`, `[provider: Z]`), answer for the value(s) the question names; if it names none and the branches differ, state which branch you're answering for and list the others — never assume the reader is on the right tab."*

---

## 5. Retrieval at full-docs scale

Keep the shipped **hybrid BM25 + vector (RRF)** core. Additions:

- **Taxonomy-filtered retrieval.** Optional pre-filter by `suite`/`product`/`feature`/`platform_topic`/`kind`. Unfiltered by default (cross-suite native).
- **Facet fan-out + merge** for multi-suite questions (§2).
- **Reranking becomes likely-necessary** (was P5-optional for APM). At whole-docs scale, top-k precision matters more; add a cross-encoder/Claude rerank pass over the fused candidates before generation. Gate it on eval, but expect to turn it on.
- **Vector store:** brute-force cosine is fine for thousands of chunks; **whole-docs is ~10–20× larger** (tens of thousands of unique vectors). Measure first — brute force may still be sub-second. If not, add an ANN index (HNSW) behind the existing `Retriever` interface. **No external vector DB unless eval forces it.**

---

## 6. Scaling: one corpus, query-time site resolution (the big cost win)

**The real scaling problem is not suites — it's the 8× per-site multiplication of a now-huge corpus.** APM × 8 builds in ~2 min; whole-docs × 8 could be 30–60+ min builds and hundreds of MB–GB of LFS.

**"Won't one corpus lose per-site context?" — No.** Site variation in the docs is **two distinct mechanisms**, handled differently; neither loses context:

| Kind | Shortcode | What varies | Handling |
|---|---|---|---|
| **Value substitution** (the bulk) | `region-param` | only an endpoint string; prose is **byte-identical** across sites | typed placeholder, resolved at serve time — there is no context to lose |
| **Structural** (rare) | `site-region` show/hide | the **content itself** differs per site | kept as **site-tagged branches** (like tabs), filtered to the asker's site at retrieval; never collapsed away |
| **Availability** | `unsupported_sites` | whole product absent on a site | already query-time today (`SiteFacts.Unsupported`) |

The old design filtered/collapsed these at **build** time → 8 corpora. The new design carries site-applicability as **metadata** and filters at **query** time → 1 corpus. **Same information, same correctness, deferred application.**

**Fix: stop baking per-site corpora.** Store **one corpus**; resolve region values + filter site-branches at answer time.

- **Today (APM):** resolver collapses `site-region` / substitutes `region-param` **per site at build time** → 8 resolved corpora; cross-site dedup recovers 88% on embeddings but still materializes 8 chunk sets. *That 88% is the proof:* the build already discovered ~88% of chunk-slots are byte-identical across sites (the value-swap majority).
- **New:**
  - **Value swaps → typed placeholders** (`⟦dd_site⟧`, `⟦dd_full_site⟧`, `⟦dd_api⟧`, …). Identical prose ⇒ one chunk is the full context.
  - **Structural `site-region` blocks → site-applicability tag** on the chunk (which sites it applies to); **filtered during retrieval** so a `us` query can't surface a `gov`-only branch, and the `gov` branch is fully present when `site=gov`. Each distinct branch keeps its own chunk + embedding (not deduped — different text).
  - **Index canonical, serve resolved:** embed + BM25-tokenize on a **canonical (`us`-resolved)** form so literal endpoint searches (`us3.datadoghq.com`) still match; resolve placeholders to the **asker's site** only when assembling the generation context.
- **Net:** embeddings + LFS collapse from ~8× to ~1× (only the genuinely site-divergent ~12% carries multiple branch-chunks); site-correctness preserved exactly.
- **Honest cost (not loss):** resolver gets more complex (tag applicability + carry a template vs. "render for site X"); a missed `region-param` key ⇒ wrong endpoint at serve time → **eval must assert site-correctness hard** (§8 site cases). If structural divergence were *large* the win would shrink — but the 88% measurement says it isn't.

This is the highest-leverage change in the plan and should land early (P1) so the corpus is never built the expensive way.

---

## 7. Generation (unchanged engine, minor additions)

Reuse the shipped generation path verbatim — it's the de-risked part:
- Cached grounding prefix + per-chunk citable `document` blocks + Citations API + **cited_text verification** (the forwardable guarantee).
- **No-truncation continuation loop** (`MaxTokens` 8192 + `max_tokens` resume, cap 8 rounds, `Truncated` flag).
- **Site label** via `SiteLabel()` (`US1 (us)`); **doc links stay site-agnostic** (`docs.datadoghq.com`, `site=us`, never a datacenter in a URL).

Additions for whole-docs:
- Inject the resolved **site values** into chunks at assembly time (§6).
- Add the **`branches`** output field (§4) and a cross-suite note when an answer spans suites.
- Grounding prefix gains a compact **taxonomy legend** so the model can name suites/products correctly in prose.

---

## 8. Eval (expand the trust gate)

The 12-case APM eval becomes a **stratified suite** — the trust gate is what lets answers go out verbatim, so it must cover the new surface area:
- **Per-suite** answer/refuse cases across all major suites + **platform topics** (tagging, dashboards, monitors, notebooks, admin/RBAC, billing).
- **Cross-suite** cases (trace↔log, RUM↔APM, infra↔APM) — assert citations from **both** suites, single coherent answer.
- **Wrong-tab cases** — question names a non-default branch; assert the answer uses the right `[selector: …]` branch and names the alternatives.
- **Site cases** — same Q across sites resolves endpoints correctly *from the single corpus* (proves §6).
- **Taxonomy-mismatch cases** — e.g. an Administrator's-Guide question resolves despite nav placement.
- Keep **citation-integrity = true on every answer** as the hard pass condition.

---

## 9. Build phases

**D0 — migration shape: PARALLEL MODULE (decided 2026-06-18).**
- Build a new module `ddocs` under `.claude/agents/documentation-agent/`, **alongside** the frozen `apm-docs-knowledge-agent` (module `apmdocs`).
- **Seed by copying** the validated engine (`resolve/ retrieve/ agent/ service/ mcp/ eval/`, `cmd/`), then generalize on the copy — APM agent is never destabilized during the build.
- **Deprecation path (DONE 2026-06-19):** whole-docs passed its eval gate (§8) and APM-as-a-slice reproduced the old APM agent's answers, so `apm-docs-knowledge-agent` (module `apmdocs` + subagent face) was retired and deleted. The parallel-module window is closed — `ddocs` is the only docs codebase now.
- **Cost of this choice (accepted):** engine duplication + a window where bug-fixes may need porting between the two. Mitigate by minimizing divergence in the copied core and porting only what the generalization requires.

| Phase | Deliverable |
|---|---|
| **P0 ✅ DONE** | **Taxonomy foundation.** `taxonomy/` package (Classify/Inventory) + `taxonomy.json` seed (80 areas, 100% coverage) + generalized `Chunk` (suite/product/feature/platform_topic/kind/selectors) + tab→`selectors` axis inference + `ka taxonomy` report. Tests green, 2026-06-18. **Review queue curated to H, 2026-06-19** (see §0 for the operator's suite-boundary calls). |
| **P1 ✅ DONE** | **Single-corpus, query-time site resolution** (§6). `ResolveText` + template render + canonical-index/serve-resolve through build/query/service/eval. Full whole-docs corpus built (29,538 chunks, 5m54s) + eval 10/12 @ k16, citation-integrity 100%. **P1b structural-`site-region` edge still noted.** |
| **P2a ✅ DONE** | **Gateway reranker** (§5): retrieve wide → rerank → keep best k. Fixed both eval residuals; whole-docs eval 12/12, citation-integrity 100%. |
| **P2b ✅ DONE** | **Taxonomy-filtered retrieval** — explicit, deterministic suite/product `Filter` on retrieval (`SearchFiltered`), wired through CLI (`-suite`/`-product`), service (`AnswerFiltered`/`AnswerSuite`), and the MCP `suite` arg. **Validated:** `suite=apm` on the Python question drops all AppSec docs and the answer cites the canonical tracing docs (was the eval's top precision finding). Unit-tested. |
| **P2c ✅ DONE** | **Facet fan-out** (§2/§5): `SearchFanout` retrieves per-suite and merges by reciprocal rank, so a cross-suite question is balanced across facets before a single rerank+generation. Service `AnswerFanout` + CLI `-suites`. **Validated:** traces↔logs 7/2→5/4, RUM↔logs 7/1→6/6 (suite skew fixed). |
| **P3 ✅ DONE** | **Expanded eval** (§8) — the go/no-go trust gate. **Deterministic, no LLM judge**: golden-citation assertions, an offline retrieval gate (golden-doc-in-pool, ~7s, never flakes) + a full gate (citation-integrity + site facts + refusals). ~100 stratified cases (every suite, platform topics, cross-suite, selector/tab, all 8 sites, refusals). **Full gate 98/98, citation-integrity 100%; retrieval gate 85/0/13.** |
| **P4 ✅ DONE** | **Faces.** MCP tool renamed to `query_datadog_docs` (whole-docs; full filter + fan-out args) and server identity to `documentation-agent`. **One** subagent face: `.claude/agents/documentation-agent.md` (scopes per question via the filter flags); the per-suite-face clone pattern is documented inside it. Per-suite faces (logs-expert, etc.) are trivial clones — mint as PSAs ask. (An `apm-docs-expert` exemplar was added then retired to avoid confusion with the frozen old APM agent, which stays until cutover.) |
| **P5 ✅ DONE** | **Freshness at scale.** Incremental refresh re-embeds only changed chunks (content-hash cache) AND now **prunes orphaned vectors** (deleted/changed chunks) so the committed cache stays bounded; build reports new/reused/pruned + the committed LFS footprint. **Validated:** shrinking a corpus pruned 661 orphans, embcache 7.9→5.9MB, unchanged refresh ran in 172ms. (Scheduling + `git pull` live in the P7 skill.) |
| **P6** | **Slack auto-responder** (in-cluster; can use gateway embeddings there) — now docs-wide, not APM-only. |
| **P7 ✅ DONE** | **Rebuild/refresh skill** — `/refresh-docs-agent` (`.claude/skills/refresh-docs-agent/`, opus/medium). Two modes: `refresh` (git pull docs → incremental re-embed → prune → retrieval gate) and `rebuild` (from scratch → full eval gate). Verifies prerequisites before the embed, builds into the committed corpus location, reports new/reused/pruned + LFS footprint, and commits only on the operator's go-ahead. The durable "keep it current" surface over the P1–P5 machinery. |

---

## 10. Open decisions / spikes

1. **D0 migration shape** (§9) — evolve-in-place vs. parallel module. **Recommend evolve-in-place.** User to confirm.
2. **Taxonomy curation scope** — how deep does `feature`-level tagging go on day one? Recommend: `suite`+`product`+`platform_topic` complete; `feature` only where it disambiguates retrieval. Earn deeper tagging with eval.
3. **Locale scope** — English only (`content/en/`), as today. Confirm.
4. **Doc-section allowlist vs. whole repo** — whole `content/en/` minus non-answerable nav/marketing/release-note noise. Produce an *exclude* list, not an include list (inverts the APM allowlist).
5. **Vector store at scale** — brute force vs. ANN. **Measure in P1** before adding a dependency.
6. **Rerank model** — cross-encoder local vs. Claude rerank via gateway. Bake-off in P2.
7. **LFS budget** — single-corpus collapses this ~8×, but whole-docs is still bigger than APM. Confirm the LFS quota target before P1 commit.

---

## 11. Key references

- **Predecessor (everything reused, now deleted 2026-06-19):** the former `apm-docs-knowledge-agent` plan + module `apmdocs` (`resolve/ retrieve/ agent/ service/ mcp/ eval/`, `cmd/ka`). All P1–P4 mechanisms were validated 2026-06-18 and live on in `ddocs`; history is recoverable from git (final commit before removal: `df2e224`).
- **Taxonomy source signals:** docs repo `content/en/**` paths + per-file front-matter; `config/_default/menus/*` (nav, for *detecting* mismatches, not for classification); workspace tag namespaces in `CLAUDE.md` (align vocabulary).
- **Site map:** `assets/scripts/config/regions.config.js` → committed `regions.json` (8 sites; `dd_datacenter`, `dd_site`, `dd_full_site`, `dd_api`, …).
- **Availability:** `config/_default/params.yaml` → `unsupported_sites`.
- **Infra:** a model gateway (`anthropic/claude-opus-4-8`, Citations + caching), ollama `nomic-embed-text` (local, offline). Same as APM build.

---

## 12. Tracker / internal tracking

New effort: **`documentation-agent`** — generalizes `apm-docs-knowledge-agent` . Foundational; supersedes the docs-half scope for cross-suite work and feeds `product-alias-docs-gap-analysis`, `expert-agent-fleet`, specialist tooling. Track in the effort tracker + an internal tracking ticket. Mark in-progress when D0 is decided and P0 starts.

