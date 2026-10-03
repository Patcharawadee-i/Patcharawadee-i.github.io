---
layout: post
lang: th
slug: docker-compose-to-microk8s-clean-host
series: infra
title: "ย้าย vLLM จาก docker-compose ไป MicroK8s: GPU หายไปไหน และทำไมห้ามตั้งชื่อ Service ว่า vllm"
date: 2026-09-11 10:00:00 +0700
description: "ย้าย vLLM จาก docker-compose ไป MicroK8s แล้วเจอสองเรื่องที่ compose ไม่เคยมี: hostPath ที่ mount /dev/kfd ให้แต่ไม่ให้สิทธิ์ cgroup และชื่อ Service ที่ไปชนกับตัวแปรภายในของ vLLM เอง"
tags: [kubernetes, microk8s, vllm, rocm, docker, amd-igpu, device-plugin]
---

## TL;DR

ย้าย vLLM จาก docker-compose ไป MicroK8s แล้วจะเจอสองอย่างที่ compose ไม่มี:
**หนึ่ง** — `hostPath` ที่ชี้ไป `/dev/kfd` กับ `/dev/dri` ไม่ได้ให้สิทธิ์ cgroup ต่างจาก
`--device` ของ docker ต้องใช้ device plugin ของ AMD แทน แล้วขอ GPU ผ่าน
`resources.limits: amd.com/gpu: 1` **สอง** — ถ้าตั้งชื่อ Service ว่า `vllm` Kubernetes
จะฉีดตัวแปร `VLLM_PORT` เข้าไปในทุก pod ซึ่งชนกับตัวแปรชื่อเดียวกันที่ vLLM ใช้เอง
แก้ด้วย `enableServiceLinks: false` หลังแก้ครบแล้ววัดได้ 12.72–12.79 tok/s
เทียบกับ 12.76 tok/s ตอนอยู่บน compose คือเท่าเดิม

## สเปกเครื่องและซอฟต์แวร์

| รายการ | ค่า |
|---|---|
| เครื่อง | มินิพีซี AMD (Strix Point) |
| CPU | AMD Ryzen AI 9 HX 370 (12 cores, 1 socket) |
| iGPU | AMD Radeon 890M — `gfx1150` |
| RAM | 64 GB DDR5-5600 SO-DIMM (dual channel) — OS มองเห็น 60 GiB |
| VRAM ที่ BIOS จองให้ | 512 MB |
| GTT | 44 GiB |
| OS | Ubuntu Server 26.04.1 LTS |
| Kernel | 7.0.0-31-generic |
| Kubernetes | MicroK8s `v1.35.6` (snap, track `1.35/stable`) |
| image | `rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1` |
| โมเดลที่ใช้ทดสอบ | `Qwen/Qwen2.5-3B-Instruct` (BF16 ไม่ได้ quantize) |

จุดตั้งต้นคือ `docker-compose.yml` จาก[โพสต์ก่อนหน้า]({{ '/th/posts/vllm-rocm-gfx1150-wrong-version/' | relative_url }})
ที่รันได้ **12.76 tok/s** เป้าหมายคือแปลงเป็น Kubernetes โดยตัวเลขต้องไม่ตก

## ทำไมต้องย้าย ทั้งที่ compose ก็รันได้อยู่แล้ว

ไม่ใช่เรื่องความเร็ว แต่เป็นกฎที่ตั้งไว้ให้เครื่องนี้: **ทุกอย่างที่รัน ต้องรันในกล่อง** บนโฮสต์มีน้อย
ที่สุด กฎนี้มาจากโพสต์ที่แล้วโดยตรง ตอนไล่หาสาเหตุที่ vLLM ช้า 10 เท่า ถ้าโฮสต์มี ROCm, virtualenv
หรือไลบรารีที่เผลอ `pip install` ทิ้งไว้ คำถามว่า "เคอร์เนลตัวไหนถูกเรียก" จะกลายเป็นงานสืบสวน

