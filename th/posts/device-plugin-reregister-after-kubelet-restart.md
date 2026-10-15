---
layout: post
lang: th
slug: device-plugin-reregister-after-kubelet-restart
series: infra
title: "device plugin ที่หลุดทะเบียนเงียบ ๆ: แก้ที่ตัว plugin และพิสูจน์ด้วยการ restart kubelet จริง"
date: 2026-10-15 14:30:00 +0000
description: "โพสต์ที่แล้วสงสัยว่า kubelet restart ทำให้ device plugin ของ NPU ยังรันอยู่แต่หลุดทะเบียน รอบนี้ restart kubelet เองเพื่อทดสอบ: plugin ทำอะไรเมื่อทะเบียนหลุด, log เป็นอย่างไร และ govulncheck เจอช่องโหว่ 8 ตัวระหว่าง rebuild"
tags: [kubernetes, microk8s, device-plugin, kubelet, npu, fastflowlm, grpc, govulncheck]
---

## TL;DR

device plugin ที่ลงทะเบียนกับ kubelet ครั้งเดียวแล้วไม่เช็กอีกเลย จะหลุดทะเบียนเงียบ ๆ ทุกครั้งที่
kubelet restart ทั้งที่ container ยังเป็น `Running` ทางแก้คือให้ plugin เช็กทุก 5 วินาทีว่าไฟล์ socket
ของตัวเองยังอยู่ไหม ถ้าหายก็ลงทะเบียนใหม่ใน process เดิม เมื่อ restart kubelet เพื่อทดสอบ plugin บันทึก
ว่า socket หาย แล้วลงทะเบียนกลับในวินาทีเดียวกัน ตัวนับ restart ยังเป็น 0 และ pod โมเดลตัวใหม่ได้ NPU

## สเปกเครื่องและซอฟต์แวร์

| รายการ | ค่า |
|---|---|
| เครื่อง | มินิพีซี AMD (Strix Point) เครื่องเดียวกับในซีรีส์ [Building an AI Station]({{ '/th/series/#infra' | relative_url }}) |
| Kubernetes | MicroK8s `v1.35.6` |
| device plugin | ตัวจาก[โพสต์ NPU]({{ '/th/posts/fastflowlm-npu-kubernetes/' | relative_url }}) rebuild ด้วย Go 1.27.2 บน Alpine 3.24.2 |
| หลังอัปเดต | grpc v1.83.2, x/net v0.60.0 ตรวจด้วย govulncheck v1.9.0 |

## เรื่องเริ่มจากตรงไหน

[โพสต์ก่อนหน้า]({{ '/th/posts/agent-reported-a-run-that-never-happened/' | relative_url }})มี pod
โมเดลที่ไม่กลับมาเองหลังโดน OOM kill มันค้างอยู่ที่ `CreateContainerConfigError` พร้อม event นี้

```
endpoint not found in cache for a registered resource: <npu-resource>
```

คำอธิบายในโพสต์นั้นคือ device plugin ลงทะเบียนกับ kubelet ครั้งเดียว ถ้า kubelet restart plugin จึงอาจ
ยังรันอยู่แต่หลุดทะเบียน โพสต์นั้นพิสูจน์ไม่ได้ว่า kubelet restart เป็นสาเหตุจริง และบอกไม่ได้ว่า plugin
หลุดทะเบียนไปนานเท่าไร โพสต์นี้ก็ยังไม่ตอบสองข้อนั้นสำหรับเหตุการณ์ครั้งนั้น แต่ตอนนี้ log ของ plugin
บอกได้แล้วว่าทะเบียนหลุดเมื่อไร

## อ่าน source ของ kubelet

ก่อนแก้อะไร ได้อ่าน source ของ kubelet `v1.35.6` ส่วนนี้มาจากการอ่าน ไม่ใช่จากการทดลอง

- kubelet ที่เพิ่งเริ่มทำงานจะลบไฟล์ socket ทุกไฟล์ในโฟลเดอร์ device-plugins (`CleanupPluginDirectory`)
  และจำนวน device ของแต่ละ resource จะเป็น 0 จนกว่า plugin จะลงทะเบียนใหม่
