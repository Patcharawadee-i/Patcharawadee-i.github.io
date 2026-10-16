---
layout: post
lang: th
slug: package-nobody-asked-for
series: agents
title: "package ที่ไม่มีใครขอ: Windmill แปลง `from google.oauth2 import ...` เป็น package ที่ไม่เกี่ยวกันบน PyPI ใน helper ที่นึกว่าครอบคลุมแล้ว"
date: 2026-10-16 12:00:00 +0000
description: "ทุกสคริปต์ที่ flow รันถูกปักเวอร์ชันแล้ว แต่ helper module ที่ไม่มี #requirements: ก็ยังได้ lockfile ที่มี package 'google' ผิดตัวอยู่ข้างใน เจอโดยบังเอิญได้อย่างไร กฎข้อไหนของ Windmill เป็นต้นเหตุ และทำไมกฎที่เขียนไว้จึงไม่เท่ากับการตรวจ"
tags: [windmill, python, dependency-pinning, supply-chain, lockfile, pip, ai-agents, claude-code]
---

## TL;DR

Windmill สร้าง lockfile ให้ทุกสคริปต์ Python รวมถึง helper module ที่ไม่มี flow ไหนรันตรง ๆ ถ้าสคริปต์ไม่มี
บล็อก `#requirements:` Windmill จะเอาส่วนแรกของแต่ละ import เป็นชื่อ package บน PyPI เว้นแต่ตารางข้อยกเว้น
ของมันจะระบุไว้อย่างอื่น `from google.oauth2 import ...` จึงกลายเป็น `google==3.0.0` ซึ่งเป็นตัวขูดผลค้นหา
Google ที่ไม่เกี่ยวกับ Google และถูกติดตั้งลง package cache ของ worker การปักเวอร์ชันเฉพาะสคริปต์ที่ flow รัน
ไม่พอ ต้องสแกน lockfile ทุกไฟล์ และเปลี่ยนกฎนี้เป็นการตรวจที่รันก่อน commit

## สเปกเครื่องและซอฟต์แวร์

| รายการ | ค่า |
|---|---|
| เครื่อง | มินิพีซี AMD เครื่องเดียวกับในซีรีส์ [Building an AI Station]({{ '/th/series/#infra' | relative_url }}) Kubernetes โหนดเดียว |
| Windmill | 1.817.0 (server และ worker) Python 3.12.13 ใน worker |
| คนเขียนโค้ด | Claude Code ด้วย `claude-opus-5-5` |
| คนรีวิวและตัดสินใจ | ฉัน (ไม่ได้ทำงานเป็น developer) |

## "pin แล้ว" หมายถึงอะไร