compose ทำข้อนี้ได้อยู่แล้ว แต่พอมีของหลายตัว การจัดการ dependency ระหว่างกัน การให้กลับมาเองหลัง
ไฟดับ และการเปิดออกภายนอกแบบมี auth คั่น เริ่มกลายเป็นสคริปต์ที่เขียนเอง ซึ่งก็คือ "ของบนโฮสต์" อีกแบบ

## แปลง compose เป็น Deployment ทีละบรรทัด

ใช้ **Deployment** เพราะ vLLM ไม่มี state ที่ผูกกับ pod (โมเดลอยู่ใน PVC) และ `replicas: 1` เพราะ
iGPU มีตัวเดียวและ GTT เป็นก้อนเดียว

ต้องตั้ง `strategy: Recreate` เอง เพราะค่าเริ่มต้น RollingUpdate จะ**สตาร์ต pod ใหม่ก่อนแล้วค่อยฆ่า
ตัวเก่า** บนเครื่องที่มี `amd.com/gpu` แค่ `1` pod ใหม่จะค้าง `Pending` เพราะตัวเก่ายังถือ GPU อยู่
ส่วนตัวเก่าก็ไม่ถูกฆ่าเพราะตัวใหม่ยังไม่ Ready การอัปเดตจะค้างไปเรื่อย ๆ `Recreate` ฆ่าตัวเก่าก่อน
แลกกับบริการขาดตอนช่วงโหลดโมเดล ซึ่งบนเครื่องส่วนตัวไม่ใช่ปัญหา

| docker-compose | Kubernetes |
|---|---|
| `image:` | `containers[].image` — ตัวเดียวกัน tag เดียวกัน |
| `container_name: vllm` | `metadata.name` + label `app: vllm` |
| `command:` | `containers[].command` — คำต่อคำ |
| `restart: unless-stopped` | `restartPolicy: Always` (Deployment คุมให้อยู่แล้ว) |
| `ipc: host` | `hostIPC: true` |
| `shm_size: "8g"` | `emptyDir` แบบ `medium: Memory`, `sizeLimit: 8Gi` mount ที่ `/dev/shm` |
| `security_opt: seccomp=unconfined` | `securityContext.seccompProfile.type: Unconfined` |
| `group_add: ["44", "991"]` | `securityContext.supplementalGroups: [44, 991]` |
| `devices: /dev/kfd, /dev/dri` | **ใช้ไม่ได้** — ต้องใช้ device plugin ดูหัวข้อถัดไป |
| `ports: 127.0.0.1:8000:8000` | Service แบบ ClusterIP `8000` → `8000` |
| `volumes: hf-cache` | PVC ชื่อ `hf-cache-pvc` |

`group_add` → `supplementalGroups` ย้ายมาตรง ๆ บทเรียนเดิมยังใช้ได้: ต้องใส่ GID เป็นตัวเลข
(`44` video, `991` render หาด้วย `getent group render video`) และฝั่ง Kubernetes รับแค่ integer
อยู่แล้ว เลยพลาดแบบ compose ไม่ได้ แถวที่แปลงตรง ๆ ไม่ได้เลยคือ `devices:`

## ทางตันที่ 1: ไฟล์อยู่ครบ สิทธิ์ถูกหมด แต่เรียกไม่ได้

ความคิดแรกคือแปลง `devices:` เป็น `hostPath` ตรง ๆ

```yaml
      volumes:
      - name: kfd
        hostPath:
          path: /dev/kfd
      - name: dri
        hostPath:
          path: /dev/dri
```

pod ขึ้นปกติ exec เข้าไปดูก็เห็นไฟล์ครบ permission ถูก กลุ่มตรงกับ `supplementalGroups` แต่พอ
vLLM เรียก GPU จริงกลับได้ `Operation not permitted` ไล่ผิดทางไปก่อนหลายรอบ ทั้ง GID, seccomp
และ `allowPrivilegeEscalation` — ไม่ใช่สักอัน

สาเหตุจริงคือ `--device` ของ docker ทำ **สองอย่าง**: bind-mount ไฟล์ device เข้าไป **และ** เพิ่มกฎ
ลงใน **cgroup device allow-list** ของ container ส่วน `hostPath` ทำแค่อย่างแรก และ cgroup device
controller บล็อกที่ระดับการเรียก driver โดยไม่สนว่า `ls -l` แสดงอะไร

