---
layout: post
lang: th
slug: fastflowlm-npu-kubernetes
series: infra
title: "รัน FastFlowLM บน NPU ใน Kubernetes: ulimit ที่ไม่มีอยู่จริง กับ device plugin ที่ต้องเขียนเอง"
date: 2026-09-11 16:00:00 +0700
description: "เอา NPU ของ Strix Point (XDNA2) มารันโมเดลใน MicroK8s เจอสองช่องว่างที่ Kubernetes ไม่มีให้เลย คือ ulimit ต่อ pod และ device plugin ของ NPU ต้องสร้างเองทั้งคู่"
tags: [kubernetes, microk8s, npu, xdna2, fastflowlm, amd, device-plugin, runtimeclass]
---

## TL;DR

จะเอา NPU มาใช้ใน Kubernetes ต้องเติมของที่ Kubernetes ไม่มีให้สองอย่าง
**หนึ่ง** — Kubernetes **ไม่มีฟิลด์สำหรับตั้ง ulimit ต่อ pod** (ต่างจาก `--ulimit` ของ docker)
แต่ FastFlowLM ต้องการ `memlock` แบบ unlimited ทางแก้คือ containerd runtime handler ตัวที่สอง
ผูกกับ RuntimeClass ให้เฉพาะ pod ที่ขอใช้ **สอง** — AMD ไม่มี device plugin ทางการสำหรับ NPU
เลยต้องเขียนเอง ~140 บรรทัด ผลคือ pod ของ FastFlowLM ใช้ NPU ได้โดยไม่ต้อง `privileged: true`

## สเปกเครื่องและซอฟต์แวร์

| รายการ | ค่า |
|---|---|
| เครื่อง | มินิพีซี AMD (Strix Point) |
| CPU | AMD Ryzen AI 9 HX 370 (12 cores, 1 socket) |
| NPU | AMD XDNA2 — `/dev/accel/accel0` |
| kernel driver | `amdxdna` |
| iGPU (ของเดิม) | AMD Radeon 890M — `gfx1150` |
| OS | Ubuntu Server 26.04.1 LTS |
| Kernel | 7.0.0-31-generic |
| Kubernetes | MicroK8s `v1.35.6` |
| engine | FastFlowLM (build เองจาก source) |
| โมเดล | Qwen2.5-3B quantize 4-bit รูปแบบ `q4nx` |

คลัสเตอร์ตัวเดียวกับที่[ย้าย vLLM จาก docker-compose มาลงไว้]({{ '/th/posts/docker-compose-to-microk8s-clean-host/' | relative_url }})
กฎเดิมยังใช้อยู่: ทุกอย่างรันเป็น pod และห้ามยกสิทธิ์เกินจำเป็น

## NPU ใช้งานได้อยู่แล้วก่อนเริ่ม

```bash
lsmod | grep amdxdna
lspci | grep -i neural
ls -l /dev/accel/accel0
```

kernel driver `amdxdna` โหลดอยู่แล้ว PCI รายงานอุปกรณ์เป็น
`Strix/Krackan/Strix Halo Neural Processing Unit` และมี device node ที่ `/dev/accel/accel0`
ถ้าสามอย่างนี้ขาดอันใดอันหนึ่ง ปัญหาอยู่ที่ kernel/BIOS ไม่ใช่ที่ Kubernetes

## เปลี่ยนแผนตั้งแต่ยังไม่เริ่ม

แผนเดิมคือใช้ตัวห่อชื่อ `lemonade-npu-toolbox` เพราะคิดว่าชิปตัวนี้อาจไม่รองรับ แต่พอไล่ดูจริงกลับพบว่า
**Strix Point เป็นแพลตฟอร์มหลักที่ FastFlowLM ใช้ทำ benchmark ของตัวเอง** เลยใช้ FastFlowLM
ตรง ๆ แทน ไม่ต้องมีชั้นที่ต้องเดาว่ามันทำอะไรให้บ้าง

> ⚠️ **สมมติฐาน "ฮาร์ดแวร์เราคงไม่มีใครรองรับ" ทำให้เสียเวลาได้พอกับสมมติฐานตรงข้าม**
> ก่อนตัดสินใจ ให้ไปดูว่าโปรเจกต์นั้นทดสอบบนอะไรจริง ๆ
{: .warn}

## ทางตันที่ 1: build image ไม่ผ่าน เพราะ test ต้องใช้ NPU จริง