ใน Windmill สคริปต์ระบุ package ไว้ในคอมเมนต์ `#requirements:` ที่หัวไฟล์ สำหรับ import ที่ *ไม่ได้* ระบุไว้
ตรงนั้น Windmill จะหาชื่อ package เอง
[เอกสาร](https://www.windmill.dev/docs/advanced/dependencies_in_python)ระบุว่า "We use a
simple heuristics to infer the package name: the import root name must be the package name"
และ "We also maintain a list of exceptions."

ก่อนถึงเรื่องนี้ ปักเวอร์ชันมาแล้วสองรอบ

1. ทุกสคริปต์ที่ flow รันโดยตรงได้บล็อก `#requirements:` ที่ระบุเวอร์ชันแบบตรงตัว (`==`) ฉันรีวิวแล้ว
2. base ของทุก container image ถูกปักด้วย digest และมีสคริปต์ตรวจก่อนทุก commit (package จาก `apt-get`
   ปล่อยให้เป็นไปตาม distro โดยตั้งใจ)

ฉันกับ Claude Code เชื่อว่า "package ถูก pin แล้ว" ซึ่งจริงเฉพาะที่ที่เคยมองเท่านั้น

## helper module ตัวหนึ่ง

Claude Code เขียน helper module สำหรับ Google Drive ชื่อ `gdrive.py` เขียนครั้งแรกวันที่ 2026-10-07 เวลา
19:12 UTC มันเป็น helper คือสคริปต์อื่น import ไปใช้ ไม่มี flow ไหนรันมันตรง ๆ จึงไม่มีบล็อก
`#requirements:` เพราะการปักเวอร์ชันรอบแรกครอบคลุมเฉพาะสคริปต์ที่ flow รัน import สองบรรทัดที่เกี่ยวข้องคือ

```
from google.oauth2 import service_account
from googleapiclient.discovery import build
```

Windmill สร้าง lockfile ให้ทุกสคริปต์ รวมถึง helper เพื่อดูว่ามันทำอะไรกับสองบรรทัดนี้ จึงอ่าน source ของ
Windmill 1.817.0 ใน `lib.rs` import ถูกตัดที่จุดแรก และตารางใน `mapping.rs` ถูกเช็กก่อน

<details markdown="1">
<summary>Windmill 1.817.0: แปลง import เป็นชื่อ package อย่างไร</summary>

```rust
let imprt = module.split('.').next().unwrap_or("").replace("_", "-");
...
let pkg = replace_full_import(&module).unwrap_or(replace_import(imprt));
```

</details>

- `googleapiclient` อยู่ในตารางข้อยกเว้น แปลงเป็น `google-api-python-client` ถูกต้อง
- `google.oauth2` ไม่อยู่ในตาราง และ `google` เดี่ยว ๆ ก็ไม่อยู่ (grep ทั้งไฟล์แล้ว) ชื่อจึงตกลงมาเป็น
  `google` ซึ่งเป็นอีกโปรเจกต์หนึ่งบน PyPI

ดังนั้นนี่ไม่ใช่การเดาสุ่ม เป็นกฎที่เขียนไว้ในเอกสาร และ import ของเราไม่เข้ากับกฎนั้น lockfile ได้
`google==3.0.0` ซึ่งเป็นตัวขูดผลค้นหา Google ของผู้เขียนอิสระ (metadata เขียนว่า "Python bindings to the
Google search engine") ไม่ใช่ client library ของ Google มันมาพร้อม `beautifulsoup4` และ `soupsieve`
บรรทัดจาก lockfile ตามที่อยู่ใน git (เฉพาะบรรทัดที่ตรง)

```
2:beautifulsoup4==4.15.0
7:google==3.0.0
23:soupsieve==2.10
```

package ถูกติดตั้งลง cache ของ worker ตัวหนึ่งวันที่ 2026-10-08 เวลา 13:00 UTC ไม่รู้ว่า deploy ครั้งไหนเป็น
ตัวกระตุ้น เอกสารบอกว่า lockfile ถูกคำนวณใหม่ตอน deploy แต่ไม่มี log ที่ผูกเวลานั้นกับ push ใด worker อีกตัว
ไม่มี package นี้

> **คำเตือน:** library ที่ import path ขึ้นต้นด้วยคำทั่วไป (`google`, `azure`, ...) อาจตกไปเป็น project
> อื่นที่ไม่เกี่ยวกันบน PyPI เมื่อเครื่องมือแปลงส่วนแรกของ import เป็นชื่อ package หน้าตาเหมือนการโจมตีแบบ
> dependency confusion ทุกอย่าง เพียงแต่รอบนี้ไม่ต้องมีผู้โจมตี

## เสียหายแค่ไหน

เท่าที่ยืนยันได้ เรียงตามความหนักของหลักฐาน

- สคริปต์ที่รันจริงมี lockfile ที่ปักไว้ของตัวเอง pipeline จึงไม่ได้รับผลกระทบ และไม่มีโค้ดใน repo ที่ import
  module ของตัวขูดนั้น (`googlesearch`)
- package ถูกติดตั้งจาก wheel (`INSTALLER` เป็น `uv` ไฟล์ `WHEEL` มี `Tag: py3-none-any`) สรุปว่าการติดตั้ง
  ไม่ได้รันโค้ดใด ๆ แต่ข้อสรุปนี้มาจากวิธีที่ wheel ทำงาน ไม่ใช่จาก log การติดตั้ง เพราะไม่มี log
- ส่วนว่ามัน *ถูกรัน* หรือไม่ พิสูจน์ไม่ได้ ค้น log เก่าของ job ไม่ได้ ไฟล์ใน cache ไม่มีร่องรอยการอ่านหลัง
  ติดตั้ง ผลด้านล่างรันในตัว worker

<details markdown="1">
<summary>ผล stat ของไฟล์ใน cache</summary>

```
$ stat -c "atime=%x mtime=%y %n" googlesearch/__init__.py googlesearch/__pycache__/__init__.cpython-312.pyc
atime=2026-10-08 13:00:42.657979851 +0000 mtime=2026-10-08 13:00:42.638066615 +0000 googlesearch/__init__.py
atime=2026-10-08 13:00:42.659979862 +0000 mtime=2026-10-08 13:00:42.661215281 +0000 googlesearch/__pycache__/__init__.cpython-312.pyc
```

</details>

นี่เป็นหลักฐานหนักว่าไม่เคยถูก import แต่หนักเพราะ `/tmp` mount แบบ `relatime` เท่านั้น ภายใต้ตัวเลือกนี้
การอ่านหลังติดตั้งควรทำให้ access time ขยับ ไม่ใช่ข้อพิสูจน์

## เจอโดยบังเอิญ

แปดวันต่อมา ระหว่างแก้เรื่องอื่น Claude Code แก้ **คอมเมนต์** ในโมดูลที่ใช้ร่วมกัน ทุกสคริปต์ที่ import โมดูล
นั้นถูกสร้าง lockfile ใหม่ และ lockfile หนึ่งขยับจาก `pythainlp` 5.3.7 เป็น 5.4.0

ฉันถามว่า *"ฉันงง ว่าเราปัก หรือ pin version ไป 2 รอบแล้ว ตัวนี้หลุดมาได้ไง"*

คำตอบแรกของ Claude Code คือ *"ไม่ได้หลุดครับ"* และบอกว่าสิ่งที่รันจริงถูก pin แล้ว ซึ่งจริงสำหรับสิ่งที่
รัน แต่ตอบไม่ตรงประเด็น คือปกป้องสภาพที่เป็นอยู่ แทนที่จะถามว่าปัญหาประเภทนี้กว้างแค่ไหน ฉันไม่ยอมจบ
ขอให้ทำให้ทุก lock ตรงกัน และพูดเพิ่มว่า *"ขนาดฉันเปิดเช็คดูแล้วนะ คิดว่าเคลียร์ยังหลุดอ่ะ"*

หลังจากนั้น Claude Code จึงสแกน lockfile ทุกไฟล์หา package ที่ปรากฏมากกว่าหนึ่งเวอร์ชัน ในโฟลเดอร์ที่มี
สคริปต์และไฟล์ `*.script.lock`

<details markdown="1">
<summary>คำสั่งสแกนและ output ทั้งหมด</summary>

```
perl -ne 'chomp; next if /^#/ || !/==/; ($p,$v)=split /==/; ($f=$ARGV)=~s/\.script\.lock//; push @{$h{lc $p}{$v}}, $f; END { for $p (sort keys %h) { next if keys %{$h{$p}} < 2; print "$p:\n"; for $v (sort keys %{$h{$p}}) { print "  $v: @{$h{$p}{$v}}\n" } } }' *.script.lock
```

```
google-api-core:
  2.41.0: <script> gdrive
  2.42.0: <script> x6
google-auth:
  2.60.0: <script> x7
  2.61.0: gdrive
psycopg:
  3.3.5: <script> x9
  3.3.6: <script>
psycopg-binary:
  3.3.5: <script> x9
  3.3.6: <script>
pycparser:
  3.1: <script> x3, gdrive
  3.11: <script> x4
pythainlp:
  5.3.7: <script> x3
  5.4.0: <script> x2
```

</details>

ชื่อสคริปต์อื่นนอกจาก `gdrive` ถูกแทนด้วย `<script>` และ `<script> x6` หมายถึงสคริปต์หกตัว การสแกนรอบนี้เกิดหลังจากสคริปต์ที่ไม่มี pin สองตัวได้บล็อก
`#requirements:` ไปแล้ว `pythainlp` 5.4.0 จึงยังโผล่ใน lockfile ของสองตัวนั้น สิ่งที่สแกนเจอ

- helper ที่ไม่มี pin เลย (`gdrive` กับอีกสองตัว)
- สคริปต์ health check ที่รันหลังทุก deploy และไม่มี pin เลย มัน resolve ได้ `psycopg` 3.3.6 ขณะที่สคริปต์
  ที่ใช้ `psycopg` และถูก pin ไว้อีกเก้าตัวได้ 3.3.5
- package ย่อยที่ไม่มีใคร pin (`google-api-core`, `pycparser`) อยู่คนละ release ในแต่ละ lockfile
- ใน lockfile ของ Drive helper มี `google==3.0.0` ที่หลุดเข้ามา

จากนั้นดูใน cache ของ worker แล้วเจอ package นั้นอยู่จริง

<details markdown="1">
<summary>ดูใน cache ของ worker</summary>

```
$ kubectl -n <ns> exec deploy/<worker> -- sh -c 'ls -d /tmp/windmill/cache/python_3_12/google==*; ...'
/tmp/windmill/cache/python_3_12/google==3.0.0
/tmp/windmill/cache/python_3_12/google==3.0.0.lock
```

</details>

(คำสั่งจริงรัน `stat` ต่อจากนี้ด้วย ตัด output ส่วนนั้นออก เวลาติดตั้งที่แสดงคือ 2026-10-08 13:00:42 UTC)

## วิธีแก้

ใส่ `#requirements:` ที่หัว helper รวมถึง package ย่อยที่ไม่มีใครเคยระบุ ไม่ได้ย้ายอะไรและไม่ได้เปลี่ยน
import

```diff
+#requirements:
+#google-api-core==2.42.0
+#google-api-python-client==2.201.0
+#google-auth==2.60.0
+#pycparser==3.11
+
 # layer: service
```

สคริปต์อื่นที่ใช้ library ของ Google และมีบล็อก `#requirements:` อยู่แล้ว ได้ package ย่อยสองตัวเดียวกัน
เพิ่มเข้าไป หลังสร้าง lockfile ใหม่ lockfile ของ Drive helper ไม่มี `google`, `beautifulsoup4` และ
`soupsieve` อีก และการสแกนเดิมพิมพ์ `packages with two versions: 0`

`pythainlp` ถูก pin กลับเป็น 5.3.7 ซึ่งเป็นเวอร์ชันที่ใช้จริง ในทุก lockfile ส่วนว่าทำไม Windmill
resolve ได้ 5.4.0 ให้ตัวที่ไม่มี pin ไม่รู้

## ทำไมถึงหลุด

1. **"pin แล้ว" หมายถึง "pin ในที่ที่เคยมอง"** กฎถูกใช้ด้วยมือกับไฟล์ที่ดูสำคัญ แต่เครื่องมือสร้าง lockfile
   ให้ *ทุก* ไฟล์ ไฟล์ที่ไม่ได้ป้องกันจึงยังถูก resolve และติดตั้ง การรีวิวของฉันจับไม่ได้ เพราะไฟล์ที่
   รีวิวไม่มีปัญหา
2. **กฎการแปลง import เป็นชื่อ package เป็นกฎของ Windmill และ import ของเราไม่เข้ากับกฎนั้น**
   ถ้าอ่านกฎของ Windmill ก่อนเขียน helper จะเห็นว่า `google.oauth2` ไม่มีรายการ
3. **package ที่ระบุชื่อถูก pin แต่ package ที่มันพึ่งพาไม่ถูก** package ย่อยลอยไปตาม release ล่าสุดในวันที่
   สร้าง lockfile ใหม่ สอง lockfile ที่สร้างคนละวันจึงไม่ตรงกัน
4. **workflow engine ไม่ใช่เส้นแบ่งความปลอดภัยของ package** Windmill ติดตั้งตามที่ lockfile บอก ด้วยสิทธิ์
   เดียวกับโค้ดของเราเอง
5. **กฎที่เขียนไว้ จะทำตามหรือไม่ทำตามก็ได้ ควรทำเป็น script จริง** โปรเจกต์นี้มีกฎอยู่แล้วว่า
   "checklist ที่สำคัญต้องเป็นสคริปต์ที่รันด้วย git hook" แต่เรื่องการ pin ไม่ได้ทำตามกฎข้อนี้ จึงไม่มี
   script ตรวจ และไม่มีอะไรหยุดเราตอนที่ลืมบางไฟล์ กฎที่อยู่แค่ในเอกสารมีโอกาสไม่ถูกทำตาม ถ้าเรื่องไหนสำคัญ
   ควรทำเป็น script ที่รันจริงทุกครั้ง
6. **model ตัวเดียวกันเขียนโค้ด รีวิว และปกป้องมัน** Claude Code เขียน `gdrive.py` ด้วย `claude-opus-5-5`
   `/code-review` ที่รันก่อนเจอเรื่องนี้ใช้ model เดียวกัน และดูแค่ diff ของ branch เทียบกับ `main`
   `gdrive.py` กับ lockfile ไม่อยู่ใน diff นั้น ข้อค้นพบ 8 ข้อจึงไม่มีข้อไหนแตะเรื่อง package คำตอบ
   "ไม่ได้หลุดครับ" ก็มาจาก model เดิมอีกครั้ง ใน session ที่ตั้ง effort เป็น medium (ช่วงวางแผนเป็น max)
   ส่วน session ที่เขียน helper ใช้ effort ระดับไหน ไม่รู้
7. **เรื่อง package ใช้ไปสี่ task** สองรอบแรกคือการ pin ที่เล่าไว้ข้างบน
   รอบสามคือเรื่องในโพสต์นี้ และรอบสี่คือ task ที่กำลังทำอยู่เพื่อปิดช่องเหล่านี้ทั้งหมด แต่ละรอบแก้เฉพาะ
   ที่ที่มองอยู่ตอนนั้น ฉันสรุปว่า *"เรื่อง package เนี่ย เท่ากับแก้ไป 4 task เต็มๆ อ่ะ ทั้งๆที่ควรจะจบแค่ 1-3 tasks"*

## ตัวเลข

| | ก่อน | หลัง |
|---|---|---|
| package ที่มีสองเวอร์ชันข้าม lockfile | 6 (`pythainlp`, `psycopg`, `psycopg-binary`, `google-auth`, `google-api-core`, `pycparser`) | 0 |
| package แปลกปลอมใน lockfile | 3 (`google`, `beautifulsoup4`, `soupsieve`) | 0 |

จากติดตั้งถึงเจอ 8 วัน สคริปต์ 11 ตัวได้ pin, 3 flow ถูกเพิ่มเลขเวอร์ชัน และ lockfile เปลี่ยนจริง 7 ไฟล์
(อีก 4 ไฟล์ตรงอยู่แล้ว) เทสต์ 533 ตัวที่ผ่านเป็นชุดเทสต์ทั้ง repo ไม่มีเทสต์ไหนตรวจการ pin

## setup ที่ดีกว่าหน้าตาเป็นอย่างไร

image อีกสองตัวในระบบเดียวกันทำถูกอยู่แล้ว

```
RUN pip install --no-cache-dir --require-hashes --no-deps -r /opt/<mcp-server>/requirements.txt
```

```
RUN npm ci --ignore-scripts --omit=optional --no-audit --no-fund && npm cache clean --force
```

ตัวแรกติดตั้งจากรายการเต็มชุดเดียว รวม package ย่อย แต่ละตัวมีเวอร์ชันและ hash แล้ว pip ปฏิเสธทุกอย่าง
นอกรายการ ตัวที่สองติดตั้งตรงตามที่ `package-lock.json` ระบุ พร้อม hash แบบ sha512

ไม่รู้ว่า Windmill ทำแบบนี้ได้หรือไม่ ยังไม่ได้ลอง และ lockfile ที่มันสร้างไม่มี hash เลย (`grep` หา `sha256`
และ `--hash` เจอ 0 ไฟล์) สิ่งที่วางแผนไว้แทนคือการตรวจก่อน commit ซึ่งยังไม่ได้เขียน เก็บไฟล์เดียวที่รวม
บรรทัด `package==version` ที่อนุญาต เทียบทุกบรรทัดของ lockfile ทุกไฟล์ของ Windmill กับไฟล์นั้น แล้วให้
commit ไม่ผ่านเมื่อมีอะไรขาดหรือต่างไป ซึ่งจะหยุด `google==3.0.0`, pin ที่ลืม และ package ย่อยที่โผล่เข้ามา
เงียบ ๆ ได้

## ที่ยังค้างอยู่ ณ วันที่เขียน

- ลบ package แปลกปลอมออกจาก cache ของ worker วางแผนไว้หลัง deploy ครั้งถัดไป
- เขียนการตรวจก่อน commit และบังคับใช้จริง และย้ายอีกสอง image ไปติดตั้งแบบล็อกด้วย hash

## สิ่งที่ได้เรียนรู้

| ความผิดพลาด | ควรทำแทน |
|---|---|
| pin เฉพาะสคริปต์ที่ flow รัน | ไล่ทุกสคริปต์ รวม helper และตรวจ lockfile ทุกไฟล์ |
| pin เฉพาะ package ที่ระบุชื่อ | pin หรือใส่ hash ให้ package ย่อยด้วย ไม่งั้นมันลอยไปตามวันที่สร้างใหม่ |
| เชื่อการรีวิวรายการของตัวเอง | สแกน lockfile ทุกไฟล์หา package ที่มีมากกว่าหนึ่งเวอร์ชัน |
| รีวิวแค่ diff | ถามว่าเครื่องมือสร้างอะไรที่ไม่อยู่ใน diff (ในที่นี้คือ lockfile ของทุกไฟล์) |
| ปกป้องคำว่า "สิ่งที่รันถูก pin แล้ว" | ถามก่อนว่าปัญหาประเภทนี้กว้างแค่ไหน แล้วค่อยอธิบาย |
| กฎที่เขียนไว้ในเอกสาร | สคริปต์ที่รันด้วย git hook |