- ผ่านไป 5 นาที (`endpointStopGracePeriod`) kubelet จะลืม plugin ไปเลย container ที่เริ่มหลังจากนั้น
  จะเจอ `endpoint not found in cache` ซึ่งเป็นข้อความเดียวกับข้างบน

เอกสาร device plugin ของ Kubernetes เขียนไว้ตรงกันในหัวข้อ "Handling kubelet restarts" ว่า plugin
ต้องรู้ตัวและลงทะเบียนใหม่เอง ดังนั้น kubelet restart ด้วยเหตุใดก็ตาม จะทำให้ plugin ที่ไม่ได้เฝ้าเรื่องนี้
หลุดทะเบียน และไม่มีใครเห็น เพราะ pod ที่ถือ device อยู่แล้วยังทำงานต่อได้

## วิธีแก้ 3 แบบ

| แบบ | ใครเป็นคนเช็ก | เมื่อ socket หาย | ข้อเสีย |
|---|---|---|---|
| liveness probe (`test -S <sock>`) | kubelet ทุก ~10 วินาที | ฆ่า container แล้วสร้างใหม่ | ตัวนับ restart เพิ่ม ถ้าเกิดบ่อยจะโดน crash-loop backoff และต้องพึ่ง kubelet ซึ่งเพิ่ง restart ไป |
| เฝ้า `kubelet.sock` (แบบที่เข้าใจว่า plugin ของ AMD และ NVIDIA ทำ) | plugin รอให้ระบบแจ้งเมื่อไฟล์เปลี่ยน | ลงทะเบียนใหม่ | ต้องเพิ่ม library (fsnotify) และจัดการกรณีพิเศษเพิ่ม |
| เช็ก socket ของตัวเองเป็นรอบ (วิธีที่เอกสาร Kubernetes แนะนำ และ generic-device-plugin ใช้) | plugin ทุก 5 วินาที | ลงทะเบียนใหม่ใน process เดิม | แทบไม่มี แต่ละรอบคือการเรียก `stat` หนึ่งครั้ง |

แถวของ AMD และ NVIDIA อิงข้อมูลที่มีตอนเขียน ไม่ได้อ่านโค้ดของสองตัวนั้นทีละบรรทัด เปรียบเป็นโรงงาน
probe คือผู้ตรวจที่เจอน็อตหลวมแล้วยกเครื่องทั้งเครื่องออก ส่วนแบบ timer คือเครื่องที่รู้ตัวเองแล้วขันน็อตกลับเอง

ที่ยังต้องเขียน plugin เองเพราะยังไม่มี plugin สำเร็จรูปสำหรับ NPU ตัวนี้ plugin ของ AMD รองรับแค่ GPU
ส่วน squat/generic-device-plugin ต้องใช้ `privileged: true` และเข้าถึง `/dev` ทั้งหมด (ตรวจเมื่อ
2026-10-14) เมื่อ AMD ออก device plugin สำหรับ XDNA NPU ก็เลิกใช้ตัวนี้ได้

## โค้ดที่เปลี่ยน

ตัดมาเฉพาะส่วนที่เปลี่ยน จาก `git diff` (บรรทัดอื่นตัดออก และไม่ได้แสดง import `"time"` ที่เพิ่มเข้ามา)

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

`ListAndWatch` เดิมรอ channel ที่ไม่มีใครปิดเลย ตอนนี้เปลี่ยนให้รอ context ของ stream แทน จึงจบเมื่อ
kubelet หายไปหรือ server หยุด

```diff
-    <-p.stop
+    // Ends when the kubelet goes away or the server stops.
+    <-stream.Context().Done()
```

`main()` ตอนนี้เป็น loop: เปิด server, ลงทะเบียน แล้วเช็ก socket ทุกรอบ ถ้าหายก็เริ่มใหม่

<details markdown="1">
<summary>main() หลังเช็ก device ฉบับเต็ม</summary>

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

