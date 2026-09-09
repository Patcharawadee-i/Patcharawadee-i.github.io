---
layout: post
lang: en
slug: vllm-rocm-gfx1150-wrong-version
title: "vLLM on a Radeon 890M was 10x slower because of the wrong ROCm version"
date: 2026-09-09 20:00:00 +0700
description: "Running vLLM on a Strix Point iGPU (gfx1150) gave me 1.17 tok/s. The bottleneck turned out not to be the hardware, but an HSA_OVERRIDE_GFX_VERSION that should never have been there."
tags: [rocm, vllm, docker, amd-igpu, benchmark]
---

## TL;DR

If you run vLLM on Strix Point (gfx1150) and you need `HSA_OVERRIDE_GFX_VERSION=11.5.1`
to keep rocBLAS from breaking, your image is too old — the hardware does not need an
override. The fix is to switch to the image AMD actually builds gfx1150 kernels for
(`rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1`) and
**delete** the `HSA_OVERRIDE_GFX_VERSION` variable. That took Qwen2.5-3B-Instruct from
1.17 tok/s to 12.61 tok/s — 10.8x faster.

## Hardware and software

| Item | Value |
|---|---|
| Machine | AMD mini PC (Strix Point) |
| CPU | AMD Ryzen AI 9 HX 370 (12 cores, 1 socket) |
| iGPU | AMD Radeon 890M — `gfx1150` |
| RAM | 64 GB DDR5-5600 SO-DIMM (dual channel) — 60 GiB visible to the OS |
| VRAM carved out by BIOS | 512 MB (536,870,912 B) |
| GTT | 44 GiB (47,244,640,256 B), set with `amd-ttm --set 44` |
| OS | Ubuntu Server 26.04.1 LTS |
| Kernel | 7.0.0-31-generic |
| Docker | 29.8.0 (build 88096ef) from the official repo |
| Test model | `Qwen/Qwen2.5-3B-Instruct` (BF16 as published on Hugging Face, not quantized) |

About that RAM figure: the machine has 64 GB installed but `free -g` reports 60. The
difference is not missing, it is reserved before the OS ever sees it. The largest single
piece is `crashkernel`, visible in `/proc/cmdline`:

```
crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M
```

Do not guess the reservation from that table — ask the kernel. `/sys/kernel/kexec_crash_size`
reports `1073741824`, so **1 GiB**, not the 2 GiB the `64G-128G` bracket suggests: by the
time the kernel picks a bracket it sees the RAM left after the firmware carve-outs, which
is under 64 GiB, so it lands in `32G-64G` instead. Add the 512 MiB the BIOS gives the iGPU
plus roughly 1.6 GiB of firmware and ACPI reservations, and `MemTotal` comes out at
60.92 GiB, which `free -g` truncates to 60.

The full accounting is in the [dual boot post]({{ '/en/posts/ubuntu-server-dual-boot-windows-11-unallocated-space/' | relative_url }}),
which is the work that came before this one. None of it affects the bandwidth arithmetic
here — capacity and read speed are unrelated.

One detail worth recording about GTT: `amd-ttm` lives at `~/.local/bin/amd-ttm` and was
not installed from apt (`dpkg -S` finds nothing), and there is no `ttm.pages_limit` in
`/proc/cmdline`. So GTT is being resized at runtime, not through a kernel parameter.
The 44 GiB I set matches the `44.0 GiB` figure that shows up in an error message later
in this post.

## It started with rocBLAS breaking

I first used the `rocm/vllm:latest` image (pulled 8 Sep 2026). Run it as-is and rocBLAS
cannot find kernel files for `gfx1150`. The workaround you find everywhere is to make
ROCm believe the machine is a `gfx1151` so it borrows that chip's kernels, by setting
`HSA_OVERRIDE_GFX_VERSION=11.5.1`.

```bash
docker run -d --name vllm \
  --device /dev/kfd --device /dev/dri \
  --group-add video --group-add render \
  --security-opt seccomp=unconfined \
  --ipc=host --shm-size 8g \
  -e HSA_OVERRIDE_GFX_VERSION=11.5.1 \
  -e VLLM_USE_TRITON_FLASH_ATTN=0 \
  -v ~/hf-cache:/root/.cache/huggingface \
  -p 127.0.0.1:8000:8000 \
  rocm/vllm:latest \
  vllm serve Qwen/Qwen2.5-3B-Instruct \
    --host 0.0.0.0 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.75
```