FastFlowLM ไม่มี image สำเร็จรูป ต้อง build เองเป็น 3 stage: build XRT (Xilinx Runtime) จาก
source, build FastFlowLM แล้วเอาเฉพาะของที่ใช้ตอนรันไปใส่ runtime image ตัวเล็ก ได้ image ราว
**150 MB** ใช้เวลา build ราว **15–25 นาที**

stage แรกพัง เพราะสคริปต์ build ของ XRT รัน **ชุดทดสอบ CTest** ต่อท้าย และมี test ที่ต้องคุยกับ
NPU จริง แต่ `docker build` ไม่มี NPU ให้ — device เข้าถึงได้เฉพาะตอน `docker run` พร้อม `--device`
ทางแก้คือ flag `-noctest` และใน stage เดียวกันมี flag พี่น้องที่มีไว้ด้วยเหตุผลเดียวกัน คือ `-nokmod`
ที่ข้ามการ build kernel module สองตัวนี้ต้องไปอ่าน `build.sh` เองถึงจะรู้ว่ามี

```
./build.sh -npu -opt -noctest     # XRT base — ข้าม test ที่ต้องใช้ NPU
./build.sh -release -nokmod       # XRT NPU plugin — ข้ามการ build kernel module
```

> ⚠️ **แยกให้ออกระหว่าง "build ไม่ผ่าน" กับ "test ไม่ผ่าน"** ใน Dockerfile ทั้งสองอย่างจบเหมือนกัน
> คือ layer ล้ม แต่รอบนี้ build สำเร็จไปแล้ว ที่ fail คือ test ถ้าเห็นคำว่า fail แล้วรีบไปแก้
> dependency จะไม่มีทางเจอ เพราะของที่ขาดคือฮาร์ดแวร์ ซึ่ง `docker build` ให้ไม่ได้โดยการออกแบบ
{: .warn}

<details markdown="1">
<summary>Dockerfile ฉบับเต็มที่ใช้จริง (3 stage)</summary>

