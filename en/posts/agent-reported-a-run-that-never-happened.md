---
layout: post
lang: en
slug: agent-reported-a-run-that-never-happened
series: agents
title: "The agent reported a run that never happened: a small local model, hidden MCP tools, and an OOM at 13 GiB"
date: 2026-10-03 19:14:36 +0000
description: "Connecting the agent to the MCP server took under a second. Then the small model wrote a finished-run report for a job that did not exist. What it took to get reports I could check: a larger model, more memory, and tool search turned off."
tags: [ai-agents, hermes-agent, mcp, local-llm, npu, fastflowlm, kubernetes]
---

## TL;DR

Connecting an agent to an MCP server is the easy part; whether you can trust what it
reports depends on the model. A small local model wrote a report for a run that never
happened, while a 12B model invented nothing but needed its memory limit raised from 13 GiB
to 14 GiB, because Hermes Agent refuses to start below a 64K context. If Hermes says an MCP
tool does not exist right after a skill told the model to use it, turn tool search off and
name the tools by their full registered names.

## Hardware and software

| Item | Value |
|---|---|
| Machine | AMD mini PC (Strix Point), the one from the [Building an AI Station]({{ '/en/series/#infra' | relative_url }}) series |
| Agent | Hermes Agent v0.21.3 (Docker image tag `v2026.9.14`), running in Kubernetes |
| Model server | FastFlowLM v1.0.5, on the NPU |
| Small model | `gemma4-it:e4b` |
| Larger model | `gemma4-it:12b`, context 65,536 |
| Workflow engine | Windmill CE 1.817.0 |
| MCP server | `<mcp-server>`, three tools, from the [previous post]({{ '/en/posts/agent-skill-narrow-mcp-door/' | relative_url }}) |

Internal names are placeholders throughout.

## Where the last post ended

The [previous post]({{ '/en/posts/agent-skill-narrow-mcp-door/' | relative_url }}) built
one narrow door: an internal MCP server with three tools, `read`, `book` and `status`.
Hermes connected to it over Streamable HTTP in 801 ms and found 3 tools.

That was the connection. This post is about the problems that came after it worked.

## The small model reported a run that never happened

The first test used the small default model, `gemma4-it:e4b`. I put a number of test files
in the folder and asked it to read the documents there.

It answered with a report saying the work was done.

What gave it away was the file count. The report claimed far more files than I had put in
the folder, which is impossible.

Checking further, there was nothing behind that report at all. The workflow engine had no
such job, and no data had been produced. The model had not called the tool. It had written
the tool's answer itself, inside its own reply, and then said "done" with a straight face
over an empty result.

I can see why it happens. The instruction and the content are in Thai, and that is a known
weak spot for small models. It matches what I saw testing Gemma 3 around July–August 2025:
the 4B model could work with Thai, but not as well as the 12B, and the 12B took roughly
twice as long as the 4B at the time. That is the trade-off, and for Thai it comes out in
favour of the 12B. One practical upside is that Gemma 3 12B runs on a Google Colab T4, so you can
test this for yourself without owning the hardware.

> ⚠️ **A report the model made up looks exactly like a real one.** This one was caught only
> because the file count was higher than what existed. Had the invented number been
> plausible, nothing would have stood out. Check the job id against the workflow engine
> every time.
{: .warn}

## The 12B model reported truthfully, but its pod was killed for lack of memory

The same test on `gemma4-it:12b` gave a different result. When it was unsure, it stopped
and asked, and it invented nothing in any of the test runs.

Then the model pod was killed: `OOMKilled`, exit code 137, at its memory limit of 13Gi.

Memory had been sampled every 20 seconds before the kill:

| Situation | Container memory |
|---|---|
| one-line prompt | 12,001 Mi |
| with the skill loaded | 11,993 → 12,301 → 12,965 → 12,916 Mi |

The limit is 13,312 Mi, and every sample is under it. The spike that killed the pod fell
between two samples. A 20-second sampler shows that the pod is running close to the limit;
it does not show the peak.

