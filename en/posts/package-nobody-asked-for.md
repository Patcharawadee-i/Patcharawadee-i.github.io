---
layout: post
lang: en
slug: package-nobody-asked-for
series: agents
title: "The package nobody asked for: Windmill turned `from google.oauth2 import ...` into an unrelated PyPI package, in a helper I thought was covered"
date: 2026-10-16 12:00:00 +0000
description: "Every script that a flow runs had pinned versions, and a helper module with no #requirements: block still got a lockfile with the wrong 'google' package in it. How it was found by accident, which Windmill rule caused it, and why a written rule was not a check."
tags: [windmill, python, dependency-pinning, supply-chain, lockfile, pip, ai-agents, claude-code]
---

## TL;DR

Windmill builds a lockfile for every Python script, including helper modules that no flow runs.
If the script has no `#requirements:` block, Windmill takes the first part of each import as the
PyPI package name unless its exception table says otherwise: `from google.oauth2 import ...`
became `google==3.0.0`, an unrelated Google-search scraper, and it was installed into a
worker's package cache. Pinning only the scripts that flows run is not enough; scan every
lockfile, and turn the rule into a check that runs before commit.

## Hardware and software

| Item | Value |
|---|---|
| Machine | AMD mini PC, the one from the [Building an AI Station]({{ '/en/series/#infra' | relative_url }}) series, Kubernetes on one node |
| Windmill | 1.817.0 (server and workers), Python 3.12.13 in the worker |
| Who writes the code | Claude Code with `claude-opus-5-5` |
| Who reviews and decides | me (I do not work as a developer) |

## What "pinned" meant