```dockerfile
# =============================================================================
# FastFlowLM on AMD Ryzen AI NPU — Ubuntu 24.04
# =============================================================================
# Runs LLMs on the AMD XDNA/XDNA2 NPU (Strix Point, Kraken Point, etc.) on Linux.
#
# Prerequisites (on the HOST, not in the container):
#   - AMD Ryzen AI processor with NPU (Strix Point / Kraken Point / etc.)
#   - Linux kernel 6.11+ with amdxdna driver (in-tree from 6.14+, or via amdxdna-dkms)
#   - NPU device visible at /dev/accel/accel0
#   - NPU firmware in /lib/firmware/amdnpu/ (or /usr/lib/firmware/amdnpu/)
#   - Docker with --device passthrough support
#
# Build:
#   docker build -t fastflowlm .
#
# Run (interactive chat):
#   docker run -it --rm \
#     --device=/dev/accel/accel0 \
#     --ulimit memlock=-1:-1 \
#     -v ~/.config/flm:/root/.config/flm \
#     fastflowlm run llama3.2:1b
#
# The model cache is in /root/.config/flm inside the container.
# Mounting it as a volume avoids re-downloading models on every run.
#
# Other examples:
#   docker run ... fastflowlm list              # list available models
#   docker run ... fastflowlm pull qwen3:1.7b   # download a model
#   docker run ... fastflowlm validate          # check NPU setup
#   docker run ... fastflowlm serve             # OpenAI-compatible API server
#
# (where "..." = --device=/dev/accel/accel0 --ulimit memlock=-1:-1 -v ~/.config/flm:/root/.config/flm)
# =============================================================================

# ---------------------
# Stage 1: Build XRT from source
# ---------------------
FROM ubuntu:24.04 AS xrt-builder

ENV DEBIAN_FRONTEND=noninteractive

# XRT build dependencies (from xdna-driver's xrtdeps.sh, trimmed for Docker)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    curl \
    file \
    git \
    ca-certificates \
    pkg-config \
    jq \
    wget \
    libboost-dev \
    libboost-filesystem-dev \
    libboost-program-options-dev \
    libcurl4-openssl-dev \
    libdrm-dev \
    libdw-dev \
    libelf-dev \
    libffi-dev \
    libgtest-dev \
    libjson-glib-dev \
    libncurses5-dev \
    libprotoc-dev \
    libssl-dev \
    libsystemd-dev \
    libudev-dev \
    libyaml-dev \
    lsb-release \
    ocl-icd-dev \
    ocl-icd-opencl-dev \
    opencl-headers \
    pciutils \
    protobuf-compiler \
    python3 \
    libpython3-dev \
    python3-pybind11 \
    pybind11-dev \
    rapidjson-dev \
    systemtap-sdt-dev \
    uuid-dev \
    && rm -rf /var/lib/apt/lists/*

# Clone xdna-driver (includes XRT as submodule)
WORKDIR /build
RUN git clone --recurse-submodules https://github.com/amd/xdna-driver.git

# Build XRT base (headers + libs)
# -noctest: skip unit tests, which need real NPU hardware not available during `docker build`
# (hardware is only passed through at `docker run` time via --device)
WORKDIR /build/xdna-driver/xrt/build
RUN ./build.sh -npu -opt -noctest

# Install XRT base .deb
RUN apt-get update && apt install -y ./Release/xrt_*.deb && rm -rf /var/lib/apt/lists/*

# Build XRT NPU plugin
WORKDIR /build/xdna-driver/build
RUN ./build.sh -release -nokmod

# Install XRT plugin .deb
RUN apt install -y ./Release/xrt_plugin*.deb

# ---------------------
# Stage 2: Build FastFlowLM
# ---------------------
FROM ubuntu:24.04 AS flm-builder

ENV DEBIAN_FRONTEND=noninteractive

# FastFlowLM build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    ninja-build \
    git \
    ca-certificates \
    curl \
    pkg-config \
    libboost-program-options-dev \
    libcurl4-openssl-dev \
    libfftw3-dev \
    libavformat-dev \
    libavcodec-dev \
    libavutil-dev \
    libswscale-dev \
    libswresample-dev \
    libreadline-dev \
    uuid-dev \
    libdrm-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Rust (needed for tokenizers-cpp FFI bindings)
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# Copy XRT installation from xrt-builder
COPY --from=xrt-builder /opt/xilinx/xrt /opt/xilinx/xrt

# Clone FastFlowLM
WORKDIR /build
RUN git clone --recurse-submodules https://github.com/FastFlowLM/FastFlowLM.git

# Build (point at XRT from source build)
WORKDIR /build/FastFlowLM/src
RUN cmake --preset linux-default \
      -DXRT_INCLUDE_DIR=/opt/xilinx/xrt/include \
      -DXRT_LIB_DIR=/opt/xilinx/xrt/lib \
    && cmake --build build -j8

# Install to /opt/fastflowlm
RUN cmake --install build

# ---------------------
# Stage 3: Runtime
# ---------------------
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# Runtime dependencies only (no -dev packages)
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    libboost-program-options1.83.0 \
    libcurl4 \
    libfftw3-single3 \
    libfftw3-double3 \
    libfftw3-long3 \
    libavformat60 \
    libavcodec60 \
    libavutil58 \
    libswscale7 \
    libswresample4 \
    libreadline8t64 \
    libdrm2 \
    && rm -rf /var/lib/apt/lists/*

# Copy XRT runtime libraries (from source build)
COPY --from=xrt-builder /opt/xilinx/xrt/lib /opt/xilinx/xrt/lib
COPY --from=xrt-builder /opt/xilinx/xrt/setup.sh /opt/xilinx/xrt/setup.sh

# Add XRT libs to linker path
ENV LD_LIBRARY_PATH="/opt/xilinx/xrt/lib:${LD_LIBRARY_PATH}"

# Copy FastFlowLM installation from builder
COPY --from=flm-builder /opt/fastflowlm /opt/fastflowlm

# Symlink so `flm` is in PATH
RUN ln -sf /opt/fastflowlm/bin/flm /usr/local/bin/flm

# Model cache directory
RUN mkdir -p /root/.config/flm

# FLM needs the NPU xclbin files at a known path
ENV FLM_XCLBIN_PATH=/opt/fastflowlm/share/flm/xclbins

ENTRYPOINT ["flm"]
CMD ["--help"]
```

</details>

บน docker การรัน image นี้ใช้แค่สอง flag คือ `--device=/dev/accel/accel0` กับ
`--ulimit memlock=-1:-1` แต่ฝั่ง Kubernetes **ไม่มีอะไรเทียบตรง ๆ เลยสักอัน** นั่นคือเนื้อหา
ที่เหลือของโพสต์นี้

