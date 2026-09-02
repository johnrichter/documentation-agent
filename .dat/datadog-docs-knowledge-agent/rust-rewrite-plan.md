---
name: Datadog Docs Knowledge Agent — Rust Rewrite Plan
description: "Tracking + resume doc for rewriting the ka knowledge agent from Go to Rust. Phases 0–6 DONE — cutover COMPLETE. Phase 7.1 (local candle reranker): steps 1–6 done (hand-rolled head merged, head weight-keys verified, backbone key-remapping fix merged, backbone load validated empirically); step 7 (tau calibration, 85/85 gate) pending. Phase 8 (GPU re-embed): code-complete. Phase 7–8 review (2026-06-26): Metal GPU RESOLVED + SUPPORTED — the block was a missing candle-nn/metal feature flag (not a candle kernel gap), plus an fp16 where_cond gap sidestepped by forcing fp32 on Metal; full embed verified on Apple Silicon (~2.4× vs CPU, cosine 0.999953). Default device now auto (GPU-preferred, CPU fallback). Phase 9 (pipeline-parallel embed): CLOSED with data — embed is forward-bound (tokenize 0.1% of wall-clock), so overlap is not worth building. Phase 10 (retire Go + .bin/+tests/ reorg + tau calibration + disk reclaim/LFS ship of reranker + metal binary): DONE — merged to main 2026-06-27 (bc85b72); Go fully removed, Rust ka is the sole runtime. REWRITE COMPLETE."
id: project:tooling:docs-agent-rust-rewrite
tags:
  - type:project
  - topic:tooling
  - topic:apm
  - status:complete
  - privacy:public
links:
  - agent:tooling:datadog-docs-knowledge-agent
  - skill:workspace:refresh-docs-agent
  - agent:workflows:model-tiering
  - agent:workflows:git
updated: 2026-06-27T07:00:00Z
---

# Datadog Docs Knowledge Agent — Rust Rewrite Plan

Rewrite the `ka` knowledge agent (~4,400 LOC Go) to Rust. Phase 1 of an eventual move of **both** agents to a shared Rust engine crate (the sibling code-agent, ~8,500 LOC Go, follows later — separate module today, no import coupling).

## Why Rust (settled)
- The embedder — the one capability where language is decisive — is **native** in Rust (candle + HF `tokenizers`), giving a **pure-Rust single binary** with zero external services and no native libs. Go has no first-party ML and forces community CGO bindings.
- Everything else (Anthropic HTTP, BM25, MCP stdio) is fully doable in Rust.
- Operator owns Rust going forward (identity updated separately).
- Acceptance is cheap to prove: the existing eval gates are a behavioral spec.

## Phase 0 — SPIKE (DONE ✅)
Hands-on results, darwin-arm64:
- **Embedder → GO.** candle `0.10.2` (released tag) + `candle_transformers::models::nomic_bert` + HF `tokenizers` crate, masked mean-pool → L2-norm → `search_query:`/`search_document:` prefixes. Parity vs Ollama: **fp32 cosine 1.00000**, **fp16 cosine 0.99994 (worst 0.99991)**. `otool -L` → **only system libs: clean single binary, no ONNX Runtime, no native deps.**
- **Anthropic client → HAND-ROLL.** No community crate supports citations + `cache_control` + custom base-URL/auth-cascade; best candidate was v0.1.3 / 0★ / none of the three. No official Rust SDK exists (Anthropic ships 7 SDKs, Rust not among them; Stainless generator winding down).

## Locked technical decisions
- **Embedder:** candle `0.10.x` (pin exact), `nomic_bert`, `tokenizers` crate. Ship **fp16 `model.safetensors` (~261MB)** via Git LFS (fp32 is the fallback for max precision). Pure-Rust single binary — **single-binary distribution is back on** (no need to relax it).
- **Anthropic client:** thin `reqwest` + `serde`, mirroring the Go `gateway.go` — `document` blocks with `citations.enabled`, parse `cited_text`/`document_index`, `cache_control: ephemeral`, the 3-method auth cascade (<internal-cli> gateway Bearer+headers / `x-api-key` / OAuth Bearer, 401 fallthrough), non-streaming `/v1/messages` + max-tokens continuation. No streaming.
- **MCP:** optional `rmcp` (Tier 2 — fine for a single-tool stdio server) as a `ka mcp` subcommand. **Primary consumption is CLI `-json`** (the Claude Code subagent shells out); MCP is a secondary surface. **No Go, no second binary.**
- **Retrieval:** port BM25 + dense-vector + RRF fusion + taxonomy filters + suite fan-out, preserving scoring behavior.
- **Assets:** reuse `chunks.jsonl`, `regions.json`, `taxonomy.json` unchanged. **Re-embed the corpus with candle** → rewrite `vectors.bin` + `embcache.bin` (keep the simple binary layout, or re-spec; old Ollama cache is invalid in the new pipeline).
- **Distribution:** cross-compile 4 targets (`cargo-zigbuild`/`cross`; candle CPU is pure Rust → no cgo), ship per-target single binaries via LFS, host-selected by `uname` (mirror the Go `ka-<os>-<arch>` pattern + `build.sh`). **Revised (Phase 8, task 7):** the strict single-portable-binary constraint is relaxed — the 4 CPU static-musl/dynamic baselines remain the always-works, blind-pick default; an optional `ka-darwin-arm64-metal` supplement is produced on Apple Silicon and is an explicit operator opt-in (not auto-picked, not required). Distribution naming: `ka-<os>-<arch>[-<accel>]` where `-metal` is the only defined accel today.
- **Acceptance:** pass the same **98-question retrieval gate + citation-integrity** gate; plus a **Go-vs-Rust parity harness** diffing answers/citations during migration.

## Phases & per-phase tiering
Tiers per `agent/workflows/governance-model-tiering.md`. Thinking depth = effort baseline + the per-step cues noted.

