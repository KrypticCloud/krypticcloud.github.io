---
layout: post
title: "Treating my coding agent as an untrusted process: sandboxing Claude Code and scoping my tailnet"
date: 2026-11-18 08:40:00 -0600
tags: [ai-security, agents, claude-code, sandboxing, tailscale, zero-trust, blue-team]
---

**Summary.** A coding agent runs commands on your machine, reads your files, and can be steered by anything it reads: a README, an issue comment, a web page. I decided to treat mine as an untrusted process and lock down what it can reach. On my laptop that meant a strict OS-level sandbox and deny rules for secrets in Claude Code, on both WSL2 and Windows. While doing that, I also found that rented GPU machines had been joining my private network as me. This post covers the patterns I used, the mistake I found, and how I tested the fix. It describes patterns, not my live config, so ports, device names and network names are left out on purpose.

## Scope

Everything here is my own laptop, my own accounts and my own private network. Nothing was tested against anyone else's systems. The config fragments below are generic examples of each pattern, not copies of my files.

## Threat model

The agent isn't malicious. The problem is that it acts on text, and some of that text comes from people I don't control. A prompt injection in a dependency's README or a scraped page can try to get the agent to read an SSH key, print a `.env` file, or send data somewhere. So the questions I asked were:

1. If the agent is tricked, what files can it read?
2. What can it run without asking me?
3. Where on the network can it, or anything else running as me, reach?

## Layer 1: an OS-level sandbox in WSL2

In WSL2 (Ubuntu), I run Claude Code with its strict sandbox on. On Linux that's built from bubblewrap for filesystem and process isolation, socat for brokering network access, and a seccomp filter to restrict system calls. Commands the agent runs go through the sandbox instead of running with my full user access.

On top of the sandbox:

- **Deny and ask rules for secrets.** The agent can't read SSH keys, API keys or credential files, and it's blocked from the secrets on my Windows drive, which WSL2 mounts into Linux by default and which is easy to forget.
- **No claude.ai connectors and no plugin sync on this machine.** Every connector is another set of tools the agent can call and another place data can go. I didn't need them here, so they're off.
- **A global CLAUDE.md** with my standing rules for how the agent works.

A trimmed version of my settings looks like this. Two settings matter most: `failIfUnavailable` means Claude Code refuses to run rather than quietly running without the sandbox, and `allowUnsandboxedCommands: false` means the agent can't decide on its own to step outside it.

```json
{
  "disableClaudeAiConnectors": true,
  "syncClaudeAiPlugins": false,
  "permissions": {
    "defaultMode": "auto",
    "deny": [
      "Read(~/.ssh/**)",
      "Read(~/.config/gh/hosts.yml)",
      "Read(~/.claude/.credentials.json)",
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(//mnt/c/Users/<me>/.ssh/**)",
      "Read(//mnt/c/Users/<me>/AppData/**)"
    ],
    "ask": [
      "Edit(~/.claude/settings.json)",
      "Edit(~/.claude/settings.local.json)",
      "Edit(~/.claude/CLAUDE.md)",
      "Edit(~/.bashrc)",
      "Edit(~/.profile)",
      "Edit(~/.bash_profile)",
      "Edit(~/.ssh/**)",
      "Edit(//mnt/c/Users/<me>/.ssh/**)",
      "Edit(//mnt/c/Users/<me>/AppData/**)",
      "Bash(sudo *)",
      "Bash(rm *)",
      "Bash(systemctl *)",
      "Bash(crontab *)",
      "Bash(docker *)",
      "Bash(git push *)",
      "Bash(powershell.exe *)",
      "Bash(cmd.exe *)",
      "Bash(wsl.exe *)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": ["docker *", "gh *", "git push *", "git fetch *", "git pull *"],
    "network": {
      "allowedDomains": ["github.com", "*.github.com", "pypi.org", "registry.npmjs.org"]
    }
  }
}
```

