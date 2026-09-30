# TSTC Local AI Project

A local LLM for authorized cybersecurity labs and NCL practice. This records **what we actually set up and tested** on the school PC, the commands used or provided along the way, and the GPU failure that interrupted the setup. The first model is an abliterated Qwen3.8-27B GGUF; Qwen3.8-Flash-Next and multi-PC pooling are later experiments.

## Setup at a glance (September 30, 2026)

- **PC:** CachyOS with KDE Plasma, Intel Core i9-14900KS, NVIDIA RTX 5090 (32 GiB-class VRAM), 192 GB installed RAM. No switch to the other operating systems proposed in an earlier draft was made.
- **Inference:** CUDA-enabled llama.cpp (`llama-server`), binding the unauthenticated API to **`127.0.0.1:8080` only**.
- **Model:** `huihui-ai/Huihui-Qwen3.8-27B-abliterated-GGUF`, `Huihui-Qwen3.8-27B-abliterated-Q6_K_L.gguf`.
- **Agent:** Hermes Agent through the local OpenAI-compatible Chat Completions endpoint, model ID `huihui-qwen3.8-27b`.
- **What worked before the incident:** The model loaded on the GPU; the endpoint advertised 65K, then 192K context in the recorded checks; Hermes generated short responses at the 192K *configuration*, including through a systemd user service.
- **Current limitation:** Following the GPU incident described **at the end of this file**, the operator says GPU usability has been restored by a **temporary fix** and the user service was temporarily taken out of use. We do not have post-recovery commands or output confirming the precise service-file/enabled state, the recovery method, or that the model is currently running. A longer-term fix and restoration of the service are planned, not completed.

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

## GPU issue, temporary workaround, and planned restoration

On **September 30, 2026**, the school PC's kernel log showed a **correctable PCIe physical-layer receive error** at **14:25:39 CDT**, followed by NVIDIA **Xid 79 — “GPU has fallen off the bus”** at **14:25:40 CDT**. `nvidia-smi` subsequently could not query the card (`Unable to determine the device handle ... Unknown Error`; `No devices were found`), although `lspci` still listed the RTX 5090 with the NVIDIA driver bound. Seeing the card in `lspci` did not mean the GPU was usable. The log then showed the model service loading at **14:26:01** and stopping at **14:28:38**. The bus failure was logged **before** the service stop, so the recorded stop did not cause that Xid.

We stopped using the `school-llm.service` user service as a **temporary precaution** during troubleshooting. The operator reports having **turned it off / removed it temporarily**; the last captured unit-status output (immediately after the stop) still showed the file present and the unit enabled, so we cannot say from these logs whether the file was later removed, the unit was disabled, or both. The operator later reported that the GPU is working again with a **temporary fix**, but did not capture the recovery steps or post-recovery `nvidia-smi` output here. **The underlying cause of Xid 79 is still unknown.** We have not verified that the service was restored or that Hermes is currently serving requests.

The next step is to develop a more durable fix, confirm stable GPU health, and **then** reinstate and test the systemd service. We will update this record with the actual fix, unit state, and fresh test output when available. Xid 79 is NVIDIA's “GPU has fallen off the bus” event; see [NVIDIA's Xid catalog](https://docs.nvidia.com/deploy/xid-errors/analyzing-xid-catalog.html) for its description.
