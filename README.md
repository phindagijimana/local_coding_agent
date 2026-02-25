# Local Coding Agent Setup

Minimal setup for local VS Code + Continue using a remote GPU model with Ollama.

## Stack

- Local: VS Code + Continue
- Remote: HPC/cluster scheduler + `ollama serve`
- Model: `<model-name>`
- Tunnel: `localhost:11435 -> remote 127.0.0.1:11434`

## Prerequisites

- Cluster access with GPU resources
- Continue extension installed in VS Code
- Ollama installed in user space on the remote node (`$HOME/.local/bin/ollama`)

## Quick Start

### 1) Start Ollama on remote GPU node

```bash
srun -p <partition> --gres=gpu:<gpu-type>:1 --cpus-per-task=<cpu-count> --mem=<memory> --time=<time-limit> bash -lc '$HOME/.local/bin/ollama serve'
```

### 2) Pull model (one time)

```bash
$HOME/.local/bin/ollama pull <model-name>
```

### 3) Create local SSH tunnel

Replace placeholders with your own username, jump host, and worker node:

```bash
ssh -N -L 11435:127.0.0.1:11434 -J <username>@<jump-host> <username>@<worker-node>
```

### 4) Configure Continue

```yaml
name: Local Config
version: 1.0.0
schema: v1
models:
  - name: local-ollama-model
    provider: ollama
    model: <model-name>
    apiBase: http://127.0.0.1:11435
```

Then reload VS Code window and select `local-ollama-model` in Continue.

## Verification

### Endpoint check

```bash
curl http://127.0.0.1:11435/api/tags
```

### Generation check

```bash
curl -s http://127.0.0.1:11435/api/generate -d '{"model":"<model-name>","prompt":"Respond with exactly READY","stream":false}'
```

Expected response contains `"response":"READY"`.

## Monitor Compute Usage

```bash
squeue -u "$USER" -o '%.18i %.12P %.20j %.8T %.10M %.10l %.25R %.30b'
```

```bash
srun --overlap --jobid=<JOB_ID> -N1 -n1 bash -lc 'nvidia-smi --query-gpu=timestamp,utilization.gpu,utilization.memory,memory.used,memory.total --format=csv,noheader'
```

Typical state:

- VRAM usage increases when model is loaded
- GPU utilization spikes while prompts run

## Troubleshooting

- Local port in use: change local tunnel port (`11435` -> `11436`) and update `apiBase`
- Continue cannot connect: confirm tunnel is running and job is active
- Wrong model identity in chat text: rely on `/api/tags`, `/api/chat` logs, and GPU usage
- Job expired: restart `srun` and re-create SSH tunnel