| Phase | Work | Model | Effort | Thinking cue |
|---|---|---|---|---|
| 0 Spike | de-risk embedder + client | Opus | high | DONE |
| 1 Scaffold + `resolve` port | cargo workspace, core types; port site/region resolution (1207 LOC, subtle correctness) | **Opus** | **high** | DONE — `rust/` workspace (`taxonomy` + `resolve` crates), all 17 ported tests green, clippy `-D warnings` clean |
| 2 Retrieval + embedder | BM25 + vector + RRF + filters/fanout (scoring **parity**); wire candle embedder (known recipe) | **Opus** retrieval / **Sonnet** embedder | high / medium | DONE — `crates/retrieve` + `crates/embed`; 9 tests green; BM25/RRF constants match Go exactly; `KAV1`/`KEC1` binary formats preserved; Ollama dependency eliminated |
| 3 Anthropic client + answer/citations | hand-rolled client; **citation verification is the product** | **Opus** | **xhigh** | DONE — `crates/client` (gateway + auth cascade + wire types) + `crates/answer` (grounding, continuation, citation verify, reranker); 15 tests green; ported the platform-hardened auth (commit `b6c51ac`) |
| 4 CLI + MCP surfaces | clap CLI `-json`; `rmcp` stdio subcommand | **Sonnet** | high (operator-directed) | DONE — `crates/cli` (`ka` binary): 11 subcommands mirroring `cmd/ka`, `query --json`, `ka mcp` rmcp stdio server; built by a Sonnet subagent, reviewed on Opus |
| 5 Eval gates + parity harness | port gates; adversarial Go-vs-Rust diff (acceptance) | **Opus** | high | DONE — `crates/eval` (full + offline retrieval gate, faithful `eval.go` port) + `ka eval` wired; Go-vs-Rust parity harness (`parity` bin + `parity.sh`); built by an Opus subagent at high effort, reviewed on Opus. **Empirical run deferred** — needs model assets + gateway (LFS pointers in-repo) |
| 6 Cross-build + distribution + docs | zigbuild matrix, LFS, build.sh, agent .md + refresh skill, cutover | **Sonnet** | low–medium | IN PROGRESS — **6a infra + 6b authoring DONE** (`rust/build.sh` cargo-zigbuild matrix + `rust/BUILD.md`; **linux=static musl**; Go→Rust cutover edits prepared on `feat/rust-phase6b-cutover`, held). **Execution operator-gated** on a toolchain+assets host: cross-build/LFS-ship, re-embed, host eval gate, then merge the cutover + retire Go |

Each phase = its own worktree per `governance-git.md`; review gate on every commit.

## Open risks / unknowns (verify in-phase, not blockers)
- **Cross-compile to 4 targets** — candle CPU is pure Rust (no cgo) so `cargo-zigbuild` should be clean; confirm per target.
- **Bulk-embed throughput** on CPU for the corpus re-embed (candle uses the `gemm` crate; Metal/CUDA available if slow). Build-time only, not query-time.
- **`vectors.bin`/`embcache.bin` format** — decide reuse-Go-layout vs re-spec; re-embed regenerates both regardless.
- **candle `nomic_bert` edge behaviors** vs the exported model — spike covered the common path (cos 1.0/0.9999); watch long-input truncation + the 2048 vs 8192 position handling. **PARTIALLY ADDRESSED (hardening):** embedder now sets tokenizer truncation to the model's `max_position_embeddings` (read from config.json, fallback 2048), removing the unbounded-input footgun. **Still open for Phase 5:** empirical Ollama-parity on long inputs is unverifiable until the model assets ship (assets are not in-repo); the parity harness must confirm the effective length matches the Go/Ollama path.
- **fp16 asset** — ship a converted fp16 safetensors (~261MB), not the 522MB fp32; verify the converted file reproduces the 0.9999 parity.
- **YAML crate** — ~~`serde_yaml` 0.9 (archived)~~ **RESOLVED** — replaced with `yaml_serde 0.10` (The YAML Project / yaml.org; pure-Rust `libyaml-rs`; API-identical drop-in). No longer a risk.

