# Operations runbook (RTX 5090 box)

Day-to-day start/stop and the failure modes actually hit on this machine.

[← back to the main README](../README.md) · [Docker notes](docker.md)

Box at time of writing: Arch (`eos-ext`), one RTX 5090 32 GiB, NVIDIA driver
615.71.09 (open kernel modules), Docker 29.8.1 / Compose 5.5.1, NVIDIA
Container Toolkit 1.20.0, image `ghcr.io/syv-ai/qwen38-27b-rtx3090:latest`,
port 18020. Local tuning lives in `.env` (gitignored).

## Start / stop / status

`single` and `batch` are compose **profiles** — a bare `compose up` starts no
server, only the `prepare` service (which exits 0 once the model is ready).
That is the "compose is up but nothing listens" trap.

```bash
docker compose --profile single up -d     # low-latency server (this box's default)
# docker compose --profile batch up -d    # throughput instead — one GPU, one at a time

docker compose --profile single ps
docker compose --profile single logs -f single
curl localhost:18020/health

docker compose --profile single stop single   # stop, keep container
docker compose --profile single down          # stop and remove (switching profiles)
```

First boot takes up to ~15 min (torch.compile + CUDA graphs + FlashInfer JIT;
healthcheck `start_period` is 900 s). Later boots take ~1 min (warm
`qwen-cache` volume). `/health` returning 200 is the ready signal; before that
`curl` gets `Connection reset by peer`, which is normal mid-boot.

The entrypoint runs `prepare` (idempotent) then `verify.sh --no-server`
before serving. A real verify FAIL refuses to serve and — with
`restart: unless-stopped` — the container **restart-loops** (`Up N seconds`,
`Exited (1)` in `ps -a`). A looping container means: read the verify output
in `logs`, don't just re-`up`.

## Incident 2026-09-19: stale CDI spec → `cuInit 999`

**Symptoms:** `single` restart-looping; logs end with
`FAIL torch cannot see a CUDA GPU` / `verify: 1 FAILURE(S)` /
`entrypoint: verify.sh FAILED`; `/health` connection-reset; `nvidia-smi`
works on the host *and* inside the container.

**Diagnosis trail** (so the next one goes faster):

1. `docker inspect` showed the GPU *was* requested (`DeviceRequests` nvidia),
   and `/dev/nvidia*` nodes existed in the container — not a passthrough
   misconfiguration.
2. Raw CUDA (nvcc-built `cudaGetDeviceCount`, not torch) also failed with
   `999 (unknown error)`; host CUDA was fine (`cudaMalloc OK`); a
   `--privileged` container worked (`cuInit → 0`). So: sandbox restriction,
   not driver/torch.
3. `--cap-add=ALL` and `seccomp=unconfined` each still failed → not caps/seccomp.
4. strace on `cuInit` showed `openat("/dev/nvidia-uvm", O_RDWR) = EPERM` —
   the cgroup device allow-list denied UVM while allowing `nvidiactl`/`nvidia0`
   (which is why NVML/`nvidia-smi` kept working).
5. Comparing nodes: container `/dev/nvidia-uvm-tools` was `237,1`, host was
   `511,1` — and `/etc/cdi/nvidia.yaml` pinned `major: 237` for both UVM
   nodes. The `nvidia-uvm` kernel module had reloaded since the spec was
   generated (dynamic major allocation: 237 → 511), so Docker built stale
   device nodes and a stale allow-list from it.

**Fix** (needs root; the checkout user has no passwordless sudo):

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
grep -A2 nvidia-uvm /etc/cdi/nvidia.yaml   # majors must match `ls -l /dev/nvidia*` on the host
docker compose --profile single up -d --force-recreate
docker compose --profile single exec -T single /app/venv/bin/python -c "import ctypes; print(ctypes.CDLL('libcuda.so.1').cuInit(0))"
# expect 0 before waiting out the boot
```

**If it recurs:** any `nvidia_uvm` unload/reload cycle (suspend/resume,
driver update, long idle) can shift the dynamic major again. Symptoms will be
identical (NVML fine, CUDA 999, restart loop). Re-run the two commands above.
If it happens often, worth automating the regen after modules settle at boot
— the spec on this box was last generated 20:10 but went stale later the same
evening, so boot-time generation alone didn't cover it.

## Quick reference

```bash
# GPU present and CUDA alive inside the running server container?
docker compose --profile single exec -T single nvidia-smi -L
docker compose --profile single exec -T single /app/venv/bin/python -c "import torch; print(torch.cuda.is_available())"

# host side
nvidia-smi --query-gpu=driver_version,name --format=csv
ls -l /dev/nvidia*                        # compare majors with /etc/cdi/nvidia.yaml
docker compose --profile single logs --tail=40 single
```