## ทางตันที่ 2: Kubernetes ไม่มี ulimit ให้ตั้งต่อ pod

FastFlowLM ต้องการ `memlock` แบบ **unlimited** เพราะการโอนข้อมูลเข้าออก NPU ใช้ DMA ซึ่งต้อง
**pin** หน่วยความจำไว้ไม่ให้ kernel ย้ายหรือ swap ออก ค่าเริ่มต้นของ pod บนคลัสเตอร์นี้คือ
**16 MiB** ไม่พอ

ฝั่ง docker จบใน flag เดียว แต่ pod spec ของ Kubernetes **ไม่มีฟิลด์ ulimit เลย** ไม่ใช่ว่าชื่อต่าง
หรืออยู่ที่อื่น (เป็นข้อเสนอที่ค้างอยู่ใน upstream มานาน) ทางที่เหลือคือลงไปแก้ที่ **container runtime**

### ทำไมไม่แก้ค่าเริ่มต้นไปเลย

แก้ config ของ containerd ให้ทุก container ได้ `memlock` unlimited ก็ได้ผล แต่แปลว่า**ทุก pod**
ในคลัสเตอร์ pin memory ได้ไม่จำกัด process ที่มีบั๊กตัวเดียวก็ล็อก RAM ทั้งเครื่องได้

วิธีที่ใช้คือทำ **runtime handler ตัวที่สอง** ชื่อ `runc-highmem` ตั้งใจให้ต่างจากค่าเริ่มต้นแค่ rlimit
ตัวนี้ (ซึ่งทีหลังพบว่าไม่จริง อยู่ท้ายหัวข้อนี้) แล้วผูกกับ **RuntimeClass** เริ่มจากสร้างแม่แบบ
OCI runtime spec จากค่าเริ่มต้นของ `runc`

```bash
mkdir -p runc-highmem
runc spec --rootless=false -b runc-highmem
```

แก้ `process.rlimits` ใน `runc-highmem/config.json` ให้เหลือ `RLIMIT_MEMLOCK` ตัวเดียว ส่วนอื่น
ปล่อยตามที่ `runc spec` สร้างมา (`18446744073709551615` คือ 2^64 − 1 ที่ kernel ตีความว่า
unlimited) แล้ววางไฟล์ไว้ที่ `/var/snap/microk8s/current/args/runc-highmem-spec.json`

```json
    "rlimits": [
      {
        "type": "RLIMIT_MEMLOCK",
        "hard": 18446744073709551615,
        "soft": 18446744073709551615
      }
    ],
```

ลงทะเบียน handler ใน `/var/snap/microk8s/current/args/containerd-template.toml` (MicroK8s
สร้าง `containerd.toml` จาก template นี้) แล้ว restart MicroK8s

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc-highmem]
  runtime_type = "io.containerd.runc.v2"
  base_runtime_spec = "/var/snap/microk8s/current/args/runc-highmem-spec.json"
```

```bash
sudo microk8s stop
sudo microk8s start
```

ฝั่ง Kubernetes ประกาศ RuntimeClass แล้วให้ pod ที่ต้องการขอใช้เอง

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: highmem
handler: runc-highmem
```

```yaml
    spec:
      runtimeClassName: highmem
```

pod อื่นทั้งคลัสเตอร์ยังได้ค่าเริ่มต้นเหมือนเดิม และอ่าน manifest ก็เห็นว่า pod ไหนขอของพิเศษ
ต่างจากการแก้ค่าเริ่มต้น ซึ่งมองไม่เห็นจาก manifest เลย

### บั๊กที่ตามมา: `cannot allocate tty`

ไฟล์จาก `runc spec` ตั้ง `process.terminal` เป็น `true` (เทียบได้กับ `docker run -t` มีไว้สำหรับ
การรันแบบ interactive) ผลคือทุก container ที่ใช้ handler นี้พังด้วย

```
cannot allocate tty
```

ทางแก้คือตั้งเป็น `false` แล้ว restart containerd

```json
"terminal": false
```

```bash
sudo snap restart microk8s.daemon-containerd
```

> ⚠️ **บั๊กนี้พังทุก pod ที่ใช้ handler นั้น ไม่ใช่แค่ตัวที่กำลังแก้** และ error ไม่มีคำว่า NPU,
> memlock หรือ rlimit เลย คนที่กำลังไล่เรื่อง ulimit จะไม่มีทางโยงได้ว่ามาจากไฟล์เดียวกัน
{: .warn}