> ⚠️ **`hostPath` ที่ชี้ไป device file ทำให้ทุกอย่างดูถูกต้อง โดยที่มันใช้งานไม่ได้จริง**
> การปฏิเสธที่ชั้น cgroup ไม่โผล่ใน `ls` ไม่โผล่ใน `id` และไม่มี event ใน `kubectl describe pod`
> ถ้าไล่ปัญหาด้วยการเช็ค permission อย่างเดียว จะไม่มีวันเจอ
{: .warn}

ทางแก้คือใช้ **device plugin** ของ AMD ซึ่งรันเป็น DaemonSet คอย advertise GPU ให้ node เวลา pod
ขอใช้ kubelet จะเรียก `Allocate()` ของ plugin แล้ว runtime จะใส่กฎ cgroup ที่ขาดไปให้ ฝั่ง pod
ไม่ต้อง mount อะไรเอง แค่ขอเป็น resource (`amd.com/gpu` คือชื่อที่ plugin ประกาศไว้)

```yaml
        resources:
          limits:
            amd.com/gpu: "1"
            memory: 32Gi
          requests:
            memory: 3Gi
```

ตัว plugin รันเป็น DaemonSet ใน `kube-system`

```bash
microk8s kubectl get ds -A
```

```
NAMESPACE     NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR              AGE
kube-system   amdgpu-device-plugin-daemonset   1         1         1       1            1           kubernetes.io/arch=amd64   24h
```

(ตัดบรรทัดอื่นออก) ตัวที่ต้องมีคือ `amdgpu-device-plugin-daemonset` ซึ่งใช้ image
`rocm/k8s-device-plugin` วิธีเช็คว่า plugin ทำงานคือดูที่ **node** ไม่ใช่ที่ pod ถ้า resource ไม่ขึ้น
pod ที่ขอ `amd.com/gpu` จะค้าง `Pending` ตลอดโดยไม่มี error บอกว่าทำไม

```bash
microk8s kubectl describe node | grep -A8 "Allocatable"
```

```
Allocatable:
  amd.com/gpu:             1
```

(ตัดบรรทัดอื่นออก) ต้องเห็นบรรทัด `amd.com/gpu: 1` ตอนใช้ `hostPath` บรรทัดนี้ไม่มี เพราะ
`hostPath` ไม่ได้บอกอะไร Kubernetes เลย

ส่วน `memory` limit `32Gi` กับ request `3Gi` เป็นของใหม่ที่ compose ไม่บังคับ request ตั้งต่ำเพราะ
weight ของโมเดลอยู่บน GTT ไม่ได้อยู่ใน RSS ของ process

## ทางตันที่ 2: ตั้งชื่อ Service ว่า `vllm` แล้ว vLLM พังทันที

สร้าง Service ชื่อ `vllm` ครอบไว้ แล้ว pod พังตอน startup ทันที

```
ValueError: VLLM_PORT 'tcp://...' appears to be a URI
```

ทั้งที่ใน Deployment ไม่มีตรงไหนตั้ง `VLLM_PORT` เลย สาเหตุคือ **service links** ฟีเจอร์เก่าของ
Kubernetes ที่ฉีดตัวแปรของทุก Service ใน namespace เข้าไปในทุก pod ในรูป `<ชื่อ SERVICE>_PORT`,
`<ชื่อ SERVICE>_SERVICE_HOST` ฯลฯ (มีมาตั้งแต่ก่อนมี DNS ในคลัสเตอร์ และยังเปิดเป็นค่าเริ่มต้น)
Service ชื่อ `vllm` จึงกลายเป็น `VLLM_PORT` ซึ่งชนกับตัวแปรภายในของ vLLM ที่ใช้สำหรับ IPC ระหว่าง
process ของ engine ค่าที่ฉีดเข้ามาเป็น address ของ Service จริง ๆ เลยดูน่าเชื่อ และชวนให้ไปแก้ `--port`
ของ vLLM ซึ่งไม่เกี่ยวเลย

