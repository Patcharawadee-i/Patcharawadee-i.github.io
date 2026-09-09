---
layout: post
lang: th
slug: vllm-rocm-gfx1150-wrong-version
title: "vLLM บน Radeon 890M ช้า 10 เท่า เพราะ ROCm ผิดเวอร์ชัน"
date: 2026-09-09
description: "รัน vLLM บน iGPU Strix Point (gfx1150) แล้วได้ 1.17 tok/s ไล่หาสาเหตุจนเจอว่าปัญหาไม่ใช่ฮาร์ดแวร์ แต่เป็น HSA_OVERRIDE_GFX_VERSION ที่ไม่ควรมีตั้งแต่แรก"
tags: [rocm, vllm, docker, amd-igpu, benchmark]
---

## TL;DR

ถ้าคุณรัน vLLM บน Strix Point (gfx1150) แล้วต้องใส่ `HSA_OVERRIDE_GFX_VERSION=11.5.1`
เพื่อไม่ให้ rocBLAS พัง แปลว่า image ที่ใช้เก่าเกินไป ไม่ใช่ว่าฮาร์ดแวร์ต้องการ override
ทางแก้คือเปลี่ยนไปใช้ image ที่ AMD build เคอร์เนลของ gfx1150 มาให้จริง
(`rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1`)
แล้ว **ลบ** ตัวแปร `HSA_OVERRIDE_GFX_VERSION` ทิ้ง
ผลคือ Qwen2.5-3B-Instruct จาก 1.17 tok/s เป็น 12.61 tok/s เร็วขึ้น 10.8 เท่า

## สเปกเครื่อง

| รายการ | ค่า |
|---|---|
| เครื่อง | มินิพีซี AMD (Strix Point) |
| CPU | AMD Ryzen AI 9 HX 370 (12 cores, 1 socket) |
| iGPU | AMD Radeon 890M — `gfx1150` |
| RAM | 60 GB, DDR5-5600 SO-DIMM (dual channel) |
| VRAM ที่ BIOS จองให้ | 512 MB (536,870,912 B) |
| GTT | 44 GiB (47,244,640,256 B) ตั้งด้วย `amd-ttm --set 44` |
| OS | Ubuntu Server 26.04.1 LTS |
| Kernel | 7.0.0-31-generic |
| Docker | 29.8.0 (build 88096ef) จาก official repo |
| โมเดลที่ใช้ทดสอบ | `Qwen/Qwen2.5-3B-Instruct` (BF16 ตามที่ Hugging Face ให้มา ไม่ได้ quantize) |

เรื่อง GTT มีจุดที่ควรบอกไว้: `amd-ttm` อยู่ที่ `~/.local/bin/amd-ttm` ไม่ได้ติดตั้งมาจาก apt
(`dpkg -S` หาไม่เจอ) และใน `/proc/cmdline` ไม่มี `ttm.pages_limit` อยู่เลย แปลว่า GTT ถูกขยาย
ตอน runtime ไม่ได้ผ่าน kernel parameter ค่า 44 GiB ที่ตั้งไว้ตรงกับเลข `44.0 GiB` ที่จะโผล่
ใน error message ช่วงหลังของโพสต์นี้พอดี

## เริ่มจาก rocBLAS พัง

ตอนแรกใช้ image `rocm/vllm:latest` (pull วันที่ 8 ก.ย. 2026) รันตรง ๆ แล้ว rocBLAS
หาไฟล์เคอร์เนลของ `gfx1150` ไม่เจอ วิธีที่เจอตามฟอรัมทั่วไปคือหลอกให้ ROCm คิดว่า
เครื่องเป็น `gfx1151` แล้วยืมเคอร์เนลของรุ่นนั้นมาใช้ ด้วยการตั้ง
`HSA_OVERRIDE_GFX_VERSION=11.5.1`

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

มันรันได้ ตอบ prompt ได้ ไม่มี error อะไรเลย — แค่ช้ามาก

## วัดความเร็วให้เป็นตัวเลขก่อน