### สิ่งที่ `runc spec` พามาด้วยโดยไม่ได้ตั้งใจ

ตอนเขียนโพสต์นี้ กลับไปเทียบ pod ที่ใช้ `highmem` กับ pod ปกติ แล้วพบว่าต่างกันมากกว่า memlock

| | pod ปกติ | pod ที่ใช้ `highmem` |
|---|---|---|
| memlock | 16 MiB | unlimited |
| capability | 14 ตัว (ค่าเริ่มต้นของ containerd) | 3 ตัว (ตามไฟล์จาก `runc spec`) |
| root filesystem | เขียนได้ | `Read-only file system` |

เพราะ `base_runtime_spec` **ใช้แทนค่าเริ่มต้นทั้งไฟล์ ไม่ใช่ patch ที่แปะทับ** ค่าเริ่มต้นของ
`runc spec` จึงติดไปกับ pod ด้วยทั้งหมด FastFlowLM ไม่สะดุดเพราะเขียนไฟล์แค่ใน PVC และไม่ต้องใช้
capability เพิ่ม ตอนนี้ยังไม่ได้แก้ ทางที่น่าจะถูกกว่าคือสร้างแม่แบบจาก spec ค่าเริ่มต้นของ containerd
แทนของ `runc` แต่ยังไม่ได้ลอง เช็คของตัวเองได้ด้วยสองคำสั่งนี้

```bash
microk8s kubectl exec deploy/fastflowlm -- grep -E 'CapEff|Max locked memory' /proc/1/status /proc/1/limits
microk8s kubectl exec deploy/fastflowlm -- sh -c 'touch /rw-test && rm /rw-test && echo writable'
```

> ⚠️ **image ที่ต้องเขียนไฟล์นอก volume หรือต้องใช้ capability ที่หายไป อาจพังด้วย error ที่ไม่ได้
> ชี้กลับมาที่ RuntimeClass เลย** สร้าง handler ใหม่ทุกครั้ง ให้เทียบกับ pod ปกติด้วยสองคำสั่งข้างบน
{: .warn}

## ทางตันที่ 3: ไม่มี device plugin ให้ ก็ต้องเขียนเอง

เหมือน[โพสต์ที่แล้ว]({{ '/th/posts/docker-compose-to-microk8s-clean-host/' | relative_url }})
Kubernetes ไม่ให้สิทธิ์ cgroup กับ device ที่ mount เข้ามาเฉย ๆ ต้องใช้ **device plugin** แต่คราวนี้
**AMD ไม่มี device plugin ทางการสำหรับ NPU** (มีแต่ของ GPU) เลยต้องเขียนเอง

### 140 บรรทัด ที่ไม่แตะ NPU เลยสักครั้ง

plugin ตัวนี้ไม่โหลด driver ไม่เปิด device ไม่ส่งงานให้ NPU มันแค่ประกาศกับ kubelet ว่า node นี้มี
device **หนึ่งตัว** และเมื่อมี pod มาขอ ก็ตอบว่าให้ bind-mount `/dev/accel/accel0` พร้อมให้สิทธิ์
cgroup ส่วน kubelet กับ runtime เป็นคนลงมือทำจริง ใช้ device plugin API **v1beta1** เขียนด้วย Go
~140 บรรทัด หัวใจของมันคือฟังก์ชัน `Allocate`

```go
func (p *npuDevicePlugin) Allocate(ctx context.Context, r *pluginapi.AllocateRequest) (*pluginapi.AllocateResponse, error) {
    resp := &pluginapi.AllocateResponse{}
    for range r.ContainerRequests {
        resp.ContainerResponses = append(resp.ContainerResponses, &pluginapi.ContainerAllocateResponse{
            Devices: []*pluginapi.DeviceSpec{
                {
                    ContainerPath: devicePath,
                    HostPath:      devicePath,
                    Permissions:   "rw",
                },
            },
        })
    }
    return resp, nil
}
```

เพราะไม่แตะฮาร์ดแวร์ ทั้งตัว plugin และ pod ของ FastFlowLM จึงรันด้วย `privileged: false` —
ประเด็นเดียวกับโพสต์ที่แล้วที่บอกว่า `privileged: true` คือการปิดกำแพงทั้งหลังเพื่อเปิดประตูบานเดียว