ทางแก้คือปิด service links ที่ pod spec ไม่ต้องเปลี่ยนชื่อ Service ซึ่งยังเป็นชื่อที่ถูกสำหรับ DNS
ในคลัสเตอร์ (`vllm.default.svc.cluster.local`)

```yaml
    spec:
      enableServiceLinks: false
```

> ⚠️ **ปัญหานี้ไม่ได้เกิดเฉพาะกับ pod ของ vLLM เอง** service links ฉีดตัวแปรเข้า **ทุก pod ใน
> namespace** pod อื่นที่เกิดหลังจากนี้ใน `default` ก็จะได้ `VLLM_PORT` ติดไปด้วย ถ้าวันหนึ่งมี pod
> ตัวอื่นที่บังเอิญอ่านตัวแปรชื่อนี้ มันจะพังแบบเดียวกันโดยไม่มีอะไรโยงกลับมาที่ Service ชื่อ `vllm`
{: .warn}

## วัดผล: ต้องไม่ช้าลง

เกณฑ์คือ "ต้องไม่ช้าลง" เพราะสิ่งที่เปลี่ยนคือชั้นการจัดการ ไม่ใช่ชั้นคำนวณ ใช้สคริปต์เดิมจากโพสต์ก่อน
ทุกตัวอักษร (ยังต้องมี `ignore_eos: true` เพื่อบังคับให้ผลิตครบ `max_tokens`) ต่างกันแค่ต้องเปิดทาง
เข้า Service ก่อน โดยรัน `port-forward` ค้างไว้ใน terminal หนึ่ง แล้วรัน `curl` ในอีก terminal

```bash
microk8s kubectl port-forward svc/vllm 8000:8000
```

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
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

ยิง 3 รอบ ได้ **12.72–12.79 tok/s** เทียบกับ **12.76 tok/s** ของ compose คือเท่าเดิมภายในความ
คลาดเคลื่อนของการวัด (รอบแรกหลัง pod เพิ่งขึ้นจะต่ำกว่านี้เพราะ warm-up) ยืนยันว่า Kubernetes
ไม่ได้แทรกอะไรในเส้นทางคำนวณ งานคำนวณยังเป็น ROCm คุยกับ iGPU ตรง ๆ เหมือนเดิม

## Deployment กับ Service ฉบับที่ใช้จริง

จุดที่ควรสังเกตในไฟล์:

- **ไม่มี `HSA_OVERRIDE_GFX_VERSION`** และห้ามใส่กลับมา ด้วยเหตุผลเดียวกับโพสต์ก่อน (ใส่แล้ว
  ช้าลง ~10 เท่าโดยไม่มี error ใด ๆ)
- **`privileged: false`** — `privileged: true` แก้ปัญหา GPU ได้จริงเพราะข้าม cgroup device
  controller ทั้งหมด แต่คือการปิดกำแพงทั้งหลังเพื่อเปิดประตูบานเดียว device plugin ให้ผลเหมือนกัน
  โดยไม่ต้องยกสิทธิ์
- **`imagePullPolicy: Never`** — image ถูกโหลดเข้า containerd ของ MicroK8s ไว้แล้ว MicroK8s ใช้
  containerd ของตัวเอง image ที่ `docker images` เห็น Kubernetes จะไม่เห็น
- **`--gpu-memory-utilization` ยังเป็น `0.75`** เท่าฝั่ง compose ทุกค่าตั้งใจให้เหมือนเดิม ไม่งั้น
  ตัวเลข throughput จะวัดสองอย่างพร้อมกัน