ก่อนจะเดาสาเหตุ ต้องมีตัวเลขที่วัดซ้ำได้ก่อน จุดสำคัญคือ `ignore_eos: true`
เพื่อบังคับให้โมเดลผลิตครบ `max_tokens` ทุกครั้ง ไม่งั้นแต่ละรอบจะจบประโยคสั้นยาวไม่เท่ากัน
เอามาเทียบกันไม่ได้

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

จำนวน token อ่านจาก `usage.completion_tokens` ที่ API คืนมา ส่วนเวลาใช้ `%{time_total}`
ของ curl ไม่ได้ไปอ่านจาก log ของ vLLM

เรื่อง port ในโพสต์นี้ กันงง: ตัวเก่าที่ยังใส่ override อยู่รันบน **8000** ตัวใหม่ที่แก้แล้ว
รันบน **8001** ตัวเลข 12.61 tok/s จึงวัดจาก 8001 พอทดสอบเสร็จก็ลบทิ้งเหลือ container
เดียวกลับไปใช้ 8000 ตาม compose ท้ายโพสต์

ค่าที่ได้จากตัวเก่า (port 8000, มี override): **1.17 tok/s**

## ไล่หาสาเหตุ: ฮาร์ดแวร์ หรือซอฟต์แวร์

สมมติฐานแรกคือ iGPU มันช้าแบบนี้เอง เลยแยกวัดสองอย่างจากใน container เดียวกัน
เพื่อดูว่าคอขวดอยู่ตรงไหน

**memory bandwidth** — copy บัฟเฟอร์ 1 GB ไปกลับ 20 รอบ

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

ได้ **71 GB/s** ซึ่งเป็นค่าที่สมเหตุสมผล DDR5-5600 dual channel มีเพดานทางทฤษฎีที่
5600 MT/s × 8 bytes × 2 = 89.6 GB/s ของจริงได้ราว 65–75 GB/s ก็ถือว่าปกติ
สรุปว่า memory ไม่ใช่ปัญหา

**compute** — GEMM ขนาด 4096×4096 bf16

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

ได้ **1.84 TFLOPS**

ตอนนั้นยังไม่มีอะไรมาเทียบว่าเลขนี้ควรเป็นเท่าไร รู้แค่ว่า memory ปกติแต่ compute ดูน้อย
เลยเก็บคำสั่งนี้ไว้รันซ้ำหลังแก้ปัญหาเสร็จ — ซึ่งกลายเป็นตัวชี้ขาดในที่สุด (เฉลยอยู่ท้ายโพสต์)

## จุดพลิก: อ่านเอกสารของ vLLM เอง

