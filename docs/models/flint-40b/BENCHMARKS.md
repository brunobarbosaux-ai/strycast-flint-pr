# Flint 40B benchmarks

AMD Strix Halo `gfx1151`, 128 GB unified memory. Extension
`brunobarbosaux-ai/flint-gufo-extension` at `3788f66`, built against Gufo main
`6cc8a42` (v0.31.0) with the `gpu-test` preset, ROCm 7.2.1. Weights:
`flint-UD-Q4_K_XL`. Measured 2026-11-05 over HTTP
(`/v1/chat/completions`), serial cold prefill with `--no-cache-prompt`,
3 runs per depth, 256 decode tokens per run.

**llama.cpp comparison: open.** The merge bar for model PRs requires Strycast
to run the model faster than llama.cpp runs the same GGUF on the same machine;
that comparison has not been run yet. llama.cpp serves this GGUF (it is the
pinned oracle for the distribution gate in [QUALITY](QUALITY.md)), so the
comparison is mechanically straightforward. This PR is docs-first; the cell
stays a TODO until measured.

## Serial cold prefill, single user

Depth is the prompt length in tokens. Prefill and decode are mean throughput
across the 3 runs; TTFT is the mean time to first token.

| Depth (tokens) | Prefill (tok/s) | Decode (tok/s) | TTFT (s) |
| ---: | ---: | ---: | ---: |
| 514 | 37.4 | 70.1 | 13.75 |
| 2,041 | 146.0 | 68.9 | 13.99 |
| 8,183 | 488.5 | 64.2 | 16.76 |
| 32,748 | 1,137.3 | 51.1 | 28.80 |
| 65,265 | 1,133.0 | 40.2 | 57.90 |
| 130,808 | 1,035.8 | 27.59 (sd 0.24) | 126.29 |

Shallow-depth TTFT is dominated by the monolithic session snapshot the
component-cache adapter copies per capture (about 15.2 s and a 13 GB payload
at the 131072-token frontier); prefill and decode rates are the signal there.
Raw runs and digests: [`depth-q4.jsonl` and `depth-q4-summary.json` in the
extension's `docs/evidence/`](https://github.com/brunobarbosaux-ai/flint-gufo-extension/tree/3788f66faf3f746fa1000e3d70742f2a98a8f434/docs/evidence).

## TODO

- [ ] llama.cpp same-GGUF same-machine comparison (merge bar).
- [ ] UD-Q6_K_XL column (artifact present locally, not yet measured).
- [ ] Warm/prefix-reuse timing (cache reuse is validated for correctness in
      [QUALITY](QUALITY.md), not yet benchmarked).
