---
layout: post
title: "Running an autonomous pentest agent against my own lab, and what stopped it"
date: 2026-11-18 18:00:00 -0600
tags: [ai-security, agents, llm, safeguards, owasp-juice-shop, docker]
---

**Summary.** I set up Strix, an open-source autonomous AI pentesting agent, against a local OWASP Juice Shop container to see how an LLM-driven attacker behaves end to end. The harness worked. Getting a model to drive it did not. A self-hosted 27B model reasoned for about 28,000 tokens without stopping and hung the agent loop. A frontier model behind an API refused on the very first turn with a content-filter error, even though the target was a deliberately vulnerable app on my own machine. This post is about those two failures: what caused each one, why the refusal is a reasonable design choice rather than a bug, and what it means for anyone building offensive security agents. It is not a findings post. The agent hasn't produced a single finding against Juice Shop yet.

## Scope and authorization

Everything ran on my own laptop. The only target was a local OWASP Juice Shop container, an app built to be attacked for training. The agent and the target ran in separate containers on their own Docker network, and my own attacker VM was never a target.

The lab network was an ordinary user-defined bridge, not an `--internal` one, so the Strix sandbox did have outbound internet access (it needs it to pull tools). The agent's only target was the Juice Shop container.

## Setup

- Strix 1.7.0, installed with pipx in WSL2 (Ubuntu) on my laptop
- Strix runs its tools inside a Docker sandbox: a kali-rolling container with nmap, sqlmap, nuclei, ffuf and subfinder
- OWASP Juice Shop in its own container
- LiteLLM as the layer between Strix and whatever model backend I pointed it at

The networking took a little thought. I created a user-defined bridge network, attached Juice Shop to it, and used Strix's `STRIX_DOCKER_SANDBOX_NETWORK` environment variable to put the agent's sandbox on the same network. That gives the sandbox Docker's internal DNS, so the target is reachable by container name:

```bash
# Z13 — WSL2 (Ubuntu)
docker network create strixnet
docker run -d --name juiceshop --network strixnet -p 3000:3000 bkimminich/juice-shop
export STRIX_DOCKER_SANDBOX_NETWORK=strixnet

# the scan
strix --target http://juiceshop:3000 -m quick -n --max-budget 5
```

From inside the sandbox, `http://juiceshop:3000` returned a 200. The harness was ready. The interesting part was the model.

## Failure 1: a model that wouldn't stop thinking

The first backend, in October 2026, was a self-hosted open-weights 27B Qwen model, served with llama.cpp on a rented A40 GPU.

Strix sends a very large agentic prompt. Mine came to roughly 282,000 tokens once the system prompt, tool definitions and context were assembled. Strix's default reasoning effort is "high", which in this setup mapped to the model's maximum reasoning setting. The model's response was a single reasoning dump of about 28,000 tokens with no tool call at the end. The agent loop sat waiting for an action that never came.

The root cause was two things stacking up:

1. **Over-reasoning.** Maximum effort on a huge prompt invites the model to plan everything up front instead of taking one step.
2. **No output cap.** LiteLLM looks up a model's limits in its model registry. A custom self-hosted model isn't in that registry, so there was no default limit on output length to stop the runaway.

The partial fix was to bound every part of the loop explicitly:

```bash
export STRIX_REASONING_EFFORT=low
export STRIX_TOOL_OUTPUT_MAX_TOKENS=4000
export STRIX_CONTEXT_SUMMARY_TOKENS=2048
```

That brought turns down to about 100 tokens each, and the loop kept moving. But at around 24 tokens per second, a full scan would take hours. It was bounded, but it wasn't practical.

The engineering lesson: an agent harness can't assume the model will stop on its own. Output caps, reasoning budgets and context limits have to be set explicitly, especially for any model the routing layer doesn't already know.

## Failure 2: a refusal on turn one

The second backend, on October 20, was the Claude API (`anthropic/claude-sonnet-5-5`, reasoning effort high), through the same LiteLLM layer. The first real scan failed on its first turn:

```text
ModelRefusalError: Response withheld by the provider's content filter
```

That turn sent about 86,200 input tokens, got 0 output tokens back, and cost about $0.22. I reran it several times and got the same refusal every time, so it wasn't a fluke.

My first reaction was frustration, because the target was a training app on my own laptop. On reflection, I think the safeguard did the right thing, and the reason is worth spelling out.

**The filter is judging the role, not the target.** Look at what the model actually receives: a system prompt telling it that it is an autonomous attacker, tool definitions for scanners and SQL injection tooling, and instructions to find and exploit vulnerabilities without a human approving each step. The only evidence that the target is legitimate is text in the prompt, and anyone can write "this is authorized" in a prompt. From the provider's side, my request is indistinguishable from someone pointing the same agent at a server they don't own. A safeguard that trusted the prompt's own claims about authorization would protect nothing.

**Autonomy changes the risk.** A model answering a question about SQL injection keeps a human in the loop. A model that runs sqlmap itself, reads the output and decides what to do next doesn't. It's reasonable for a provider to draw a firmer line around the second case.

**The right fix is verified access, not a cleverer prompt.** Rewording the system prompt until the filter stops firing would be trying to get around a safety control, and it would teach me nothing useful. The sanctioned path for this kind of research is access tied to a verified identity and an accountable context. For me that means research access through my advisor's lab, which is still pending. Since the refusal, I've also been accepted into Anthropic's Cyber Verification Program, which exists for exactly this kind of authorized security work.

## What this means for people building security agents

1. **Budget everything.** Reasoning effort, output tokens, tool output and context summaries all need explicit limits. A runaway turn costs money and hangs the loop, and the harness won't notice unless you tell it what "too long" means.
2. **Treat refusals as an expected state.** A frontier backend can refuse an offensive agent's prompt at any turn. Strix surfaced it as a clear `ModelRefusalError`, which is the right behavior. A harness that silently retried or swallowed it would make the failure much harder to debug.
3. **Authorization lives outside the prompt.** If you're building or evaluating offensive agents, plan for verified access from the start: an organization account, a research program, or a self-hosted model under your own accountability. Don't plan on the prompt convincing the model.
4. **Isolate the agent like you'd isolate malware.** An autonomous agent with sqlmap and nuclei will act on what it decides. Put it on a network where the only reachable thing is the target you mean, and keep your own machines off that list.

## What's next

The harness is ready and the target is scoped to my own lab. What I need is a backend that's both fast enough to finish a scan and appropriate for this use. Once I have that through verified access, I'll run a full scan against Juice Shop and write up what the agent finds, what it misses, and where its reasoning goes wrong.

## Lessons learned

- **The harness was the easy part.** Containers, networking and tooling worked. Model behavior was where everything broke.
- **Defaults assume a known model.** Limits that exist for registered models silently vanish for custom ones.
- **A refusal is information.** It told me exactly how a provider sees an autonomous attacker, which matters more for AI security work than one more scan result.
