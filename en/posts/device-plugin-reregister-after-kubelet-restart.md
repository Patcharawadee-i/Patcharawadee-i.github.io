---
layout: post
lang: en
slug: device-plugin-reregister-after-kubelet-restart
series: infra
title: "A device plugin that quietly lost its registration: fixing it in the plugin, and proving it with a real kubelet restart"
date: 2026-10-15 14:30:00 +0000
description: "The last post suspected that a kubelet restart had left the NPU device plugin running but unregistered. This time kubelet was restarted on purpose: what the plugin now does about it, what the log showed, and the eight vulnerabilities govulncheck found while rebuilding."
tags: [kubernetes, microk8s, device-plugin, kubelet, npu, fastflowlm, grpc, govulncheck]
---

## TL;DR

A device plugin that registers with kubelet once and never checks again goes silently
unregistered the next time kubelet restarts, while its container stays `Running`. The fix
is for the plugin to check that its own socket file still exists every 5 seconds and to
register again, in the same process, when it is gone. I restarted kubelet to test it: the
plugin logged the missing socket and registered again within the same second, its restart
counter stayed at 0, and a new model pod got the NPU.

## Hardware and software

| Item | Value |
|---|---|
| Machine | AMD mini PC (Strix Point), the one from the [Building an AI Station]({{ '/en/series/#infra' | relative_url }}) series |
| Kubernetes | MicroK8s `v1.35.6` |
| Device plugin | the one from the [NPU post]({{ '/en/posts/fastflowlm-npu-kubernetes/' | relative_url }}), rebuilt with Go 1.27.2 on Alpine 3.24.2 |
| After the update | grpc v1.83.2, x/net v0.60.0, checked with govulncheck v1.9.0 |

## Where this started

The [previous post]({{ '/en/posts/agent-reported-a-run-that-never-happened/' | relative_url }})
had a model pod that did not come back after an OOM kill. It sat in
`CreateContainerConfigError` with this event:

```
endpoint not found in cache for a registered resource: <npu-resource>
```

The explanation there was that the device plugin registers with kubelet once, so a kubelet
restart could leave it alive but unregistered. It could not prove that a kubelet restart was
what happened, and it could not say how long the plugin had been unregistered. This post
settles neither for that incident, but the plugin's log can now say when it loses the
registration.

## What the kubelet source says

I read the kubelet `v1.35.6` source before changing anything. This is from reading, not from
an experiment:

- a kubelet that has just started deletes every socket file in the device-plugins directory
  (`CleanupPluginDirectory`), and the device count for each resource stays at 0 until a plugin
  registers again
- after 5 minutes (`endpointStopGracePeriod`) kubelet forgets the plugin entirely, and a
  container that starts after that fails with `endpoint not found in cache`, the message above

The Kubernetes device plugin documentation says the same under "Handling kubelet restarts":
the plugin is expected to notice and register again. So any kubelet restart, for whatever
reason, drops the registration of a plugin that does not watch for it, and nobody sees it,
because pods that already hold the device keep working.

## Three ways to fix it

| Approach | Who checks | When the socket is gone | Downside |
|---|---|---|---|
| liveness probe (`test -S <sock>`) | kubelet, about every 10 seconds | kills the container and creates a new one | the restart counter goes up, frequent failures hit crash-loop backoff, and it depends on kubelet, which is what just restarted |
| watch `kubelet.sock` (what I understand the AMD and NVIDIA plugins to do) | the plugin, on a file-change notification | registers again | needs another library (fsnotify) and more special cases |
| check its own socket on a timer (what the Kubernetes documentation suggests, and generic-device-plugin does) | the plugin, every 5 seconds | registers again in the same process | close to none; each round is one `stat` call |

The AMD and NVIDIA row is based on the information I had when writing, not on a line-by-line
read of their code. A probe is an inspector who finds a loose bolt and has the whole machine
pulled out; the timer is a machine that tightens the bolt itself.

I am still writing my own plugin because no ready-made one exists for this NPU. AMD's plugin
covers GPUs only, and squat/generic-device-plugin needs `privileged: true` and all of `/dev`
(checked 2026-10-14). When AMD ships a device plugin for the XDNA NPU, this one can go.

## The change

Changed parts only, from `git diff` (other lines trimmed, and the new `"time"` import is not
shown):

```diff
     socketDir    = "/var/lib/kubelet/device-plugins"
+    sockPath     = socketDir + "/" + socketName
+    checkEvery   = 5 * time.Second // well inside the kubelet's 5-minute grace for a stopped plugin
 )

 type npuDevicePlugin struct {
     pluginapi.UnimplementedDevicePluginServer
-    stop chan struct{}
 }
```

`ListAndWatch` used to wait on a channel that nothing ever closed. It now waits on the
stream's own context, so it ends when kubelet goes away or the server stops:

```diff
-    <-p.stop
+    // Ends when the kubelet goes away or the server stops.
+    <-stream.Context().Done()
```

`main()` now loops: serve, register, then check the socket on every tick, and start over
when it is gone.

<details markdown="1">
<summary>main() after the device check, in full</summary>

