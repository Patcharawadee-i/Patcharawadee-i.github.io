---
layout: post
lang: en
slug: docker-compose-to-microk8s-clean-host
title: "Moving vLLM from docker-compose to MicroK8s: where the GPU went, and why you must not name the Service vllm"
date: 2026-09-11 10:00:00 +0700
description: "Two things bite you moving vLLM from docker-compose to MicroK8s: a hostPath that mounts /dev/kfd without granting cgroup access, and a Service name that collides with one of vLLM's own internal environment variables."
tags: [kubernetes, microk8s, vllm, rocm, docker, amd-igpu, device-plugin]
---

## TL;DR

Moving vLLM from docker-compose to MicroK8s surfaces two problems compose never had.
**One** — a `hostPath` pointing at `/dev/kfd` and `/dev/dri` does not grant cgroup device
access the way docker's `--device` does. Use the AMD device plugin and request the GPU
through `resources.limits: amd.com/gpu: 1` instead. **Two** — name the Service `vllm` and
Kubernetes injects a `VLLM_PORT` variable into every pod, which collides with a variable
of the same name that vLLM uses internally. Fix it with `enableServiceLinks: false`. After
both fixes I measured 12.72–12.79 tok/s against the 12.76 tok/s compose baseline — the
same number.

## Hardware and software

| Item | Value |
|---|---|
| Machine | AMD mini PC (Strix Point) |
| CPU | AMD Ryzen AI 9 HX 370 (12 cores, 1 socket) |
| iGPU | AMD Radeon 890M — `gfx1150` |
| RAM | 64 GB DDR5-5600 SO-DIMM (dual channel) — 60 GiB visible to the OS |
| VRAM carved out by BIOS | 512 MB |
| GTT | 44 GiB |
| OS | Ubuntu Server 26.04.1 LTS |
| Kernel | 7.0.0-31-generic |
| Kubernetes | MicroK8s `v1.35.6` (snap, track `1.35/stable`) |
| Image | `rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1` |
| Test model | `Qwen/Qwen2.5-3B-Instruct` (BF16, not quantized) |

The starting point is the `docker-compose.yml` from the [previous post]({{ '/en/posts/vllm-rocm-gfx1150-wrong-version/' | relative_url }}),
running at **12.76 tok/s**. The goal: translate it into Kubernetes without that number dropping.

## Why move at all, when compose already worked

Not for speed. It is a rule I set for this machine: **everything that runs, runs in a box**, and
the host carries as little as possible. The rule came straight out of the last post: chasing the
10x slowdown, a host with its own ROCm, a virtualenv, and whatever got `pip install`ed would have
turned "which kernel did it just call?" into detective work.

compose already gives you that. What it does not give you is what comes after: once several
things are running, managing their dependencies, getting them back after a power cut, and
exposing them with auth in front turns into hand-written scripts — just another kind of stuff
living on the host.

## Translating compose into a Deployment, line by line

A **Deployment**, because vLLM holds no state tied to a pod identity (the model sits in a PVC),
and `replicas: 1`, because there is one iGPU and one GTT pool.

`strategy: Recreate` has to be set deliberately. The default, RollingUpdate, **starts the new pod
before killing the old one**. With exactly `1` of `amd.com/gpu` on the node, the new pod sits in
`Pending` because the old one still holds the GPU, and the old one is never killed because the new
one never becomes Ready. The update just hangs. `Recreate` kills the old pod first, at the cost
of downtime while the model loads — not a problem on a personal machine.

| docker-compose | Kubernetes |
|---|---|
| `image:` | `containers[].image` — same image, same tag |
| `container_name: vllm` | `metadata.name` plus the label `app: vllm` |
| `command:` | `containers[].command` — word for word |
| `restart: unless-stopped` | `restartPolicy: Always` (the Deployment already handles this) |
| `ipc: host` | `hostIPC: true` |
| `shm_size: "8g"` | an `emptyDir` with `medium: Memory`, `sizeLimit: 8Gi`, mounted at `/dev/shm` |
| `security_opt: seccomp=unconfined` | `securityContext.seccompProfile.type: Unconfined` |
| `group_add: ["44", "991"]` | `securityContext.supplementalGroups: [44, 991]` |
| `devices: /dev/kfd, /dev/dri` | **does not translate** — see the next section |
| `ports: 127.0.0.1:8000:8000` | a ClusterIP Service, `8000` → `8000` |
| `volumes: hf-cache` | a PVC named `hf-cache-pvc` |

`group_add` → `supplementalGroups` carries over directly, and so does the lesson: numeric GIDs
only (`44` video, `991` render, from `getent group render video`). Kubernetes only accepts
integers there anyway, so the compose mistake is not available. The row that does not translate
at all is `devices:`.