<details markdown="1">
<summary>Deployment และ Service ฉบับเต็ม</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: vllm
    spec:
      # บรรทัดนี้คือทางแก้ของทางตันที่ 2 ถ้าลบออก pod จะพังตอน startup
      # เพราะ Service ชื่อ vllm จะฉีดตัวแปร VLLM_PORT ทับของ vLLM เอง
      enableServiceLinks: false
      hostIPC: true
      securityContext:
        # GID เป็นตัวเลขเท่านั้น ของเครื่องนี้เป็น 44/991 เช็คของตัวเองด้วย
        #   getent group video render
        supplementalGroups:
        - 44
        - 991
      containers:
      - name: vllm
        image: rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1
        imagePullPolicy: Never
        command:
        - vllm
        - serve
        - Qwen/Qwen2.5-3B-Instruct
        - --host
        - 0.0.0.0
        - --max-model-len
        - "8192"
        - --gpu-memory-utilization
        - "0.75"
        env:
        - name: TOKENIZERS_PARALLELISM
          value: "false"
        - name: SAFETENSORS_FAST_GPU
          value: "1"
        - name: HIP_FORCE_DEV_KERNARG
          value: "1"
        ports:
        - containerPort: 8000
          protocol: TCP
        resources:
          limits:
            # มาจาก device plugin ของ AMD ไม่ใช่ชื่อที่ตั้งเอง
            # และนี่คือสิ่งที่มาแทน devices: /dev/kfd, /dev/dri ของ compose
            amd.com/gpu: "1"
            memory: 32Gi
          requests:
            memory: 3Gi
        securityContext:
          allowPrivilegeEscalation: false
          privileged: false
          seccompProfile:
            type: Unconfined
        volumeMounts:
        - mountPath: /root/.cache/huggingface
          name: hf-cache
        - mountPath: /dev/shm
          name: shm
      volumes:
      - name: hf-cache
        persistentVolumeClaim:
          claimName: hf-cache-pvc
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 8Gi
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vllm
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: vllm
  ports:
  - port: 8000
    targetPort: 8000