```go
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    tick := time.NewTicker(checkEvery)
    defer tick.Stop()

    for {
        server, err := serve()
        if err != nil {
            log.Fatalf("failed to start device plugin grpc server: %v", err)
        }
        // A failed register exits; the DaemonSet restarts us and this registers again.
        if err := registerWithKubelet(); err != nil {
            log.Fatalf("failed to register with kubelet: %v", err)
        }
        log.Printf("npu-device-plugin registered %s -> %s, serving on %s", resourceName, devicePath, socketName)

    wait:
        for {
            select {
            case <-sigCh:
                server.Stop()
                return
            case <-tick.C:
                if _, err := os.Stat(sockPath); err != nil {
                    log.Printf("%s is gone (kubelet restarted?), registering again", sockPath)
                    break wait
                }
            }
        }
        server.Stop()
    }
```

</details>

The old code ended with `server.GracefulStop()`; the new code uses `Stop()`. My reasoning from
the code is that `GracefulStop()` waits for every stream to finish, and the `ListAndWatch`
stream now stays open as long as kubelet is connected. I did not test whether it would hang.

One edge is not covered: if the plugin notices its socket is gone before kubelet has recreated
`kubelet.sock`, registering fails and the plugin exits, and the DaemonSet restarts it. That did
not happen in the test below.

## The rebuild found vulnerabilities

The plugin had to be rebuilt anyway. The old Dockerfile uses `golang:1.23-alpine` and
`alpine:3.20`, both past their support, so I moved to Go 1.27.2 and Alpine 3.24.2 and pinned
them by digest. Then I ran govulncheck v1.9.0 on the old build, which had grpc v1.65.0 and
x/net v0.26.0:

```
Your code is affected by 8 vulnerabilities from 2 modules.
```

The [NPU post]({{ '/en/posts/fastflowlm-npu-kubernetes/' | relative_url }}) pins grpc v1.65.0 in
its Dockerfile, and x/net v0.26.0 came in with it, so anyone who copied that Dockerfile got
the same versions.