This memory spike is a real problem, and it is not new with this model. It was already
there with Gemma 3: when I tested Gemma 3 on a Google Colab T4, I sat and watched it run,
and memory spiked the same way. A T4 is very quick to run out of memory, so it showed up
there clearly. On this machine the pod was killed by it once.

### Lowering the context was not an option

The obvious way to save memory is a smaller context window. Hermes does not allow it. This
never triggered here; I found it by reading the source inside the v0.21.3 image (I did not
compare it against the public repository):

- the constant, in `agent/model_metadata.py`: `MINIMUM_CONTEXT_LENGTH = 64_000`
- the check, in `agent/agent_init.py`: `_enforce_minimum_context()`, which raises `ValueError`
- the only exception: provider `lmstudio` with an explicit positive `model.context_length`

The message it raises (trimmed at the end):

```
Model {model} has a context window of {ctx:,} tokens, which is below the minimum 64,000 required by Hermes Agent.  Choose a model with at least 64K context.  If your server reports a window smaller than the model's true window, set model.context_length in config.yaml to the real value ...
```

So the context stays at 65,536, and the only thing left to change is the limit:

```
-            memory: 13Gi
+            memory: 14Gi
```

After that, 45 samples at 20-second intervals: steady at 11,990 Mi (11.7 GiB), peak
12,468 Mi, 0 restarts. The next day, 60 samples over 20 minutes stayed between 11,190 and
12,096 Mi.

### The model pod did not come back by itself

After the OOM kill the model pod should have restarted. It did not. Its status was
`CreateContainerConfigError`, with an event along these lines (from my notes, not a raw
paste):

```
endpoint not found in cache for a registered resource: <npu-resource>
```

The pod asks for the NPU through a device plugin — the hand-written one from the
[NPU post]({{ '/en/posts/fastflowlm-npu-kubernetes/' | relative_url }}). kubelet no longer
had a working endpoint for that resource, so it could not hand the device to the new
container. Restarting the plugin fixed it.

Normally a broken DaemonSet pod restarts by itself. This one could not, and the reason is in
the plugin code that post published. It registers with kubelet **once**, at startup, and
then waits. It never checks whether the registration is still there.

That matters because kubelet clears the plugin sockets in its device-plugins directory when
it restarts, and the Kubernetes device plugin documentation expects a plugin to notice a
kubelet restart and register again. A plugin that does not do this ends up in a state where:

- its process is alive and its container is `Running`, so the DaemonSet has no reason to
  restart it
- kubelet no longer knows about its resource
- pods that already hold the device keep working, so nothing looks wrong
- the first pod that has to **start** fails with `CreateContainerConfigError`

I cannot prove that a kubelet restart is what happened here, only that the code allows it.
How long the plugin had been unregistered is unknown, and its log could not have told me:
it writes one line at startup and nothing afterwards.

The OOM did not cause any of this. It only made it visible, because it was the first time
that pod had to start again.

> ⚠️ **If you copied the device plugin from the NPU post, it has this flaw.** It works until
> kubelet restarts, then silently stops being registered while still showing as `Running`.
> The real fix is to register again on a kubelet restart: have the plugin watch the
> device-plugins directory and re-register when `kubelet.sock` is recreated, which is what
> upstream plugins do. A liveness probe that fails when the plugin's own socket file is gone
> is a workaround that forces the same result, by getting the container restarted so that
> it registers again on startup.
{: .warn}

## Hermes said the tool does not exist

With the skill loaded, the next run failed in a new way. The model called one of the MCP
tools, by the name the skill gave it, and Hermes answered with an error saying the tool
does not exist.

My first reading was that the model was lying again. It was not. Hermes itself returned
that error, and the model reported it honestly.

The cause is a Hermes feature called tool search. When it is on, MCP and plugin tools are
not put in front of the model directly; they are deferred behind a search step. Hermes says
so in its startup line (the two numbers are replaced with placeholders):