In Windmill a script lists its packages in a `#requirements:` comment at the top. For an import
that is *not* listed there, Windmill works out the package itself. The
[documentation](https://www.windmill.dev/docs/advanced/dependencies_in_python) says: "We use a
simple heuristics to infer the package name: the import root name must be the package name",
and "We also maintain a list of exceptions."

Two rounds of pinning had been done before this story starts:

1. Every script that a flow runs directly got a `#requirements:` block with exact versions
   (`==`). I reviewed it.
2. Every container image's base was pinned by digest, and a script checks that before every
   commit. (`apt-get` packages were left to the distro on purpose.)

Both of us believed "packages are pinned". That was true only where we had looked.

## A helper module

Claude Code wrote a helper module for Google Drive, `gdrive.py`. The first write was
2026-10-07 at 19:12 UTC. It is a helper: other scripts import it, and no flow runs it directly.
So it got no `#requirements:` block, because the first round of pinning covered only the
scripts that flows run. Two of its imports matter:

```
from google.oauth2 import service_account
from googleapiclient.discovery import build
```

Windmill builds a lockfile for every script, helpers included. I read the Windmill 1.817.0
source to see what it does with these two lines. In `lib.rs` the import is cut at the first
dot, and a table in `mapping.rs` is checked first:

<details markdown="1">
<summary>Windmill 1.817.0: how an import becomes a package name</summary>

```rust
let imprt = module.split('.').next().unwrap_or("").replace("_", "-");
...
let pkg = replace_full_import(&module).unwrap_or(replace_import(imprt));
```

</details>

- `googleapiclient` is in the exception table, mapped to `google-api-python-client`. Correct.
- `google.oauth2` is not in the table, and neither is plain `google` (checked by grepping the
  whole file). So the name falls through to `google`, which is a different project on PyPI.

So this was not a random guess. It is a documented rule that our import did not fit. The
lockfile got `google==3.0.0`, a Google-search scraper by an independent author (its metadata
says "Python bindings to the Google search engine"), not Google's client library. It came with
`beautifulsoup4` and `soupsieve`. From the lockfile as it was in git (matching lines only):

```
2:beautifulsoup4==4.15.0
7:google==3.0.0
23:soupsieve==2.10
```

The package was installed into one worker's cache on 2026-10-08 at 13:00 UTC. I do not know
which deploy triggered it: the documentation says the lockfile is recomputed on deploy, but no
log ties that time to a specific push. A second worker did not have it.

> **Warning:** a library whose import path starts with a generic word (`google`, `azure`, ...)
> can fall through to an unrelated PyPI project when the tool maps the first part of the import
> to a package name. This is the same shape as a dependency-confusion attack, except that no
> attacker was needed.

## How bad was it

What I could establish, in order of how strong the evidence is:

- the scripts that actually run had their own pinned lockfiles, so the pipeline was not
  affected, and nothing in the repo imports the scraper's module (`googlesearch`)
- the package was installed from a wheel (`INSTALLER` is `uv`, the `WHEEL` file has
  `Tag: py3-none-any`). I conclude that installing it ran no code, but that comes from how wheels
  work, not from an install log, because there is none
- whether it was ever *run* I cannot prove. I could not search old job logs. The files in the
  cache show no read after the install. Output below, from inside the worker:

<details markdown="1">
<summary>stat output for the cached files</summary>

```
$ stat -c "atime=%x mtime=%y %n" googlesearch/__init__.py googlesearch/__pycache__/__init__.cpython-312.pyc
atime=2026-10-08 13:00:42.657979851 +0000 mtime=2026-10-08 13:00:42.638066615 +0000 googlesearch/__init__.py
atime=2026-10-08 13:00:42.659979862 +0000 mtime=2026-10-08 13:00:42.661215281 +0000 googlesearch/__pycache__/__init__.cpython-312.pyc
```

</details>

That is strong evidence it was never imported, but only because `/tmp` is mounted with
`relatime`: under that option, a read after the install should have moved the access time. It
is not proof.

## Found by accident

Eight days later, while fixing something unrelated, Claude Code edited **a comment** in a shared
module. Every script that imports that module had its lockfile rebuilt, and one lockfile moved
from `pythainlp` 5.3.7 to 5.4.0.

I asked, in Thai: *"ฉันงง ว่าเราปัก หรือ pin version ไป 2 รอบแล้ว ตัวนี้หลุดมาได้ไง"*
("We pinned versions twice already. How did this one get through?")

Claude Code's first answer, in Thai, was *"ไม่ได้หลุดครับ"* ("It didn't slip"), and that what
runs is pinned. That was true for what runs, and it missed the point: it defended the existing
state instead of asking how large the class of problem was. I pushed anyway, asking to
make every lock agree, and added: *"ขนาดฉันเปิดเช็คดูแล้วนะ คิดว่าเคลียร์ยังหลุดอ่ะ"* ("I even
opened it and checked. I thought it was clear, and it still slipped.")

Only then did Claude Code scan every lockfile for packages that appear with more than one
version. In the folder with the scripts and their `*.script.lock` files:

<details markdown="1">
<summary>The scan command and its full output</summary>

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

Script names other than `gdrive` are replaced with `<script>`, and `<script> x6` means six such scripts. This run happened after two of
the unpinned files had already been given `#requirements:` blocks, so `pythainlp` 5.4.0 still
shows up in their lockfiles. The scan found:

- helper modules with no pins at all (`gdrive` and two others)
- a health-check script that runs after every deploy with no pins at all; it had `psycopg` 3.3.6
  while the nine scripts that use `psycopg` and are pinned had 3.3.5
- sub-packages pinned nowhere (`google-api-core`, `pycparser`), at different releases in
  different lockfiles
- inside the Drive helper's lockfile, the stray `google==3.0.0`

Then it looked inside the worker's cache. The package was there:

<details markdown="1">
<summary>Looking inside the worker's cache</summary>

```
$ kubectl -n <ns> exec deploy/<worker> -- sh -c 'ls -d /tmp/windmill/cache/python_3_12/google==*; ...'
/tmp/windmill/cache/python_3_12/google==3.0.0
/tmp/windmill/cache/python_3_12/google==3.0.0.lock
```

</details>

(the real command ran `stat` after this and that part of the output is trimmed; the install time it showed was 2026-10-08 13:00:42 UTC)

## The fix

`#requirements:` at the top of the helper, including the sub-packages that nobody had named.
Nothing was moved and no import was changed:

```diff
+#requirements:
+#google-api-core==2.42.0
+#google-api-python-client==2.201.0
+#google-auth==2.60.0
+#pycparser==3.11
+
 # layer: service
```

The other scripts that use Google libraries and already had a `#requirements:` block got the same
two sub-packages added. After the lockfiles were rebuilt, the Drive helper's lockfile no longer
had `google`, `beautifulsoup4` or `soupsieve`, and the same scan printed
`packages with two versions: 0`.

`pythainlp` was pinned back to 5.3.7, the version actually in use, in every lockfile. Why Windmill
had resolved 5.4.0 for the unpinned ones, I do not know.

## Why it slipped

1. **"Pinned" meant "pinned where we looked".** The rule was applied by hand to the files that
   seemed to matter. The tool builds lockfiles for *every* file, so the unguarded ones were
   still resolved and installed. My review could not have caught it: the files being
   reviewed were fine.
2. **The import-to-package rule is Windmill's, and our import did not fit it.** Reading
   Windmill's rule before writing the helper would have shown that `google.oauth2` has no entry.
3. **Named packages were pinned; their dependencies were not.** Sub-packages float to whatever
   is newest on the day a lockfile is rebuilt, so two lockfiles rebuilt on different days
   disagree.
4. **A workflow engine is not a boundary for packages.** Windmill installs what the lockfile
   says, with the same rights as your own code.
5. **A written rule can be followed or skipped; a script runs every time.** The project already
   had a rule: "a checklist that matters must be a script run by a git hook". The pinning checklist
   did not follow that rule, so no script checked it, and nothing stopped us when a file was missed.
   A rule that lives only in a document may or may not be followed. If it matters, make it a real
   script.
6. **The same model wrote the code, reviewed it and defended it.** Claude Code wrote
   `gdrive.py` with `claude-opus-5-5`. The `/code-review` run before this was found used the
   same model, and it only looked at the branch diff against `main`; `gdrive.py` and the
   lockfiles were not in it, so none of its 8 findings touched packages. The answer "ไม่ได้หลุด
   ครับ" came from the same model again, in a session set to medium effort (planning had been
   at max). I do not know the effort level of the session that wrote the helper.
7. **This one subject took four tasks.** The first two were the pinning rounds described above, the
   third is the incident in this post, and the fourth is the task under way now, closing all of these
   gaps. Each round fixed only the place we were looking at.
   In my words, *"เรื่อง package เนี่ย เท่ากับแก้ไป 4 task เต็มๆ อ่ะ ทั้งๆที่ควรจะจบแค่ 1-3 tasks"*
   ("This package thing took four full tasks when it should have ended in one to three.")

## Numbers

| | Before | After |
|---|---|---|
| Packages with two versions across lockfiles | 6 (`pythainlp`, `psycopg`, `psycopg-binary`, `google-auth`, `google-api-core`, `pycparser`) | 0 |
| Stray packages in lockfiles | 3 (`google`, `beautifulsoup4`, `soupsieve`) | 0 |

From install to discovery: 8 days. 11 scripts were given pins, 3 flows had their version
number bumped, and 7 lockfiles actually changed (the other 4 already matched). The 533 tests
that pass are the whole repo's suite; none of them checks pinning.

## What a better setup looks like

Two other images in the same station already did this properly:

```
RUN pip install --no-cache-dir --require-hashes --no-deps -r /opt/<mcp-server>/requirements.txt
```

```
RUN npm ci --ignore-scripts --omit=optional --no-audit --no-fund && npm cache clean --force
```

The first installs from one full list, sub-packages included, each with a version and a hash,
and pip refuses anything else. The second installs exactly what `package-lock.json` says,
with its sha512 hashes.

I do not know whether Windmill can be made to do the same: I have not tried, and none of the
lockfiles it produced contains a hash (`grep` for `sha256` and `--hash` finds 0 files). What I
plan instead is a pre-commit check, not written yet: keep one file of approved
`package==version` lines, compare every line of every Windmill lockfile with it, and fail the
commit on anything missing or different. That would have stopped `google==3.0.0`, a forgotten
pin and a sub-package that arrives unannounced.

## Still open at the time of writing

- removing the stray package from the worker's cache, planned after the next deploy
- writing the pre-commit check and making it enforced, and moving two more images to
  hash-locked installs

## What I learned

| Mistake | What to do instead |
|---|---|
| Pinning only the scripts a flow runs | List every script, helpers included, and check each lockfile |
| Pinning the packages you named | Pin or hash the sub-packages too, or they float on rebuild day |
| Trusting a review of your own list | Scan every lockfile for packages with more than one version |
| Reviewing only the diff | Ask what the tool builds that is not in the diff (here, a lockfile for every file) |
| Defending "what runs is pinned" | Ask how big the class of problem is before explaining |
| A rule in a document | A script run by a git hook |
