# TSTC Local AI Project

A local LLM for authorized cybersecurity labs and NCL practice. This records **what we actually set up and tested** on the school PC, the commands used or provided along the way, and the results. **Phase 1** was an abliterated Qwen3.8-27B GGUF on llama.cpp. **Phase 2 (measured October 5, 2026)** runs Qwen3.8-Flash-Next — a 125B mixture-of-experts model — on [Strata](https://github.com/Niko1221/Strata) at **80.4 tok/s on a single RTX 5090**; see the Phase 2 section below. The September 30 GPU incident has since been resolved (details at the end).

## Setup at a glance (September 30, 2026)

- **PC:** CachyOS with KDE Plasma, Intel Core i9-14900KS, NVIDIA RTX 5090 (32 GiB-class VRAM), 192 GB installed RAM. No switch to the other operating systems proposed in an earlier draft was made.
- **Inference:** CUDA-enabled llama.cpp (`llama-server`), binding the unauthenticated API to **`127.0.0.1:8080` only**.
- **Model:** `huihui-ai/Huihui-Qwen3.8-27B-abliterated-GGUF`, `Huihui-Qwen3.8-27B-abliterated-Q6_K_L.gguf`.
- **Agent:** Hermes Agent through the local OpenAI-compatible Chat Completions endpoint, model ID `huihui-qwen3.8-27b`.
- **What worked before the incident:** The model loaded on the GPU; the endpoint advertised 65K, then 192K context in the recorded checks; Hermes generated short responses at the 192K *configuration*, including through a systemd user service.
- **GPU incident, resolved:** The September 30 Xid 79 event described at the end of this file has been resolved; the GPU has been stable since, including through a 9.5-minute sustained Phase 2 inference run at 37 °C with no recurrence. The GPU power limit is held at 450 W as a precaution.

This is a lab record, not a claim that a near-192K prompt, reliable tool use, model checksum, or later auto-start were tested.

## What we did, in order

The school-PC commands and outputs below were provided by the operator; this repository is documentation, not a checkout of the school machine. The exact package-install commands for CachyOS, the llama.cpp CUDA build, and Hermes were **not captured**, so none are invented here. Commands shown for downloading or checking a file reproduce the provided procedure; where no command output was saved, that is stated.

### 1. Confirm the GPU and executable

The operator installed llama.cpp with CUDA support and Hermes, then ran:

```bash
llama-server --list-devices
command -v llama-server
```

`--list-devices` reported `CUDA0: NVIDIA GeForce RTX 5090 (32202 MiB, 31640 MiB free)`; `command -v llama-server` returned `/usr/bin/llama-server`. We also used `nvidia-smi` during model-load checks. These are observations from **before** the GPU incident, not current health checks.

### 2. Choose and download one model file

We chose Q6_K_L to leave more GPU headroom than Q8_0. The operator reported that the download finished and the service later used a file at this location. The pinned download procedure supplied was:

```bash
mkdir -p "$HOME/models/huihui-qwen3.8-27b"
hf download huihui-ai/Huihui-Qwen3.8-27B-abliterated-GGUF \
  Huihui-Qwen3.8-27B-abliterated-Q6_K_L.gguf \
  --revision 3f101cd22b7999228bbd5d79a33975414eb9758b \
  --local-dir "$HOME/models/huihui-qwen3.8-27b"
```

The chosen file's publisher LFS SHA-256 is `20b1214a23f0f5f401b6117cbd63336cfff6c07be4df93bd6e374d8fd8d12938`. **No school-PC checksum output was recorded**; verify it before treating the local file as checked:

```bash
sha256sum "$HOME/models/huihui-qwen3.8-27b/Huihui-Qwen3.8-27B-abliterated-Q6_K_L.gguf"
```

Keep the model weights out of this Git repository. Review the [publisher's model card](https://huggingface.co/huihui-ai/Huihui-Qwen3.8-27B-abliterated-GGUF) for model provenance and licensing.

### 3. Start llama.cpp manually and validate the API

We first tested a **65,536-token** context. The operator's checks were:

```bash
curl -fsS http://127.0.0.1:8080/v1/models
nvidia-smi
```

`/v1/models` reported `n_ctx: 65536`; `nvidia-smi` showed **25,486 MiB / 32,607 MiB** GPU memory in use, with **25,422 MiB** attributed to `llama-server`. The 128K trial used `-c 131072`; the operator reported approximately **28 GB / 32 GB** VRAM in btop and a short Hermes reply, but that Hermes session still displayed `65.5K pinned` until its configuration was adjusted. This was not a long-prompt test.

We then selected a **196,608-token (192K) server configuration**, with one parallel slot and q8_0 KV caches. The manual launch was:

```bash
llama-server \
  -m "$HOME/models/huihui-qwen3.8-27b/Huihui-Qwen3.8-27B-abliterated-Q6_K_L.gguf" \
  --alias huihui-qwen3.8-27b \
  -ngl 99 -c 196608 -np 1 \
  --flash-attn on \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --jinja --host 127.0.0.1 --port 8080
```

The next `/v1/models` result showed `n_ctx: 196608`, and `nvidia-smi` showed **30,478 MiB / 32,607 MiB** in use, with **30,414 MiB** attributed to `llama-server`. That leaves **2,129 MiB idle** in this snapshot. An advertised window and a short generated reply do **not** establish that requests approaching 192K tokens fit or run reliably. If memory headroom is inadequate, evaluate `-c 131072` and match Hermes's context to that lower value. Do not run this manual command while another server owns port 8080.

### 4. Connect Hermes Agent

In the `hermes model` setup, we chose a **Custom endpoint** at `http://127.0.0.1:8080/v1`, **Chat Completions** compatibility, and model ID `huihui-qwen3.8-27b`. The server was bound to loopback; it was not meant to be accessible across the campus network. We set the agent's context to match the 192K server:

```bash
hermes config set model.context_length 196608
hermes chat -q "Reply with one short sentence."
```

Hermes returned a short answer and displayed `196.6K pinned`. Earlier, after the manually launched `llama-server` terminal was closed, Hermes reported `Transient APIConnectionError on custom`; restarting the server restored the connection. This is why we tried running it as a user service. See the [Hermes provider documentation](https://hermes-agent.nousresearch.com/docs/integrations/providers/) for the current provider wizard.

### 5. Put the working server under systemd (historical setup)

Before the unit was created, `command -v llama-server` showed `/usr/bin/llama-server` and `systemctl --user cat school-llm.service` returned `No files found for school-llm.service.` The operator ran:

```bash
systemctl --user edit --force --full school-llm.service
```

The unit used this configuration (with `%h` standing for the current user's home):

```ini
[Unit]
Description=School Qwen3.8-27B llama.cpp server

[Service]
Type=exec
ExecStart=/usr/bin/llama-server -m %h/models/huihui-qwen3.8-27b/Huihui-Qwen3.8-27B-abliterated-Q6_K_L.gguf --alias huihui-qwen3.8-27b -ngl 99 -c 196608 -np 1 --flash-attn on --cache-type-k q8_0 --cache-type-v q8_0 --jinja --host 127.0.0.1 --port 8080
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

After stopping the manual server, the operator enabled and started the unit and checked it:

```bash
systemctl --user enable --now school-llm.service
systemctl --user status school-llm.service --no-pager
curl -fsS http://127.0.0.1:8080/v1/models
hermes chat -q "Reply with one short sentence."
```

The first `curl` was issued **8 ms after start** and got `Connection refused` while the model loaded. A later check showed the unit **enabled and active**, the server listening on loopback, `/v1/models` showing `n_ctx: 196608`, and Hermes giving a short response without a manual server terminal. This verified that the service worked **then**; a new-login auto-start test was not recorded. **This is historical configuration, not proof that the service is currently enabled or running.**

For ordinary operation, `systemctl --user stop school-llm.service` unloads the model; `systemctl --user start school-llm.service` loads it again **if the unit exists and the GPU is healthy**. `stop` alone does not prevent auto-start at a later login. After the GPU incident we did **not** verify whether the unit file was deleted or just disabled, so inspect `systemctl --user cat school-llm.service` and `systemctl --user is-enabled school-llm.service` before reusing these commands. Do not re-enable it automatically based solely on the earlier success.

## What remains to test

- Confirm the GPU's post-recovery health with fresh `nvidia-smi` and kernel logs; verify the local model checksum.
- Benchmark short and progressively longer prompts, generation speed, VRAM and reliability. The 192K setting has only been used for model load and short responses.
- Verify Hermes tool calls. Test MTP on/off with measured draft acceptance and tokens/sec instead of assuming a speedup.
- Once the GPU issue has a durable fix, decide whether to restore a user service and test startup after a fresh login. If the old unit still exists, inspect it; if it was removed, recreate it from the historical configuration above. **Only then**, with a healthy GPU and an intentional auto-start decision, use `systemctl --user enable --now school-llm.service`.
- Evaluate Qwen3.8-Flash-Next separately; pooling a second PC is not necessary for this 27B milestone.

The repo should contain documentation and reproducible, sanitized results, **not** weights, passwords/API tokens, campus IPs/hostnames, raw diagnostic bundles, or private notes. This project does not yet specify a license for its own repository contents; an upstream model license is a separate matter.

## GPU incident (September 30, 2026) — resolved

On **September 30, 2026**, the school PC's kernel log showed a **correctable PCIe physical-layer receive error** at **14:25:39 CDT**, followed by NVIDIA **Xid 79 — "GPU has fallen off the bus"** at **14:25:40 CDT**. `nvidia-smi` could not query the card (`Unable to determine the device handle ... Unknown Error`), although `lspci` still listed the RTX 5090 with the NVIDIA driver bound. The model service loaded at **14:26:01** and stopped at **14:28:38**; the bus failure was logged **before** that stop, so the service stop did not cause the Xid.

**Resolution:** the GPU was restored and has been stable since. The most demanding test to date — the Phase 2 Strata run below, a continuous 9.5-minute generation at the 450 W power limit — completed with the GPU holding **37 °C and no Xid recurrence**, and with zero disk-read stalls. The root cause was not formally diagnosed, and the 450 W power cap (below the 576 W stock TDP) is kept in place as a precaution. Xid 79 is NVIDIA's "GPU has fallen off the bus" event; see [NVIDIA's Xid catalog](https://docs.nvidia.com/deploy/xid-errors/analyzing-xid-catalog.html) for its description.

---

## Phase 2 — Strata with UD-Q4_K_XL (measured, October 5, 2026)

Phase 1 ran a 27B model on llama.cpp. **Phase 2 runs Qwen3.8-Flash-Next — a 125B mixture-of-experts model — in Unsloth's UD-Q4_K_XL quantization on [Strata](https://github.com/Niko1221/Strata), on a single consumer GPU.**

**Why this is hard, and why this PC:** UD-Q4_K_XL has 71.7 GiB of routed expert weights — far more than fits in a 32 GB GPU. Strata keeps a resident budget of those experts in system RAM (`RAM − 24 GB`, capped at the full 71.7 GiB) and streams only the ones each token needs onto the GPU. On a 64 GB machine most experts are read from the SSD on every token, which caps this quant at **7–8.5 tok/s**. With 192 GB of RAM, every expert stays resident and nothing is read from disk during inference — the first hardware class where this quant runs at full speed.

### Configuration
`--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5 --mtp rt --max-context 131072 --kv int8 --resident-budget-gib 71`, served OpenAI-compatible on loopback (`127.0.0.1:8080`), consumed through Hermes Agent.

### Measured results

A single sustained generation — 45,469 output tokens over 566.6 s at 131K-token context:

| Metric | Value |
|---|---:|
| **Sustained decode** | **80.4 tok/s** |
| Median of 554 engine samples | 80.6 tok/s (range 53.3–87.3) |
| Prefill | 136 tok/s |
| Expert cache in VRAM | 7,534 experts · 22.0 GiB |
| VRAM expert-cache hit rate | 90.9% (6.3% of routed experts over PCIe) |
| **Disk read during the 9.5-minute run** | **0.0 MB/s — zero SSD spillover** |
| System RAM used | 62.4 / 188 GB |
| GPU temperature / power | 37 °C @ 450 W |
| Cold-start warm-up | 53.3 tok/s first sample → ~80 by 14 s |

This result is at 131K context; the model is trained for 256K, so the window can be doubled — measurement at full depth is the next step.

### What the numbers show
- **80.4 tok/s on a 125B model, one consumer GPU, at a reduced 450 W power limit.** That is **~2.6×** the only comparable published figure for this quant (31 tok/s on an RTX 3090 + 165 GB, from Strata's docs) and **~10×** what it does on a 64 GB machine.
- **0.0 MB/s disk read for the whole run** is the proof the design works: with 192 GB of RAM, no expert is ever read from the SSD during inference.
- A calibration sweep found a **non-obvious optimum**: letting ~20% of the expert computation go over PCIe to the CPU is **+8.9% faster** than forcing everything onto the GPU, and pushing it to 75% is slower than 0% — a real optimum, not a trend. On a 5090 + 192 GB rig the CPU/RAM path is an asset, not a bottleneck.

### Honesty notes
These are informal measurements from the engine log and server UI, under a reasoning-heavy workload (~65% of output was reasoning tokens) — not the formal greedy-decode protocol. The protocol run (greedy, 256-token cap, 3 runs each at 4K/32K/128K, with telemetry) is still to be recorded before submitting these as a community benchmark to the Strata project.