<details markdown="1">
<summary>The eight findings (a table I built from the tool's output)</summary>

I did not keep every trace line, and the title of one finding was not saved.

| ID | Module | Fixed in | Title |
|---|---|---|---|
| GO-2026-6617 | x/net v0.26.0 | v0.60.0 | Double flow control refund on HTTP/2 server streams in net/http |
| GO-2026-6612 | x/net v0.26.0 | v0.60.0 | Excessive CPU consumption from repeated initial window changes |
| GO-2026-6611 | x/net v0.26.0 | v0.60.0 | HTTP/2 server memory exhaustion due to Trailer headers |
| GO-2026-6603 | x/net v0.26.0 | v0.60.0 | Server panic via missing authority or Host headers |
| GO-2026-6443 | grpc v1.65.0 | v1.82.2 | Heap Memory Exhaustion (OOM) via HTTP/2 DATA Frame Fragmentation |
| GO-2026-6348 | grpc v1.65.0 | v1.83.1 | (title not kept) |
| GO-2026-6061 | grpc v1.65.0 | v1.82.1 | Vulnerabilities in the xDS RBAC authorization engine and the HTTP/2 transport server |
| GO-2026-4762 | grpc v1.65.0 | v1.79.3 | Authorization bypass in gRPC-Go via missing leading slash in :path |

</details>

I first moved grpc to v1.83.1, and the scan still reported one vulnerability. The end of the
real output:

```
More info: https://pkg.go.dev/vuln/GO-2026-6443
Module: google.golang.org/grpc
  Found in: google.golang.org/grpc@v1.83.1
  Fixed in: google.golang.org/grpc@v1.83.2
  Example traces found:
    #1: main.go:104:25: npu.serve calls grpc.Server.Serve, which eventually calls transport.http2Server.HandleStreams

Your code is affected by 1 vulnerability from 1 module.
```

The first scan had listed v1.82.2 as the fix for GO-2026-6443, yet v1.83.1 still showed it. I
do not know why. After grpc v1.83.2 and x/net v0.60.0:

```
No vulnerabilities found.
```

None of the plugin's source needed to change for this.

> ⚠️ **Scan again after every update, and do not trust the version number alone.** A version
> that the first scan listed as the fix still had a finding on the second scan.
{: .warn}

## Proving it on the real machine

### Step A: deploy the new image

The output of the push, apply and rollout status was not kept. The result: one `registered`
line in the log, and both resources on the node.

```bash
microk8s kubectl get node <node> -o jsonpath='{.status.allocatable}'; echo
```

```
{"amd.com/gpu":"1","cpu":"24", ... "fastflowlm.local/npu":"1", ... "pods":"110"}
```

The real output is a single line. The keys that do not matter here are cut, and the `...`
marks where.

### Step B: restart kubelet

Before restarting I checked that no model swap was running, then noted the time and the
plugin's pod:

```bash
date -u; microk8s kubectl get pods -n kube-system | grep npu-device
```

```
Thu Oct 15 08:04:05 AM UTC 2026
npu-device-plugin-97b2n                    1/1     Running   0              9m4s
```

Then I restarted kubelet, which on MicroK8s runs inside `kubelite`. I did not record the exact
time:

```bash
sudo snap restart microk8s.daemon-kubelite
```

I went off to other work after that, so I do not know how long it was before I looked at the
results. The plugin's log:

```bash
microk8s kubectl logs -n kube-system ds/npu-device-plugin
```

```
2026/10/15 07:55:02 npu-device-plugin registered fastflowlm.local/npu -> /dev/accel/accel0, serving on fastflowlm-npu.sock
2026/10/15 08:06:07 /var/lib/kubelet/device-plugins/fastflowlm-npu.sock is gone (kubelet restarted?), registering again
2026/10/15 08:06:07 npu-device-plugin registered fastflowlm.local/npu -> /dev/accel/accel0, serving on fastflowlm-npu.sock
```

The socket was missing and the plugin registered again in the same second. The check runs
every 5 seconds, so kubelet came back up to 5 seconds before 08:06:07. Later:

```bash
microk8s kubectl get pods -n kube-system | grep -E 'npu-device|amdgpu'
```

```
amdgpu-device-plugin-daemonset-8pcnz       1/1     Running   4 (55m ago)    29d
npu-device-plugin-97b2n                    1/1     Running   0              66m
```

The NPU plugin's restart counter is still `0`, so it repaired itself without a restart, and
the allocatable list again showed `"amd.com/gpu":"1"` and `"fastflowlm.local/npu":"1"`.

The AMD GPU plugin recovered too, but I think by restarting its container. This is an
inference from the ages, not something I watched: the NPU plugin started at 07:55 and was 66
minutes old, so this check ran at about 09:01, and `amdgpu` showed its last restart 55 minutes
earlier, about 08:06.

### Step C: does a new pod get the NPU

The log and the allocatable list say the registration is back, not that a new pod can use it.
A probe on the old pod could not answer that, because it already held the device. The model
Deployment uses `Recreate`, so the old pod stops first:

```bash
microk8s kubectl rollout restart -n <namespace> deploy/fastflowlm
microk8s kubectl rollout status -n <namespace> deploy/fastflowlm --timeout=600s
```

```
deployment "fastflowlm" successfully rolled out
```

Only the last line of the rollout output is shown here. The "Waiting for deployment ..." lines
before it are trimmed. Then the pod:

```bash
microk8s kubectl get pods -n <namespace> | grep fastflowlm
```

```
fastflowlm-575fccb464-snnxd        1/1     Running   0          65s
```

Running, with 0 restarts. Then a request to the model:

```bash
microk8s kubectl run curltest -n <namespace> --rm -i --restart=Never --image=curlimages/curl -- curl -s -X POST http://fastflowlm.<namespace>.svc.cluster.local:<port>/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"gemma4-it:12b","messages":[{"role":"user","content":"hello"}],"max_tokens":20}'
```

```
"content":"Hello! How can I help you today?"
"usage":{"prompt_tokens":10,"completion_tokens":10,"total_tokens":20, ... "decoding_speed_tps":5.422593561429308}
```

The response is trimmed to the fields that matter here. Decoding ran at about 5.4 tokens per
second.

I did not record how long after the kubelet restart I started this step. That limits what it
proves. Within the first 5 minutes kubelet still remembers a plugin that has not registered
again, so a pod started in that window could succeed even without the fix. This step shows that
a new pod gets the NPU once the plugin has registered again. It does not show that the same pod
would have failed with the old plugin.

## What is proven and what is not

- **Proven:** a kubelet restart removes the plugin's socket, and the new plugin registers again
  by itself without a container restart.
- **Proven:** a new pod that asks for the NPU can start after the plugin has registered again,
  and answers requests.
- **Not proven:** that the incident in the previous post was a kubelet restart. The old
  plugin's log was not kept before I changed it, and that version is gone.
- **Not tested:** the restart was done on the new plugin only, so what the old plugin does comes
  from the source reading, not from an observation. Also untested: `GracefulStop()` versus
  `Stop()`, and `kubelet.sock` not being back yet when the plugin tries to register.

Detecting the problem is a separate question, and I built nothing for it. This post only makes
the plugin repair itself and log when it lost the registration. I added a checklist line for
after any kubelet restart (the allocatable list shows the NPU as 1 and the GPU as 1), and a
runbook line: if a model pod is stuck in `CreateContainerConfigError`, read the plugin's log
first.

## What I learned

| Symptom | Actual cause | Fix |
|---|---|---|
| a new pod is stuck with `endpoint not found in cache for a registered resource` | the plugin registered once, and a kubelet restart deleted its socket | the plugin checks its own socket every 5 seconds and registers again |
| the plugin's container is `Running` but the resource is unusable | nothing checks the registration, and the old log had only one line | the log now says when the registration was lost; check allocatable after any kubelet restart |
| old libraries with known vulnerabilities | the plugin was built with grpc v1.65.0 and x/net v0.26.0 | run govulncheck on every rebuild, and scan again after the update |

If you copied the plugin from the [NPU post]({{ '/en/posts/fastflowlm-npu-kubernetes/' | relative_url }}),
it has both problems: it registers once, and it carries the old library versions.