## Phase 1 — DONE ✅ (resolve port)
- **Layout:** `rust/` cargo workspace under the agent module (coexists with Go until Phase 6 cutover). Members: `crates/taxonomy` (path→Class overlay) + `crates/resolve` (the 1207-LOC port). Go code untouched.
- **Ported faithfully:** `regions`, `params` (unsupported), `frontmatter`, `lex`, `parse` (container/leaf tree — value-semantics stack reproduces Go's pointer build incl. unmatched-close + unwind order), `render` (shortcode engine, template + per-site modes), `links` (ref-inline + absolutize), `chunk` (heading/label split, selector axes, sha256 id), `resolvetext` (query-time site resolve), `corpus` (build orchestration; std recursive walk, lexically sorted to mirror `filepath.WalkDir`). `Chunk.selectors` uses `BTreeMap` for Go-matching sorted-key JSON.
- **Tests:** all Go `resolve` + `taxonomy` table tests ported (17 total) — green. `cargo clippy --all-targets -D warnings` clean; `cargo fmt` applied.

## Phase 2 — DONE ✅ (retrieve + embed crates)
- **`crates/retrieve`:** `bm25` (K1=1.5, B=0.75; token regex `[a-z0-9_]+`; IDF + TF exact match to Go), `vector` (KAV1 binary format, `normalize` + `dot` + `vector_search`), `embcache` (KEC1 binary format; 64-char hex key, 32-byte raw on disk; `prune(impl Fn)`), `filter` (suite/product/platform_topic/kind), `rank` (`Ranked`, `top_ranked`), `embed` (`Embedder: Send` trait + `prefix_for`), `index` (`Index` — hybrid BM25+vector+RRF; `search`, `search_filtered`, `search_fanout`; `open`, `open_canonical` with `CANONICAL_SITE` resolve), `build` (embed-once dedup across sites by content_hash; `materialize_vectors`, `ensure_vectors`, `build_all_vectors`, `build_corpus_vectors`). RRF: `rrfK=60.0`, 0-indexed rank, arm widths 50 (unfiltered) / 400 (filtered). `vectors.bin`/`embcache.bin` byte-identical to Go for cross-compat.
- **`crates/embed`:** `CandleEmbedder` — nomic-embed-text-v1.5 on CPU via candle 0.10; masked mean-pool → L2-norm; fp16/fp32 selectable; mmap weights; no Ollama/ONNX runtime. Implements `retrieve::Embedder`. Eliminates the undocumented `localhost:11434` runtime requirement.
- **Tests:** 9 tests in `retrieve::tests` — BM25 tokenization, vectors round-trip, bad-magic rejection, hybrid RRF fusion, filter matching, EmbCache round-trip, `build_all_vectors_dedup` (cross-site dedup + cache reuse + lossless materialize). All green, `cargo clippy -D warnings` clean, `cargo fmt` applied.
- **MSRV:** workspace `rust-version` lowered to `"1.82"` (candle + yaml_serde requirement). `yaml_serde 0.10` replaces archived `serde_yaml 0.9`.

## Phase 1 + 2 — parity audit + hardening ✅ (pre-Phase-3 gate)
Full Go↔Rust parity audit (7 module-group reviewers) before building Phase 3 on this base. Phase 1 (Opus-built) verified faithful incl. the subtlest `parse` unwind/attach order. Phase 2 scoring math + KAV1/KEC1 formats verified faithful. Fixes applied (own worktree, 28 tests green, clippy `-D warnings`):
- **Deterministic ranking** — `rank::top_ranked` + both `index` fused/fan-out sorts now break score ties by ascending idx / chunk id. Go uses unstable `sort.Slice` over map-derived lists (non-deterministic run-to-run); the explicit tiebreak makes retrieval reproducible — a strict improvement, not a ranking divergence.
- **Embedder truncation** — see risks above (tokenizer capped at model context).
- **L2 zero-vector guard** — epsilon before the in-graph normalize divide so a degenerate all-masked input can't yield NaN/Inf (Go guarded `sum==0`).
- **`dot` dimension assert** — `debug_assert_eq!` on operand lengths (Go panics on a short operand; Rust `zip` had silently truncated).
- **Walk symlink parity** — `collect_md` uses `symlink_metadata` (lstat) so symlinked dirs are not descended, matching `filepath.WalkDir`; `.md` match is now suffix-based like Go's `HasSuffix`.
- **Front-matter tolerance** — a single bad-typed field no longer discards the others (mirrors Go's partial `yaml.Unmarshal`).
- **Parity-harness notes (Phase 5):** compare `embcache.bin` by record-set, NOT bytes — both Go and Rust serialize it from unordered maps (non-reproducible byte order; `vectors.bin` IS byte-stable). Long-input embedder parity is the open verification (see risks).

## Phase 3 — DONE ✅ (client + answer/citations)
- **`crates/client`:** hand-rolled `reqwest::blocking` + `serde` gateway, port of Go `gateway.go`. `wire` (the `/v1/messages` request/response types — `document` blocks + `citations.enabled`, `cache_control: ephemeral`, `cited_text`/`document_index`; renames mirror the Go `json` tags, `skip_serializing_if` reproduces `,omitempty`), `auth` (3-method cascade <internal-cli> Bearer+org/source headers / `x-api-key` / Claude Code OAuth Bearer; availability probed without I/O), `gateway` (`complete` with `anthropic-version: 2023-06-01`, 401 fall-through, error envelope; `Mutex`-guarded token cache, 3h TTL, shareable via `Arc`). **rustls-tls / blocking** chosen for the pure-Rust single-binary goal and to mirror Go's synchronous design.
- **Ported the platform-hardened auth** (Go commit `b6c51ac`): Claude Code OAuth falls back to the credentials file when the Keychain is unreadable, registers on Linux/Windows (not only where `security` exists), and reports every source tried on failure. This is the fix for the keychain-in-subprocess bug — the port is of the corrected logic, not the original.
- **`crates/answer`:** port of Go `answer.go` + `rerank.go`. `Agent` (per-site grounding prefix, cited-documents request, max-tokens **continuation loop** so an answer is never silently truncated, `assemble` with per-URL citation dedup, `verify_citation` whitespace-normalized substring = the **forward-verbatim guarantee**), `GatewayReranker` (gateway-as-cross-encoder; wide pool → relevant indices; empty result honored as a refusal signal), `SiteFacts`/`Citation`/`Answer` + `build_site_facts`/`site_label`.
- **Tests:** 15 green (client: 6 credentials-parse cases; answer: verify, assemble maps/dedup/dedup-unverified, empty-answer, site-label, rerank-pool clamp, parse-index-array, snippet). Full workspace `cargo clippy --all-targets -D warnings` clean; `cargo fmt` applied; 43 tests total pass.
- **Divergences (faithful, documented):** reranker is infallible (Go's advisory error folded into always-usable return); `complete` takes the request by value and sets `model` from the active method (as Go does); enum-dispatched auth methods replace Go's `tokenFn` closures.

## Phase 4 — DONE ✅ (CLI + MCP surfaces)
- **`crates/cli`** (the `ka` binary), built by a **Sonnet** subagent at operator-directed high effort, reviewed on Opus (gates re-run independently; code read; site-correctness invariant verified). `main.rs` (clap), `pipeline.rs` (shared orchestration, port of `service.go` + `cmd/ka` query wiring), `mcp.rs` (rmcp stdio).
- **CLI:** 11 subcommands mirroring `cmd/ka/main.go` 1:1 — `version`, `build`, `refresh`, `resolve`, `index`, `search`, `query`, `materialize`, `taxonomy`, `eval`, `mcp` — with matching flag names/defaults and self-locating data via `exe_dir`. `query --json` serializes `answer::Answer` with **byte-identical field names** (so JSON consumers are unaffected). Composes `resolve → retrieve → (candle) → rerank → answer` over a shared `Arc<Gateway>`; serve-resolves survivors to the asker's site (verified: `open_canonical` keeps template chunks, so non-`us` answers resolve correctly).
- **MCP:** `ka mcp` stdio server (`rmcp`), single tool `query_datadog_docs` with Go-faithful schema + `format_answer`; async↔blocking bridge via `tokio::task::spawn_blocking`.
- **Supporting-crate changes (reviewed):** `retrieve::Embedder: Send → Send + Sync` (so `Arc<Pipeline>` crosses the MCP async handler; compiler-proven sound, candle weights are read-only mmap); `taxonomy::inventory`/`Report`/`Entry` ported for `ka taxonomy` (was Go-only; uses the same lstat walk as the Phase 1+2 symlink hardening).
- **Deps:** `clap 4`, `rmcp 1.8` (server+macros+transport-io), `tokio 1` (rt-multi-thread+io-std+macros+sync) — MCP path only.
- **Tests:** 55 total green (12 new: cli 11 + taxonomy inventory); `cargo clippy --all-targets -D warnings` clean; `cargo fmt` applied. Independently re-verified on review.
- **Deferred / cutover notes:** `eval` exits non-zero pointing at Phase 5 (eval-gate crate not yet ported). **Flag-style change:** clap long flags (`--json`, `--site`) replace Go's single-dash (`-json`) — the agent `.md` + `refresh-docs-agent` skill that shell out to `ka` MUST be updated at the **Phase 6 cutover**. MCP protocol version is rmcp's default (negotiated) vs Go's pinned `2024-11-05`.

## Phase 5 — DONE ✅ (eval gates + parity harness)
- **`crates/eval`:** faithful port of the Go `eval` package. `check` (citation-integrity ALWAYS required = the forward-verbatim guarantee; case-insensitive `must_contain`/`must_not_contain`; `expect_no_citation`; deliberately does NOT assert `cite_url_contains` at the full gate — the LLM's citation choice varies run-to-run), `run` (full gate: wide pool → optional rerank → serve-resolve to the question's site → grounded answer → check), `run_retrieval` (OFFLINE deterministic gate: hybrid retrieval only, asserts every golden `cite_url_contains` lands in the top-N candidate pool; golden-less cases skipped), `load_questions` (reads the frozen `eval/questions.yaml`, ~98 cases), `open_corpus` (`ensure_corpus_vectors` → `open_canonical`, BM25-only fallback). All 5 `eval_test.go` cases ported + 1 load test.
- **`ka eval` wired:** replaced the Phase-5 stub; mirrors Go `cmdEval` — both gate branches, byte-identical stdout formats, non-zero exit on any failure. **Faithful divergence:** semantic retrieval is `--semantic` opt-in via candle (`--model-dir`/`KA_MODEL_DIR` + `--fp16`) instead of Go's always-on `--ollama`/`--model`, consistent with the rest of the Rust CLI (query/search/mcp).
- **Go-vs-Rust parity harness** (acceptance): `parity` bin (`crates/eval/src/bin/parity.rs`) + thin `eval/parity.sh` wrapper. `diff` runs both binaries' `eval -v` over the same questions+corpus and diffs per-question pass/verified/cites/answer + the summary line. `artifacts` compares `vectors.bin` by **bytes** (byte-stable, ordered slice) and `embcache.bin` by **record-set** (hash→vector; byte order non-reproducible from an unordered map) — with a byte-identical short-circuit for unmaterialized LFS pointers. `EmbCache::keys()` added for the record-set compare.
- **Deferred (Phase 6, honest boundary):** the full gate (needs model assets + live gateway) and the offline gate (needs `vectors.bin`) can't run end-to-end here — corpus files are Git-LFS pointers — so the empirical Go-vs-Rust diff is deferred to asset availability; gates fail with clear actionable errors, no faked passes. The **long-input embedder-truncation parity check** is wired behind `parity long-input` and skips with a Phase-6 runbook when assets are absent.
- **Tests:** 61 total green (+6 eval crate); `cargo clippy --all-targets -- -D warnings` clean (verified raw, no pipe masking); `cargo fmt` applied. Independently re-verified on review.

## Phase 6 — DONE ✅ (cross-build + distribution + docs + semantic-default cutover)
**6a — build infrastructure DONE ✅** (reversible slice; Sonnet subagent, reviewed on Opus):
- **`rust/build.sh`:** cargo-zigbuild cross-matrix mirroring the Go `build.sh` — 4 targets (`aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`), emitting the **same `ka-<os>-<arch>` output names** so the agent `.md` platform-selection survives cutover unchanged; host-native `ka` copy; preflight that exits with an install hint if `cargo-zigbuild`/targets are missing.
- **`rust/BUILD.md`:** operator toolchain bootstrap (rustup + 4 triples + cargo-zigbuild + zig); host-only build needs none of it.
- **Verified:** host release build (`cargo build --release -p ka --bin ka`) + `ka version` on darwin-arm64; `build.sh` syntax-clean. (Toolchain has since been installed on this host — the cross-matrix musl build is now verified; see the De-risk run below.)

**6b — authoring DONE ✅ (on `main` / prepared); execution operator-gated (each effectively irreversible):**
- **6b.1 linux build = static musl DONE ✅** (operator decision): `rust/build.sh` + `BUILD.md` now target `x86_64-unknown-linux-musl` + `aarch64-unknown-linux-musl` (fully static, matches the Go `CGO_ENABLED=0` binaries, runs on any linux incl. Alpine/distroless). macOS stays dynamic (Apple has no static libc). Output names `ka-<os>-<arch>` unchanged. On `main`.
- **Cutover edits PREPARED (held, NOT merged)** on branch `feat/rust-phase6b-cutover`: agent `.md` + `refresh-docs-agent` skill switched Go→Rust — every `-flag`→`--flag`, the Go-build fallback → `cd rust && cargo build --release -p ka --bin ka`, `./build.sh` → `rust/build.sh`, and **ollama → candle**. Reviewed on Opus. **Superseded by 6c** (semantic became the default; see below) and merged there atomically with the binary ship.
- **Candle model assets — acquisition RESOLVED:** get the assets by `git clone https://huggingface.co/nomic-ai/nomic-embed-text-v1.5` (HF repos are git+LFS → `config.json` + `tokenizer.json` + `model.safetensors`); **no `huggingface-cli` dependency** (operator decision). HF ships fp32; the shipped form is fp16 (~261MB). **In-repo distribution RESOLVED in 6c** (below): fp16 ships via LFS at `<module>/model/`, self-located.

### De-risk run (2026-06-24, this host — toolchain now installed)
- **✅ musl static build VERIFIED — closes the plan's open candle-on-musl risk.** `cargo zigbuild --release --target x86_64-unknown-linux-musl` compiled the whole tree incl. `candle-core`/`candle-nn`/`candle-transformers`/`gemm`/`tokenizers`/`onig` in **1m41s**. Static musl is buildable.
- **✅ Toolchain PATH quirk RESOLVED (operator fix).** The earlier blocker — `cargo`/`rustc` on PATH not being the rustup-managed versions, so the target std (`core`/`std`) was unreachable and only an explicit `RUSTC=…/.rustup/… cargo` pin built — is gone. `cargo`/`rustc` now resolve to the rustup proxy with `RUSTUP_HOME`/`CARGO_HOME` set and the stable toolchain active. **No pin needed:** `cargo zigbuild` builds every target straight from PATH. `rust/build.sh` needs no toolchain-pin guard and `BUILD.md` needs no quirk note (neither was ever added).
- **✅ Full 4-target matrix built pin-free.** All four `ka` binaries compiled via PATH `cargo zigbuild` with no pinning: `aarch64-apple-darwin` (~14MB Mach-O), `x86_64-apple-darwin` (~15MB Mach-O), `x86_64-unknown-linux-musl` (~13MB ELF, **static** + stripped), `aarch64-unknown-linux-musl` (~12MB ELF, **static** + stripped). Both musl targets confirmed statically linked. Tools: `rustup 1.29.0`, `cargo-zigbuild`, `zig 0.16.0`. **Built to `target/` only — committed `ka-*` LFS binaries untouched (this was de-risk, not the ship).**
- **✅ Offline retrieval smoke test.** Rust **BM25-only** gate (no model assets present): 83 passed / 2 failed / 13 skipped of 98 — misses `apm-ssi-overview`, `x-deploy-tracking`. The Go **hybrid** gate (BM25+vector via ollama — current production path) passes **85/85**, including both. The 2 misses are the **absent vector arm** in the BM25-only run, **not a Rust regression**: BM25 was audited byte-parity in Phase 1+2 and candle≈ollama at 0.9999 cosine, so Rust hybrid should also reach 85/85.
- **✅ Empirical Rust-candle hybrid eval (acceptance gate).** With the HF fp32 model: offline hybrid **85/85** (= Go), and the **full gate 98/98 with citation-integrity true across all answers** (live gateway, <internal-cli> leg, ~30–40 min) — the forward-verbatim guarantee holds end-to-end in Rust. This cleared the last unknown before cutover.

### Phase 6c — semantic-default cutover EXECUTED ✅ (2026-06-24)
Operator chose **hybrid-by-default** for parity with the retired Go/ollama path. Built on `feat/rust-phase6b-cutover` (rebased onto `main`), merged atomically (binaries + model + code + docs):
- **CLI: semantic is now the DEFAULT** for `query`/`search`/`mcp` (was opt-in `--semantic`). New **`--no-semantic`** forces BM25-only. `eval`/`parity` keep explicit `--semantic` (test harness). Model dir resolves **`--model-dir` → `KA_MODEL_DIR` → `<exe_dir>/model/`** (self-located, no flag). Weight dtype **auto-detected** from the safetensors header (fp16/fp32). **Graceful fallback:** model unavailable → one-line stderr warning → BM25-only, never a hard fail. (Sonnet-built, Opus-reviewed: build clean, 76 tests, clippy `-D warnings` clean.)
- **Model distribution RESOLVED:** fp16 `model.safetensors` (~261MB, converted from HF fp32) + `config.json` + `tokenizer.json` ship **in-repo via Git LFS** at `<module>/model/` (`*.safetensors` added to `.gitattributes`), self-located by `ka`. `vectors.bin` still auto-materializes from the committed `embcache.bin` — **re-embed DEFERRED** as planned (query parity already proven).
- **Binaries:** all 4 `ka-<os>-<arch>` rebuilt with the new code via `rust/build.sh` and shipped (overwriting the Go LFS binaries). Fixed a `build.sh` preflight bug (`cargo zigbuild --version` → `--help`; the subcommand has no `--version`).
- **Docs:** agent `.md` + `refresh-docs-agent` skill updated to semantic-default + self-located in-repo model; refresh eval gates now run `--semantic`.
- **Acceptance (post-change, fp16 self-located):** offline hybrid **85/85**; behavioral spot-check `ka search` → hybrid by default, `ka search --no-semantic` → bm25-only.

## Resume pointer — CUTOVER COMPLETE ✅
Phases 0–6 done. The Rust `ka` is live: **semantic-hybrid by default**, self-contained (platform binary + corpus + in-repo fp16 model + the 3-method auth cascade), **no ollama, no second binary**. Go retired. Acceptance: offline hybrid 85/85 + full gate 98/98 citation-integrity true. Open follow-ups (none blocking):
- **Phase 7** (below) — runtime LLM-call optimization (local candle reranker + semantic answer cache), eval-gated.
- **Optional re-embed** — current vectors are the proven Go/ollama embeddings, query parity verified, so not urgent. **Update (2026-06-25):** a candle re-embed was attempted (Option A — also drops the now-private legacy OP page, 29,538→29,528 chunks) but abandoned mid-embed (the sequential CPU embed stalled the agent watchdog twice). Now folded into **Phase 8.0** — batch the embed loop first, then GPU — rather than retried as-is.
- **`$CLAUDE_PLUGIN_ROOT` in a packaged plugin** — still unverified that it reaches a subagent's Bash; the agent `.md` documents the workspace-relative fallback that works today.

### ▶ Resume here (next session)
Cutover done; agent live. **Status (2026-06-26):**
- **Phase 7.1 reranker — steps 1–4 MERGED** (`8768f79`): `crates/rerank` `CandleReranker` over **ettin-reranker-150m-v1 (ModernBERT)** + the `Reranker` trait, self-location, gateway fallback, adaptive-skip (default OFF). **No model asset ships yet → the gateway reranker is still the live default; runtime behavior unchanged.** Opus-reviewed.
- **Phase 8 (GPU) — SCOPED** (`19be66b`): batching-first, then GPU; decisions settled. See Phase 8.
- **Phase 8 task 1 — batch the embed forward MERGED** (`2f17bf6`): `crates/embed` now runs a padded `[B, L]` batched forward (`plan_batches` + `forward_batch`) replacing the per-chunk loop; env caps `KA_EMBED_BATCH_SEQS`/`KA_EMBED_BATCH_TOKENS` (32/8192). **Batch-of-1 is bit-identical to the old path → query/search/eval parity with the committed corpus is preserved; no re-embed shipped.** CPU-only (device selector is task 2). Delivery-agent-team gated: test-engineer PASS (105 tests, +4 adversarial), principal-engineer ACCEPT (right-padding rotary-position parity + the `embed_one` pooling op-sequence verified against candle `nomic_bert` source). **Empirical 85/85 + re-embed wall-clock + `--device cpu` byte-stability DEFERRED to an asset host** (model + `vectors.bin` are LFS pointers).
- **Re-embed — PARKED/superseded** by Phase 8.0 (see the Optional-re-embed note above).
- **Phase 8 task 2 — device selector MERGED** (`01e812b`, merge `f016619`): `crates/device` (`DevicePref` enum, `select()`, `from_flag_or_env()`); `--device` flag on all 6 embedding subcommands; `Device::Cpu` no longer hardcoded; CUDA arm compile-gated. 116 tests green, clippy `-D warnings` clean. **Empirical acceptance deferred to asset host.**
- **Phase 8 task 3 — Metal feature build code-complete** (`feat/phase8-metal-reembed`): `cargo build --features metal` verified on Darwin/arm64; `build.sh` extended with Metal build step producing `ka-darwin-arm64-metal`. **Empirical acceptance (GPU re-embed << CPU time; 85/85 on GPU vectors) deferred to asset host.**
- **Phase 8 task 4 — cross-device cosine parity gate DONE** (`feat/phase8-parity-gate`): `eval::cosine_similarity` + `eval::cosine_parity_check`; `parity cross-device` subcommand (reads raw f32 LE vec files, asserts cosine ≥ threshold, default 0.9999); 7 new tests (identity, orthogonal, empty/mismatch, pass-at-threshold, fail-on-large-perturbation, pass-on-fp16-noise). 123 tests green.

- **Phase 8 tasks 5+6 — code-complete from task 2** (see task list above; no separate worktrees — wiring landed in `feat/phase8-device-select`).
- **Phase 8 task 7 — distribution docs DONE** (`feat/phase8-dist-metal`): `BUILD.md` GPU section, agent `.md` Metal opt-in, Distribution locked decision revised.

**Phase 8 — code-complete ✅.** All 7 tasks done. Empirical acceptance items outstanding: GPU re-embed timing, 85/85 on GPU vectors, `parity cross-device` run, `cargo check --features cuda` on CUDA host, ettin reranker steps 6–7.

**Metal GPU re-embed — RESOLVED, Metal works (2026-06-26, Phase 7–8 review on branch `feat/datadog-docs-agent-p78`):** the earlier `Metal error no metal implementation for layer-norm` was NOT a candle limitation — it was a local Cargo feature-wiring omission. The workspace `metal` feature enabled `candle-core/metal` but never `candle-nn/metal` (which owns the LayerNorm Metal kernel that ships in candle 0.10.2). Fix: enable `candle-nn/metal` + `candle-transformers/metal` on the embed + rerank crates (upstream-confirmed, [huggingface/candle#2463](https://github.com/huggingface/candle/issues/2463)). A SECOND gap then surfaced empirically: candle 0.10.2's Metal backend has no fp16 `where_cond` kernel (the attention-mask select), so the fp16 model failed with `Metal where_cond U32 F16 not implemented`; the embedder (and reranker backbone) now force **fp32 on Metal** (`device.is_metal() → F32`). Verified on Apple Silicon: full embed runs end-to-end on Metal, ~2.4× faster than CPU on a 150-doc sample, cross-device cosine 0.999953 (≥0.9999 gate). Default device flipped CPU→**auto** (GPU-preferred, loud CPU fallback). Reranker-on-Metal is prophylactic (ettin asset host-gated); full-corpus `--device metal` re-embed still to be run on an asset host. **Metal acceleration is SUPPORTED, not blocked.**

**Rewrite COMPLETE (2026-06-27). Remaining items below are optional enhancements, none blocking:**
- **Phase 8 full-corpus Metal re-embed wall-clock** — Metal is wired + working; a full `--device metal` re-embed timing + byte-stability measurement was never formally captured (the corpus ships from the proven Go/ollama-parity embeddings; query parity verified). Optional.
- **Phase 7.1 step 7 — DONE (via Phase 10/M2, 2026-06-27):** tau calibration sweep run on the ettin asset — tau non-sensitive across 0.1–0.9, all 85/85, `DEFAULT_TAU = 0.3` validated unchanged. See `tests/tau-results.json` + `<internal-tracking>`.
- **Phase 7.2 semantic answer cache** — unstarted (optional; explicitly out of scope of the Go-retirement project).
- **Process:** per-item `feat/…` worktree; review gate per commit; reconcile this plan + `/memory-hygiene` at close.
- **Acceptance (regression guard):** `./ka eval --semantic` — offline hybrid **85/85** + full-gate citation-integrity true.
- **Run the agent:** `./ka-<os>-<arch> query --site <site> --json "<q>"` (semantic hybrid default; `--no-semantic` for BM25). Full design/state: this file.

## Phase 7 — runtime optimization (post-cutover backlog)
**Not part of the rewrite.** The rewrite ships at Go-parity first; these reduce per-query LLM calls afterward. Production currently makes **~2 gateway/LLM calls per query** — rerank + grounded answer. The **answer-generation call is irreducible** (it is the product: the grounded, verbatim-cited answer). Everything below is gated by the eval harness (retrieval + citation-integrity) as a regression guard before landing.

- **7.1 Local cross-encoder reranker (candle).** Replace the gateway-as-cross-encoder rerank with an in-process candle reranker → **2 calls/query → 1**, $0 for the rerank step, no new services. **Model: `cross-encoder/ettin-reranker-150m-v1` (ModernBERT, Apache-2.0)** — BAAI/bge dropped (US BIS Entity-List risk), Jina v2/v3 dropped (CC-BY-NC), mxbai-v2/Qwen dropped (LLM backbone won't fit candle's existing model support). **Steps 1–6 DONE:** the `Reranker` trait + `crates/rerank` (base ModernBERT + hand-rolled raw-logit head, load-time hard-fail shape guard), self-location, gateway fallback (steps 1–4, `8768f79`); head weight-key names verified, per-module VarBuilder loading fixed (step 5, `feat/phase71-key-fix`, merged 2026-06-26); backbone pulled (569 MB curl), key-remapping fix (ettin backbone omits `model.` prefix; prepend before `VarBuilder::from_tensors`) merged (`feat/phase71-backbone-keymap`, 2026-06-26); `CandleReranker::load` confirmed empirically — reranker runs before LLM in live query (step 6). **Remaining (step 7):** `tau` calibration sweep, ordering-fidelity + 85/85 gates. **Refinement — adaptive rerank:** skip the reranker when the hybrid score distribution is confidently separated; default OFF until the calibration sweep proves zero regression.
- **7.2 Semantic answer cache.** Cache `query → answer (+citations)` keyed on the locally-computed candle query embedding (cosine ≥ threshold = hit) → **0 LLM calls on repeat / FAQ-style traffic** — the highest-leverage reduction for real usage, since docs-question repetition is high. Needs a TTL + corpus-refresh invalidation so a refresh drops stale answers.
- **Discarded:** dropping rerank entirely (RRF/dense-only top-k). Rejected — loses top-k precision and diverges from parity behavior; the recall risk isn't worth the saved call when 7.1 gets to 1 call without the tradeoff.

## Phase 8 — GPU-accelerated re-embed + GPU-capable runtime (post-cutover; build-time tooling + a distribution-model decision)
**Not part of the Go-parity rewrite.** Two related tracks: **8.1** a host-local GPU build for the build-time corpus re-embed (never shipped), and **8.2** an operator-approved relaxation of the strict single-portable-binary constraint so the *shipped* binary may use the GPU with a runtime CPU fallback. Today the embedder hardcodes `Device::Cpu` (`crates/embed/src/lib.rs`) and no candle GPU feature is enabled — a deliberate choice for the pure-Rust, cross-compiled, static-musl-Linux portable binary. Embedding is build-time-only, so GPU was never wired.

### 8.0 — Batch the embed loop FIRST (the real bottleneck; do before GPU)
**Operator decision (2026-06-25): batching before GPU.** The embedder loops one chunk at a time (`crates/embed/src/lib.rs:243-246`) — the dominant reason the CPU re-embed ran ~10–15 min and tripped the agent watchdog. A true **batched forward** (one padded `[N,len]` pass) may make the CPU re-embed fast enough on its own, with zero distribution complexity. Implement + measure this first; GPU then compounds it. Acceptance: re-embed wall-clock materially down, offline **85/85** unchanged. **Update (2026-06-26): batched CPU alone is NOT fast enough** — batched CPU re-embed ran 3.5h+ on 29,538 chunks with no completion; aborted. **Metal GPU is the answer and is now working** (Phase 7–8 review: the "missing LayerNorm Metal kernel" was a feature-flag gap, not a candle limitation; fixed + fp32-on-Metal sidesteps the fp16 `where_cond` gap). Re-run the re-embed with `--device metal` (auto-default on the `-metal` binary); full-corpus Metal timing to be measured on an asset host. Q8 quantized weights are an optional future speed-up, not a prerequisite.

### 8.1 — Host-local GPU re-embed binary (NOT shipped)
- **Goal:** cut the build-time corpus re-embed (~25.5k chunks, ~10–15 min CPU) using the host GPU. candle backends: **Metal** (Apple Silicon — this host) / **CUDA** (NVIDIA). Feature-gated; built on and for the operator's machine, used only to regenerate `embcache.bin`/`vectors.bin` faster. **Not** part of the 4-target shipped matrix.
- **Wiring:** add candle `metal`/`cuda` features (gated); a `--device {cpu,metal,cuda,auto}` flag + `KA_DEVICE` env (default `cpu`); replace the hardcoded `Device::Cpu` with the selector. `auto` tries the GPU and falls back to CPU on init failure with a one-line stderr warning — mirrors the model-absent → BM25 and reranker-absent → gateway degradation patterns.
- **Numerical parity:** GPU fp16 matmul can differ slightly from CPU `gemm`. Re-embedded vectors must still pass the **offline 85/85** gate (the wash-out check); spot-check a few cosines vs a CPU run; tolerance is the established ~0.9999.
- **Tiering:** device-selector + feature-gating wiring **Sonnet**; `rust/build.sh` + `BUILD.md` feature-matrix updates **Sonnet**; numerical-parity sign-off **Opus**.

### 8.2 — Relax single-binary: GPU-capable shipped runtime with CPU fallback (operator-approved)
- **Decision (operator, 2026-06-25):** it is acceptable to break the strict single-contained-binary constraint to add GPU acceleration to the shipped runtime, **provided a runtime CPU fallback** kicks in when GPU libs/deps are absent. Applies to the query-time embedder (and the Phase-7.1 reranker), not just build-time.
- **Distribution implication (the real design work):** GPU backends break static portability — CUDA needs glibc + the CUDA runtime (cannot link static musl); Metal links Apple frameworks (macOS only). So the shipped artifact set likely becomes the **portable CPU build (static musl) as the always-works baseline** + **optional GPU variants** (macOS-Metal, linux-CUDA, glibc-dynamic). Runtime fallback: detect GPU-backend init; on failure → CPU + stderr warning.
- **Tiering:** the distribution-model design (artifact matrix + fallback semantics) **Opus/high** (architecturally significant — it revises the **Distribution** locked decision); implementation **Sonnet**.

### Acceptance
- **8.1:** GPU re-embed produces vectors passing **offline 85/85**; CPU fallback verified when the GPU backend is forced off; the shipped 4-target CPU matrix stays byte-unaffected (8.1 changes are feature-gated, off by default).
- **8.2:** each shipped variant passes its platform's gate; the CPU-portable baseline still builds static musl and runs on Alpine/distroless; the fallback path is exercised (GPU libs absent → CPU, no hard fail).

### Decisions (settled 2026-06-25, design spike)
- **Sequencing:** **batching first** (8.0), then GPU. Batching may solve the re-embed speed alone; GPU compounds it + adds query-time value.
- **Ship scope:** do 8.1 **and** ship an **optional macOS `-metal` variant** (8.2); the 4 CPU static-musl baseline binaries stay the always-works, blind-pick default. **CUDA: not built/shipped** (no NVIDIA test host, can't static-link) — wired as a compile-gated abstraction only.
- **Backends:** **Metal-now + CUDA-ready abstraction** — `DevicePref::Cuda` is a real enum arm compiled only under the `cuda` feature (absent in all shipped/tested builds). Don't claim CUDA works.
- **Default device:** shipped default = **`cpu`**; GPU is explicit opt-in (`--device metal|auto` / `KA_DEVICE`). The host-local 8.1 re-embed binary may default to `auto`.
- **Canonical GPU vectors:** **accepted** — GPU-built `embcache.bin`/`vectors.bin` may be committed as canonical once they pass a **cosine ≥0.9999 spot-check vs a CPU reference + the offline 85/85 gate** (they won't byte-match CPU; the embcache contract already compares by record-set).
- **Device selection:** shared `device::select(DevicePref) -> Device` helper (leaf crate so `embed` + `rerank` share it); `auto` probes the backend and degrades to CPU + one-line stderr warning — mirrors the model-absent→BM25 / reranker-absent→gateway idiom. Replaces the hardcoded `Device::Cpu` (`embed/src/lib.rs:141`); threaded into both candle models.
- **Distribution naming:** extend `ka-<os>-<arch>` → `ka-<os>-<arch>[-<accel>]`; the agent's blind pick stays the baseline name, `-metal` chosen only on explicit GPU opt-in when present.

### Task plan (batching-first; each its own worktree; the offline 85/85 gate is the regression guard throughout)
1. **Batch the embed forward** — `feat/phase8-batch-embed` — **Sonnet/M**. Padded `[N,len]` batched pass. Acceptance: re-embed wall-clock materially down; 85/85 unchanged; `--device cpu` byte-stable. — **DONE ✅ (`2f17bf6`)** `plan_batches`+`forward_batch`, env caps, batch-of-1 byte-identical. Code+test gates green; **empirical acceptance (85/85, wall-clock, byte-stability) deferred to asset host** (see Resume "Phase 8 task 1 asset-host verification").
2. **Device selector + thread into embedder** — `feat/phase8-device-select` — **Sonnet/M**. `device::select`+`DevicePref`, `--device`/`KA_DEVICE`, replace `Device::Cpu`. Acceptance: `cpu` byte-identical to today; `auto` w/o GPU → CPU + warning; 85/85 unchanged. — **DONE ✅ (`01e812b`)** `crates/device` (11 tests, `DevicePref` enum, `select()`, `from_flag_or_env()`); `--device` flag wired into all 6 embedding subcommands; `Device::Cpu` no longer hardcoded in embed/rerank. CUDA arm compile-gated. **Empirical acceptance deferred to asset host.**
3. **Metal feature build + host GPU re-embed (8.1)** — `feat/phase8-metal-reembed` — **Opus/L**. — **DONE ✅ (code-complete)** `cargo build --features metal` verified on Darwin/arm64; `build.sh` extended with Metal build step producing `ka-darwin-arm64-metal`. **Empirical acceptance (GPU re-embed << CPU time; 85/85 on GPU vectors; cosine ≥0.9999 vs CPU) deferred to asset host** (model assets are LFS pointers).
4. **Cross-device cosine parity gate** — `feat/phase8-parity-gate` — **Sonnet/M**. Extend `eval`/`parity` with the cosine spot-check. Acceptance: passes on GPU vectors, fails on an injected >1e-4 perturbation. — **DONE ✅** `eval::cosine_similarity` + `eval::cosine_parity_check` (pure Rust, no assets); `parity cross-device` subcommand reads raw f32 vec files; 7 adversarial tests (identity, orthogonal, empty/mismatch errors, pass at threshold, fail on large perturbation, pass on fp16-noise-level perturbation). **Empirical run on GPU vectors deferred to asset host.**
5. **CUDA-ready abstraction (compile-gated, untested)** — `feat/phase8-cuda-stub` — **Sonnet/S**. — **DONE ✅ (code-complete from task 2)** `DevicePref::Cuda` arm + `cuda` feature chain (`cli→embed/rerank→device→candle-core/cuda`) wired and compile-gated; `Device::new_cuda(0)` called when feature enabled. Default build unchanged (feature absent from all shipped builds). `cargo check --features cuda` deferred to a CUDA host.
6. **Reranker (7.1) device wiring** — `feat/phase8-rerank-device` — **Sonnet/M**. — **DONE ✅ (code-complete from task 2)** `CandleReranker::load_with_tau` takes `device_pref: DevicePref` and calls `device::select()`; `load()` delegates to `load_with_tau(…, DevicePref::Cpu)`. Acceptance (Metal + CPU-fallback with live ettin model) deferred to asset host (steps 5–7 of Phase 7.1).
7. **Ship optional Metal variant + build/release + docs** — `feat/phase8-dist-metal` — **Opus/M**. — **DONE ✅** `BUILD.md` GPU section (when to use Metal, operator opt-in steps, parity check runbook); agent `.md` Metal opt-in comment + note; **Distribution** locked decision revised (see below). Baseline 4-target build unchanged; `ka-darwin-arm64-metal` is an explicit opt-in, not auto-picked.

8.2/task 7 revises the **Distribution** locked decision (single portable binary → baseline-CPU + optional GPU supplements) — update that bullet and reconcile per `governance-memory.md` when task 7 lands.

## Phase 9 — Pipeline-parallel embed (tokenize-ahead-of-forward) — CLOSED with data ✅

**CLOSED, not built (2026-06-26, Phase 7–8 review).** Profiling on the real model (small synthetic sample, Metal-host) measured tokenize at **0.1%** of per-batch wall-clock — embed is overwhelmingly forward-bound (forward ≈ 1600× tokenize). Overlapping tokenization with the forward pass would yield ~0.1% improvement, far below the ≥15% gate, so the producer/consumer pipeline is **not worth building**. The real re-embed speed-up is the GPU/Metal path (Phase 8), now working. The profiling harness ships behind `#[ignore]` (`cargo test -p embed -- --ignored`). Original design retained below for the record.

**First `build-with-agents` target.** The Phase 8 batch forward made embedding correct and multi-core, but each batch is still fully sequential: tokenize batch → forward batch → repeat. On a 29,538-chunk cold re-embed (~924 batches), tokenization is pure-CPU work that sits idle while the forward pass runs. Overlapping them could cut wall-clock proportionally to tokenization's share of per-batch time — worth measuring first.

### Goal
Overlap `encode_batch` tokenization of batch N+1 with `forward_batch` computation of batch N using a bounded channel. Producer thread pre-tokenizes ahead; consumer (main thread) drains and runs forward passes. Output order is preserved; byte-identity for same-input is unchanged.

### Design sketch
- Producer: thread calling `tokenizer.encode_batch` for each `plan_batches` range, sending `Vec<(Vec<u32>, Vec<u32>)>` into a bounded `std::sync::mpsc` or `crossbeam::channel` (bound = 2–4 batches ahead, so memory stays bounded).
- Consumer: current forward-pass loop drains the channel.
- Constraint: `Tokenizer` must be `Send` (it is — `tokenizers` crate is `Send + Sync`); the candle model is NOT `Send`, so the forward loop stays on the main thread.
- Fallback: if the channel approach adds measurable complexity for negligible gain (profile first!), document and close.

### Prerequisite: profile
Before implementing, measure tokenization vs forward-pass time per batch on the current code (add per-batch timing to the progress line). If tokenization is <10% of per-batch wall-clock, the gain is marginal and the phase should be closed as "profiled + not worth it." If ≥15%, proceed.

### Acceptance
- Re-embed wall-clock materially lower than Phase 8 baseline (same corpus, same machine, both with empty embcache).
- Offline **85/85** retrieval gate unchanged on the resulting `vectors.bin`.
- `--device cpu` byte-stability: two pipeline-parallel re-embed runs produce identical `vectors.bin`.
- No regression on single-chunk query-time embed (single batch path untouched).

### Tiering
- Profile + design: **Sonnet/M**
- Implementation + tests: **Sonnet/M**
- Principal review: **principal-engineer**

## Phase 10 — Retire Go + obsolete-artifact cleanup (DONE ✅)

**Status: DONE — merged to `main` 2026-06-27 (`bc85b72`, signed `--no-ff`).** Delivered via the `docs-agent-go-retirement` build-with-team project (full design/plan/execution: `<internal-tracking>`). The Go layer is gone; the Rust `ka` is the sole runtime. Acceptance: `cargo test` 165 pass, clippy/fmt clean, hybrid offline **85/85**, self-located query verified; 13/13 plan success-criteria audited ACCEPT.

### What shipped (4 milestones)
- **M1 — Retire Go + reorg:** removed `go.mod`/`go.sum` + the 38 `.go` files + Go dirs `agent/ cmd/ eval/ mcp/ resolve/ retrieve/ service/ taxonomy/`. Reorganized: binaries → `<module>/.bin/`; golden set `eval/questions.yaml` → `tests/questions.yaml` (the trap file — relocated + the two `crates/cli/src/main.rs` defaults updated before `eval/` was removed). `build.sh` → thin `exec rust/build.sh` wrapper. parity.rs Go-vs-Rust `diff` path neutered, Rust-only subcommands kept.
- **M2 — Tau calibration:** swept the ettin reranker over the 85 golden questions — **tau non-sensitive across 0.1–0.9 (all 85/85, ordering 0.60); `DEFAULT_TAU = 0.3` validated unchanged.** Landed a ~9× sweep optimization (score-once + in-memory threshold), a `score_chunks()` primitive, and a `--json-out` flag → `tests/tau-results.json`. (This closes **Phase 7.1 step 7**, the last open reranker item.)
- **M3 — Disk reclaim + LFS hygiene:** `model-rerank/` was an untracked local HF clone (0 files tracked) — purged its `.git` (3.2 GB) + `onnx/` (859 MB) + `openvino/` (2 MB) and `corpus/_template` (~4.1 GB reclaimed), then committed the 6 candle-loaded reranker files via Git LFS so **the local reranker now ships**. All large committed binaries are LFS; `rust/target` + `corpus/vectors.bin` gitignored.
- **M4 — Rebuild + reship binaries:** rebuilt all platform binaries via `rust/build.sh` so shipped artifacts match source. **`ka-darwin-arm64-metal` is now SHIPPED via LFS and the default pick on Apple Silicon** (Metal, CPU auto-fallback); the unqualified host-native `ka` is **removed** — all invocations use fully-qualified `ka-<os>-<arch>[-metal]`.

### Bonus fix (regression caught during M3)
The M1 `.bin/` move silently broke `ka`'s self-location: `exe_dir()` returned `.bin/` so flagless `ka query`/`search` couldn't find `regions.json`/`corpus/`/`model/`. Fixed (`d3a6623` + unit test): `exe_dir()` strips a trailing `.bin` segment → resolves the module root. Also fixed the agent.md `$KA` resolver path (still pointed at the agent root, not `.bin/`). Shipped in the M4 rebuild.

### Remaining (optional — NOT part of the rewrite)
- **Phase 7.2 semantic answer cache** — unstarted; explicitly out of scope of the retirement project.
- **CUDA `cargo check --features cuda`** — host-gated (no NVIDIA host); compile-gated code only.
- **Cross-platform binary execution** — the 4 non-darwin-arm64 binaries are committed LFS objects, not yet executed on their native hosts (deferred per this plan).
