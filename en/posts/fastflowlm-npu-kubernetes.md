---
layout: post
lang: en
slug: fastflowlm-npu-kubernetes
title: "Running FastFlowLM on the NPU under Kubernetes: the ulimit that does not exist, and the device plugin I had to write"
date: 2026-09-11 16:00:00 +0700
description: "Putting the Strix Point NPU (XDNA2) to work inside MicroK8s. Two gaps Kubernetes does not fill at all — a per-pod ulimit and an NPU device plugin — both of which had to be built from scratch."
tags: [kubernetes, microk8s, npu, xdna2, fastflowlm, amd, device-plugin, runtimeclass]
---

## TL;DR

Using an NPU from Kubernetes means building two things Kubernetes does not provide.
**One** — Kubernetes has **no field for a per-pod ulimit** (unlike docker's `--ulimit`), and
FastFlowLM needs `memlock` unlimited. The fix is a second containerd runtime handler bound to
a RuntimeClass, so only pods that ask for it are affected. **Two** — AMD ships no official
device plugin for the NPU, so I wrote one, about 140 lines. The result is a FastFlowLM pod
using the NPU without `privileged: true`.

## Hardware and software

| Item | Value |
|---|---|
| Machine | AMD mini PC (Strix Point) |
| CPU | AMD Ryzen AI 9 HX 370 (12 cores, 1 socket) |
| NPU | AMD XDNA2 — `/dev/accel/accel0` |
| Kernel driver | `amdxdna` |
| iGPU (from before) | AMD Radeon 890M — `gfx1150` |
| OS | Ubuntu Server 26.04.1 LTS |
| Kernel | 7.0.0-31-generic |
| Kubernetes | MicroK8s `v1.35.6` |
| Engine | FastFlowLM, built from source |
| Model | Qwen2.5-3B quantized to 4-bit, `q4nx` format |

This is the same cluster [vLLM was migrated to from docker-compose]({{ '/en/posts/docker-compose-to-microk8s-clean-host/' | relative_url }}),
with the same rules: everything runs as a pod, and nothing gets more privilege than it needs.

## The NPU already worked before any of this

```bash
lsmod | grep amdxdna
lspci | grep -i neural
ls -l /dev/accel/accel0
```

The `amdxdna` kernel driver was already loaded, PCI reports the device as
`Strix/Krackan/Strix Halo Neural Processing Unit`, and the device node is at
`/dev/accel/accel0`. If any of those three is missing, the problem is in the kernel or the
BIOS, not in Kubernetes.

## Changing the plan before starting

The original plan was a wrapper called `lemonade-npu-toolbox`, on the assumption that this
chip might not be supported. Looking properly, it turned out **Strix Point is the primary
platform FastFlowLM benchmarks on**, so I used FastFlowLM directly, without a layer whose
behaviour I would have to guess at.

> ⚠️ **"Nobody supports my hardware" wastes as much time as the opposite assumption.**
> Before deciding, check what the project actually tests on.
{: .warn}

## Dead end 1: the build fails because a test needs real NPU hardware

FastFlowLM has no prebuilt image, so I built one in three stages: XRT (Xilinx Runtime) from
source, then FastFlowLM, then a minimal runtime image with only what is needed to run. The
result is around **150 MB** and takes roughly **15–25 minutes** to build.

Stage 1 broke because XRT's build script runs a **CTest suite** afterwards, and one test
needs a real NPU — which `docker build` does not have; devices only exist at `docker run` with
`--device`. The fix is the `-noctest` flag, and the same stage has a sibling flag for the same
reason, `-nokmod`, which skips building a kernel module. You only find either by reading
`build.sh` yourself.

```
./build.sh -npu -opt -noctest     # XRT base — skip tests that need an NPU
./build.sh -release -nokmod       # XRT NPU plugin — skip building the kernel module
```

> ⚠️ **Learn to tell "the build failed" apart from "a test failed."** Inside a Dockerfile both
> end with a dead layer, but here the build had already succeeded and only the test failed.
> Skim the log, see "fail", go fix dependencies, and you will never get there — what was
> missing is hardware, which `docker build` cannot provide by design.
{: .warn}

<details markdown="1">
<summary>The full Dockerfile I use (3 stages)</summary>

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

On docker, running this image takes two flags, `--device=/dev/accel/accel0` and
`--ulimit memlock=-1:-1`. **Neither has a direct equivalent in Kubernetes**, and that is the
rest of this post.

## Dead end 2: Kubernetes has no per-pod ulimit

FastFlowLM needs `memlock` set to **unlimited**, because moving data in and out of the NPU uses
DMA, which requires **pinning** memory so the kernel cannot move or swap it. The default for a
pod on this cluster is **16 MiB**, nowhere near enough.

On docker this is one flag. A Kubernetes pod spec **has no ulimit field at all** — not renamed,
not moved (it has been an open upstream proposal for a long time). That leaves going one layer
down, to the **container runtime**.

### Why not just change the default

Editing containerd's config so every container gets `memlock` unlimited works, but it gives
**every pod in the cluster** unlimited memory pinning. One buggy process can then pin the
machine's entire RAM.

What I did instead was register a **second runtime handler** called `runc-highmem`, meant to
differ from the default in that one rlimit only (which turned out not to be true — see the end
of this section), and bind it to a **RuntimeClass**. First, an OCI runtime spec template from
`runc`'s defaults:

```bash
mkdir -p runc-highmem
runc spec --rootless=false -b runc-highmem
```

Edit `process.rlimits` in `runc-highmem/config.json` down to a single `RLIMIT_MEMLOCK`, leave
the rest as `runc spec` generated it (`18446744073709551615` is 2^64 − 1, which the kernel
reads as unlimited), and put the file at `/var/snap/microk8s/current/args/runc-highmem-spec.json`:

```json
    "rlimits": [
      {
        "type": "RLIMIT_MEMLOCK",
        "hard": 18446744073709551615,
        "soft": 18446744073709551615
      }
    ],
```

Register the handler in `/var/snap/microk8s/current/args/containerd-template.toml` (MicroK8s
generates `containerd.toml` from this template) and restart MicroK8s:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc-highmem]
  runtime_type = "io.containerd.runc.v2"
  base_runtime_spec = "/var/snap/microk8s/current/args/runc-highmem-spec.json"