<details markdown="1">
<summary>โค้ด Go ฉบับเต็ม (~140 บรรทัด)</summary>

```go
// npu-device-plugin: a minimal Kubernetes device plugin that advertises
// exactly one host device, /dev/accel/accel0 (the AMD XDNA NPU), as the
// schedulable resource "fastflowlm.local/npu".
//
// This plugin never opens or touches the device itself. All it does is
// tell kubelet "this device path exists" (ListAndWatch) and, when a pod
// requests it, tell kubelet which host device to bind-mount and grant
// cgroup permission for (Allocate). Kubelet does the actual mounting and
// cgroup-rule work for the REQUESTING pod, not for this plugin's own
// container. That's why this plugin itself needs no special privilege,
// no /dev mount, and no "privileged: true".
package main

import (
    "context"
    "fmt"
    "log"
    "net"
    "os"
    "os/signal"
    "syscall"

    "google.golang.org/grpc"
    pluginapi "k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1"
)

const (
    resourceName = "fastflowlm.local/npu"
    devicePath   = "/dev/accel/accel0"
    deviceID     = "accel0"
    socketName   = "fastflowlm-npu.sock"
    socketDir    = "/var/lib/kubelet/device-plugins"
)

type npuDevicePlugin struct {
    pluginapi.UnimplementedDevicePluginServer
    stop chan struct{}
}

func (p *npuDevicePlugin) GetDevicePluginOptions(ctx context.Context, e *pluginapi.Empty) (*pluginapi.DevicePluginOptions, error) {
    return &pluginapi.DevicePluginOptions{}, nil
}

func (p *npuDevicePlugin) ListAndWatch(e *pluginapi.Empty, stream pluginapi.DevicePlugin_ListAndWatchServer) error {
    devices := []*pluginapi.Device{
        {ID: deviceID, Health: pluginapi.Healthy},
    }
    if err := stream.Send(&pluginapi.ListAndWatchResponse{Devices: devices}); err != nil {
        return err
    }
    // Device is always present and healthy for the pod's lifetime; the host
    // device node either exists (checked at startup) or this plugin exits.
    <-p.stop
    return nil
}

func (p *npuDevicePlugin) GetPreferredAllocation(ctx context.Context, r *pluginapi.PreferredAllocationRequest) (*pluginapi.PreferredAllocationResponse, error) {
    return &pluginapi.PreferredAllocationResponse{}, nil
}

func (p *npuDevicePlugin) Allocate(ctx context.Context, r *pluginapi.AllocateRequest) (*pluginapi.AllocateResponse, error) {
    resp := &pluginapi.AllocateResponse{}
    for range r.ContainerRequests {
        resp.ContainerResponses = append(resp.ContainerResponses, &pluginapi.ContainerAllocateResponse{
            Devices: []*pluginapi.DeviceSpec{
                {
                    ContainerPath: devicePath,
                    HostPath:      devicePath,
                    Permissions:   "rw",
                },
            },
        })
    }
    return resp, nil
}

func (p *npuDevicePlugin) PreStartContainer(ctx context.Context, r *pluginapi.PreStartContainerRequest) (*pluginapi.PreStartContainerResponse, error) {
    return &pluginapi.PreStartContainerResponse{}, nil
}

func serve() (*grpc.Server, error) {
    sockPath := socketDir + "/" + socketName
    _ = os.Remove(sockPath)

    lis, err := net.Listen("unix", sockPath)
    if err != nil {
        return nil, fmt.Errorf("listen on %s: %w", sockPath, err)
    }

    server := grpc.NewServer()
    pluginapi.RegisterDevicePluginServer(server, &npuDevicePlugin{stop: make(chan struct{})})

    go func() {
        if err := server.Serve(lis); err != nil {
            log.Fatalf("grpc serve failed: %v", err)
        }
    }()
    return server, nil
}

func registerWithKubelet() error {
    conn, err := grpc.NewClient("unix://"+socketDir+"/kubelet.sock", grpc.WithInsecure())
    if err != nil {
        return fmt.Errorf("dial kubelet: %w", err)
    }
    defer conn.Close()

    client := pluginapi.NewRegistrationClient(conn)
    _, err = client.Register(context.Background(), &pluginapi.RegisterRequest{
        Version:      pluginapi.Version,
        Endpoint:     socketName,
        ResourceName: resourceName,
    })
    if err != nil {
        return fmt.Errorf("register with kubelet: %w", err)
    }
    return nil
}

func main() {
    if _, err := os.Stat(devicePath); err != nil {
        log.Fatalf("device %s not found on this node: %v", devicePath, err)
    }

    server, err := serve()
    if err != nil {
        log.Fatalf("failed to start device plugin grpc server: %v", err)
    }

    if err := registerWithKubelet(); err != nil {
        log.Fatalf("failed to register with kubelet: %v", err)
    }

    log.Printf("npu-device-plugin registered %s -> %s, serving on %s", resourceName, devicePath, socketName)

    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh
    server.GracefulStop()
}
```