```

</details>

## ผลลัพธ์: workload ทุกตัวเป็น pod แต่ Docker ยังถอดไม่ได้

หลังย้ายเสร็จ workload ทุกตัวเป็น pod หมด ตั้งใจให้โฮสต์เหลือแค่ Ubuntu Server กับ MicroK8s
แต่พอเช็ค service ที่ enable อยู่บนโฮสต์ ก็เจอสองตัวนี้

```bash
systemctl list-unit-files --type=service --state=enabled --no-pager
```

```
containerd.service    enabled
docker.service        enabled
```

(ตัดบรรทัดอื่นออก) **Docker ยังรันอยู่** ตอนแรกคิดว่าเป็นของค้างจากยุค compose แต่มันยังจำเป็น
ด้วยสองเหตุผลที่ตอนวางแผนไม่ได้คิดถึง

- **`imagePullPolicy: Never` ต้องมีคนเอา image มาวางให้** image ของ vLLM มาจาก `docker save`
  แล้ว import เข้า containerd ของ MicroK8s ต้นฉบับจึงอยู่ใน image store ของ Docker ที่เดียว ถ้า
  containerd ล้าง image ทิ้งแล้ว pod ถูก schedule ใหม่ สิ่งที่กู้คืนได้คือ Docker เท่านั้น
- **Docker คือเครื่องมือ build** image ที่ build เองบนเครื่องนี้ (อย่าง image ของ NPU ใน[โพสต์ถัดไป]({{ '/th/posts/fastflowlm-npu-kubernetes/' | relative_url }}))
  ใช้ `docker build` แล้ว `docker push` เข้า registry ภายในของ MicroK8s

> ⚠️ **"ทุกอย่างรันในกล่อง" ไม่ครอบคลุมเรื่องว่ากล่องมาจากไหน** Kubernetes รัน container ได้
> แต่ **สร้าง** container ไม่ได้ และเอา image จากเครื่องตัวเองเข้า containerd ด้วยตัวมันเองไม่ได้
> ใครที่ไล่ถอน Docker ทิ้งหลังย้ายเสร็จ จะพบว่า pod ที่ใช้ `imagePullPolicy: Never` กู้กลับมาไม่ได้อีกเลย
{: .warn}

สรุปคือ "ทุกอย่างรันในกล่อง" **จริงในระดับ workload แต่ไม่จริงในระดับเครื่องมือ** สิ่งที่ได้จริงจาก
การย้ายไม่ใช่ความเร็ว แต่คือ**ปิดเครื่องแล้วเปิดใหม่โดยไม่ต้องทำอะไร** MicroK8s สตาร์ตตอนบูตแล้ว
Deployment ทุกตัวพา pod กลับมาเอง และ**ถอดของออกง่ายพอ ๆ กับติดตั้ง** `kubectl delete` แล้วหาย
หมด ไม่ทิ้ง config ใน `/etc` หรือ service ค้างใน systemd

## สิ่งที่ได้เรียนรู้

| อาการ | สาเหตุจริง | ทางแก้ |
|---|---|---|
| `Operation not permitted` ตอนเรียก GPU ทั้งที่ไฟล์ device อยู่ครบและสิทธิ์ถูก | `hostPath` ทำแค่ bind-mount ไม่ได้ใส่กฎ cgroup ต่างจาก `--device` ของ docker ที่ทำทั้งสองอย่าง | ใช้ device plugin ของ AMD แล้วขอผ่าน `resources.limits: amd.com/gpu: 1` |
| แก้ปัญหา GPU ได้ด้วย `privileged: true` | privileged ข้าม cgroup device controller ทั้งหมด | ได้ผลจริงแต่เป็นการยกสิทธิ์เกินจำเป็น ใช้ device plugin แทน |
| `ValueError: VLLM_PORT ... appears to be a URI` ทั้งที่ไม่เคยตั้ง `VLLM_PORT` | Service ชื่อ `vllm` ทำให้ Kubernetes ฉีด `VLLM_PORT` เข้าทุก pod แล้วชนกับตัวแปรภายในของ vLLM | `enableServiceLinks: false` ที่ pod spec |
| (ป้องกันไว้ก่อน) RollingUpdate จะสตาร์ต pod ใหม่ก่อนฆ่าตัวเก่า | iGPU มี GTT ก้อนเดียว pod ใหม่จะขอไม่ได้ | ตั้ง `strategy: Recreate` ตั้งแต่แรก |
| image ที่ `docker images` เห็น Kubernetes ไม่เห็น | MicroK8s ใช้ containerd ของตัวเอง คนละที่กับ docker | โหลด image เข้า containerd ของ MicroK8s แล้วใช้ `imagePullPolicy: Never` |
| pod ค้าง `Pending` ตลอด ไม่มี error บอกสาเหตุ | node ยังไม่ประกาศ `amd.com/gpu` เพราะ device plugin ไม่ได้ทำงาน | ดูที่ node ไม่ใช่ที่ pod — `microk8s kubectl describe node` แล้วหาหัวข้อ `Allocatable` |
| ย้ายมา Kubernetes แล้วแต่ถอน Docker ทิ้งไม่ได้ | Kubernetes build image ไม่ได้ และเอา image จากเครื่องเข้า containerd เองไม่ได้ ยิ่ง pod ใช้ `imagePullPolicy: Never` ยิ่งพึ่ง Docker เต็มตัว | ยอมรับว่า Docker เป็น build/load tool ของคลัสเตอร์ หรือย้ายไป push เข้า registry จริงแล้วเลิกใช้ `Never` |
| ย้ายเสร็จแล้วความเร็วเท่าเดิมเป๊ะ | Kubernetes จัดการแค่ชั้นรอบนอก ไม่ได้แทรกในเส้นทางคำนวณ | เป็นผลลัพธ์ที่ถูกต้องแล้ว ถ้าตัวเลขเปลี่ยนต่างหากที่ควรสงสัย |

บทเรียนนี้กลับด้านกับโพสต์ก่อน คราวก่อนคือ *workaround ที่ทำให้มันรันได้ไม่ได้แปลว่ามันถูก*
คราวนี้คือ **config ที่ดูถูกทุกอย่าง ไม่ได้แปลว่ามันทำงาน** `hostPath` ผ่านทุกการตรวจที่มองเห็นด้วยตา
แต่ขาดสิ่งที่ `ls` มองไม่เห็น การจะรู้ต้องเข้าใจว่าเครื่องมือเดิม (`--device` ของ docker) ทำอะไรให้บ้าง
ไม่ใช่แค่รู้ว่าใส่แล้วมันทำงาน
