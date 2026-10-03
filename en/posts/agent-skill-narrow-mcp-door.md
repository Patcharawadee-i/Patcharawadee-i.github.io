---
layout: post
lang: en
slug: agent-skill-narrow-mcp-door
series: agents
title: "A sandboxed agent cannot curl your internal services: one narrow MCP door instead of a skill full of commands"
date: 2026-10-03 17:57:10 +0000
description: "I wanted to tell my self-hosted agent to run a document pipeline. The agent's sandbox is built so that commands cannot reach internal services, so a skill made of curl commands starts nothing. What I built instead: a small internal MCP server with three tools and no generic one."
tags: [ai-agents, hermes-agent, mcp, kubernetes, networkpolicy, windmill, sandbox]
---

## TL;DR

If your agent runs its commands in a sandbox that is meant to have no route to internal
services, a skill made of `curl` commands cannot start a workflow, and opening the sandbox
is the wrong fix. Put a small internal MCP server between the agent and the workflow engine,
give it a few narrowly named tools and no generic "run anything" tool, and keep rules like
"no second run while one is running" in the server, not in the skill text. If you then
restrict who may call that server with a Kubernetes NetworkPolicy, remember policies are
additive: you have to exclude the pod from the wide allow rule **and** add the narrow one,
both or neither.

## Hardware and software

| Item | Value |
|---|---|
| Machine | AMD mini PC (Strix Point), the one from the [Building an AI Station]({{ '/en/series/#infra' | relative_url }}) series |
| Agent | Hermes Agent v0.21.3 (Docker image tag `v2026.9.14`), running in Kubernetes |
| Workflow engine | Windmill CE 1.817.0 |
| Pipeline | two workflows: `read` (OCR plus three readers) and `book` (extracts fields, runs checks, stores rows) |
| MCP server | Python, standard library only, 18 MiB idle, 27 unit tests against a fake workflow engine |
| Transport | Streamable HTTP |

Names of internal services are replaced with placeholders throughout, for example
`<mcp-server>`.

## The goal

I wanted to say one sentence to the agent, in plain Thai — "read the documents in the
folder" — and have it run my document pipeline and report back. The pipeline already existed
as two Windmill workflows, `read` and `book`. What was missing was the part that lets an
agent start them.

## Step 1: test the flows before wrapping anything around them

The rule I set before writing any skill: **a skill built on untested flows just moves the
failure somewhere harder to read.** If the workflow is broken, I want to see that in the
workflow engine, not as a vague sentence from an agent.

So the flows were run for real first, on 8 test files (9 pages):

| Run | Result |
|---|---|
| `read`, 8 files | 42.3 min |
| `book`, the same files | 35.7 min |
| `read` with nothing new in the folder | under half a second (269, 275, 305 and 419 ms over four runs), no GPU use |
| model swap | 301 s in, 242 s back |
| OCR model | 65–140 s per page |

A note on these numbers: `read` and `book` were still prototypes at this point, and the
times above include extra time added by errors that had not been fixed yet. That has since
been fixed, and `read` now takes roughly 20–30 minutes.

Most of the `read` time is not reading. It goes to swapping models in and out (about 4–5
minutes each way) and to the OCR model.

Two behaviours were worth confirming before an agent ever touched this:

- **Nothing new means nothing happens.** With no new file in the folder the run stops in
  under half a second and never touches the GPU. An agent that calls `read` too often costs
  almost nothing.
- **One file can be re-read by its hash**, and the other 7 stay untouched. A fix for one bad
  file does not mean another 42 minutes.

## Step 2: the wall

The obvious way to write a skill is a list of commands: `curl` the workflow engine's API to
start a flow, `curl` again to poll it. That does not work here, and it is not a bug.

Hermes can run every terminal command in a separate sandbox pod. The whole point of that
sandbox is that whatever runs in it — including code the model wrote a moment ago — has no
route to internal services. And Hermes v0.21.3 has no generic HTTP tool of its own. Put
those two together and a skill made of `curl` commands has nowhere to run from: the only
place the agent can run `curl` is the one place that must not reach the workflow engine.

### Options I rejected

- **Run commands in the agent pod itself.** That throws the sandbox away.
- **Open a hole from the sandbox to the workflow engine.** Then any untrusted code in the
  sandbox can use that hole too. The hole does not know who is asking.
- **Make the workflow trigger reachable from the internet.** No.

All three "fix" the problem by removing the thing that made the setup safe to begin with.

### What I chose: an internal MCP server

MCP is the one extension point Hermes has that does not go through the sandbox: MCP tools
are called by the agent itself, not by a command the agent runs. So the door goes there — a
small MCP server inside the cluster, with no public route, that knows how to start exactly
the flows I decided it may start.

## Designing the door

I expect more skills from other domains later, so this is one shared server, `<mcp-server>`.
What is shared is the container, never the verbs: each domain gets its own narrowly named
tools.

The rules I designed it around:

- **Three tools, named for what they do: `read`, `book`, `status`.** There is no generic
  "run any flow" tool and no "call any URL" tool.
- **The workflow engine's token belongs to the server only.** The agent never needs to hold
  it, and nothing in the sandbox ever needs to see it.
- **The tools take no `model` argument.** An argument that is not there cannot be abused.
  The pipeline should enforce the same thing on its own side too, and refuse any model that
  is not local, so the choice never depends on what an agent passes in.
- **The agent asks for a flow by name and nothing more.** It gets no GPU access of its own;
  GPU work queues on one worker.