</details>

Dockerfile ของ plugin มีแค่ `go build` แล้ว copy ไปวางบน alpine (`CGO_ENABLED=0` ทำให้ได้ binary
แบบ static) ไม่ต้องมี XRT หรือ driver เพราะมันคุยแค่กับ kubelet ผ่าน gRPC

```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /src
COPY main.go .
RUN go mod init npu-device-plugin && \
    go get k8s.io/kubelet@v0.31.3 google.golang.org/grpc@v1.65.0 && \
    go mod tidy && \
    CGO_ENABLED=0 go build -o /npu-device-plugin main.go

FROM alpine:3.20
COPY --from=builder /npu-device-plugin /npu-device-plugin
ENTRYPOINT ["/npu-device-plugin"]
```

plugin รันเป็น DaemonSet ใน `kube-system` ส่วนที่สำคัญคือ mount สองตัวนี้

```yaml
        securityContext:
          privileged: false
        volumeMounts:
        - mountPath: /var/lib/kubelet/device-plugins
          name: device-plugins
        - mountPath: /dev/accel/accel0
          name: accel0
          readOnly: true
      volumes:
      - hostPath:
          path: /var/lib/kubelet/device-plugins
        name: device-plugins
      - hostPath:
          path: /dev/accel/accel0
          type: CharDevice
        name: accel0
```

ตัวแรกคือที่ที่ kubelet เปิด `kubelet.sock` ให้ plugin มาลงทะเบียน ตัวที่สองขัดกับ comment ในโค้ด Go
ที่เขียนว่าไม่ต้อง mount `/dev` — ต้อง mount เพราะ `main()` เรียก `os.Stat` เช็คว่ามี device จริง
ถ้าไม่เห็นไฟล์จะ exit ทันที (comment นั้นผิด แต่ปล่อยไว้ตามไฟล์จริง) `hostPath` ให้ได้แค่ "เห็น"
แต่ "เปิดใช้" ไม่ได้ ซึ่งเป็นอาการของทางตันที่ 1 ใน[โพสต์ที่แล้ว]({{ '/th/posts/docker-compose-to-microk8s-clean-host/' | relative_url }})
คราวนี้กลับเป็นสิ่งที่ต้องการพอดี เพราะ `os.Stat` ต้องการแค่เห็นไฟล์

ตรวจผลที่ node

```bash
microk8s kubectl describe node | grep -A8 "Allocatable"
```

```
Allocatable:
  amd.com/gpu:             1
  fastflowlm.local/npu:    1
```

(ตัดบรรทัดอื่นออก) node ประกาศ GPU กับ NPU เป็น resource คนละตัว pod จะขอตัวไหนก็เขียนใน
`resources.limits` — รูปแบบเดียวกัน ไม่ว่า plugin จะมาจาก AMD หรือเขียนเอง

## ผลลัพธ์

FastFlowLM เปิด API แบบ OpenAI-compatible ยิง request จริงเข้าไปแล้วได้คำตอบกลับมาครบ
ใน Deployment มีสามบรรทัดที่เป็นหัวใจของโพสต์นี้: `runtimeClassName: highmem` (ทางแก้ของ
ทางตันที่ 2), `fastflowlm.local/npu: '1'` (ทางแก้ของทางตันที่ 3) และ `privileged: false` ที่สองอัน
แรกทำให้เป็นไปได้ ส่วน `enableServiceLinks: false` ใส่ไว้ตั้งแต่แรกตามบทเรียนจาก[โพสต์ที่แล้ว]({{ '/th/posts/docker-compose-to-microk8s-clean-host/' | relative_url }})