It ran. It answered prompts. Not a single error anywhere — it was just very slow.

## Get a number before guessing

Before theorizing about causes I needed a number I could reproduce. The important part
is `ignore_eos: true`, which forces the model to emit the full `max_tokens` every time.
Without it each run stops at a different length and the runs are not comparable.

```bash
curl -s http://127.0.0.1:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-3B-Instruct","max_tokens":100,"ignore_eos":true,"messages":[{"role":"user","content":"Write a long essay about the ocean."}]}' \
  -w '\n---\ntotal: %{time_total}s\n' | python3 -c "
import sys,json
raw=sys.stdin.read().split('\n---\n')
d=json.loads(raw[0]); t=float(raw[1].split()[1].rstrip('s'))
n=d['usage']['completion_tokens']
print(f'{n} tokens / {t:.2f}s = {n/t:.2f} tok/s')
"
```

The token count comes from `usage.completion_tokens` in the API response, and the elapsed
time from curl's `%{time_total}` — I am not reading vLLM's own log for this.

About the ports in this post, so they do not confuse you: the old container with the
override ran on **8000**, the fixed one on **8001**. That is why 12.61 tok/s is measured
against 8001. Once the comparison was done I deleted the extra container and went back to
a single one on 8000, which is what the compose file at the end of this post describes.

Result from the old container (port 8000, override in place): **1.17 tok/s**

## Narrowing it down: hardware or software

My first assumption was that the iGPU is simply this slow. So I measured two things
separately from inside the same container to find out where the ceiling was.

**Memory bandwidth** — copying a 1 GB buffer, 20 iterations:

```bash
docker exec vllm python3 -c "
import torch, time
x = torch.empty(2**30, dtype=torch.uint8, device='cuda')
y = torch.empty_like(x)
for _ in range(3): y.copy_(x)
torch.cuda.synchronize()
t = time.time()
for _ in range(20): y.copy_(x)
torch.cuda.synchronize()
dt = time.time() - t
print(f'{20 * 2 * 2**30 / dt / 1e9:.1f} GB/s')
"
```

**71 GB/s**, which is exactly what this machine should do. DDR5-5600 in dual channel has a
theoretical ceiling of 5600 MT/s × 8 bytes × 2 = 89.6 GB/s, and 65–75 GB/s in practice is
normal. Memory was not the problem.

**Compute** — a 4096×4096 bf16 GEMM:

```bash
docker exec vllm python3 -c "
import torch, time
a = torch.randn(4096, 4096, dtype=torch.bfloat16, device='cuda')
b = torch.randn(4096, 4096, dtype=torch.bfloat16, device='cuda')
for _ in range(3): torch.mm(a, b)
torch.cuda.synchronize()
t = time.time()
for _ in range(50): torch.mm(a, b)
torch.cuda.synchronize()
dt = time.time() - t
print(f'{50 * 2 * 4096**3 / dt / 1e12:.2f} TFLOPS')
"
```

**1.84 TFLOPS**

At that moment I had nothing to compare it against. All I knew was that memory looked
normal while compute looked low. So I kept the command around to re-run after fixing
whatever was wrong — which is what eventually settled the question (the answer is further
down).

## The turning point: reading vLLM's own docs