## Dead end 1: every file present, every permission correct, still denied

My first instinct was to turn `devices:` into a `hostPath`:

```yaml
      volumes:
      - name: kfd
        hostPath:
          path: /dev/kfd
      - name: dri
        hostPath:
          path: /dev/dri
```

The pod came up fine. Exec into it and every device file is there, permissions right, group
matching `supplementalGroups`. Then vLLM made a real GPU call and got `Operation not permitted`.
I chased the wrong GID, then seccomp, then `allowPrivilegeEscalation` — none of it.

The real cause is that docker's `--device` does **two** things: it bind-mounts the device file
**and** adds a rule to the container's **cgroup device allow-list**. A `hostPath` only does the
first, and the cgroup device controller blocks at the driver call regardless of what `ls -l` shows.

> ⚠️ **A `hostPath` pointing at a device file makes everything look correct while being
> non-functional.** The cgroup refusal does not show up in `ls`, does not show up in `id`, and
> produces no event in `kubectl describe pod`. Debug this by checking permissions and you will
> never find it.
{: .warn}

The fix is AMD's **device plugin**, which runs as a DaemonSet and advertises the GPU to the
node. When a pod requests one, kubelet calls the plugin's `Allocate()`, and the runtime adds the
missing cgroup rule. The pod mounts nothing itself; it just asks for the resource (`amd.com/gpu`
is the name the plugin advertises):

```yaml
        resources:
          limits:
            amd.com/gpu: "1"
            memory: 32Gi
          requests:
            memory: 3Gi
```

The plugin runs as a DaemonSet in `kube-system`:

```bash
microk8s kubectl get ds -A
```

```
NAMESPACE     NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR              AGE
kube-system   amdgpu-device-plugin-daemonset   1         1         1       1            1           kubernetes.io/arch=amd64   24h
```

(Other lines trimmed.) The one you need is `amdgpu-device-plugin-daemonset`, running the
`rocm/k8s-device-plugin` image. To check it is working, look at the **node**, not the pod: if the
resource is missing, any pod requesting `amd.com/gpu` sits in `Pending` forever without saying why.

```bash
microk8s kubectl describe node | grep -A8 "Allocatable"
```

```
Allocatable:
  amd.com/gpu:             1
```

(Other lines trimmed.) `amd.com/gpu: 1` is the line you want. With `hostPath` it does not exist,
because `hostPath` tells Kubernetes nothing at all.

The `32Gi` memory limit and `3Gi` request are new; compose never required them. The request is
low because the model weights live in GTT, not in the process's RSS.

## Dead end 2: naming the Service `vllm` broke vLLM instantly

I put a Service named `vllm` in front of it, and the pod died at startup:

```
ValueError: VLLM_PORT 'tcp://...' appears to be a URI
```

Nothing in the Deployment sets `VLLM_PORT`. The cause is **service links**, an old Kubernetes
feature that injects variables for every Service in the namespace into every pod —
`<SERVICE NAME>_PORT`, `<SERVICE NAME>_SERVICE_HOST`, and so on (it predates cluster DNS and is
still on by default). A Service named `vllm` becomes `VLLM_PORT`, which collides with vLLM's own
internal variable for IPC between engine processes. The injected value really is your Service's
address, so it looks plausible and sends you off editing vLLM's `--port`, which is unrelated.

The fix is one line on the pod spec, and the Service keeps its name, which is still the right one
for cluster DNS (`vllm.default.svc.cluster.local`):

```yaml
    spec:
      enableServiceLinks: false
```

> ⚠️ **This is not limited to vLLM's own pod.** Service links inject into **every pod in the
> namespace**, so any pod created later in `default` also gets `VLLM_PORT`. If something else
> ever reads a variable by that name, it breaks the same way, with nothing pointing back to a
> Service called `vllm`.
{: .warn}

## Measuring: the bar is "no slower"

The bar was "not slower", because what changed is the orchestration layer, not the compute
layer. Same script as the previous post (`ignore_eos: true` still forces the full `max_tokens`);
the only difference is a way into the Service first. Leave `port-forward` running in one
terminal and run `curl` in another:

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

Three requests: **12.72–12.79 tok/s**, against **12.76 tok/s** on compose — the same number within
measurement noise (the first request after a fresh pod comes in lower; that is warm-up). It
confirms Kubernetes is not in the compute path; the compute is still ROCm talking to the iGPU.

## The Deployment and Service I actually run

Things worth pointing at in the file:

- **No `HSA_OVERRIDE_GFX_VERSION`**, and it must not come back, for the same reason as the last
  post (~10x slower with no error of any kind).