**The tradeoff I accepted.** Docker, `gh` and `git push`/`fetch`/`pull` run outside the sandbox. They need network access and sockets the sandbox doesn't give them. That's a real gap, and I want to be honest about it. Access to the Docker socket is close to root on the host, so an agent that can run arbitrary `docker` commands can get around a lot of the sandbox. The mitigation is to keep the permission layer in front of those commands. In my rules, `docker` and `git push` still need my approval every time. `gh`, `git fetch` and `git pull` don't, which is the part of this gap I'd tighten next.

## Layer 2: a stricter default on Windows

On the Windows side, Claude Code runs in a more conservative mode: manual approval is the default, and bypass mode is disabled so it can't be switched on mid-session. Connectors and plugin sync are off here too. Deny rules cover keys, `.env` files, browser profiles (which hold session cookies) and an archive of old files I keep for recovery.

While setting this up I also deleted `.env` files left behind by old personal assistant projects. A deny rule protects a secret from the agent, but deleting a secret nobody uses protects it from everything.

## Layer 3: the network, and the mistake I found

My laptop is on a Tailscale tailnet, and the tailnet policy is an allowlist. Only two kinds of traffic reach the laptop: my phone to a remote-desktop service, and my own devices to a file-sync service. Everything else is denied by default.

In October 2026, a public disclosure about Tailscale SSH ("Tailworm") made me go back and audit who was actually on my tailnet. I started with the script that sets up the rented GPU pods I use for model experiments. It ran `tailscale up` with no tag, and the admin console showed the auth key it used was reusable and ephemeral but untagged. That meant every pod joined the tailnet as my own user, so to Tailscale it looked exactly like one of my personal devices. The key itself was handled carefully (typed at a hidden prompt, passed over stdin, never written to disk), but that didn't matter: whatever joined with it got my access. Any rule that let "my devices" in also let in a container on a machine I rent from someone else.

The fix was to give those machines their own identity and the smallest possible access:

1. **A dedicated tag.** Pods now join with a tagged key, so they're identified by the tag instead of my user. Only the admin role can assign the tag.
2. **One grant.** My own devices can reach the pods on the single port the pods' service uses. Nothing grants the pods access to anything.
3. **Policy tests.** Tailscale policies support a `tests` block that's checked every time the policy is saved. I added tests that assert the tag is denied to my home server and my desktop. If a future edit accidentally opens a path, the policy won't save.

A sanitized version of the pattern:

```json
{
  "tagOwners": {
    "tag:gpu": ["autogroup:admin"]
  },
  "grants": [
    { "src": ["autogroup:member"], "dst": ["tag:gpu"], "ip": ["tcp:<service-port>"] }
  ],
  "tests": [
    { "src": "tag:gpu", "deny": ["<home-server>:<admin-ports>", "<desktop>:<ssh-port>"] }
  ]
}
```

Then I revoked the old key. The new one lives in my password manager and is passed to the pod over stdin, so it never touches disk.

## What this means for defenders

1. **Treat the agent as untrusted input with a shell.** Assume anything it reads might be trying to steer it, and decide ahead of time what it can reach if that works.
2. **Use layers that fail independently.** The OS sandbox, the permission rules and the network policy each catch different mistakes. Don't rely on one.
3. **Write down every exception.** Each command that runs outside the sandbox is a hole. Knowing exactly which ones you allowed is what lets you watch them.
4. **Untagged reusable keys are a lateral movement path.** If a machine you don't fully control joins your tailnet as you, it gets everything you get. Give infrastructure its own tag and grant it only what it needs.
5. **Test your access policy like code.** Tests that assert what must stay denied turn an easy-to-miss mistake into a failed save.

## Lessons learned

- **The biggest exposure wasn't the agent.** I went looking for agent risk and found an identity problem in my network.
- **Defaults are generous.** A reusable key with no tag does exactly what it says, and that's the problem.
- **Deleting beats denying.** Old secrets I didn't need were a liability no matter how well they were fenced off.