```
🔎 Tool Search (tier 1): <number of tools> MCP/plugin tools deferred (~<number of tokens> tokens) behind tool_search/describe/call
```

So the skill told the model to use a tool the model could not see.

Two changes fixed it. First, turn tool search off in Hermes's `config.yaml`:

```yaml
tools:
  tool_search:
    enabled: "off"
```

The cost is that the definitions of those tools now go into every prompt, so each prompt
uses more tokens. I did not measure how many.

Second, the skill names the tools by their full registered names. Hermes registers an MCP
tool as:

```
mcp__<server>__<tool>
```

Both parts are sanitized, and a name longer than 64 characters is clamped with a hash
suffix (`mcp_prefixed_tool_name()` in `tools/mcp_tool_schema.py`). The short name a skill
author thinks of is not the name the model has to call.

## The server refused `book` when `read` looked finished

Once, the server seemed to be wrong. It was right.

The sequence: a `read` was started. As soon as the documents had been read, Hermes asked to
`book`. The server answered "already running" — a job is still in progress — and started
nothing.

It looked like a bug, because the `read` appeared to be over.

It was not over. After the documents are read, the flow has one more step: swapping the main
model back in, which takes about 4 more minutes. During that time the job still counts as
running, so the refusal was correct.

The real problem is that nothing says so. There is no notice that the flow is busy swapping
the model, so from the outside the run looks finished when it is not.

Hermes did not invent anything here either. It reported plainly that the request had been
refused, and waited for the first job to end.

The "no second run" rule was put in the server in the previous post. This is where it paid
off: the server held the line at a moment when the run looked finished to everyone outside
it.

## The final runs

The last runs were on the 12B model. I compared the numbers in the agent's report with what
the workflow engine had recorded, and every one matched. For example, the agent reported
that the job took 249.3 s, and the engine recorded `duration_ms` as 249302, which is the
same value. No document text appeared in the report.

Two things were still not good:

- the summary written for a human reader was shorter and less detailed than the skill asks
  for
- the JSON in the developer details had broken quoting

Both are limits of the model. Neither comes from the connection between the agent and the
server.

## What I learned

| Symptom | Actual cause | Fix |
|---|---|---|
| the agent reports a finished run with more files than exist, and the workflow engine has no such job and no data | the small model wrote the tool result itself instead of calling the tool; Thai content is a weak spot for small models | use a 12B model for this, and check every job id against the engine |
| model pod `OOMKilled` (exit code 137) while a 20-second sampler never showed the limit being reached | the spike fell between samples | treat "close to the limit" as over it; raise the limit (13Gi → 14Gi here) |
| wanting to cut context to save memory | Hermes Agent refuses to start below 64,000 tokens of context | keep the context, change the memory limit |
| pod stuck in `CreateContainerConfigError` after a restart, `endpoint not found in cache for a registered resource`, while the device plugin shows `Running` | the plugin registers with kubelet once and never again, so a kubelet restart leaves it alive but unregistered | restart the plugin to recover; the real fix is to re-register when `kubelet.sock` is recreated (a liveness probe on the plugin's socket file is a workaround that forces the same result) |
| Hermes says an MCP tool does not exist right after the skill named it | tool search deferred the MCP tools behind a search step, so the model could not see them | `tools.tool_search.enabled: "off"`, and use full `mcp__<server>__<tool>` names in the skill |
| the server refuses `book` with "already running" after `read` looked finished | the flow was still swapping the main model back, and nothing reports that it is doing so | the refusal was correct; what is missing is a notice that the model swap is in progress |
| numbers match but the summary is thin and the JSON quoting is broken | model quality | not a connection problem; it takes a better model |

Connecting was the easy part: under a second to connect, and all three tools found. Whether the
report can be trusted depends on the model. So every figure in a report has to be checkable
against a job id, by something other than the agent that wrote it.
