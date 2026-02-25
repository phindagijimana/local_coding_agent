# Local Coding Agent on OOD

Minimal setup for local VS Code + Continue using a remote OOD GPU model with Ollama.

## Stack

- Local: VS Code + Continue
- Remote: OOD + SLURM + `ollama serve`
- Model: `qwen3-coder:30b-a3b-q4_K_M`
- Tunnel: `localhost:11435 -> remote 127.0.0.1:11434`

## Prerequisites

- OOD/SLURM access with GPU (`gpu:l40s.24g:1`)
- Continue extension installed in VS Code
- Ollama installed in user space on OOD (`$HOME/.local/bin/ollama`)

## Quick Start

### 1) Start Ollama on OOD (2 days, general partition)

```bash
srun -p general --gres=gpu:l40s.24g:1 --cpus-per-task=12 --mem=64G --time=2-00:00:00 bash -lc '$HOME/.local/bin/ollama serve'
```

### 2) Pull model (one time)

```bash
$HOME/.local/bin/ollama pull qwen3-coder:30b-a3b-q4_K_M
```

### 3) Create local SSH tunnel

Replace node name with your current worker node from `squeue`:

```bash
ssh -N -L 11435:127.0.0.1:11434 -J pndagiji@ood.urmc-sh.rochester.edu pndagiji@smdodwork03
```

### 4) Configure Continue

```yaml
name: Local Config
version: 1.0.0
schema: v1
models:
  - name: qwen3-coder-local
    provider: ollama
    model: qwen3-coder:30b-a3b-q4_K_M
    apiBase: http://127.0.0.1:11435
```

Then reload VS Code window and select `qwen3-coder-local` in Continue.

## Verification

### Endpoint check

```bash
curl http://127.0.0.1:11435/api/tags
```

### Generation check

```bash
curl -s http://127.0.0.1:11435/api/generate -d '{"model":"qwen3-coder:30b-a3b-q4_K_M","prompt":"Respond with exactly READY","stream":false}'
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

- ~19 GB VRAM used when model is loaded
- GPU utilization spikes while prompts run

## Troubleshooting

- Local port in use: change local tunnel port (`11435` -> `11436`) and update `apiBase`
- Continue cannot connect: confirm tunnel is running and job is active
- Wrong model identity in chat text: rely on `/api/tags`, `/api/chat` logs, and GPU usage
- Job expired: restart `srun` and re-create SSH tunnel