โค้ดเดิมจบด้วย `server.GracefulStop()` โค้ดใหม่ใช้ `Stop()` เหตุผลที่อนุมานจากโค้ดคือ `GracefulStop()` รอ
ให้ทุก stream จบก่อน แต่ stream ของ `ListAndWatch` ตอนนี้เปิดค้างไว้ตราบที่ kubelet ยังต่ออยู่ ยังไม่ได้
ทดสอบว่าถ้าใช้ `GracefulStop()` จะค้างจริงหรือไม่

ยังมีกรณีหนึ่งที่ไม่ได้ครอบคลุม: ถ้า plugin เห็นว่า socket หายก่อนที่ kubelet จะสร้าง `kubelet.sock`
กลับมา การลงทะเบียนจะล้มเหลว plugin จะปิดตัว แล้ว DaemonSet สั่งสตาร์ตใหม่ กรณีนี้ไม่ได้เกิดขึ้นใน
การทดสอบข้างล่าง

## rebuild แล้วเจอช่องโหว่

plugin ต้อง rebuild อยู่แล้ว Dockerfile เดิมใช้ `golang:1.23-alpine` และ `alpine:3.20` ซึ่งหมดช่วง
support ทั้งคู่ จึงย้ายไป Go 1.27.2 กับ Alpine 3.24.2 และ pin ด้วย digest จากนั้นรัน govulncheck v1.9.0
กับ build เดิมที่ใช้ grpc v1.65.0 และ x/net v0.26.0

```
Your code is affected by 8 vulnerabilities from 2 modules.
```

[โพสต์ NPU]({{ '/th/posts/fastflowlm-npu-kubernetes/' | relative_url }})ใน Dockerfile pin grpc v1.65.0
ไว้ และ x/net v0.26.0 มากับมัน ใครที่ copy Dockerfile นั้นไปจึงได้เวอร์ชันเดียวกัน

<details markdown="1">
<summary>ช่องโหว่ทั้ง 8 ตัว (ตารางที่จัดเองจาก output ของเครื่องมือ)</summary>

ไม่ได้เก็บ trace ครบทุกบรรทัด และชื่อของช่องโหว่หนึ่งตัวไม่ได้ถูกบันทึกไว้

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

ตอนแรกอัปเดต grpc เป็น v1.83.1 แต่สแกนแล้วยังเหลือหนึ่งรายการ ท้าย output จริงเป็นแบบนี้

```
More info: https://pkg.go.dev/vuln/GO-2026-6443
Module: google.golang.org/grpc
  Found in: google.golang.org/grpc@v1.83.1
  Fixed in: google.golang.org/grpc@v1.83.2
  Example traces found:
    #1: main.go:104:25: npu.serve calls grpc.Server.Serve, which eventually calls transport.http2Server.HandleStreams

Your code is affected by 1 vulnerability from 1 module.
```

สแกนครั้งแรกบอกว่า GO-2026-6443 แก้แล้วใน v1.82.2 แต่ v1.83.1 ก็ยังเจอ ยังอธิบายสาเหตุไม่ได้
หลังย้ายเป็น grpc v1.83.2 และ x/net v0.60.0 ผลเป็นดังนี้

```
No vulnerabilities found.
```

ไม่ต้องแก้โค้ดของ plugin เลยสักบรรทัด

> ⚠️ **สแกนซ้ำหลังอัปเดตทุกครั้ง อย่าเชื่อแค่เลขเวอร์ชัน** เวอร์ชันที่สแกนรอบแรกบอกว่าเป็นตัวที่แก้แล้ว
> ยังเจอช่องโหว่ในการสแกนรอบสอง
{: .warn}

## พิสูจน์บนเครื่องจริง

### ขั้น A: deploy image ใหม่

output ของ push, apply และ rollout status ไม่ได้เก็บไว้ เหลือแค่ผล คือใน log มีบรรทัด `registered`
หนึ่งบรรทัด และ node แสดง resource ครบทั้งสองตัว

```bash
microk8s kubectl get node <node> -o jsonpath='{.status.allocatable}'; echo
```

```
{"amd.com/gpu":"1","cpu":"24", ... "fastflowlm.local/npu":"1", ... "pods":"110"}
```

(output จริงอยู่บรรทัดเดียว ตัด key ที่ไม่เกี่ยวออก)

### ขั้น B: restart kubelet

