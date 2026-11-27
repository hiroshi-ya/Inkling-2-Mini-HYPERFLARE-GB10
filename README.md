---
license: apache-2.0
pipeline_tag: image-text-to-text
tags:
- hyperflare
- flare
- inkling-2-mini
- sm121
- gb10
- long-context
- image-text-to-text
base_model:
- thinkingmachines/Inkling-2-Mini
---

# Inkling-2-Mini HYPERFLARE — TP1, GB10
 
A prepared [Hyperflare](https://github.com/NVIDIA/hyperflare-inference) (at `b9cac839`) runtime
package for **Inkling-2-Mini** running on a single **DGX Spark** (GB10 / SM121). The base checkpoint for the model is
[`thinkingmachines/Inkling-2-Mini`](https://huggingface.co/thinkingmachines/Inkling-2-Mini)
at `ed40e7da`. This package can be served with vision support, a full **1,048,576-token** context, and around **4.1 million KV tokens**.

[![Static Badge](https://img.shields.io/badge/github-repo-black?style=for-the-badge&logo=github)](https://github.com/hiroshi-ya/Inkling-2-Mini-HYPERFLARE-GB10) &nbsp; [![Static Badge](https://img.shields.io/badge/hugging%20face-model%20card-%23FFD21E?style=for-the-badge&logo=huggingface)](https://huggingface.co/hiroshiya/Inkling-2-Mini-HYPERFLARE-GB10)

## Before you get your hands on

The current implementation has a very low decode throughput. Check [`llama-benchy`](https://github.com/eugr/llama-benchy) results at [`BENCH.md`](BENCH.md). These results were obtained via:
```console
$ flare run $INKLING_2_MINI_MODEL_PATH \
  --mtp-draft-tokens 5 \
  --host 0.0.0.0 \
  --port 8000
```

[`BENCH_MTP_5_NoHistoryDraft.md`](BENCH_MTP_5_NoHistoryDraft.md) and [`BENCH_NO_MTP.md`](BENCH_NO_MTP.md) are also available. The former results were obtained via:
```console
$ flare run $INKLING_2_MINI_MODEL_PATH \
  --mtp-draft-tokens 5 \
  --no-history-drafts \
  --mtp-adaptive-verification-alpha 0.50
  --host 0.0.0.0 \
  --port 8000
```
And the latter results were obtained via the core that has its drafter disabled via `ignition export --no-mtp`.

## How to serve

Prerequisites:
1. A relatively new pair of `rustc` and `cargo`.
2. A working DGX Spark with a working Nvidia driver (R580 or later).
3. At least ~25 GiB of available storage space: `hyperflare` needs about 10 GiB to compile, and this package is about 21 GiB.
4. A way to download large files from huggingface.

### Package properties
| Model                | Inkling-2-Mini                   |
| -------------------- | -------------------------------- |
| license              | apache-2.0      |
| architecture         | sm_121                           |
| context              | 1,048,576 tokens                 |
| prefill chunk        | 4,096 tokens                     |
| concurrency          | 8 requests · 1 ranks             |
| default output limit | remaining context                |
| vocabulary           | 200,058                          |
| cache histories      | continuation state, paged key value  |
| speculation          | mtp · widths 1, 6    |

> [!NOTE]
> As of `b9cac839`, Hyperflare does not support serving Inkling-2-Mini via a TP=2 setup. Besides, one cannot freely adjust the amount of draft tokens, meaning that `--mtp-draft-tokens <N>` can either be the exported core's  configuration (`N=5` in this case) or zero. If you want to adjust the model properties, you have to export your own package. Here is a detailed [guide](https://github.com/hiroshi-ya/GLM-5.3-Flash-HYPERFLARE-TP2/blob/main/README.md#reproduction) on how to export your own Hyperflare core.

### Obtain `flare`
Install the following build dependencies:
```console
# apt install build-essential cmake ninja-build pkg-config protobuf-compiler libssl-dev python3-dev liburing-dev libibverbs-dev
```
If on Fedora:
```console
# dnf install @development-tools cmake ninja-build pkgconf-pkg-config protobuf-compiler openssl-devel python3-devel liburing-devel rdma-core-devel libnl3
```

Then, obtain `flare` from source:
```console
$ git clone https://github.com/NVIDIA/hyperflare-inference
$ cd hyperflare-inference
$ # git checkout b9cac839 # in case a breaking change happens
$ cargo build --release --manifest-path flare-cli/Cargo.toml
```
After a successful compilation, the `flare` executable binary will be generated in `hyperflare-inference/flare-cli/target/release/`.

### Obtain this model package
If you have the `hf` CLI tool:
```console
$ hf download hiroshiya/Inkling-2-Mini-HYPERFLARE-GB10 \
    --revision gb10 \
    --local-dir /your/desired/path/
```

> [!IMPORTANT]
> You have to specify `--revision` as the `main` branch does not include the model.

If you do not have `hf`, `flare` itself comes with a `download` function:
```console
$ flare download hiroshiya/Inkling-2-Mini-HYPERFLARE-GB10
```

`flare` (running on a GB10 system) will automatically pull the `gb10` branch from this huggingface repository.

The `gb10` branch (in the huggingface repository) contains the Hyperflare core that was exported via:
```console
$ ignition export thinkingmachines/Inkling-2-Mini \
    --gpus 0 \
    --max-concurrency 8 \
    --mtp-draft-tokens 5 \
    --prefill-chunk-size 4096 \
    --speculative-token-capacity 96 \
    --speculative-graph-budget 64
```

If you would like to use a core with its drafter completely disabled, you can pull the `gb10-no-mtp` branch instead:
- Replace `--revision gb10` with `--revision gb10-no-mtp` if using `hf download`
- Replace `hiroshiya/Inkling-2-Mini-HYPERFLARE-GB10` with `hiroshiya/Inkling-2-Mini-HYPERFLARE-GB10@gb10-no-mtp` if using `flare download`

### Serve
Verify the existence of the model:
```console
$ export $MODEL_PATH= # if using /a/customized/path
$ flare inspect $MODEL_PATH
$ # flare inspect thinkingmachines/Inkling-2-Mini # if you downloaded the model using `flare download`
```

Then serve:
```console
$ export SERVER_IP=     # IP address of your serving address
$ export SERVER_PORT=   # Port of your serving address
$ flare run $MODEL_PATH \
    --publisher-keys $MODEL_PATH/publisher.json \
    --host $SERVER_IP \
    --port $SERVER_PORT
```

When serving for the first time, `flare` will ask you if you trust the publisher (on both nodes). After you trust the publisher, `flare` will generate/update the `trust.toml` in `~/.config/flare/` by default, and the `--publisher-keys` argument will no longer be needed in the future.

After the server is ready:
```console
$ curl http://$SERVER_IP:$SERVER_PORT/v1/models # check if alive
$ curl http://$SERVER_IP:$SERVER_PORT/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "inkling-2-mini",
    "messages": [
      {"role": "user", "content": "hi how are you"}
    ]
  }'
```

## Reproduction

If you want to export the Hyperflare core by yourself, I wrote a detailed step-by-step [guide](https://github.com/hiroshi-ya/GLM-5.3-Flash-HYPERFLARE-TP2/blob/main/README.md#reproduction) on exporting a Hyperflare package. It uses `GLM-5.3-Flash` as an example. However, as previously stated, `Inkling-2-Mini` cannot be exported as a TP=2 core by Hyperflare yet, so the steps will be slightly different (and easier).

## Credits

This repository is a packaging, validation, and publication effort only. The model, runtime, recipes, kernels, and benchmark tooling are the work of others. No authorship claim is made over any upstream work.

- **Local Inference Lab and the NVIDIA Hyperflare (formerly Flare) authors and contributors** — runtime, recipes, package format, and documentation ([`NVIDIA/hyperflare-inference`](https://github.com/NVIDIA/hyperflare-inference)).
- **Thinking Machines Lab** — the Inkling-2-Mini checkpoint ([`thinkingmachines/Inkling-2-Mini`](https://huggingface.co/thinkingmachines/Inkling-2-Mini)).

Packaged, validated, and published on an Asus GX10. The package's publisher identity is a signing identity, not an authorship claim.

All credits for the model, engine, kernels, and fixes belong to the projects and people listed above.

## License

Weights were derived from
[`thinkingmachines/Inkling-2-Mini`](https://huggingface.co/thinkingmachines/Inkling-2-Mini), licensed under `Apache-2.0`.

Hyperflare is licensed under `Apache-2.0`.