<details markdown="1">
<summary>Deployment และ Service ฉบับที่ใช้จริง</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastflowlm
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastflowlm
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: fastflowlm
    spec:
      containers:
      - command:
        - flm
        - serve
        - qwen2.5-it:3b
        - --host
        - 0.0.0.0
        image: localhost:32000/fastflowlm:latest
        name: fastflowlm
        ports:
        - containerPort: <port>
          protocol: TCP
        resources:
          limits:
            fastflowlm.local/npu: '1'
            memory: 12Gi
          requests:
            memory: 4Gi
        securityContext:
          privileged: false
        volumeMounts:
        - mountPath: /root/.config/flm
          name: flm-cache
      enableServiceLinks: false
      runtimeClassName: highmem
      securityContext: {}
      volumes:
      - name: flm-cache
        persistentVolumeClaim:
          claimName: fastflowlm-cache-pvc
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastflowlm
  namespace: default
spec:
  ports:
  - port: <port>
    protocol: TCP
    targetPort: <port>
  selector:
    app: fastflowlm
```

</details>

เครื่องนี้ตอนนี้รัน inference engine สองตัวพร้อมกันบนฮาร์ดแวร์คนละชิ้น ต่างจากตอนที่ลองรัน vLLM
สองตัวบน iGPU แล้วชน GTT กันเองในโพสต์แรก แต่ทั้งคู่ยังใช้ RAM ก้อนเดียวกัน ยังไม่ได้วัดว่ารันพร้อมกัน
แล้วแต่ละตัวช้าลงเท่าไร และยังไม่ได้วัดความเร็วของ NPU ด้วยวิธีที่เทียบกับโพสต์ vLLM ได้

## สิ่งที่ได้เรียนรู้

| อาการ | สาเหตุจริง | ทางแก้ |
|---|---|---|
| `docker build` ล้มตอน build XRT | สคริปต์ build รัน CTest ต่อท้าย และมี test ที่ต้องใช้ NPU จริง ซึ่ง `docker build` ไม่มีให้ | ใส่ flag `-noctest` — แยกให้ออกว่า build ผ่านแล้ว ที่ fail คือ test |
| อยากตั้ง ulimit ให้ pod แต่หาฟิลด์ไม่เจอ | Kubernetes ไม่มีฟิลด์นี้จริง ๆ ไม่ใช่หาไม่เจอ | ลงไปทำที่ containerd runtime handler แล้วผูกกับ RuntimeClass |
| แก้ ulimit แล้วทุก container พังด้วย `cannot allocate tty` | `process.terminal` ใน base OCI spec เป็น `true` ซึ่งมีไว้สำหรับ interactive | ตั้ง `"terminal": false` |
| pod ที่ใช้ RuntimeClass ได้ capability เหลือ 3 ตัว และ root filesystem อ่านได้อย่างเดียว | `base_runtime_spec` ใช้แทน spec ค่าเริ่มต้นทั้งไฟล์ และไฟล์จาก `runc spec` มีค่าพวกนี้ติดมา | เทียบ `/proc/1/status` และลอง `touch` กับ pod ปกติทุกครั้งที่สร้าง handler ใหม่ |
| จะเปิด `memlock` ให้ทั้งคลัสเตอร์เลยดีไหม | ง่ายกว่าจริง แต่ให้สิทธิ์ pin memory ไม่จำกัดกับ pod ที่ไม่เกี่ยวข้องด้วย | ใช้ RuntimeClass ให้ opt-in เฉพาะตัวที่ต้องการ |
| NPU ไม่มี device plugin ทางการ | AMD ทำให้เฉพาะ GPU | เขียนเอง ~140 บรรทัด plugin ไม่ต้องแตะ device เลย แค่บอก kubelet ว่าจะ mount อะไร |

**บางครั้งสิ่งที่ขาดไม่ใช่ config ที่ตั้งผิด แต่คือฟีเจอร์ที่ไม่มีอยู่จริง** หา `ulimit` ใน pod spec
หรือ device plugin ของ NPU ไม่เจอ ไม่ใช่เพราะค้นไม่เก่ง มันไม่มีจริง ทางที่เหลือคือลงไปทำที่ชั้นล่างกว่า
แล้ว**ทำให้เป็นทางเลือกที่ pod ต้องขอเอง ไม่ใช่ค่าเริ่มต้นของทั้งระบบ** ซึ่ง RuntimeClass กับ
device plugin ทำหน้าที่นี้เหมือนกัน