ก่อน restart ตรวจว่าไม่มีงานสลับโมเดลค้างอยู่ แล้วจดเวลาและ pod ของ plugin ไว้

```bash
date -u; microk8s kubectl get pods -n kube-system | grep npu-device
```

```
Thu Oct 15 08:04:05 AM UTC 2026
npu-device-plugin-97b2n                    1/1     Running   0              9m4s
```

จากนั้น restart kubelet ซึ่งบน MicroK8s อยู่ใน `kubelite` ไม่ได้จดเวลาที่สั่งไว้

```bash
sudo snap restart microk8s.daemon-kubelite
```

หลังจากนั้นแวะไปทำงานอื่น จึงไม่รู้ว่ากว่าจะกลับมาดูผลผ่านไปนานเท่าไร log ของ plugin

```bash
microk8s kubectl logs -n kube-system ds/npu-device-plugin
```

```
2026/10/15 07:55:02 npu-device-plugin registered fastflowlm.local/npu -> /dev/accel/accel0, serving on fastflowlm-npu.sock
2026/10/15 08:06:07 /var/lib/kubelet/device-plugins/fastflowlm-npu.sock is gone (kubelet restarted?), registering again
2026/10/15 08:06:07 npu-device-plugin registered fastflowlm.local/npu -> /dev/accel/accel0, serving on fastflowlm-npu.sock
```

socket หาย และ plugin ลงทะเบียนกลับในวินาทีเดียวกัน plugin เช็กทุก 5 วินาที kubelet จึงกลับมาก่อน
08:06:07 ไม่เกิน 5 วินาที ผลที่ดูในภายหลัง

```bash
microk8s kubectl get pods -n kube-system | grep -E 'npu-device|amdgpu'
```

```
amdgpu-device-plugin-daemonset-8pcnz       1/1     Running   4 (55m ago)    29d
npu-device-plugin-97b2n                    1/1     Running   0              66m
```

ตัวนับ restart ของ plugin NPU ยังเป็น `0` แปลว่าซ่อมตัวเองโดยไม่ต้อง restart และ allocatable
แสดง `"amd.com/gpu":"1"` กับ `"fastflowlm.local/npu":"1"` อีกครั้ง

plugin GPU ของ AMD ก็กลับมาเหมือนกัน แต่น่าจะใช้วิธี restart container เป็นข้อสรุปที่อนุมานจากอายุ pod
ไม่ได้เห็นด้วยตา plugin NPU เริ่มตอน 07:55 และอายุ 66 นาที แปลว่าเช็กราว 09:01 ส่วน `amdgpu` แสดงว่า
restart ล่าสุดเมื่อ 55 นาทีก่อน คือราว 08:06

### ขั้น C: pod ตัวใหม่ได้ NPU ไหม

log กับ allocatable บอกว่าทะเบียนกลับมาแล้ว แต่ไม่ได้บอกว่า pod ใหม่ใช้ได้จริง probe บน pod เดิมก็ตอบ
คำถามนี้ไม่ได้ เพราะ pod เดิมถือ device อยู่แล้ว Deployment ของโมเดลใช้แบบ `Recreate` ตัวเก่าจึงหยุดก่อน

```bash
microk8s kubectl rollout restart -n <namespace> deploy/fastflowlm
microk8s kubectl rollout status -n <namespace> deploy/fastflowlm --timeout=600s
```

```
deployment "fastflowlm" successfully rolled out
```

(ตัดบรรทัด "Waiting for deployment ..." ก่อนหน้าออก)

```bash
microk8s kubectl get pods -n <namespace> | grep fastflowlm
```

```
fastflowlm-575fccb464-snnxd        1/1     Running   0          65s
```

pod เป็น `Running` และ restart 0 ครั้ง จากนั้นลองถามโมเดล

```bash
microk8s kubectl run curltest -n <namespace> --rm -i --restart=Never --image=curlimages/curl -- curl -s -X POST http://fastflowlm.<namespace>.svc.cluster.local:<port>/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"gemma4-it:12b","messages":[{"role":"user","content":"hello"}],"max_tokens":20}'
```

```
"content":"Hello! How can I help you today?"
"usage":{"prompt_tokens":10,"completion_tokens":10,"total_tokens":20, ... "decoding_speed_tps":5.422593561429308}
```

