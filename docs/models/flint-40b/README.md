# Flint 40B

Text generation on gfx1151 from the Flint 40B MoE GGUF, served by a standalone
external Gufo extension. The extension is not part of Strycast's tree: it
builds against an unmodified Gufo checkout via
`-DGUFO_EXTENSION_DIRS`, so it does not need porting when Strycast rebases.

[Benchmarks](BENCHMARKS.md) · [Quality](QUALITY.md)

## Extension source

- Repository: [brunobarbosaux-ai/flint-gufo-extension](https://github.com/brunobarbosaux-ai/flint-gufo-extension),
  pinned at `3788f66faf3f746fa1000e3d70742f2a98a8f434`.
- Manifest: `api_version >= 0.2.7` (component-cache-only continuation,
  reasoning delimiters, provider-defined sampling defaults), Linux, HIP.
- Contains its own operators, tokenizer support and chat framing under
  `gufo::vendor::*`; the only private header it consumes is host-generated
  from its manifest (`gufo/ext/generated/flint_constants.hpp`).
- Validated against Gufo main `6cc8a42ca84438a0566ee498a315f35c6190dc8e`
  (v0.31.0, extension API series 0.2).

**Compatibility note.** Strycast currently pins Gufo main `1c75887b`, whose
extension API is series 0.1. The extension builds against series 0.2.7 and
above — that is, against Strycast's next rebase onto Gufo main, or against a
series-0.1 compat branch of the extension on the current pin. See the PR for
the current arrangement.

## Build and run

From a Gufo checkout with its documented dependencies:

```sh
git clone https://github.com/brunobarbosaux-ai/flint-gufo-extension
cmake --preset gpu-test -DGUFO_EXTENSION_DIRS="$PWD/flint-gufo-extension"
cmake --build --preset gpu-test --target gufo --parallel 16
ctest --preset gpu-test -R '^flint\.' --output-on-failure
build/gpu-test/gufo diagnose --section extensions --json
```

A CPU build still discovers the artifact and reports the provider as
unavailable (probe-only registration).

```sh
MODEL=/path/to/flint-UD-Q4_K_XL.gguf
build/gpu-test/gufo serve llm --model "$MODEL" --context 131072 \
  --served-model-name flint
```

The server exposes OpenAI-compatible chat, streaming and reasoning controls.
See [server configuration](../../SERVER.md) and [CLI options](../../CLI.md).

## Weights

The extension consumes a GGUF artifact keyed `gguf:flint` (architecture
`flint`). The validated artifact is `flint-UD-Q4_K_XL.gguf` (Unsloth-style
UD Q4_K_XL quantization). Public distribution of the weights is pending;
they are not part of the runtime package.

## Modes

- Autoregressive text generation; no speculative decoding yet.
- Context validated to 131072 tokens over HTTP.
- Reasoning: the model's `<think>`/`</think>` framing is registered with the
  host, so responses split into `reasoning_content` and `content`, and
  `--reasoning-budget` can force an earlier close of the thinking block.

## Evidence

Quality results and open items: [QUALITY](QUALITY.md). Measurements:
[BENCHMARKS](BENCHMARKS.md). Gate evidence logs with digests live in the
extension repository under `docs/evidence/`.