After a long detour I went back to vLLM's own installation page
([GPU installation — ROCm](https://docs.vllm.ai/en/latest/getting_started/installation/gpu/#prebuilt-wheels))
and found two lines that changed everything.

In the supported hardware list:

> MI200s (gfx90a), MI300 (gfx942), MI350 (gfx950), Radeon RX 7900 series (gfx1100/1101),
> Radeon RX 9000 series (gfx1200/1201), Ryzen AI MAX / AI 300 Series (gfx1151/1150)

And in the version requirements:

> Ryzen AI MAX / AI 300 Series requires ROCm 7.0.2 or above

(Everything else on that list only needs ROCm 6.3 or above. This chip asks for more.)

So gfx1150 has been officially supported all along and needs no override whatsoever. What
I had actually been doing was running an image built against an older ROCm, then treating
the symptom — rocBLAS not finding its kernel files — by making it borrow another chip's
kernels. That runs, but the kernels were never tuned for this chip, so it is slow.

The part that stings: this was not buried in a forum thread or a deep GitHub issue. It is
on the official installation page, which I should have read before typing `docker run` the
first time.

## Swap the image, delete the override

AMD publishes an image built specifically for gfx1150:

```
rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1
```

Switching to it and removing `HSA_OVERRIDE_GFX_VERSION` gives **12.61 tok/s**.

### Confirming which chip the container actually sees

```bash
docker exec vllm rocminfo | grep -E 'Name:|gfx|Marketing'
```

```
  Name:                    AMD Ryzen AI 9 HX 370 w/ Radeon 890M
  Marketing Name:          AMD Ryzen AI 9 HX 370 w/ Radeon 890M
  Vendor Name:             CPU
  Name:                    gfx1150
  Marketing Name:          AMD Radeon 890M Graphics
  Vendor Name:             AMD
      Name:                    amdgcn-amd-amdhsa--gfx1150
      Name:                    amdgcn-amd-amdhsa--gfx11-generic
```

The line you want is `amdgcn-amd-amdhsa--gfx1150`: the runtime is talking to the chip as
gfx1150 directly, not impersonating something else. (`gfx11-generic` is a fallback target
that also ships in the image.)

Now the memory side:

```bash
docker exec vllm rocm-smi --showmeminfo vram gtt
```

```
GPU[0]          : VRAM Total Memory (B): 536870912
GPU[0]          : VRAM Total Used Memory (B): 158003200
GPU[0]          : GTT Total Memory (B): 47244640256
GPU[0]          : GTT Total Used Memory (B): 36192239616
```

There is only 512 MB of real VRAM; the weights all live in GTT (36 GB out of 44 GB). That
is why two containers cannot run at once, which comes up again below.

### Re-running the GEMM — what 1.84 TFLOPS actually meant

Running the exact same GEMM script on the new image:

```
6.52 TFLOPS
```

**1.84 → 6.52 TFLOPS** on identical hardware with identical code. The only difference is
which kernels rocBLAS picked. That closes the "maybe the iGPU is just this slow" theory
for good.

Worth noting: GEMM improved 3.5x while end-to-end throughput improved 10.8x. So most of
the win did not come from GEMM alone — other kernels (attention, KV cache handling) are
presumably faster on the new image too. I have not measured those separately, so I am not
going to claim it.

> ⚠️ **Warning — putting `HSA_OVERRIDE_GFX_VERSION` back with the new image costs you
> roughly 10x speed and reports no error at all.** Nothing in the log, no startup warning,
> and the model still answers normally. The only difference is the tok/s number. Unless
> you deliberately benchmark the two configurations against each other, there is no way to
> notice you are affected.
{: .warn}

The 12.61 tok/s figure matches the ceiling I had calculated in advance: a 3B model in BF16
is about 6.2 GB, and generating each token means reading the whole weight set once. So the
ceiling is

```
71 GB/s ÷ 6.2 GB ≈ 11.5 tok/s
```

Put differently, after the fix the bottleneck moved to DDR5 memory bandwidth, which is
where it belongs — not to the kernels. Going faster from here means reading less data per
token (quantization), not more time spent fighting ROCm.

## The docker-compose.yml I actually use

```yaml
services:
  vllm:
    image: rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1
    container_name: vllm
    command:
      - vllm
      - serve
      - Qwen/Qwen2.5-3B-Instruct
      - --host
      - "0.0.0.0"
      - --max-model-len
      - "8192"
      - --gpu-memory-utilization
      - "0.75"
    restart: unless-stopped
    ipc: host
    shm_size: "8g"
    security_opt:
      - seccomp=unconfined
      - label=disable
    # Numeric GIDs, not group names — see the "unable to find group render"
    # section below. These are the values on my machine; check your own with:
    #   getent group render video
    group_add:
      - "44"    # video   (getent group video)
      - "991"   # render  (getent group render)
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    ports:
      - "127.0.0.1:8000:8000"
    volumes:
      - /home/<user>/hf-cache:/root/.cache/huggingface
    # Do NOT put HSA_OVERRIDE_GFX_VERSION back in! This image genuinely ships
    # rocBLAS kernels for gfx1150. Setting the override back to 11.5.1 drops you
    # back to borrowing gfx1151 kernels and costs ~10x performance
    # (12.61 tok/s -> 1.17 tok/s).
    environment:
      # VLLM_USE_TRITON_FLASH_ATTN / HSA_NO_SCRATCH_RECLAIM: workarounds for older
      # ROCm releases. Left commented out to verify ROCm 7.13 no longer needs them.
      # VLLM_USE_TRITON_FLASH_ATTN: "0"
      # HSA_NO_SCRATCH_RECLAIM: "1"
```

## Everything that got in the way

### You cannot run both versions side by side on different ports

The plan was to run the old and new images at the same time on different ports and
benchmark them back to back. That does not work: the iGPU has a single GTT pool shared
with the system, and once the first container reserves it there is nothing left for the
second.

```
ValueError: Free memory on device cuda:0 (10.45/44.0 GiB) on startup is less than desired GPU memory utilization (0.9, 39.6 GiB)
```

There is no shortcut around this. You have to `docker stop` the old one before starting
the new one, which means no side-by-side comparison — you alternate and measure one at a
time.

### `unable to find group render`

Moving from `docker run` to `docker compose` produced this:

```
Error response from daemon: unable to find group render: no matching entries in group file
```

Even though `--group-add render` had worked fine with `docker run`. The reason is that
compose resolves `group_add` names against the **image's** `/etc/group`, not the host's,
and the new image has no group named `render` inside it.

The fix is to use numeric GIDs instead of names. Get them from the host:

```bash
getent group render video
```

and put those numbers in `group_add`.

### Container name collisions

`docker run --name vllm` **creates** a container every time; it does not start the
existing one. As soon as the old container is sitting there stopped, the name collides —
and plain `docker ps` will not show it to you. You need:

```bash
docker ps -a
```

This one cost me several confused rounds before it clicked.

### Trusting the first measurement after startup is wrong

The first measurement taken immediately after the container came up was **9.18 tok/s**.
Running it again gave **12.76 tok/s**. That is pure warm-up. If you compare a first-run
number against another configuration you will reach the wrong conclusion. Fire the request
several times and only use the numbers once they settle.

### Downloading the same model twice by accident

While switching configurations back and forth I pointed the cache at two different places
(`/srv/models` and `~/hf-cache`), so the same model got pulled to disk twice and wasted
5.8 GB. Pick one cache directory up front and mount the same one every time.

## Before / after

| | Before | After |
|---|---|---|
| image | `rocm/vllm:latest` (pulled 8 Sep 2026) | `rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1` |
| `HSA_OVERRIDE_GFX_VERSION` | `11.5.1` | not set |
| kernels in use | borrowed from `gfx1151` | native `gfx1150` |
| throughput | 1.17 tok/s | 12.61 tok/s |
| speedup | 1x | 10.8x |
| GEMM 4096³ bf16 | 1.84 TFLOPS | 6.52 TFLOPS (3.5x) |
| memory bandwidth | 71 GB/s | 71 GB/s (unchanged) |
| remaining bottleneck | kernels not built for the chip | memory bandwidth (~71 GB/s) |

## What I learned

| Symptom | Actual cause | Fix |
|---|---|---|
| rocBLAS breaks unless you set `HSA_OVERRIDE_GFX_VERSION` | the image predates the ROCm release that supports this chip | find an image built for `gfx1150` and delete the override |
| 10x slower with no error | the override borrows kernels from another chip: they run, but they are not tuned | benchmark tok/s after every config change |
| second container refuses to start, complains about free memory | the iGPU shares one GTT pool | `docker stop` the old one and measure one at a time |
| `unable to find group render` under compose | `group_add` resolves against the image's `/etc/group`, not the host's | use numeric GIDs from `getent group render video` |
| container name collides while `docker ps` looks empty | `docker run` always creates; the old one is stopped, not gone | `docker ps -a` |
| 9.18 tok/s, then 12.76 tok/s later | warm-up | measure repeatedly, use the settled value |
| model eats 5.8 GB of disk twice | two different cache paths | always mount one shared cache |

The expensive lesson: **a workaround that makes it run does not mean it is correct.**
`HSA_OVERRIDE_GFX_VERSION` kept the system from crashing, so it looked like a fix, when
all it did was mask the symptom of the real problem — and I burned time benchmarking
bandwidth and TFLOPS before going back to check which ROCm version this chip actually
requires.