(ตัดเหลือเฉพาะ field ที่เกี่ยว) decode ได้ประมาณ 5.4 tokens ต่อวินาที

ขั้นนี้ไม่ได้จดไว้ว่าเริ่มหลัง restart kubelet นานเท่าไร ซึ่งจำกัดสิ่งที่พิสูจน์ได้ ใน 5 นาทีแรก kubelet
ยังจำ plugin ที่ยังไม่ได้ลงทะเบียนใหม่ได้ pod ที่เริ่มในช่วงนั้นจึงอาจสำเร็จได้แม้ไม่มี fix ขั้นนี้
พิสูจน์ได้แค่ว่า pod ใหม่ได้ NPU เมื่อ plugin ลงทะเบียนกลับแล้ว ไม่ได้พิสูจน์ว่า pod ตัวเดียวกันจะล้มเหลว
กับ plugin ตัวเก่า

## อะไรพิสูจน์แล้ว อะไรยังไม่ได้

- **พิสูจน์แล้ว:** kubelet restart ลบ socket ของ plugin และ plugin ตัวใหม่ลงทะเบียนกลับเองโดยไม่ต้อง
  restart container
- **พิสูจน์แล้ว:** pod ใหม่ที่ขอ NPU เริ่มทำงานได้หลังจาก plugin ลงทะเบียนกลับ และตอบคำถามได้
- **ยังพิสูจน์ไม่ได้:** ว่าเหตุการณ์ในโพสต์ก่อนหน้าเกิดจาก kubelet restart จริงหรือไม่ ไม่ได้เก็บ log
  ของ plugin ตัวเก่าไว้ก่อนแก้ และตัวนั้นไม่มีแล้ว
- **ยังไม่ได้ทดสอบ:** restart ทำกับ plugin ตัวใหม่เท่านั้น สิ่งที่ plugin ตัวเก่าจะทำจึงมาจากการอ่าน source
  ไม่ใช่การสังเกต และยังไม่ได้ทดสอบ `GracefulStop()` เทียบกับ `Stop()` หรือกรณีที่ `kubelet.sock`
  ยังไม่กลับมาตอน plugin ลงทะเบียน

การตรวจจับปัญหานี้เป็นอีกเรื่องหนึ่ง และไม่ได้สร้างอะไรขึ้นมา โพสต์นี้ทำแค่ให้ plugin ซ่อมตัวเองและ
บันทึกว่าทะเบียนหลุดเมื่อไร สิ่งที่เพิ่มคือบรรทัดใน checklist หลัง kubelet restart (allocatable ต้องแสดง
NPU 1 และ GPU 1) และบรรทัดใน runbook ว่า ถ้า pod โมเดลติด `CreateContainerConfigError` ให้อ่าน log
ของ plugin ก่อน

## สิ่งที่ได้เรียนรู้

| อาการ | สาเหตุจริง | ทางแก้ |
|---|---|---|
| pod ใหม่ติด `endpoint not found in cache for a registered resource` | plugin ลงทะเบียนครั้งเดียว แล้ว kubelet restart ลบ socket | plugin เช็ก socket ของตัวเองทุก 5 วินาทีแล้วลงทะเบียนใหม่ |
| container ของ plugin ยัง `Running` แต่ใช้ resource ไม่ได้ | ไม่มีอะไรตรวจทะเบียน และ log เดิมมีบรรทัดเดียว | log บอกเวลาที่ทะเบียนหลุดแล้ว และให้ตรวจ allocatable หลัง kubelet restart ทุกครั้ง |
| library เก่ามีช่องโหว่ | plugin build ด้วย grpc v1.65.0 และ x/net v0.26.0 | รัน govulncheck ทุกครั้งที่ rebuild และสแกนซ้ำหลังอัปเดต |

ถ้า copy plugin จาก[โพสต์ NPU]({{ '/th/posts/fastflowlm-npu-kubernetes/' | relative_url }})ไปใช้
จะเจอทั้งสองปัญหา คือลงทะเบียนครั้งเดียว และใช้ library เวอร์ชันเก่า