หลังจากไล่ผิดทางอยู่พักใหญ่ กลับไปอ่านหน้าติดตั้งของ vLLM เอง
([GPU installation — ROCm](https://docs.vllm.ai/en/latest/getting_started/installation/gpu/#prebuilt-wheels))
แล้วเจอสองบรรทัดที่เปลี่ยนทุกอย่าง

ในรายการฮาร์ดแวร์ที่รองรับ:

> MI200s (gfx90a), MI300 (gfx942), MI350 (gfx950), Radeon RX 7900 series (gfx1100/1101),
> Radeon RX 9000 series (gfx1200/1201), Ryzen AI MAX / AI 300 Series (gfx1151/1150)

และในเงื่อนไขเวอร์ชัน:

> Ryzen AI MAX / AI 300 Series requires ROCm 7.0.2 or above

(ค่ามาตรฐานของฮาร์ดแวร์ตัวอื่นคือ ROCm 6.3 ขึ้นไป — ชิปตัวนี้ต้องการสูงกว่านั้น)

แปลว่า gfx1150 ถูกรองรับอย่างเป็นทางการอยู่แล้ว ไม่ต้อง override อะไรทั้งนั้น
สิ่งที่ทำมาตลอดคือเอา image ที่ ROCm เก่ากว่านั้นมาใช้ แล้วแก้อาการ (rocBLAS หาไฟล์
ไม่เจอ) ด้วยการหลอกให้ไปยืมเคอร์เนลของชิปอีกตัว — ซึ่งมันรันได้ แต่ได้เคอร์เนลที่ไม่ได้
tune มาสำหรับชิปตัวนี้ ก็เลยช้า

ข้อสังเกตที่เจ็บที่สุดคือ ข้อมูลนี้ไม่ได้อยู่ในฟอรัม ไม่ได้อยู่ใน issue ลึก ๆ ของ GitHub
มันอยู่ในหน้าติดตั้งอย่างเป็นทางการของ vLLM ที่ควรอ่านตั้งแต่ก่อนพิมพ์ `docker run`
ครั้งแรกด้วยซ้ำ

## เปลี่ยน image แล้วลบ override ทิ้ง

AMD build image สำหรับ gfx1150 มาให้โดยเฉพาะ:

```
rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1
```

เปลี่ยน image แล้วเอา `HSA_OVERRIDE_GFX_VERSION` ออก ผลคือ **12.61 tok/s**

### เช็คว่า container เห็นชิปเป็นอะไรจริง ๆ

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

บรรทัด `amdgcn-amd-amdhsa--gfx1150` คือสิ่งที่ต้องการเห็น — runtime คุยกับชิปเป็น gfx1150
ตรง ๆ ไม่ได้สวมรอยเป็นรุ่นอื่น (`gfx11-generic` เป็น target สำรองที่มีมาให้ใน image ด้วย)

ดูฝั่ง memory ต่อ

```bash
docker exec vllm rocm-smi --showmeminfo vram gtt
```

```
GPU[0]          : VRAM Total Memory (B): 536870912
GPU[0]          : VRAM Total Used Memory (B): 158003200
GPU[0]          : GTT Total Memory (B): 47244640256
GPU[0]          : GTT Total Used Memory (B): 36192239616
```

VRAM จริง ๆ มีแค่ 512 MB ส่วน weight ทั้งก้อนไปนั่งอยู่บน GTT (36 GB จาก 44 GB)
นี่คือเหตุผลที่รัน container สองตัวพร้อมกันไม่ได้ ซึ่งจะเล่าต่อในหัวข้ออุปสรรค

### รัน GEMM ซ้ำ — เฉลยของเลข 1.84 TFLOPS

รันสคริปต์ GEMM ตัวเดิมทุกตัวอักษรบน image ใหม่:

```
6.52 TFLOPS
```

จาก **1.84 → 6.52 TFLOPS** บนฮาร์ดแวร์เดียวกัน โค้ดเดียวกัน ต่างกันแค่เคอร์เนลที่ rocBLAS
หยิบมาใช้ ตรงนี้ปิดคดีเรื่อง "iGPU มันช้าแบบนี้เอง" ไปเลย

น่าสังเกตว่า GEMM ดีขึ้น 3.5 เท่า แต่ throughput จริงดีขึ้น 10.8 เท่า แปลว่ากำไรส่วนใหญ่
ไม่ได้มาจาก GEMM อย่างเดียว น่าจะมีเคอร์เนลอื่น (attention, การจัดการ KV cache) ที่ได้ผล
จาก image ใหม่ด้วย — ส่วนนี้ยังไม่ได้แยกวัด เลยยังไม่ฟันธง

> ⚠️ **คำเตือน — ถ้าใส่ `HSA_OVERRIDE_GFX_VERSION` กลับเข้าไปกับ image ใหม่ จะช้าลง
> ประมาณ 10 เท่าโดยไม่มี error ใด ๆ แจ้ง** ไม่มีข้อความเตือนใน log ไม่มี warning
> ตอน startup โมเดลตอบได้ปกติทุกอย่าง ต่างกันแค่ตัวเลข tok/s เท่านั้น
> ถ้าไม่ได้ตั้งใจวัดความเร็วเทียบกัน จะไม่มีทางรู้เลยว่าโดนอยู่
{: .warn}

ตัวเลข 12.61 tok/s ตรงกับเพดานที่คำนวณไว้ล่วงหน้า: โมเดล 3B ที่ BF16 มีขนาดราว 6.2 GB
และการผลิตแต่ละ token ต้องอ่าน weight ทั้งก้อนหนึ่งรอบ ดังนั้นเพดานคือ

```
71 GB/s ÷ 6.2 GB ≈ 11.5 tok/s
```

พูดอีกแบบคือหลังแก้เสร็จ คอขวดย้ายไปอยู่ที่ memory bandwidth ของ DDR5 ซึ่งเป็นที่ที่
มันควรจะอยู่ ไม่ใช่ที่เคอร์เนล การจะเร็วกว่านี้ต้องลดขนาดข้อมูลที่ต้องอ่านต่อ token
(quantize) ไม่ใช่ไปงมกับ ROCm ต่อ

## docker-compose.yml ฉบับที่ใช้จริง

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
    # ต้องใช้ GID เป็นตัวเลข ไม่ใช่ชื่อกลุ่ม — ดูหัวข้อ "unable to find group render"
    # ด้านล่าง ตัวเลขนี้เป็นของเครื่องผม ให้เช็คของตัวเองด้วย:
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
    # ห้ามใส่ HSA_OVERRIDE_GFX_VERSION กลับมา! image นี้ build เคอร์เนล
    # rocBLAS ของ gfx1150 มาให้จริงแล้ว ถ้าใส่ override กลับไปเป็น 11.5.1
    # จะตกกลับไปยืมเคอร์เนลของ gfx1151 เหมือนเดิม ทำให้ช้าลง ~10 เท่า
    # (12.61 tok/s -> 1.17 tok/s)
    environment:
      # VLLM_USE_TRITON_FLASH_ATTN / HSA_NO_SCRATCH_RECLAIM: workaround ของ
      # ROCm รุ่นเก่า ปิดไว้ (comment out) เพื่อทดสอบว่า ROCm 7.13 ไม่ต้องใช้แล้ว
      # VLLM_USE_TRITON_FLASH_ATTN: "0"
      # HSA_NO_SCRATCH_RECLAIM: "1"
```

## อุปสรรคระหว่างทาง

### รันสองตัวเทียบกันคนละ port ไม่ได้

แผนเดิมคือรัน image เก่ากับ image ใหม่พร้อมกันคนละ port แล้วยิง benchmark เทียบ
แต่ทำไม่ได้ เพราะ iGPU ใช้ GTT ก้อนเดียวร่วมกับระบบ ตัวแรกจองไปแล้วตัวที่สองก็ไม่เหลือ

```
ValueError: Free memory on device cuda:0 (10.45/44.0 GiB) on startup is less than desired GPU memory utilization (0.9, 39.6 GiB)
```

ไม่มีทางลัด ต้อง `docker stop` ตัวเก่าก่อนแล้วค่อยสตาร์ตตัวใหม่ ซึ่งแปลว่าเทียบแบบ
side-by-side ไม่ได้ ต้องสลับไปมาแล้ววัดทีละรอบ

### `unable to find group render`

พอย้ายจาก `docker run` มาเป็น `docker compose` ก็เจอ

```
Error response from daemon: unable to find group render: no matching entries in group file
```

ทั้งที่ `--group-add render` ใน `docker run` เคยใช้ได้ สาเหตุคือ `group_add` ใน compose
แปลชื่อกลุ่มจาก `/etc/group` **ของ image** ไม่ใช่ของ host และ image ตัวใหม่ไม่มีกลุ่ม
ชื่อ `render` อยู่ข้างใน

แก้ด้วยการใส่ GID เป็นตัวเลขแทนชื่อ หาจาก host ด้วย

```bash
getent group render video
```

แล้วเอาตัวเลขที่ได้ไปใส่ใน `group_add`

### container ชื่อชนกัน

`docker run --name vllm` คือการ **สร้างใหม่** ทุกครั้ง ไม่ใช่การสตาร์ตของเดิม พอตัวเก่า
ยังอยู่ในสถานะ stopped ก็ชื่อชนทันที และ `docker ps` เฉย ๆ จะไม่เห็นมัน ต้อง

```bash
docker ps -a
```

ถึงจะเห็น เรื่องนี้กินเวลาไปหลายรอบกว่าจะรู้ตัว

### วัดรอบแรกหลังสตาร์ตแล้วเชื่อเลย = ผิด

วัดครั้งแรกทันทีหลัง container ขึ้นได้ **9.18 tok/s** รันซ้ำได้ **12.76 tok/s**
เป็นผลของ warm-up ล้วน ๆ ถ้าเผลอเอาเลขรอบแรกไปเทียบกับ config อื่น จะสรุปผิดทันที
ต้องยิงซ้ำหลายรอบแล้วดูค่าที่นิ่งแล้วเท่านั้น

### โหลดโมเดลซ้ำสองที่โดยไม่ตั้งใจ

ระหว่างสลับ config ไปมา เผลอชี้ cache คนละที่ (`/srv/models` กับ `~/hf-cache`)
ผลคือโมเดลตัวเดียวกันถูกโหลดลงดิสก์สองรอบ เปลืองไป 5.8 GB
ควรกำหนด cache กลางที่เดียวตั้งแต่แรกแล้ว mount ที่เดิมทุกครั้ง

## สรุป before/after

| | ก่อน | หลัง |
|---|---|---|
| image | `rocm/vllm:latest` (pull 8 ก.ย. 2026) | `rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1` |
| `HSA_OVERRIDE_GFX_VERSION` | `11.5.1` | ไม่ตั้ง |
| เคอร์เนลที่ใช้จริง | ยืมของ `gfx1151` | ของ `gfx1150` โดยตรง |
| throughput | 1.17 tok/s | 12.61 tok/s |
| เทียบเป็นเท่า | 1× | 10.8× |
| GEMM 4096³ bf16 | 1.84 TFLOPS | 6.52 TFLOPS (3.5×) |
| memory bandwidth | 71 GB/s | 71 GB/s (ไม่เปลี่ยน) |
| คอขวดที่เหลือ | เคอร์เนลไม่ตรงชิป | memory bandwidth (~71 GB/s) |

## สิ่งที่ได้เรียนรู้

| อาการ | สาเหตุจริง | ทางแก้ |
|---|---|---|
| ต้องใส่ `HSA_OVERRIDE_GFX_VERSION` ไม่งั้น rocBLAS พัง | image เก่ากว่า ROCm ที่รองรับชิปตัวนี้ | หา image ที่ build ให้ `gfx1150` แล้วลบ override ทิ้ง |
| ช้า 10 เท่าโดยไม่มี error | override ทำให้ยืมเคอร์เนลผิดรุ่น ซึ่งรันได้แต่ไม่ได้ tune | วัด tok/s เทียบทุกครั้งที่แก้ config |
| สตาร์ตตัวที่สองไม่ขึ้น ฟ้อง free memory ไม่พอ | iGPU แชร์ GTT ก้อนเดียว | `docker stop` ตัวเก่าก่อน วัดทีละตัว |
| `unable to find group render` ใน compose | `group_add` อ่าน `/etc/group` ของ image ไม่ใช่ host | ใส่ GID เป็นตัวเลข หาด้วย `getent group render video` |
| ชื่อ container ชนทั้งที่ `docker ps` ว่าง | `docker run` สร้างใหม่เสมอ ตัวเก่ายัง stopped อยู่ | `docker ps -a` |
| วัดได้ 9.18 แล้วต่อมาได้ 12.76 | warm-up | วัดหลายรอบ ใช้ค่าที่นิ่งแล้ว |
| โมเดลกินดิสก์ซ้ำ 5.8 GB | cache คนละ path | mount cache กลางที่เดียวเสมอ |

บทเรียนที่แพงที่สุดคือ: **workaround ที่ทำให้ "มันรันได้" ไม่ได้แปลว่ามันถูก**
`HSA_OVERRIDE_GFX_VERSION` ทำให้ระบบไม่ crash เลยดูเหมือนแก้ปัญหาได้ ทั้งที่จริงมันแค่
กลบอาการของปัญหาที่แท้จริง — และเสียเวลาไปกับการวัด bandwidth กับ TFLOPS ก่อนที่จะ
กลับไปอ่านเอกสารว่าชิปตัวนี้ต้องการ ROCm เวอร์ชันอะไรกันแน่