```

```bash
sudo microk8s stop
sudo microk8s start
```

On the Kubernetes side, a RuntimeClass points at the handler, and a pod has to ask for it:

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

Every other pod keeps the default, and a manifest shows which pods asked for something special
— unlike changing the default, which no manifest reveals.

### The bug that followed: `cannot allocate tty`

The file from `runc spec` sets `process.terminal` to `true` (the equivalent of `docker run -t`,
meant for interactive use). Every container on that handler then failed with:

```
cannot allocate tty
```

The fix is to set it to `false` and restart containerd:

```json
"terminal": false
```

```bash
sudo snap restart microk8s.daemon-containerd
```

> ⚠️ **This breaks every pod on that handler, not just the one you are working on.** And the
> error mentions no NPU, memlock, or rlimit, so someone debugging a ulimit problem has no reason
> to connect it to the same file.
{: .warn}

### What `runc spec` brought along uninvited

While writing this post I compared a pod on `highmem` against a normal pod, and they differ in
more than memlock:

| | Normal pod | Pod on `highmem` |
|---|---|---|
| memlock | 16 MiB | unlimited |
| Capabilities | 14 (containerd's default) | 3 (from the `runc spec` file) |
| Root filesystem | writable | `Read-only file system` |

`base_runtime_spec` **replaces the defaults wholesale; it is not a patch on top of them**, so
every `runc spec` default goes along with the pod. FastFlowLM never tripped over it, because it
only writes into its PVC and needs no extra capabilities. It is not fixed yet. Building the
template from containerd's default spec instead of `runc`'s is probably the better route, but I
have not tried it. To check your own:

```bash
microk8s kubectl exec deploy/fastflowlm -- grep -E 'CapEff|Max locked memory' /proc/1/status /proc/1/limits
microk8s kubectl exec deploy/fastflowlm -- sh -c 'touch /rw-test && rm /rw-test && echo writable'
```

> ⚠️ **An image that writes outside its volumes or needs a missing capability can fail with an
> error that points nowhere near the RuntimeClass.** Every time you create a handler, compare
> it against a normal pod with the two commands above.
{: .warn}

## Dead end 3: no device plugin exists, so write one

As in the [previous post]({{ '/en/posts/docker-compose-to-microk8s-clean-host/' | relative_url }}),
Kubernetes does not grant cgroup access for a device you merely mount; that takes a
**device plugin**. This time **AMD publishes no official device plugin for the NPU** (only for
the GPU), so I wrote one.

### 140 lines that never touch the NPU

The plugin loads no driver, opens no device, and submits no work. It tells kubelet this node
has **one** device, and when a pod requests it, replies that `/dev/accel/accel0` should be
bind-mounted with cgroup permission. kubelet and the runtime do the actual work. It uses the
device plugin **v1beta1** API in about 140 lines of Go, and the heart of it is `Allocate`:

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

Because it never touches the hardware, both the plugin and the FastFlowLM pod run with
`privileged: false` — the same point as the last post: `privileged: true` tears down the whole
wall to open one door.

<details markdown="1">
<summary>The full Go code (~140 lines)</summary>

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

The plugin's Dockerfile is a `go build` and a copy onto alpine (`CGO_ENABLED=0` gives a static
binary) — no XRT, no driver, because it only talks to kubelet over gRPC:

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

It runs as a DaemonSet in `kube-system`. The part that matters is these two mounts:

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

The first is where kubelet exposes `kubelet.sock` for plugins to register with. The second
contradicts the comment in the Go code, which says no `/dev` mount is needed — it is needed,
because `main()` calls `os.Stat` to confirm the device exists and exits if it cannot see the
file. (The comment is wrong; I have left it as it is in the real file.) A `hostPath` lets you
*see* a device but not *open* it — the exact symptom of dead end 1 in the
[previous post]({{ '/en/posts/docker-compose-to-microk8s-clean-host/' | relative_url }}). Here it
is precisely what is needed, because `os.Stat` only has to see the file.

Checking on the node:

```bash
microk8s kubectl describe node | grep -A8 "Allocatable"
```

```
Allocatable:
  amd.com/gpu:             1
  fastflowlm.local/npu:    1
