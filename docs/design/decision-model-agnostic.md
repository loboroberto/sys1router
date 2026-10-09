# Design: make sys1router decision-model agnostic (reviewed)

> Status: approved roadmap, tracked on the GitHub issue and milestone board. This document is the reviewed design: verdict, keeps, ranked changes, the revised PR roadmap, answers to the open owner decisions, and (Appendix A) the original plan under review. Gate G decisions D1 to D4 were recorded on 2026-10-09 (see "Gate G decisions"); they override Appendix A wherever the two disagree.

## Gate G decisions (recorded 2026-10-09)

| Issue | Decision | What it changes |
|---|---|---|
| D1 fork (#2) | **Hard-fork** from BillionsBobby/JevRouter. | No upstream merges. Drop the `upstream-sync` label exemption from the frozen-file guard and the rule that keeps `buildQuestions`/`describeCapability` in `provider.ts` (P1 may move them to `src/decision/questions.ts`). "Byte-for-byte unchanged" in Appendix A becomes "behaviour pinned by the P0 goldens, the legacy-contract test and the `legacy-suite` job". |
| D2 identifiers (#3) | Env prefix `SYS1ROUTER_*`. User config `$XDG_CONFIG_HOME/sys1router/config.json` (win32: `%APPDATA%\sys1router\config.json`). Project config `sys1router.config.json`. `receipt_schema: "sys1router.receipt/2"`. The `./spec` subpath is defined relative to the package; its import prefix follows the npm name chosen at P-R. | P1 defines all five as constants in `src/product.ts`. Appendix A's `jevrouter.config.json` and `jevrouter/spec` are superseded. npm name, bin, publishing, version mapping and `.jevrouter/` stay at P-R (#22). |
| D3 Node floor (#4) | **Raise `engines.node` to `>=22`.** Node 20 reached end of life in April 2026. | P0 updates `package.json` `engines`, the README badge and both "Node.js 20+" lines (English and 中文), and CONTRIBUTING. CI stays on Node 22 with no 20/22 matrix. P0 still adds one `windows-latest` job on Node 22 for the win32 config-dir fallback; symlink-escape tests skip on win32. Agent-render snapshots stay as checked-in JSON. The CHANGELOG created in P1 lists the floor change as user-facing. |
| D4 playground host (#5) | **Pause auto-deploy now; choose the host at P10.** | P0 switches `pages.yml` to `workflow_dispatch` only, so merges stop deploying `site/` under the `www.jevrouter.co` CNAME. P0 is unblocked. The host question stays open on #5 and must be answered before P10 (#21): Cloudflare Pages means P10 commits `wrangler.toml` and a deploy doc; GitHub Pages only means cutting `functions/api/jev.js` and moving the demo client-side or removing it. |

Still open in M0: the fixture spike D5 (#6), and the host half of D4.


## Context

The owner asked for a fresh set of eyes on their design plan (Appendix A, verbatim) before implementation. This file is the review: verdict, what to keep, what to change, a revised roadmap, and answers to the five open owner decisions.

**The product of this plan is a GitHub issue and milestone board** on `loboroberto/sys1router` that tracks the revised roadmap: one milestone per wave, one issue per PR, decision issues for the owner gates, and one tracking epic with sub-issues. Nothing is implemented in one shot; each PR is picked up from its issue later. The board design and the creation steps are in "Board design" below.

Method: every `src/` file was read directly; the plan's line citations all match this checkout. A 46-agent review ran on top (1 Haiku fact-checker, 5 critics on distinct lenses at the session model, 34 Sonnet skeptics trying to refute each finding, 1 completeness pass). 30 findings survived, none were fully refuted, 6 gaps were added. Everything below was cross-checked against the code.

## Verdict

The architecture is right and the security posture is unusually good for a local CLI. Three things need fixing before any code is written:

1. **Sequencing.** Two owner decisions the plan defers to P-R (hard-fork vs upstream, and the product identifiers) are silently assumed from P0 onward. "Early value at P4" is not real: about 2,000 lines of refactor and test infrastructure precede the first non-TypeSafe route, P4 itself cannot fit the 800-line rule, and agent users cannot reach a new provider until P8 because the v2 helper pins `JEV_ROUTER_PROVIDER`.
2. **Three spec-level ambiguities that would cause rework:** whether a legacy `JevProvider` goes through the bridge or the unchanged path; what `gateFor` does when a native provider returns probabilities but no confidence; and how chunked staging picks finalists without comparing probabilities across calls (the very bug owner decision 4 names).
3. **Size.** The repo is ~2,900 lines of `src/` and CONTRIBUTING says "small on purpose". The plan adds roughly 3 to 4 times that. P11's integrity-pinned plugin loader, P12's automation, seven emulation flavors, eleven policy knobs and a four-layer trust lattice have no current user. Cutting them keeps all four binding owner decisions intact.

## Keep (credited by the reviewers, confirmed in code)

- P0 characterization tests before any refactor; the HTTP wire and error mapping have no tests today.
- "Legacy frozen, new triggers only", narrow public unions with widening in new names, `defaultPolicy` gains no keys (the shallow spread at `manifest.ts:149` passes an unknown nested key through untouched, so `policy_hash` stays stable).
- `DecisionProviderError` as one class with a `Symbol.for` brand and `.of(kind)`; `router.ts:325,431` are the only `instanceof` sites.
- `CompiledQuestion`: adapters never see `CapabilityManifest`. Zero-I/O preflight gated by declared primitives doubles as spec minor-versioning.
- Normalization rules (never rescale, partial gives null, Vercel sentinel) match the "never re-normalize" contract.
- Two-tier config trust, `--endpoint` keyless-only, no `Authorization` to keyless presets, attachments bytes-only with hashes in receipts, MCP `model` input deny-by-default.
- `status_cap` in finalize; `jev_probability` null unless native so v1 readers fail closed (no in-repo reader dereferences it: dashboard reads `provenance.jev_provider`, `runtime.*`, `status`, `decision.selected` only).
- `upcastReceipt()` instead of rewriting append-only files; `buildRuntime()` for serve/MCP receipts closes a real gap (serve saves the bare result today, so the dashboard undercounts it).
- `createProvider` staying sync and legacy-only; cache key unchanged for the default endpoint.
- The P6/P9 adapter diff guard is enforceable today: `--model` already flows through every surface, and `src/providers/**` does not exist yet.

## Changes, ranked

### A. Decide first, then build (zero-code gate before P0)

1. **Hard-fork vs track upstream.** Every upstream merge since 2026-09-19 touched `router.ts`, `provider.ts`, `cli.ts` or `types.ts` plus one of the ten test files the plan freezes. No upstream remote is configured, the repo is renamed, and 1.0.0 ships a new receipt schema. The evidence says hard-fork. Decide it now: if hard-fork, drop the `upstream-sync` CI exemption and the "keep `buildQuestions` in `provider.ts` to limit merge conflicts" rule, and reword "byte-for-byte unchanged" (P1 already edits `JevProviderError` and `CachedJevProvider`) to "behaviour pinned by the P0 goldens and legacy-contract test". **Decided: hard-fork.**
2. **Identifiers that get persisted or published before P-R.** `receipt_schema: "sys1router.receipt/2"`, `$XDG_CONFIG_HOME/sys1router`, `SYS1ROUTER_*`, `jevrouter.config.json` and the `jevrouter/spec` subpath are mutually inconsistent and land in P2 to P4, in append-only receipts and committed config. Fix the receipt schema id, env prefix, user-config dir, project-config filename and export prefix in one table now, define them as constants in one module (`src/product.ts`), and move "point install docs at this fork" plus the CHANGELOG into P1. Leave npm name, bin, publishing and version mapping at P-R. **Decided: the proposed `sys1router` set (table above).**
3. **Node floor and Windows.** `engines` says >=20 but CI runs only 22, and P0 adds snapshots, a `--import` preload and `AbortSignal.any` (20.3+). Either raise `engines` to >=22 at this gate or add a 20/22 matrix in P0. Define the config dir with a win32 fallback (`%APPDATA%`), skip symlink-escape tests on win32. **Decided: raise to >=22, no matrix.**
4. **Where the playground runs.** `pages.yml` deploys `site/` to GitHub Pages on every push to `main`; `functions/api/jev.js` is a Cloudflare Pages Function with no `wrangler.toml` or workflow in the repo. P10's `env.AI` binding is unreachable from the only deploy the repo knows, and `site/index.html` still points at upstream five times. Set `pages.yml` to `workflow_dispatch` until names settle; if Cloudflare Pages is the real host, commit `wrangler.toml` in P10; if not, cut the function from the roadmap. **Decided: dispatch-only in P0; host chosen before P10.**
5. **Record one real fixture from each first-wave target before P3** (llama.cpp `/v1/systemone`, Strands, Clef). The plan says nothing hard-depends on the unverified facts, but P4's "early value" is exactly those presets. An hour of manual recording de-risks three phases.

### B. Sequencing and scope

6. **Re-sequence for real early value:** P0 → hardening PR → P1 → P3a → P4a → P8-min → P4b → P2 → P5 → P6 → P7 → P9 → P10 → P-R → P11-lite → P13. Details in the revised roadmap below. Split P3 (types/bridge/normalizer/constructor first; policy knobs with their first consumer), split P4 (adapter + http + credentials + `--model`/`--endpoint` wiring first; config files, registry, presets, CLI commands second), and pull the v3 helper (`--model` injection) up to right after the first adapter so agent users get it at the same time as CLI users.
7. **Serve hardening is its own first PR,** not part of P1. It is the one intentional default-on change to a legacy surface (DNS rebinding against a loopback server); say so in Approach. Apply the same Host allowlist to the dashboard, which serves every receipt's `routing_input.context` over HTTP with no check, and require `Content-Type: application/json` on `POST /route`.
8. **Cut P12; slim P11.** The weekly catalog job gates itself on >10 entries that the roadmap never reaches. Keep `./provider-sdk` (types), the loader, and `docs/provider-authoring.md`; keep conformance as an internal test helper used by built-in adapters; drop the `conformance` CLI, the provider-template package with its own lockfile, `models refresh|diff` and both workflows to the deferred list with their trigger written down.
9. **Trim P9 to four flavors** recordable offline today (`openai-chat`, `vllm`, `llamacpp-chat`, `ollama-native`); `together`, `fireworks`, `workers-ai-chat` become example config rows added when a fixture exists. Drop the memoized whole-decision argmax fallback from the first cut.
10. **Ship four policy knobs, not eleven:** `require_native_probabilities`, `min_confidence_by_calibration`, `on_unsupported`, `allowed_models`. Add `max_calls_per_decision` and `concurrency` with P5, `min_coverage` with P9. Defer the rest until a provider needs them.
11. **Collapse the four-layer capability lattice to three steps:** adapter defaults (alone own `probability.*`, `locality`, transport) → one data row per model id where a user-config row replaces the catalog row wholesale (plugins and user-added vendors are just row sources) → project config and policy apply `narrow()` only. One function to test; the 8-image opt-in becomes "copy the row and edit it".

### C. Spec corrections

12. **Cut `legacyEnvelope: JevRawResponse` from `DecisionResult`.** It leaks Jev into the neutral seam, duplicates `raw[0]` for every System One dialect, and makes emulation adapters fabricate a Jev body that v1 readers would trust while the same receipt nulls `jev_probability`. Derive `raw_jev` in `receipts.ts`: `raw[0]` (and stages) when the adapter dialect is in the systemone family (OpenRouter envelope included), null otherwise. Re-point `route-command.ts:40,93,103,125` (`provider_response_received`, `collectPlanUsage`, `probeJev`) to `raw.length`/`usage_normalized` so a null `raw_jev` does not read as "no response".
13. **Pin the legacy path explicitly in P3:** `new JevRouter(p)` with a duck-typed `JevProvider` (has `decide`, no `specificationVersion`) runs today's `decideWithStages`/`finalize`/`thresholdFor` unchanged; the router never inserts the bridge. The bridge serves `evaluateDecision()`, conformance and explicit callers only, reads raw `confidence` presence before `getChoiceAnswer` (which hides it at `provider.ts:304`) to set `confidence_source`, and keeps strict `malformed` throwing. Add one test: same stub through both paths, legacy receipt deep-equals the golden.
14. **Between P3 and P5 a spec model has no cap.** Add a ten-line guard in P3: spec models route through a single call only; if `N > max_options` (or `max_options` is null and `N > single_stage_max_candidates`) return `no_decision` with `error.kind: unsupported_capability` and zero calls, until staging replaces it.
15. **Retry, per-call timeout, concurrency live in the guarded client,** so adapters get them for free: per-call timeout = `min(adapter default, remaining decision deadline)`; retry `rate_limited`/`overloaded`/`network` only, jittered, honours `Retry-After`, every attempt counted against `max_calls_per_decision`; fan-out and chunking use a `concurrency` knob (default 2). Make `defineProvider` an identity type helper (`satisfies ProviderPlugin` with `import type`) so "jevrouter is only a types dev dependency" is true.
16. **SDK surface:** resolve the contradiction between "`createProvider` stays sync and legacy-only" and "`resolveDecisionModel` at `api.ts:82`". `createSdkProvider` stays sync and throws `invalid_request` if `modelRef` is set; `route()`/`plan()` branch on `modelRef`; `provider` plus `modelRef` is `invalid_request`; `evaluate()` with a spec model rejects in favour of `evaluateDecision()`. Pin all four in `compat-identity`.
17. Trim enums: drop `unspecified` from `ProbabilitySource` (legacy gets an explicit source) and `heuristic` from `CalibrationClass` (demo is `uncalibrated` and exempt). Low priority.

### D. Decision semantics corrections

18. **One confidence ladder in `gateFor`,** replacing the contradictory "missing confidence fails closed" and `derived_concentration`: (1) native confidence when present; (2) otherwise `max(p)` over answered labels, only when coverage >= `min_coverage`, recorded as `derived_concentration`, and subject to the non-vendor floor; (3) otherwise unavailable → `probability_free_answers` (confirm|reject). Without this, a native provider that omits `confidence` (llama.cpp and Strands are unverified on exactly this) returns `no_decision` on every route. Make the P3 gate test assert `fallback.type` and `confidence_source`, with a vector that passes `max(p)` but fails an entropy form so the estimator is pinned.
19. **Gate composition and calibration table.** `gate = max(clamp(min_confidence ?? 0.55), min_confidence_by_model[ref] ?? min_confidence_by_calibration[class] ?? (class === "vendor" ? 0 : 0.75))`, demo and legacy exempt from the second term. Add the table: typesafe/openrouter-decisions → vendor; keyless System One presets and workers-ai → vendor_claimed (0.75 until a live conformance run promotes the entry); openai-compatible → uncalibrated; foreign-host `JEV_API_URL` → unknown.
20. **Write chunked staging as arithmetic:** `chunks = ceil(N/cap)`, balanced; finalists = `max(chunks, K)` ids by round-robin over within-chunk rank (every rank-1, then every rank-2…), never by comparing probabilities across calls; if finalists > cap, recurse. Add `calls(N, cap, K, Q, max_questions)` to preflight, give `max_calls_per_decision` a default derived from it, reject above it with zero I/O. Worked example: 300 at cap 24 → 13 chunks → 13 finalists → 1 final call = 14 calls; at the P9 cap of 10 → 30 + 3 + 1 = 34. Put it in the preflight test next to 49 → 17/16/16.
21. **`context-fit` has no estimator anywhere.** Wave 1: bytes only, `chunks = max(ceil(N/cap), ceil(rendered_bytes/max_request_bytes))`, `context_tokens` informational, preflight refuses when `truncates_silently` and no byte bound is known.
22. **P9 estimators:** coverage := summed probability mass of valid labels (not label count); the no-rescale invariant starts at the adapter boundary (raw logprobs to `raw_provider`, within-label-set normalized distribution plus coverage to `answers`); `max_options` declared by the adapter from the flavor's real `top_logprobs` cap (OpenAI and vLLM default 20 → the plan's `/2` gives 10 options per call); per-flavor label-slot divisor, with `logit_bias` label restriction only where fixtures show logprobs stay faithful; score rejects >10 levels under the digit strategy; boolean `P(yes) = p_yes/(p_yes+p_no)`.
23. Beam skips only on the probability-free flag (not any null; `beamSelectSequence` already maps missing to 0). Spec-path hierarchical non-members get null, stage `coarse`, unselectable. Move the 70-questions-at-64 test to `evaluateDecision` (plan steps are capped at 10).

### E. Compatibility corrections

24. **Env conflict is a warning, not an error.** The v2 helper spreads `process.env` then pins `JEV_ROUTER_PROVIDER` (`agent-setup.ts:109`), helpers are committed under `.agents/.claude/.cursor` (not gitignored), and SKILL.md tells the agent to continue unrouted on error. So a user-level `SYS1ROUTER_PROVIDER` would break every repo with a v2 helper. Pin wins; receipt `warnings[]` gets `env_conflict`; stderr gets one line; `agent doctor` reports it; `SYS1ROUTER_STRICT_CREDENTIALS=1` opts into the error. The P4 env-precedence test asserts the pin is honoured and the warning emitted.
25. **Replace the byte-freeze with a `legacy-suite` CI job:** run the merge-base `tests/*.test.ts` (explicit `node --import tsx --test <base>/tests/*.test.ts`, not head's `npm test` script) against head `src/`. That proves the real invariant (old assertions pass on new code) and permits in-place fixes (`router.test.ts:165` writes `.jevrouter/.cache-tests` under the repo root; `:275` uses a fixed `/tmp` path). Keep `tests/frozen.txt` as a CODEOWNERS path.
26. **Single-emit raw bodies; dual-emit scalars.** A two-stage receipt would otherwise store the final body four times, and the dashboard parses every receipt every 5 s. `raw_jev`/`raw_jev_stages` keep today's content; `raw_provider*` only when the wire body differs from the envelope, else `provenance.decision_model.raw_in: "raw_jev"`. `jev_stage` stays populated (router fact, asserted at `router.test.ts:138`); null only `jev_probability`/`jev_confidence` when not native. This answers owner decision 5 now.
27. **`policy_hash` stays the hash of the loaded policy** (goldens hold, cookbook 08's audit claim stays true); add `provenance.decision_model.effective_policy_hash` plus the small resolved `decision_model` block.
28. **Logging:** `src/log.ts` in P1, stderr only, one format, `SYS1ROUTER_VERBOSE=1` adds per-call lines; rule that only `cli.ts`/`mcp-server.ts` write stdout (one stray `console.log` breaks serve-mcp framing). Receipt `warnings[]` mirror to stderr once and to MCP `content`. Extract `startRouteServer()` so serve-security runs in-process on port 0 like `dashboard.test.ts`.
29. **Catalog and schemas must ship.** `catalog/decision-models.json` at repo root is outside `rootDir: "src"` and npm `files`; an npm-published package would lose it, and preflight would do a `readFile` per CLI start. Keep it under `src/catalog/` with `resolveJsonModule` (static import, zero I/O, emitted to `dist`). Add `exports` entries for `./spec`, `./provider-sdk` (types first) and an `npm pack --dry-run` assertion to `compat-exports`. Keep `examples/provider-template` out of `files`.
30. MCP: drop the `--mcp-tools legacy|both|neutral` tri-state; keep `jev_route`, add `decision_route` as a second name in P10, flip the instructions text at P13.

### F. Security corrections

31. **One credential rule, tested twice.** Today the plan pins well-known keys to vendor hosts yet lets legacy `JEV_API_URL` send the TypeSafe key cross-host, and never says whether trusted user config may do the same. Rule: pinned by default; the user tier (env `JEV_API_URL`, or an explicit user-config endpoint) may rebind with a `credentials.cross_host` warning and calibration `unknown`; the project tier never can. Pins are per host, not per path (so P9's OpenRouter chat completions on the pinned host are fine). Tests: user-tier rebinding accepted and warned; project-tier refused with zero fetches.
32. **SSRF: structural, not content filtering.** "`169.254.169.254` in state never reaches the wire" cannot hold for opaque state that is the wire, and rejecting `{type:"image_url"}`-shaped objects fails legitimate context. Invariant: System One adapters send `state` verbatim; OpenAI-compatible adapters render state as one `{type:"text"}` part and never spread caller objects into `content`; images come only from the bytes-only encoder. Test by snapshotting the emulation body.
33. **Plugin loader without "integrity".** npm's SRI covers the tarball and cannot be recomputed from an extracted tree; `import("pkg")` from `dist/` resolves through the workspace `node_modules` the plan forbids. User config lists `{name, version, path}` with an absolute path; loader realpaths it, refuses anything under the project root, checks `version` and `jevrouter.spec`, `import(pathToFileURL(entry))`. Optional directory digest later.
34. **Policy narrowing asymmetry.** A cloned repo can still set `min_confidence: 0` and `confirmation_risk_levels: []` in `.jevrouter/policy.json`, so narrowing-only on `policy.decision_model` protects little. Keep the two floors in code (probability-free never auto-selects; `require_native_probabilities` and `allowed_models` cannot be loosened by project files), let thresholds be project-owned like today, and say so in one sentence: "project files choose policy; project files never choose credentials, hosts or plugins."
35. **Invariant 5 as written is unprovable:** receipts store caller `context` verbatim by design. Reword to router-held credential values, media bytes, account ids and endpoint userinfo, and test by searching written files and thrown messages for the configured key values. Give the P7 serve body cap a number (`max_items × bytes_each × 4/3 + margin`; the existing 1 MB cap is below four images).
36. Pages function: candidate count and per-field length caps plus a same-site `Origin`/`Sec-Fetch-Site` check (browser abuse only; note that real abuse protection is Cloudflare rate limiting).

## Revised roadmap

| # | PR | Contents | ~src lines |
|---|---|---|---|
| G | Owner gate | Items 1 to 5 above. Zero code. | 0 |
| P0 | Characterization | Goldens including a deliberately named `legacy-two-stage-coarse-only-selected` divergence golden; legacy-contract via the existing `provider-stub.mjs` fetch-replacement pattern; agent-render snapshots as checked-in JSON; `legacy-suite` CI job; `engines` >=22 with README/CONTRIBUTING, one `windows-latest` job; `pages.yml` to dispatch; design doc; manual fixture recording. | 0 |
| P0.5 | Hardening | serve + dashboard Host allowlist, Content-Type, `--token`. The one default-on legacy change. | ~60 |
| P1 | Names | Aliases, `DecisionProviderError` kind/brand/of, `getBooleanAnswer`, `src/product.ts`, `src/log.ts`, install docs → fork, CHANGELOG. | ~250 |
| P3a | Spec | `src/spec/*` + `exports`, normalize, bridge (evaluateDecision/conformance only), constructor union with the single-call guard, gate ladder, probability-free rule, `status_cap`, minimal receipt provenance (`provenance.decision_model`, `warnings[]`, `receipt_schema`). | ~450 |
| P4a | First adapter | Guarded http (retry, per-call timeout, bind hosts), `ScopedCredentials` with the one rule, systemone adapter + 2 dialect rows, `--model systemone:<m> --endpoint <loopback>` wired into every surface, SDK branch on `modelRef`. **First Strands / llama.cpp route here.** | ~500 |
| P8-min | Agents | v3 helper injecting `--model`, flag stripping, `integration-v3.json`, env-conflict warning + doctor. | ~120 |
| P4b | Config | User/project config, presets as rows, `providers list`, `config validate`, `models probe`, cookbook 10. | ~450 |
| P2 | Receipts v2 | Full scalar dual-emit, single-emit bodies, `usage_normalized`, `effective_policy_hash`, upcaster, `agentView`. | ~350 |
| P5 | Capabilities | 3-step resolution, `src/catalog/*.json`, preflight with `calls()`, staging as arithmetic, bytes-only context-fit, fan-out with concurrency. | ~600 |
| P6 | Cloudflare text | As planned. | ~250 |
| P7 | Images | Structural state invariant, inspector, encoder, `--attach`, numeric body cap. | ~500 |
| P9 | Emulation | 4 flavors, logprobs|argmax, mass coverage, adapter-declared `max_options`. | ~500 |
| P10 | Surfaces | `decision_route` alias, dashboard grouping, Pages only if Cloudflare is the host. | ~250 |
| P-R | Release gate | npm name/scope, bin, publishing, version mapping, `.jevrouter/`. | 0 |
| P11-lite | SDK | `./provider-sdk` types, path-based loader, internal conformance helper, authoring doc. | ~300 |
| P13 | Neutral-first | As planned. | small |

Deferred with written triggers: P12 (catalog >10 entries or first external plugin), remaining P9 flavors (a recorded fixture), the other seven policy knobs (a provider that needs one), Bedrock CMI, extra dialects, more encoders, `x-*` primitives, fallback chains, calibration fitting.

### Change footprint: existing files vs new modules

About 4,600 `src/` lines in total: ~800 in existing files (17%), ~3,800 in new files (83%). Of the ~800: ~60 change legacy behaviour by default (serve and dashboard Host allowlist, the one intentional exception); ~250 restructure legacy logic behaviour-neutrally (a legacy branch must reproduce today's output, pinned by the P0 goldens); ~480 are purely additive inside an existing file.

| Existing file | Lines touched (est.) | Kind |
|---|---|---|
| `router.ts` (544) | ~200 across P3a, P2, P5 | Constructor union, spec branch in `finalize`, `gateFor` with unchanged legacy branch, `status_cap`, brand check at `:325,431`, staging hook. |
| `cli.ts` (297) | ~150 | Serve/dashboard hardening (default-on), new flags and commands, `--attach`, extracted `startRouteServer`. |
| `agent-setup.ts` (228) | ~100 | v3 path added alongside v2; v2 code and templates untouched. |
| `types.ts` (263) | ~80 | Additive only: aliases, `policy.decision_model`, `attachments`, optional receipt fields. |
| `route-command.ts` (126) | ~50 | `modelRef` branch at three sites, receipt assembly via `receipts.ts`, re-pointed `raw_jev` readers. |
| `dashboard.ts` (205) | ~40 | Host allowlist, upcaster, two groupings. |
| `mcp-server.ts` (148) | ~40 | `decision_route` alias, path attachments behind a flag. |
| `provider.ts` (350) | ~35 | Error class `kind`/`.of`/brand, two exports, `wrapped` getter. |
| `api.ts` (84) | ~30 | `modelRef` branch, `createSdkProvider` guard, `evaluateDecision`. |
| `manifest.ts` (192) | ~30 | Additive validation of the policy block; shallow spread stays. |
| `credentials.ts` (59) | ~30 | `CredentialSpec[]` iteration (P8). |
| `index.ts` (12) | ~10 | New exports. |

Never touched: `runtime.ts`, `store.ts`, `utils.ts`, `planning.ts`, `discovery.ts`, `mcp.ts`, `functions/api/jev.js`, all ten existing test files, the v2 `SKILL.md` and helper template.

The original plan has a similar existing-file footprint; it adds roughly 6,500 to 8,000 `src/` lines, and the difference is almost entirely new modules the revision cuts or defers.

## Answers to the open owner decisions

1. **Fork:** hard-fork, decided at gate G. The identifier subset is decided at G too; npm name, bin and publishing stay at P-R.
2. **Env prefix:** `SYS1ROUTER_*` is fine; define once in `src/product.ts`. `JEV_ROUTER_PROVIDER` wins with a warning, never an error by default.
3. **Floors:** 0.75 via `max()` composition, demo and legacy exempt, justified as "runner-up holds at most 0.25 under `max(p)`", and recorded as an owner-tunable heuristic. `min_coverage` 0.9 is fine once coverage means mass.
4. **Legacy two-stage:** it is a real selection bug (merged coarse/final map sorted together at `router.ts:359,385`; when finalists are filtered a coarse-only candidate becomes `topSafe`, gated by the final call's confidence). Pin it deliberately in P0 as the one golden expected to flip at 2.0. In 1.x, apply the staging helper to legacy providers only when `policy.decision_model` is present (a new trigger under the plan's own rule), so TypeSafe/OpenRouter users can opt in.
5. **Receipt size:** decide now, per item 26. Nothing to drop at 2.0.

## Things I would not change

- Dual emission for scalar fields, `jev_probability` null unless native, `status_cap`, zero-I/O preflight, the two-tier trust model, bytes-only attachments, the adapter diff guard, no new runtime dependencies.
- P7 before or after P9 is a taste call (P7 implements the walkthrough; P9 is a binding decision). Either order works.
- Keep `SYS1ROUTER_STRICT_CREDENTIALS` and calibration `unknown` for cross-host; one skeptic argued to drop them and I agree with the other that they are cheap and honest.

## Board design (the deliverable)

Repo state checked today: Issues are **disabled** on the fork (`has_issues: false`, the GitHub default for forks), there are no milestones and no issues, the default labels exist, and authenticated REST writes work through `gh api` (GraphQL is blocked, so `gh issue`/`gh label` subcommands do not; the GitHub MCP `issue_write` tool does, and it takes `milestone` and `parent_issue_number`).

### Prerequisite
Enable Issues on the repository (`PATCH /repos/loboroberto/sys1router` with `has_issues: true`, or the owner flips Settings → Features → Issues). It is a one-line reversible repo setting; I will do it via REST as step 1 unless the owner prefers to.

### Milestones (7 waves, in order)
| Milestone | Contains | Exit criterion |
|---|---|---|
| M0 Decide and pin | D1 to D5, P0, P0.5 | Owner decisions recorded; goldens, legacy-suite job and hardening merged; design doc committed |
| M1 Neutral seam | P1, P3a, D6 | `DecisionModelV1` accepted by the router; legacy receipt deep-equals golden through both paths |
| M2 First new providers | P4a, P8-min, P4b | `route --model systemone:<m> --endpoint <loopback>` works from CLI, SDK, serve, MCP and the v3 agent helper |
| M3 Receipts and capabilities | P2, P5, D7 | Receipts v2 dual-emit; preflight, staging and `calls()` with the 300-candidate worked example |
| M4 Cloudflare | P6, P7 | Clef text and image routes recorded as fixtures; walkthrough passes |
| M5 Emulation and surfaces | P9, P10 | Four flavors with mass coverage; `decision_route`; dashboard groupings; Pages only if Cloudflare hosts it |
| M6 Release and SDK | P-R, P11-lite, P13 | Named, published 1.0.0; `./provider-sdk` and path-based loader; neutral-first docs |

### Issues (23)
- **1 epic:** "Decision-model agnostic roadmap" — body holds the verdict, the milestone table and the ordered PR list; every other issue is a sub-issue of it so GitHub shows progress.
- **7 decision issues** (label `decision`): D1 hard-fork vs upstream; D2 product identifiers (receipt schema id, env prefix, config dir, project config filename, export prefix); D3 Node floor; D4 playground host; D5 record first-wave fixtures (llama.cpp, Strands, Clef) — a spike, labelled `spike`; D6 confidence floors and calibration table (M1); D7 legacy two-stage opt-in fix (M3). P-R is an eighth decision issue in M6 (npm name, bin, publishing, version mapping).
- **15 PR issues** (label `pr`), one per row of the revised roadmap: P0, P0.5, P1, P3a, P4a, P8-min, P4b, P2, P5, P6, P7, P9, P10, P11-lite, P13.

Each PR issue body uses one template:
```
## Goal            one paragraph
## Scope           modules and files it may touch
## Diff guard      files it must not touch (router.ts, src/spec/**, preflight, normalize, staging for adapter PRs)
## Acceptance      - [ ] named test files   - [ ] typecheck, test, build   - [ ] legacy-suite green   - [ ] docs/examples/help shipped
## Budget          ~N src lines (from the roadmap table)
## Depends on      #… (previous PR issue and any decision issue)
## Design          docs/design/decision-model-agnostic.md § …
## Review notes    item numbers from this review that apply
```

Labels to create: `decision`, `spike`, `pr`, `area:router`, `area:providers`, `area:receipts`, `area:security`, `area:surfaces`, `area:ci`. Size is carried by the budget line, not a label.

### Creation steps after approval (all idempotent; nothing in `src/` changes)
1. Enable Issues (REST PATCH).
2. Create the 9 labels (REST, skip existing).
3. Create the 7 milestones with descriptions (REST `POST /milestones`), record their numbers.
4. Push the review as `docs/design/decision-model-agnostic.md` on `claude/blissful-dijkstra-wlq9q0` so issues can link to a stable section anchor (one commit, docs only; the P0 PR later lands it on `main`). Skip this step if the owner prefers the doc to arrive only with P0.
5. Create the epic issue (MCP `issue_write`).
6. Create the 22 remaining issues with `milestone`, `parent_issue_number` = epic, labels and the template body; fill `Depends on` with the numbers as they are created, in roadmap order.
7. Post the ordered list with links as a comment on the epic and report the epic URL.

### Verification of the board
- `gh api repos/loboroberto/sys1router/milestones` returns 7, in order, each with its issue count.
- `list_issues` returns 23 open issues; every non-epic issue has a milestone and a parent; no issue lacks a `Depends on` except D1 and P0.
- The epic's sub-issue progress bar reads 0/22.

## Verification of this review

- Every code citation above was read in this checkout (`src/*.ts`, `tests/*.test.ts`, `skills/`, `functions/`, `.github/workflows/`).

## Appendix A: Plan under review (verbatim)

# Plan: make sys1router decision-model agnostic

## Context
sys1router (a fork of JevRouter) puts a fast "System One" decision model in front of an agent's tools. Today it can only reach TypeSafe's Jev, through two hard-wired HTTP providers (`src/provider.ts`) and a closed `typesafe|openrouter|demo` switch (`src/runtime.ts`). The Jev name is built into public contracts:
- types: `JevProvider`, `JevRouter`;
- receipt fields: `raw_jev`, `jev_probability`, `provenance.jev_provider`;
- `jev_*` error codes, the `jev_route` MCP tool, `.jevrouter/`, and `JEV_*` env vars.

The core also quietly assumes Jev's behaviour:
- every candidate gets a probability, and confidence is one scalar gated at 0.55 (`router.ts:373`, `:541`);
- several questions go in one call, and the coarse pass sends every candidate (`router.ts:238`, `:336`);
- state is text only (`router.ts:477`);
- auth is one Bearer key. `JEV_API_URL` even sends the TypeSafe key to whatever host it names.

The research (as of 2026-10-03) shows a field that is moving fast:
- **Shared wire format.** TypeSafe's `/v1/systemone` format `{model, state, questions}→{answers, usage}` is now the de-facto standard. Many vendors and local servers speak it, each with small dialect differences: Liquid, Upstage, Telnyx, Perplexity, OpenRouter, Vercel, Kev, Ollaya and llama.cpp.
- **Cloudflare** launched Clef / Clef-flash on Workers AI. They use that format plus an `images` extension (at most 4 images, data URLs only). This is the owner's "image capability" example.
- **AWS's** decision model is Strands Decider 2B, which is open-weight and runs locally. Bedrock offers no hosted decision API.
- **General LLMs** can emulate a decision model through logprobs or constrained output. Probability quality varies, and frontier closed APIs are dropping logprobs.

**Outcome:** new models, and new features on existing models, arrive as config, catalog or adapter changes, never a core rewrite.

## Owner decisions (binding)
1. **Neutral names are primary.** Every Jev name stays as a working, deprecated alias, and receipts keep emitting the Jev fields for one major version.
2. **Emulation is allowed with honest labels.** Every answer records its probability source and calibration, and policy can require native probabilities.
3. **First wave:** Cloudflare, AWS, local/open-weight, and generic OpenAI-compatible.
4. **Packaging:** a few built-in adapters, plus a stable provider SDK so adapters can ship as separate npm packages loaded through config.

## Approach: migration-first, every PR green
- **Legacy stays frozen.** The legacy path (`HttpJevProvider`, `OpenRouterJevProvider`, `DemoProvider`, `CachedJevProvider`, custom `JevProvider`s) is byte-for-byte unchanged in 1.x.
- **New triggers only.** New behaviour switches on only through something new: a spec-model or new provider id, a declared capability, attachments, a new policy knob, or a flag.
- **Frozen tests.** The 10 existing test files are frozen (`tests/frozen.txt`, enforced by a PR-only CI guard). New assertions go in new files.
- **Narrow public types.** Public TS unions (`ProviderKind`, `fallback.type`, the `JevProviderError` code union) never widen. Widening goes into new names: `ProviderId`, `RouteOptions.modelRef`, `fallback.detail`, `error.kind`.
- **`defaultPolicy` gains no keys.** New knobs live under an optional `policy.decision_model` block.
- **Small PRs.** Each PR is about 800 lines of `src/` or less, ships the docs, examples and help for whatever it makes usable, and passes `npm run typecheck && npm test && npm run build`.
- **Early value.** The first non-TypeSafe providers (AWS Strands, llama.cpp) are usable at P4.

## Target architecture

```
Surfaces  api.ts · cli.ts · route-command.ts · serve · mcp-server.ts · agent-setup.ts · dashboard.ts · functions/api/decide.js
Router    src/router.ts  DecisionRouter (= JevRouter): gates, staging, plans, finalize, status_cap
Decision  src/decision/{questions,normalize,confidence,preflight,staging,fanout,state,media,usage,bridge}.ts
Spec      src/spec/*  (exported as "jevrouter/spec"; no node:* imports, enforced by a test)
Catalog   src/catalog/{load,resolve}.ts + catalog/decision-models.json (static, narrowing-only data)
Registry  src/providers/{registry,config,trust,credentials,http,presets}.ts
Adapters  src/providers/systemone/* (one adapter + dialect table) · src/providers/openai-compatible/* (emulation)
Receipts  src/receipts.ts (buildReceipt dual-emit, buildRuntime, upcastReceipt, agentView)
```

### Spec (the seam every adapter, built-in or plugin, implements)
```ts
interface DecisionModelV1 {
  readonly specificationVersion: "v1";
  readonly provider: string;              // "cloudflare" | "strands" | "ollama" | ...
  readonly modelId: string | null;
  readonly adapter: { name: string; version: string; dialect?: string };
  readonly defaultCapabilities: PartialCapabilities;
  readonly policyDefaults?: DecisionModelPolicy;   // merged narrowing-only
  doDecide(call: DecisionCall): Promise<DecisionResult>;
}
interface DecisionCall {
  state: StateValue;                 // opaque caller state
  media: MediaPart[];                // bytes only, never URLs or paths
  questions: Record<string, CompiledQuestion>;     // choice | score | boolean ("noul" on the System One wire)
  capabilities: EffectiveCapabilities;
  providerOptions?: Record<string, unknown>;       // user config only, never from requests
  abortSignal: AbortSignal;          // one deadline across stages, chunks and fan-out
}
interface DecisionResult {
  answers: Record<string, NormalizedAnswer>;   // probabilities/confidence nullable + AnswerProvenance
  raw: unknown[];                    // verbatim bodies → raw_provider*
  legacyEnvelope: JevRawResponse;    // translated System One envelope → raw_jev
  response: { resolved_model; endpoint_host; elapsed_ms; calls; authenticated };
  usage: { input_tokens; output_tokens; cost_usd; calls };   // summed over all calls
  warnings: DecisionWarning[];       // unsupported | compatibility | deprecated | degraded | other
}
```

**Errors.** `DecisionProviderError` is one class with two names: `JevProviderError` is the same class with the same positional constructor.
- It adds `kind`: auth, timeout, malformed, http, rate_limited, quota, overloaded, model_not_found, request_too_large, invalid_request, unsupported_capability, network.
- It adds a static `.of(kind)` and a `Symbol.for` brand, so plugins built against another copy of the module still classify correctly.
- `router.ts:325,431` switch from `instanceof` to the brand check.

**Bridge.** Legacy `JevProvider`s are wrapped in `src/decision/bridge.ts`. The request object passes through unchanged, and `boolean` is rewritten to `noul`.

### Capabilities: declared, layered, fail-closed
```ts
interface DecisionCapabilities {
  primitives: { choice?: {min_options; max_options|null; distribution: "full"|"partial"|"none"; confidence: "native"|"derived"|"none"};
                score?: {...}; boolean?: { probability: boolean } };
  batch: { max_questions: number|null; mode: "native"|"fanout" };
  state: { context_tokens: number|null; max_request_bytes: number|null; truncates_silently?: boolean };
  inputs: Partial<Record<string /*modality*/, MediaCapability>>;   // absent = unsupported
  probability: { source: ProbabilitySource; calibration: CalibrationClass };
  locality: "hosted"|"gateway"|"local"; status: "live"|"staging"|"deprecated";
}
interface MediaCapability { transport: string /* encoder id, e.g. "systemone.images" */; max_items; media_types; encodings; limits: Record<string, number> /* bytes_each, pixels_each, duration_ms_each… */ }
```

**Resolution layers.** Capabilities resolve in four layers: adapter/dialect defaults, then the bundled catalog, then user config (the user's own assertion), then project config and policy, which can only narrow.
- **Trust lattice.**
  - `probability.*`, `locality` and transport come only from the adapter.
  - `status` comes from the catalog or user config.
  - Data layers can only narrow or downgrade.
  - `null` means unknown, so policy caps apply.
- **`capabilities_hash`** goes into receipts and cache keys.
- **Preflight** (`src/decision/preflight.ts`) is pure and runs before every spec-model call. It rejects unsupported primitives, modalities and limits with **zero I/O**.

### Provenance and policy
- **Probability source and calibration.**
  - `ProbabilitySource` is one of: native, logprobs, score_endpoint, sampled, verbalized, heuristic, none, unspecified.
  - `CalibrationClass` is one of: vendor, vendor_claimed, uncalibrated, heuristic, unknown.
  - `confidence_source` is one of: native, derived_concentration, legacy_max_probability, unavailable.
- **Normalization** (`src/decision/normalize.ts`):
  - probabilities are never rescaled;
  - partial distributions give `null`, not 0;
  - the Vercel sentinel `{confidence:0, probabilities:{}}` means unavailable;
  - legacy providers keep today's strict parsing and the max(p) confidence fallback.
- **`policy.decision_model`** has these knobs: `require_native_probabilities` (CLI `--require-native`), `native_required_for_risk`, `min_confidence_by_calibration` (default 0.75 for non-vendor), `min_confidence_by_model`, `min_coverage`, `probability_free_answers: confirm|reject`, `on_unsupported: fail|degrade`, `allowed_status`, `allowed_models`, `max_calls_per_decision`, `max_decision_ms`.
- **`gateFor()` replaces `thresholdFor` (`router.ts:541`).**
  - Legacy providers, demo and vendor-calibrated models keep `min_confidence ?? 0.55`, so the goldens are unchanged.
  - Everything else gets the stricter floor.
  - Probability-free answers are never auto-selected.
  - Missing confidence fails closed.
- **`status_cap`** is computed in `finalize`. Diversity re-rank (`router.ts:168`) and the beam override (`router.ts:272`) can never raise status above it.
- **Probability-free answers** select only `answer.choice` when it is unfiltered. There is no alphabetical "highest-probability safe" substitution.

### State, media, SSRF
- `RouteInput.attachments` holds base64 bytes, or a `path` (CLI and stdio MCP only, realpath'd under the project root, behind a policy flag). URLs and `file:` are always rejected.
- Media goes through an inspector registry. Wave 1 has a PNG/JPEG/WebP sniffer and nothing else. An encoder registry is keyed by transport, and wave 1 has `systemone.images` only.
- Caller state is opaque. Typed-part-shaped objects in state are rejected for dialects that read parts.
- Receipts and cache keys store `{modality, media_type, sha256, metrics}`, never bytes.

### Registry, config trust, credentials
- **User config (trusted).** `$XDG_CONFIG_HOME/sys1router/config.json` (or `SYS1ROUTER_CONFIG`) is the only place that can bind credentials, add credentialed endpoints, or allow plugins.
- **Project config (restricted).** The committed `jevrouter.config.json` can reference ids, set aliases and a default, declare **keyless loopback** providers, and narrow capabilities or policy. Anything else is refused before any fetch.
- **Model refs** use `provider:model`, split at the first colon only when the prefix is a known id (`ollama:qwen3.5:4b` works). Built-in and preset ids are reserved.
- **Resolution order** (route-command only, matching today's env reads):
  1. `--provider` / `--model`
  2. a non-empty `JEV_ROUTER_PROVIDER`. A conflicting `SYS1ROUTER_*` value is a hard error, which protects existing v2 helper pins.
  3. `SYS1ROUTER_PROVIDER` / `_MODEL`
  4. config `default`
  5. today's key auto-detect
- **Async construction.** `resolveDecisionModel()` is the async constructor used by every construction site. `createProvider` stays synchronous and legacy-only.
- **Credentials and HTTP.**
  - `ScopedCredentials`: well-known secrets are pinned to their vendor hosts (TypeSafe→api.typesafe.ai, OpenRouter→openrouter.ai, Cloudflare→api.cloudflare.com).
  - The guarded `http` client enforces `bind_hosts`, `redirect:"error"` and HTTPS-or-loopback. It sends no `Authorization` to keyless presets and counts every call.
  - `--endpoint` is accepted only for keyless providers.
  - Legacy `JEV_API_URL` on a foreign host keeps working, but adds a `credentials.cross_host` warning and calibration `unknown`. `SYS1ROUTER_STRICT_CREDENTIALS=1` makes it an error.

### Receipts v2 (`src/receipts.ts`): dual-emit, upcast, never rewrite
| Neutral (primary) | Legacy (still emitted in 1.x) |
|---|---|
| `decision.model_choice` | `decision.jev_choice` |
| `candidates[].model_probability / model_confidence / model_stage` (+ `model_probability_source`) | `jev_probability / jev_confidence / jev_stage`: **null unless the answer is native**, so v1 readers fail closed |
| `raw_provider[]`, `raw_provider_stages`, `raw_provider_batches[]` | `raw_jev` (translated envelope), `raw_jev_stages` |
| `provenance.decision_provider` + `provenance.decision_model{provider, adapter, dialect, requested/resolved model, endpoint_host, locality, authenticated, capabilities_hash}` | `provenance.jev_provider` |
| `error.kind`, `fallback.detail` | `error.code`, `fallback.type` |

Other receipt changes:
- **New fields:** `receipt_schema: "sys1router.receipt/2"`, `decision.answer_provenance`, `inputs.attachments[]` (hashes only), `warnings[]`, and `usage_normalized` (summed over every call).
- **Runtime block.** serve and MCP receipts gain `runtime` through `buildRuntime()`.
- **Upcasting.** `upcastReceipt()` gives the dashboard a v2 view of v1 files without rewriting them.
- **Output size.** `agentView()` de-duplicates raw bodies in stdout and MCP output.
- **Cache key.** `CachedJevProvider` keeps today's key for the default endpoint, so caches stay warm, and adds the host for a non-default `JEV_API_URL`. The v2 cache key adds adapter@version, dialect, `capabilities_hash`, media hashes and estimator.

### Naming and aliases
| Surface | Neutral (primary) | Legacy alias kept in 1.x |
|---|---|---|
| Types | `DecisionProvider`, `DecisionRequest`, `RawDecisionResponse`, `ChoiceQuestion/ScoreQuestion/BooleanQuestion` | `JevProvider`, `JevRouteRequest`, `JevRawResponse`, `Jev*Question/Answer` (`JevRouteQuestion` not widened) |
| Router / errors | `DecisionRouter`, `DecisionProviderError` | `JevRouter`, `JevProviderError` (same objects) |
| Readers | `getBooleanAnswer` | `getNoulAnswer` |
| MCP tools | `decision_route`, `decision_capabilities`, `decision_models` | `jev_route`, `jev_capabilities`; `--mcp-tools legacy\|both\|neutral` (default `legacy` until P13; legacy instructions stay byte-identical) |
| Env | `SYS1ROUTER_PROVIDER/_MODEL/_CACHE/_CONFIG/_STRICT_CREDENTIALS/_VERBOSE` | `JEV_ROUTER_PROVIDER` (wins or errors), `JEV_ROUTER_CACHE`; vendor key vars not deprecated |
| Agent integration | `integration-v3.json`, `SKILL.v3.md`, marker `skill-cli-v3` | v2 files stay byte-identical for typesafe and openrouter installs |
| Pages | `/api/decide` | `/api/jev` frozen |
| Package / bin / `.jevrouter/` | decided at release gate P-R | unchanged until then |

## Phased PR roadmap

**P0. Characterization net (tests and CI only; zero `src/` diff)**
- `tests/legacy-contract.test.ts` pins the HTTP wire, headers, error mapping and the OpenRouter envelope. Nothing covers these today.
- `tests/golden-receipts.test.ts` covers route, two-stage, a 300-candidate route, batch, serial, beam, diversity and hierarchical plans. It compares a whitelisted legacy projection with ids masked.
- `tests/agent-render.test.ts` takes normalized snapshots, with `process.execPath`→`<NODE>` and cliPath→`<CLI>`.
- Also: `compat-exports`, `cache-key`, and `pages-jev` (typed dynamic import with a fake fetch).
- Infrastructure: a redacting HTTP replay harness `tests/fixtures/http-replay.ts`, and an env-isolating preload `tests/support/isolate-env.ts` wired into `npm test`.
- `tests/frozen.txt` plus the PR-only guard job in `ci.yml` (`fetch-depth: 0`, `--diff-filter=MDR`, `upstream-sync` label exemption).
- Commit the full design doc as `docs/design/decision-model-agnostic.md`.
- TS tests never import a literal `.mjs` path, because of TS7016 under strict with no `allowJs`.

**P1. Neutral names, error class, serve hardening**
- Neutral type aliases with `@deprecated` JSDoc, `DecisionRouter`, `DecisionProviderError` and brand checks.
- `buildQuestions`/`describeCapability` stay in place in `provider.ts` to limit upstream merge conflicts, and are re-exported as `renderQuestions`/`describeCandidate`. Add `getBooleanAnswer`, plus a readonly `wrapped` getter on the cache.
- Export `probeDecisionModel`.
- serve gets Host and Origin checks plus an optional `--token`.
- Tests: `compat-identity`, `error-kind` (both constructor forms; brand across a module copy), `serve-security`.

**P2. Receipts v2 dual emission**
- `src/receipts.ts`: dual-emit at `candidateView`, `baseResult`, `errorResult`, `finalize`, the empty result, and `plan()` assembly.
- `usage_normalized`, with `collectPlanUsage` going through `src/decision/usage.ts`.
- The dashboard reads through the upcaster. `schemas/receipt.v2.json` is documentation only.
- Tests: `dual-fields`, `receipts-upcast`, `agent-view`, `usage`.

**P3. Spec, bridge, normalizer; the router accepts `DecisionModelV1`**
- `src/spec/*`, `src/decision/{bridge,normalize,confidence}.ts`.
- The router constructor takes `JevProvider | DecisionModelV1`.
- Spec-model `finalize` handles null probabilities, the probability-free rule and `status_cap`. Re-rank and beam see only single/final-stage candidates.
- `policy.decision_model` is typed and validated in `manifest.ts`.
- `gateFor`, plus `resolveDecisionPolicy()` for the narrowing-only merge. The shallow spread at `manifest.ts:149` is left alone.
- `--require-native`.
- `planBatch` skips beam with a warning when probabilities are null.
- `evaluateDecision()` gives the typed path. The `./spec` export lands.
- Tests: `normalize`, `bridge` (requests equal the P0 logs), `injection`, `status-cap`, `gate` (p=(0.56, 0.44) without confidence does not auto-select).

**P4. Registry, two-tier config, scoped credentials, systemone dialect, first new providers: AWS Strands and local**
- `src/providers/{registry,config,trust,credentials,http}.ts` with hand-written validators. There is no new runtime dependency; the JSON Schemas are for editors only.
- `src/providers/systemone/*` with dialect rows `systemone` and `openrouter-decisions`. Config `dialect.extends` can override path, model field, usage casing and envelope unwrap.
- HTTP status→kind mapping: 400/422 invalid, 401/403 auth, 402 quota, 404 model, 408/504/524 timeout, 413 too large, 429 rate_limited, 501 unsupported, 529 overloaded.
- Presets, all keyless loopback with an optional local bearer: `strands-decider`, `llamacpp`, `ollaya`, `kev`.
- `resolveDecisionModel` is used at every construction site: `cli.ts:171,202`, `route-command.ts:35,83,115`, `api.ts:82`.
- CLI: `--model ref|alias`, `--endpoint` (keyless only), `--config`, `providers list|add`, `config validate`, `models probe --identity`.
- Docs: a CONTRIBUTING "add a provider" decision tree (config entry → `dialect.extends` → dialect row → encoder → plugin), `docs/cookbook/10-local-decision-models.md`, `examples/config/{strands,llamacpp}.json`, and a README provider table.
- Tests:
  - `trust`: a cloned-repo config that binds `TYPESAFE_API_KEY` to a foreign host is refused with 0 fetches.
  - `credentials`: no `Authorization` reaches loopback even with every key set.
  - `systemone-dialects`, `config-only-vendor`.
  - `env-precedence`: the frozen v2 `route.mjs` never switches silently.
  - `config-validate`: rejects `__proto__`.

**P5. Capabilities, static catalog, preflight, capacity-aware staging**
- `catalog/decision-models.json` holds only the entries needed now: clef, clef-flash, strands, plus informational entries for typesafe and openrouter. Unverified entries are `staging`.
- `catalog:validate` runs in CI.
- `src/catalog/{load,resolve}.ts`, and `capabilities_hash` goes into receipts.
- Preflight runs on every spec-model call path: route stages, both hierarchical stages, batch, and `evaluateDecision`.
- `src/decision/staging.ts` (spec models only):
  - The chunk cap is `cap = min(single_stage_max_candidates, max_options, context-fit)`.
  - Chunks are balanced, and `K = clamp(top_k, min, max)`.
  - Finalists are each chunk's winner plus a per-chunk quota. When there are too many, a bounded tournament runs.
  - Coarse-only candidates are never selectable.
  - `routeHierarchical` uses the same helper.
- `src/decision/fanout.ts` splits question batches by `max_questions`.
- CLI `models list|show`.
- Tests:
  - `preflight`: 49 candidates at cap 24 give 17/16/16; 70 questions at cap 64 make 2 calls; an undeclared primitive makes 0 calls.
  - `hierarchical-caps`.
  - `legacy-untouched`: a 300-candidate legacy route still makes 2 calls.

**P6. Cloudflare (text)**
- `workers-ai` dialect row plus the `cloudflare-workers-ai` preset:
  - path `/client/v4/accounts/{id}/ai/run/{model}`;
  - selector body `clef|clef-flash`;
  - tolerates a bare body or a `{result}` envelope.
- Model refs must match `^@cf/[a-z0-9-]+/[a-z0-9][a-z0-9._-]*$`, with `.`, `..` and `%` rejected, segments encoded, and the final pathname asserted.
- Fixed bind host `api.cloudflare.com`, token `CLOUDFLARE_WORKERS_AI_TOKEN`, account id validated and redacted.
- Docs: `docs/cookbook/11-cloudflare.md` and an example config.
- Tests: `cloudflare` covers envelopes, path traversal, 401/413/429, and other vendors' keys never being sent.
- Adapter diff guard: this PR may touch only `src/providers/**`, `catalog/**`, docs, examples, README and tests.

**P7. Attachments and images (Clef)**
- `src/decision/{state,media/*}.ts`: the image inspector, the `systemone.images` encoder, and the opaque-state guard.
- `RouteInput` and `EvaluateInput` gain `attachments`. `on_unsupported:"degrade"` is available.
- Clef's `inputs.image` goes in the catalog.
- CLI `--attach`, stdio MCP `path` attachments behind a flag, an MCP line-size cap. serve accepts base64 only, with a body cap.
- Tests:
  - `attachments`: a 5th image fails preflight; a legacy provider with an image makes 0 calls; MIME spoofing is caught; symlink escape is caught; no base64 appears in receipts, cache or errors.
  - `ssrf-state`: `169.254.169.254` and `file://` in state never reach the wire.

**P8. Agent setup v3**
- A conflict dry-run runs before any write. Switching from v2 requires `--upgrade`.
- `integration-v3.json` records `provider_ref`, `env[]` and `project_root`.
- The v3 helper injects `--model` and rejects agent-supplied `--provider`, `--model`, `--config` and `--endpoint`.
- MCP args are `--model --project --mcp-tools both`.
- SKILL.v3 tells agents to report the provider and probability source.
- `credentials.ts` iterates `CredentialSpec[]` through an injectable prompt seam.
- `doctor` reports v3, config, env, loopback reachability, deprecations and warnings.
- Tests: `agent-setup-v3` with a spawned keyless stub that fails on any `Authorization`. v2 snapshots are unchanged.

**P9. Generic OpenAI-compatible and local emulation**
- `src/providers/openai-compatible/*`. Flavors are data: `openai-chat`, `vllm`, `llamacpp-chat`, `ollama-native`, `workers-ai-chat`, `together`, `fireworks`.
- The strategy (`logprobs` or `argmax`) is explicit in config, with an optional `fallback: "argmax"`. The fallback restarts the whole decision and is memoized per endpoint.
- Labels are A–Z only. Only variants of the same letter are merged. `max_options = min(26, top_logprobs/2)`, and larger sets go to P5 chunking.
- Absent labels are `null`. Coverage is recorded, and coverage below `min_coverage` counts as probability-free.
- Score uses digits and the expected index. Boolean is P(yes).
- Requests use temperature 0 and thinking off, and never include tools.
- Estimator-owned keys (`logit_bias`, `temperature`, `logprobs`, …) are refused in `providerOptions`.
- Presets ship `policyDefaults`: a 0.75 floor, and native required for high/critical risk.
- Docs: `docs/cookbook/12-emulation.md`.
- Tests: `emulation-logprobs` (recorded fixtures), `emulation-policy` (`require_native` makes 0 calls; argmax never auto-selects), `emulation-safety`.

**P10. Surfaces**
- MCP: `--mcp-tools`, `instructionsFor(toolNames)`, and a `decision_models` tool that exposes no env names or account ids. The `model` input is denied unless `allowed_models` is set.
- The dashboard groups by model and by probability source.
- `functions/api/decide.js` is hand-written and small:
  - it uses the `env.AI` binding (Clef-flash) when bound, otherwise OpenRouter Jev;
  - it has a fixed choice question and a 64 KB cap, with images off by default;
  - it ships with a `.d.ts` shim and a test;
  - `playground.html` is updated, and `docs/pages-deploy.md` is added.

**P-R. Release and naming gate (owner decision; blocks P11)**
- npm name and scope, the bin name and the plugin prefix.
- Point install docs at this fork rather than upstream `BillionsBobby/JevRouter`.
- Version mapping: 1.0.0 = dual emission, 2.0 = alias removal. Add a CHANGELOG.
- Decide whether to hard-fork or keep tracking upstream.

**P11. Provider SDK, plugin loader, conformance kit**
- Exports: `./provider-sdk` (types only) and `./conformance`.
- Plugins default-export `defineProvider({meta, kinds, dialects?, catalog?})`. They get runtime helpers through `ctx` (`http`, `credentials`, `errors.of`, `systemone.create`, `normalize`), so `jevrouter` is only a types dev dependency for them.
- `src/providers/plugins.ts`:
  - user-config allowlist with exact version plus integrity;
  - never loaded from workspace `node_modules`;
  - loaded at startup only;
  - spec-version and engines gates.
- Documented as **fully trusted in-process code**, not a sandbox.
- Conformance: `describeDecisionModel(factory, {fixtures, claims})` runs replay-mode groups driven by claimed capabilities.
- `examples/provider-template/` has its own lockfile; there are no npm workspaces.
- CLI `conformance --model ref [--live --record]`, plus `docs/provider-authoring.md`.

**P12. Keeping current**
- Live probes.
- `models refresh|diff` (on command only; narrowing or informational).
- `catalog-refresh.yml`, weekly:
  - it runs the full suite inside the job and opens PRs that are never auto-merged;
  - it starts once the catalog has more than about 10 entries.
- `conformance-live.yml`: manual, needs secrets, re-records redacted fixtures.
- Generated `docs/decision-models.md`.

**P13. Neutral-first defaults**
- `serve-mcp` defaults to `both`.
- Docs are rewritten neutral-first, and `docs/primitives.md` replaces `docs/jev-primitives.md`.
- ADRs: spec, receipt v2, config trust.

**Deferred follow-ups (not scheduled):**
- a Bedrock Custom Model Import plugin (out of repo, SigV4 through `ctx.http`);
- `perplexity-decisions` and `vercel-evaluate` dialects;
- more media encoders;
- `x-*` primitives;
- fallback chains;
- calibration fitting from receipt outcomes.

## First-wave adapters at a glance
| Family | What ships | Phase |
|---|---|---|
| **Cloudflare** | `workers-ai` dialect row + preset (text) → `systemone.images` (Clef images) → Pages `env.AI` binding; `workers-ai-chat` emulation flavor | P6, P7, P10, P9 |
| **AWS** | `strands-decider` keyless loopback preset (AWS's actual decision model; no AWS SDK in repo). Bedrock CMI as a later plugin | P4 (+ follow-up) |
| **Local / open-weight** | Native System One presets (llama.cpp, Ollaya, Kev, Strands). Emulation via `ollama-native`, `vllm`, `llamacpp-chat` | P4, P9 |
| **Generic OpenAI-compatible** | One adapter, flavor table, logprobs/argmax strategies, fan-out batching. Hosted flavors need an explicit `auth.env` | P9 |

## How the repo keeps up with what comes next
- **New System-One vendor:** add a user-config entry, using `dialect.extends` for quirks. Verify with `conformance --live --record`, then optionally open a catalog PR. No code change.
- **New feature on an existing model** (e.g. more images, or video): a catalog diff plus fixtures, and at most one inspector or encoder. Preflight checks limits generically. No router, spec or preflight change.
- **New wire shape:** a dialect row, or a plugin for exotic auth or transport. If TypeSafe changes its own format, user config can switch the built-in `typesafe` id to `wire: "systemone@2"`.
- **New primitive** (rank, multi-select): an additive spec v1.x change (types, normalizer, preflight bounds, a conformance group). It is promoted once two providers support it.
- **Third parties:** they can publish their own adapter packages against the provider SDK plus the conformance kit.

### Walkthrough: "Cloudflare introduces image capability"
1. **Today's state.** After P6 and P7, `clef-flash` carries `inputs.image {transport:"systemone.images", max_items:4, limits}`.
2. **Routing an image.** `route --model vision --attach shot.png` sniffs the image, passes preflight with zero I/O, and sends `images:[dataURL]` to the pinned `api.cloudflare.com` path. The receipt records the image hash, `capabilities_hash`, and native/vendor provenance.
3. **Text-only provider.** The same request to `typesafe:jev-latest` fails closed (`capability_unsupported`, 0 calls). With `on_unsupported: degrade`, the image is dropped with a warning and the status is capped at `needs_confirmation`.
4. **Later changes:**
   - If Cloudflare raises the limit to 8 images, that is one catalog PR plus a fixture. Users can opt in the same day through user config.
   - If Cloudflare adds video, that is a video inspector, a catalog entry and fixtures.
   - In neither case does the core change.

## Security invariants (each one tested)
1. **Who can bind credentials.** Only user config, env and flags. Project files can never bind a secret, point a credentialed provider elsewhere, or enable a plugin.
2. **Where secrets go.**
   - Well-known secrets go only to their pinned hosts.
   - `--endpoint` never moves a secret.
   - All new-adapter I/O goes through the guarded client: `bind_hosts`, no redirects, HTTPS-or-loopback, and no `Authorization` to keyless providers.
3. **Preflight failures make zero I/O calls.**
4. **No caller-controlled URLs, paths or typed media parts reach any wire.**
5. **No secrets in stored output.** No base64, secrets, account ids or endpoint userinfo appear in receipts, caches or error text.
6. **Emulation bodies are inert.** They never contain tools. Estimator keys cannot be overridden, and requests never carry `providerOptions`.
7. **Provenance is honest.**
   - Data layers can only narrow.
   - `jev_*` probabilities are null for non-native answers.
   - Cached entries never raise calibration.
   - Status caps cannot be bypassed by re-rank or beam.
8. **Agent-reachable inputs are deny-by-default.** That covers the MCP and HTTP `model` input, serve Host/Origin, helper flag stripping, and env conflicts. Plugins are trusted code, loaded only from an integrity-pinned allowlist.
9. **The original invariants still hold:**
   - decision-only;
   - the router owns availability, permissions, risk and confirmation;
   - filtered candidates are never re-normalized;
   - append-only receipts;
   - HTTPS-or-loopback.

## Verification
- **Every PR:**
  - `npm run typecheck && npm test && npm run build`;
  - the frozen-test guard;
  - P0 golden legacy projections;
  - normalized agent-render snapshots;
  - export superset;
  - default cache key;
  - `defaultPolicy` deepEqual;
  - the `policy_hash` of `examples/policy.json`.
- **Adapter PRs (P6, P9):** the diff guard forbids edits to `router.ts`, `src/spec/**`, preflight, normalize and staging. Those change only in P3, P5 and P7.
- **Conformance:** every built-in adapter passes in replay mode inside `npm test`. Live conformance runs on manual dispatch only.
- **Manual smoke before promoting a catalog entry to `live`:**
  - Cloudflare text and image (REST), plus the Pages binding;
  - Strands local;
  - llama.cpp native;
  - Ollama and vLLM emulation;
  - one hosted OpenAI-compatible host.

  Each run re-records redacted fixtures and sets `last_verified`.
- **End to end per phase:**
  - `npm run dev -- route --provider demo …` (unchanged output);
  - `route --model strands:… --stdin` against a local stub;
  - `dashboard` showing the provider and probability-source breakdowns.

## Open owner decisions (raised at the relevant phase)
1. **P-R:** npm name, bin and plugin prefix, publishing, and whether to hard-fork or keep tracking upstream.
2. **Env prefix:** `SYS1ROUTER_*` is proposed.
3. **Default floors:** 0.75 for non-vendor calibration, and `min_coverage` 0.9.
4. **Legacy two-stage re-rank:** today it compares coarse and final probabilities taken from different calls. The plan fixes this for spec models only, to keep the goldens. Should the fix also apply to legacy providers in 1.x?
5. **Receipt size:** whether persisted receipts should drop the duplicated `raw_jev` before 2.0.

## Unverified facts (confirm against live docs and record fixtures at implementation time)
The design never hard-depends on any of these. Each one exists only as staging catalog data, a preset default or a dialect switch.
- **Cloudflare:** the Workers AI REST envelope and the selector rule; Clef's limits and its open-weights status; video support; the account id format.
- **AWS:** the Strands CLI, port and `model` tolerance; whether a hosted AWS decision API exists (none found); Bedrock CMI `return_logprobs`.
- **llama.cpp:** the `/v1/systemone` build and its 501 semantics.
- **Ollaya and Kev:** their ports.
- **Ollama:** native logprobs.
- **vLLM:** `max_logprobs` and `logprob_token_ids`, both of which vary by version.
- **Node 20/22:** whether the `--import` preload propagates to test subprocesses.
- **Hosted logprobs:** OpenAI with reasoning set to none, Together, Fireworks, DeepInfra, and Workers AI per-model chat.
- **TypeSafe's confidence formula:** used only as the "derived" confidence.
- **Reference shapes (used for vocabulary only):** the AI SDK `Experimental_EvaluationModelV4` shape and the OpenAI Decisions schema (no adapter until the schema is published).
