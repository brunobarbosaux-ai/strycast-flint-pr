# Flint 40B quality

**Validated as an external extension against Gufo main `6cc8a42` (v0.31.0) on
gfx1151: config/probe/ops tests, distribution parity vs a pinned llama.cpp
oracle, concurrency determinism, and greedy run-to-run determinism all pass.
Logit-parity rows against the pre-rebase reference fork are stale, not
failing.** Evidence logs with digests live in the extension repository under
[`docs/evidence/`](https://github.com/brunobarbosaux-ai/flint-gufo-extension/tree/3788f66faf3f746fa1000e3d70742f2a98a8f434/docs/evidence).

Scope: what is validated, with what evidence, and what remains open. Failed or
inconclusive checks stay visible until they pass.

## Environment

- GPU: AMD Strix Halo (gfx1151), ROCm 7.2.1
- Gufo: main `6cc8a42ca84438a0566ee498a315f35c6190dc8e`, v0.31.0 (extension API series 0.2)
- Build: `gpu-test` preset (`RelWithDebInfo`, HIP on, tests on)

## Passing checks

| Check | What it covers | Evidence |
| --- | --- | --- |
| `flint.config` | Architecture config parsing from GGUF metadata (in-memory containers) | [config_test.cpp](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/tests/config_test.cpp) |
| `flint.probe` | Artifact probe: `general.architecture == "flint"` detection, negative cases | [probe_test.cpp](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/tests/probe_test.cpp) |
| `flint.ops` | Vendored HIP operators vs host reference: `RmsNormRows` (grouped), `SiluScale` | [ops_test.cpp](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/tests/ops_test.cpp), gfx1151 device |
| Registration | `gufo diagnose --section extensions` lists `flint` with provider available; in-tree models coexist in the same binary | manual run, 2026-11-04 |
| Link isolation | No symbol collisions with in-tree `qwen38_flash_next`, `gufo_qwen_text`, `gufo_core` (namespaces moved under `gufo::vendor::*`) | link of `gufo` with all extensions included |
| G5 (distribution) | T1.0 p0.95 k20 word distribution vs the pinned llama.cpp oracle, 200 rows/arm on the final binary: same support, same ordering, TV=0.0650 (<= 0.25), no missing rows | [g5-verdict.log](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/docs/evidence/g5-verdict.log), 2026-11-05 |
| G8 (concurrency determinism) | Identical greedy output across 1..8 concurrent sessions (314-char stream, 1 distinct) | re-run 2026-11-05 on the final binary: [g8-full.log](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/docs/evidence/g8-full.log) |
| GX (greedy determinism) | Two independent greedy captures bit-identical per binary — the extension and the reference fork each self-check PASS on all five prompts (no run-to-run drift) | [gx-determinism-and-parity.log](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/docs/evidence/gx-determinism-and-parity.log) |
| Component-cache reuse | Warm-path output byte-identical to cold path for the same prompt; serve logs show `reuse=ram`, `cached_tokens=1208`, `prefill_tokens=0` at ~2.8x lower latency | manual run, 2026-11-05 |
| Reasoning split | Flint's XML `<think`/`</think` framing is registered as both `ReasoningBudgetTags` and `ReasoningDelimiters`. The host splits the block: `reasoning_content` holds the thought, `content` the answer, and `reasoning_tokens` counts the block (7/61 tokens across probes) | manual probes, 2026-11-05 |
| Reasoning budget | `--reasoning-budget 16` forces the close at 16 reasoning tokens then emits the answer; requires `TokenizeControl` to resolve the special-token pair, so it is overridden with `parse_special_tokens` | manual probes, 2026-11-05 |
| API boundary | Series-0.2 component-cache-only continuation; every include lands on the audited `gufo/ext/SURFACE` (the only exception, `gufo/ext/generated/flint_constants.hpp`, is host-generated from `extension.json`); no private `gufo/` includes | grep over the extension tree vs `include/gufo/ext/SURFACE` |
| Sampling defaults | One function sets both `TextRunnerDescriptor::sampling_defaults` and `TextProvider::sampling_defaults` (temp 0.6, top_p 0.95, top_k 20) | [runner.cpp](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/runner.cpp) `FlintSamplingDefaults` |

## Gate notes

- **G4 (logit-parity vs recorded baseline)**: 7 of 8 rows pass against the
  Oct 30 recording of the reference fork; each fork is a rank-2 near-tie
  (delta 0.014-0.355). The single failing row (`whitespace`, rank 3, delta
  0.5978) is a stale-baseline artifact, not an extension defect: rerunning the
  reference fork binary itself against the same recording fails the row
  identically — same rank, same delta 0.5978, byte-identical continuation
  text — because the reference was rebuilt on Oct 31, after the baseline was
  captured. The rebuilt reference also drifts further from the recording than
  the extension does on `tool_xml`. The row stays visible; refreshing the
  baseline from the current reference clears it. Evidence:
  [g4-ours-vs-baseline.log](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/docs/evidence/g4-ours-vs-baseline.log),
  [g4-reference-self-parity.log](https://github.com/brunobarbosaux-ai/flint-gufo-extension/blob/3788f66faf3f746fa1000e3d70742f2a98a8f434/docs/evidence/g4-reference-self-parity.log).
- **Snapshot cost**: Flint exposes only a monolithic session snapshot, so the
  component-cache adapter copies the live KV (proportional to the frontier) on
  each capture. Measured `cache_snapshot_ms` ~15.2 s at the 131072-token depth
  (13 GB payload); shallow-depth TTFT in the curve is dominated by the same
  copy, so the curve reports prefill/decode throughput and treats TTFT as a
  benchmark cost, not a correctness signal. No per-row KV access exists in the
  Flint engine API to make this incremental.
- **Bit-exactness vs the reference fork is not a gate**: both binaries are
  run-to-run deterministic (4 captures), and each reproduces the current
  reference on near-tie rows, but the reference no longer reproduces its own
  Oct 30 recordings. GX against the fork therefore checks determinism, not
  exactness; see the G4 note.

## Open / not yet validated

- [ ] Re-record the G4/GX baselines from the current reference build (the
      Oct 30 records predate the reference's Oct 31 rebuild; see gate notes).
- [ ] llama.cpp same-GGUF speed comparison (merge bar; see [BENCHMARKS](BENCHMARKS.md)).
- [ ] Tokenizer parity gates (G1) against the Strycast in-tree reference.
- [ ] Nix flake for reproducible extension builds.