```

(Other lines trimmed.) The node advertises the GPU and the NPU as separate resources, and a pod
asks for either in `resources.limits` — the same shape whether the plugin came from AMD or
was written by hand.

## The result

FastFlowLM exposes an OpenAI-compatible API; a real request goes in and a complete answer comes
back. Three lines in the Deployment carry this whole post: `runtimeClassName: highmem` (the fix
from dead end 2), `fastflowlm.local/npu: '1'` (the fix from dead end 3), and `privileged: false`,
which the first two make possible. `enableServiceLinks: false` is there from the start, a lesson
from the [previous post]({{ '/en/posts/docker-compose-to-microk8s-clean-host/' | relative_url }}).

<details markdown="1">
<summary>The Deployment and Service I run</summary>

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

This machine now runs two inference engines at once on separate hardware, unlike the first post,
where two vLLM containers on the iGPU collided over the same GTT pool. They still share the same
RAM, though. I have not measured how much each slows down when both are busy, nor measured the
NPU's speed in a way that compares with the vLLM posts.

## What I learned

| Symptom | Actual cause | Fix |
|---|---|---|
| `docker build` dies while building XRT | the build script runs CTest afterwards, and one test needs a real NPU, which `docker build` cannot provide | pass `-noctest` — recognise that the build passed and a test is what failed |
| want to set a ulimit on a pod, cannot find the field | Kubernetes genuinely has no such field; you are not missing it | do it at the containerd runtime handler and bind it to a RuntimeClass |
| after the ulimit change every container fails with `cannot allocate tty` | `process.terminal` in the base OCI spec is `true`, which is for interactive use | set `"terminal": false` |
| pods on the RuntimeClass have only three capabilities and a read-only root filesystem | `base_runtime_spec` replaces the default spec wholesale, and a file from `runc spec` carries those values | compare `/proc/1/status` and try a `touch` against a normal pod every time you create a handler |
| tempting to enable `memlock` cluster-wide | simpler, but hands unlimited memory-pinning to unrelated pods | RuntimeClass, so only pods that opt in are affected |
| no official device plugin for the NPU | AMD only publishes one for the GPU | write one, ~140 lines; it never touches the device, it only tells kubelet what to mount |

**Sometimes what is missing is not a misconfigured setting but a feature that does not exist.**
Failing to find `ulimit` in a pod spec, or an NPU device plugin, is not a search failure —
neither exists. The move is to work one layer down and **make it opt-in rather than the system
default**, which is what RuntimeClass and device plugins both do.