- **The server has no reason to talk to the Kubernetes API**, so it gets no Kubernetes
  credentials.

> ⚠️ **One generic tool undoes all of this.** The moment the server has a "run any flow" or
> "call any URL" tool, it stops being a door and becomes a hole in the wall with extra
> steps: everything the sandbox was built to prevent is reachable again, just through a
> tool call.
{: .warn}

### Rules that matter go in the server, not in the skill

One rule is easy to get wrong: do not start a second run while one is queued or running.

The tempting place for it is the skill text — "check status before you start a run". But
instructions in a skill are only manners, and an agent can skip manners. So the check lives
in the server. A start request that arrives while a job is queued or running starts nothing
and gets the existing job id back. The shape of that answer (names replaced with
placeholders):

```json
{"ok": false, "state": "already running", "job_id": "<job-id>", "what": "<flow-path>", "running": <true|false>,
 "note": "a <domain> job is already queued or running; nothing was started. Follow that job with <status-tool>."}
```

The agent is free to be wrong about whether a run is in progress. The server is not.

## NetworkPolicies are additive

A server like this should accept calls from one caller only: the agent. In Kubernetes that
is a NetworkPolicy, and the obvious move — add a narrow policy saying "only the agent may
call this pod" — changes nothing if a wide "same-namespace traffic is allowed" policy still
selects the pod.

NetworkPolicies are additive. What a pod accepts is the union of everything any policy
selecting it allows. A narrow policy does not override a wide one; it only adds to it.

So it takes two changes, and they only work together. This is a generic version, every name
a placeholder:

```yaml
# Wide rule: same-namespace traffic allowed, except for listed pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: <wide-policy>, namespace: <namespace>}
spec:
  podSelector:
    matchExpressions:
      - {key: app, operator: NotIn, values: [<protected-app>]}
  policyTypes: [Ingress]
  ingress:
    - from: [{podSelector: {}}]
---
# Narrow rule: only one caller may reach the protected pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: <narrow-policy>, namespace: <namespace>}
spec:
  podSelector: {matchLabels: {app: <protected-app>}}
  policyTypes: [Ingress]
  ingress:
    - from: [{podSelector: {matchLabels: {app: <caller-app>}}}]
      ports: [{port: <port>, protocol: TCP}]
```

| What you apply | What the protected pod accepts |
|---|---|
| narrow rule only, wide rule still selecting the pod | everything the wide rule allows — the narrow rule changed nothing |
| exclusion from the wide rule only | everything — a pod selected by no policy is open to all |
| exclusion **and** narrow rule | the one caller, on the one port |

> ⚠️ **Doing half of this is worse than doing none of it.** Exclude the pod from the wide
> rule without adding the narrow one and no policy selects it at all, which in Kubernetes
> means it accepts traffic from everywhere. Nothing warns you. Test from both sides: the
> caller that should get through, and some other pod that should not.
{: .warn}

## The skill's description is its trigger

The agent picks a skill by matching my words against the skill's one-line description. That
makes the description part of the safety design, not documentation.

A description as broad as "Read documents" would also match annual reports and contracts.
The pipeline is not built for those; it would turn them into nonsense rows and burn GPU time
doing it. So the description names the specific kind of document the pipeline handles, and
the skill has a "When to Use" section that lists what **not** to use it for. If the agent is
unsure, the skill tells it to ask first.

## The report: counts only

What comes back from a run is counts, never document text:

```
done — 8 files read, 6 booked, 2 need a look
```

plus a dev block with the job id, the time per stage, and any errors.

The reason is where a report goes. It lands in the agent conversation, then goes to the
model, then into traces. A chatty report that quotes what it read carries document content
into all three. Counts and a job id are enough to check the run against the workflow engine,
and they say nothing about what the documents contain.

## Where this part ends

The connection worked: Hermes connected to the server over Streamable HTTP in 801 ms and
found 3 tools.

Then the model reported a run that never happened. That is the next post.

## What I learned

| Situation | What goes wrong | What to do |
|---|---|---|
| wrapping a skill around flows you have not run end to end | the failure moves from the workflow engine into an agent's sentence, where it is harder to read | run the flows for real first, and write the numbers down |
| a skill made of `curl` commands, with commands running in a sandbox | the sandbox is the one place that must not reach internal services, so nothing starts | use an extension point the agent calls directly — here, MCP |
| "just let the sandbox reach that one service" | untrusted code in the sandbox can use the same route | keep the sandbox closed; put a server with named tools in between |
| adding a generic "run any flow" tool to save time later | the server becomes a hole in the wall with extra steps | one narrowly named tool per thing the agent may do |
| a `model` argument on a tool | the agent, or a prompt it read, can choose where the data goes | leave the argument out, and enforce it in the pipeline as well |
| "do not start two runs" written in the skill text | skill instructions are manners an agent can skip | enforce it in the server and return the existing job id |
| adding a narrow NetworkPolicy next to a wide one | policies are additive; nothing changes | exclude the pod from the wide rule **and** add the narrow rule |
| excluding the pod from the wide rule, and stopping there | a pod selected by no policy accepts everything | both changes or neither, then test from both sides |
| a broad skill description | the skill triggers on documents the pipeline was not built for | name the specific document kind, and list what not to use it for |
| a report that quotes what it read | document text ends up in the conversation, the model's context and the traces | report counts and a job id only |

The thread through all of it: every rule I cared about had to live somewhere the agent
cannot talk its way past — in the server, in the network, in an argument that does not
exist. The skill text is where the agent is told what to do. It is not where anything is
enforced.