- **`privileged: false`** — `privileged: true` does fix GPU problems, because a privileged
  container bypasses the cgroup device controller entirely. But that tears down the whole wall to
  open one door; the device plugin gets the same result without escalating anything.
- **`imagePullPolicy: Never`** — the image is already loaded into MicroK8s's containerd, which is
  separate from docker's store. An image `docker images` lists is not one Kubernetes can see.
- **`--gpu-memory-utilization` is still `0.75`**, the same as compose. Every value is kept
  identical, or the throughput number would be measuring two changes at once.

<details markdown="1">
<summary>The full Deployment and Service</summary>

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
      # This line is the fix for dead end 2. Remove it and the pod dies at
      # startup, because the Service named vllm injects VLLM_PORT over the
      # variable vLLM uses internally.
      enableServiceLinks: false
      hostIPC: true
      securityContext:
        # Numeric GIDs only. These are this machine's; check your own with:
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
            # Advertised by the AMD device plugin, not a name I chose, and this
            # is what replaces compose's devices: /dev/kfd, /dev/dri
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

## The result: every workload is a pod, but Docker cannot go

After the migration every workload is a pod. The intent was for the host to carry nothing but
Ubuntu Server and MicroK8s, but the services enabled on the host included these two:

```bash
systemctl list-unit-files --type=service --state=enabled --no-pager
```

```
containerd.service    enabled
docker.service        enabled
```

(Other lines trimmed.) **Docker is still running.** I assumed it was left over from the compose
era, but it is still load-bearing, for two reasons I had not thought about when planning:

- **`imagePullPolicy: Never` means something has to put the image there.** The vLLM image got into
  MicroK8s's containerd via `docker save` and an import, so the original lives only in Docker's
  image store. If containerd ever evicts it and the pod is rescheduled, Docker is the only thing
  that can put it back.
- **Docker is the build tool.** Images built on this machine (like the NPU ones in the [next post]({{ '/en/posts/fastflowlm-npu-kubernetes/' | relative_url }}))
  go through `docker build` and `docker push` into MicroK8s's internal registry.

> ⚠️ **"Everything runs in a box" says nothing about where the boxes come from.** Kubernetes runs
> containers; it does not *build* them, and it cannot get a local image into its containerd on
> its own. Uninstall Docker after the migration and pods using `imagePullPolicy: Never` can never
> be recovered.
{: .warn}

So "everything runs in a box" is **true for workloads, not for tooling**. What the move actually
bought is not speed but that **you can power-cycle the machine and do nothing**: MicroK8s starts
at boot and every Deployment brings its pod back. And **removing something is as easy as
installing it** — `kubectl delete` leaves no config in `/etc` and no orphaned systemd unit.

## What I learned

| Symptom | Actual cause | Fix |
|---|---|---|
| `Operation not permitted` on GPU calls even though the device files are present with correct permissions | `hostPath` only bind-mounts; it adds no cgroup rule, unlike docker's `--device` which does both | use the AMD device plugin and request `resources.limits: amd.com/gpu: 1` |
| `privileged: true` makes the GPU problem go away | privileged bypasses the cgroup device controller entirely | it works, but it over-escalates — use the device plugin instead |
| `ValueError: VLLM_PORT ... appears to be a URI` when you never set `VLLM_PORT` | a Service named `vllm` makes Kubernetes inject `VLLM_PORT` into every pod, colliding with vLLM's internal variable | `enableServiceLinks: false` on the pod spec |
| (avoided, not hit) RollingUpdate starts the new pod before killing the old one | there is a single GTT pool, so the new pod could never get it | set `strategy: Recreate` from the start |
| an image that `docker images` lists is invisible to Kubernetes | MicroK8s runs its own containerd, separate from docker's store | load the image into MicroK8s's containerd and use `imagePullPolicy: Never` |
| pod sits in `Pending` forever with no error explaining why | the node is not advertising `amd.com/gpu` because the device plugin is not working | look at the node, not the pod — `microk8s kubectl describe node` and find `Allocatable` |
| migrated to Kubernetes but Docker still cannot be removed | Kubernetes cannot build images, and cannot load a local image into its containerd by itself; a pod using `imagePullPolicy: Never` depends on Docker completely | accept Docker as the cluster's build/load tool, or push to a real registry and stop using `Never` |
| performance is identical after all that work | Kubernetes only manages the outer layer; it is not in the compute path | that is the correct result — a changed number would be the thing to worry about |

This is the mirror image of the last post. That one was *a workaround that makes it run does not
mean it is correct*. This one is **a config that looks correct does not mean it works**. The
`hostPath` passed every check you can do by eye while missing the part `ls` cannot show — and
catching that means knowing what your old tool (docker's `--device`) was actually doing for you,
not just that it worked.
