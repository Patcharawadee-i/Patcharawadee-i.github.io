# Project

A bilingual (Thai/English) GitHub Pages blog documenting technical journeys building
a self-hosted AI station on an AMD mini PC (Strix Point iGPU, Ubuntu Server +
Windows 11 dual boot).

Audience: people hitting the same problems who searched and found nothing useful.
The goal is that a reader can actually fix their problem, not that the author looks
competent.

# Language policy

Every post exists in both Thai and English. Thai is the source of truth — it is
written first, English is a translation of it.

Structure:

```
/th/posts/<slug>.md
/en/posts/<slug>.md
```

The two files share the same slug. Each front matter block declares:

```yaml
lang: th | en
slug: <shared-slug>
title: ...
date: ...
description: ...
tags: [...]
```

Tags are always lowercase English, identical in both language versions, so a
future tag index can group the pair as one topic. They are collected now but
not rendered anywhere yet.

A language toggle in the page header links to the same slug in the other language.
If the counterpart file does not exist yet, the toggle is disabled rather than
linking to a 404.

## Translation rules

- Do not machine-translate literally. The English version should read as if written
  by an engineer in English, not as translated Thai.
- Never translate: command output, error messages, file paths, config file contents,
  package names, flag names. These stay byte-identical across both versions so
  search engines match them.
- Keep code blocks identical in both versions. Only prose and code comments are
  translated.
- Technical terms stay in English inside Thai prose (container, image, kernel,
  throughput) — do not invent Thai equivalents.

# Writing style

- Chronological narrative, including approaches that failed. Not a clean tutorial
  where everything works on the first try.
- Every command shown must be one that was actually run. Never invent commands or
  output.
- Include real error messages verbatim — that exact string is what people paste into
  a search box.
- Include measured before/after numbers wherever a claim is made about performance.
- Avoid a triumphant tone. Dead ends and wrong assumptions are the most valuable
  part of the post.
- Show only what the reader needs to fix the problem. Trim command output to the
  relevant lines and say so ("other lines trimmed"). Show the changed part of a
  config file rather than the whole file when the rest is a tool's default. Leave
  out inventories of what runs on the author's machine.

# Post structure

Each post follows this order:

1. TL;DR — the fix, in three sentences or fewer
2. Hardware/software specs
3. Chronological narrative
4. What I learned / gotchas table

Use warning callouts for mistakes that cause real damage or silent performance loss.

# Hard rules

- Never fabricate commands, output, version numbers, or benchmark results. If
  something is unclear, ask instead of guessing.
- This repo is public. Never commit: real IP addresses, hostnames, usernames,
  personal file paths, recovery keys, or any credential. Use placeholders:
  `<your-ip>`, `<user>`, `<hostname>`.
- Before any commit, grep the diff for these patterns and flag anything that looks
  like a real value.

# Publication safety review

Run this review before every commit and whenever asked to check posts. Review
**every** post in `th/posts/` and `en/posts/`, not only new or changed files —
a new post can expose something when read together with an old one.

1. **No real secret values.** Client secrets, cookie secrets, tokens, passwords,
   private keys, tunnel or ingress credentials. Git keeps them in history even
   after the file is edited, so they must never be committed at all. If one is
   found in history, stop and tell the user; do not try to fix it quietly.
2. **No descriptions of the author's own security posture.** Neither weaknesses
   (a service with no auth, a workload that can read secrets, a disk left
   unencrypted, which ports are listening) nor the protections that exist
   (where auth sits, which layers ask for a password, who may push to a
   registry). The author leaves these out on purpose. Explaining how a public
   tool behaves in general is fine; stating the state of this machine is not.
   Check for inferences a reader could make by combining sentences, code, and
   output across posts.
3. **No internal commands or machine-specific paths.** Host paths under a home
   directory or custom locations, and inventory output (service lists, unit
   lists, installed software) beyond what the reader needs to fix their
   problem. Standard paths (`/dev/...`, `/proc/...`, `/var/lib/kubelet/...`)
   and paths inside a container are fine.

Commit messages are public too: apply the same three rules to them.

End commit messages with the `Co-Authored-By:` trailer only. Do not add a
`Claude-Session:` trailer — it links to the conversation, which holds details
of the author's system that are deliberately kept out of this repo.

Report findings to the user with file and line, ranked by risk, and let them
decide. Edit posts only after they choose, and apply the same change to both
language versions. Never use real details of the author's system as examples
in this file — it is public.

# Build

GitHub Pages with Jekyll. Prefer a custom layout over a prebuilt theme so the
language toggle and code styling can be controlled directly. Keep the CSS in one
file; this is a technical blog, so readable code blocks and a table of contents
matter more than visual polish.
