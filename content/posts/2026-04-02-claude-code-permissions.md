---
title: "Testing Claude Code's permission system"
date: 2026-04-02
draft: false
tags: [dev, ai, tools]
---

Claude Code asks permission before running bash commands. Except when it doesn't. Some commands run silently, others prompt every time, and the documentation doesn't fully explain which is which. I emptied my allow list, tested 40+ commands, enabled the sandbox, broke `gh`, and ended up back where I started — with finely tuned permission rules and no sandbox. Here's what I found.

## The four layers

Claude Code has four independent security layers, evaluated in order:

1. **Permission rules** — your `settings.json` allow/ask/deny lists
2. **Built-in safe command sandbox** — an undocumented curated allowlist
3. **User-configurable sandbox** — opt-in OS-level isolation (Seatbelt on macOS)
4. **Auto mode classifier** — an AI that evaluates each action (opt-in)

Most users only interact with layer 1. Layer 2 runs silently. Layers 3 and 4 are opt-in and come with trade-offs.

## The undocumented safe list

I set `"allow": []` and tested which commands ran without prompting. There's a curated list of commands that Claude runs inside `sandbox-exec` with network disabled and a read-only filesystem. This list isn't documented anywhere.

### Auto-approved

**Simple commands:** `cal`, `whoami`, `date`, `pwd`, `echo`, `ls`, `uname`, `which`, `type`

**File readers:** `cat`, `head`, `tail`, `grep`, `wc`, `diff`, `sort`, `cut`, `sed`, `find`, `file`, `stat`, `du`, `df`, `dirname`, `basename`, `realpath`

**Version checks (selective):** `python3 --version` and `node --version` pass. `swift --version`, `ruby --version`, `brew --version`, `npm --version`, `curl --version` don't.

**Git read-only:** `git status`, `git log`, `git diff`, `git show`, `git blame`, `git branch`

**Safe compound commands:** `ls | head`, `ls | grep "git"`, `git blame file | head -3`. Piping safe commands together stays safe.

### Always prompted

Shell features trigger warnings regardless of the underlying command:

| Feature | Warning |
|---------|---------|
| `echo $(whoami)` | "Command contains $() command substitution" |
| `` echo `whoami` `` | "Command contains backticks for command substitution" |
| `(echo "subshell")` | "Uses shell operators that require approval" |
| `tr 'a-z' 'A-Z' < file` | "Input redirection (<) could read sensitive files" |
| `echo "x" > file` | Path-based prompt: "allow access to X" |

Tools not in the safe list also prompt: `gh` (anything, needs network), `awk`, `env` (exposes secrets), `curl`, `npm`, `pip3`, `brew`, `make`, `touch`, `mkdir`, `rm`.

### The pattern

The safe list is read-only, local, no network, no sensitive data exposure, and on a specific allowlist. The choices aren't always obvious: `sed` is safe but `awk` isn't. `python3 --version` passes but `ruby --version` doesn't. `env` is read-only but exposes environment variables.

## The sandbox experiment

The sandbox (`/sandbox`) wraps bash commands in macOS Seatbelt. With auto-allow mode, commands that stay within boundaries run without prompting. It's both more permissive and more restrictive than the default:

| | No sandbox | Sandbox |
|---|---|---|
| Write to project dir | prompted | **auto** |
| `$()`, subshells, `<` | prompted | **auto** |
| `awk`, `env`, `brew --version` | prompted | **auto** |
| Write outside project | prompted | **OS blocked** |
| Write to `/tmp` | auto | **OS blocked** |
| Network access | prompted (command) | **prompted (domain)** |
| Settings files | unprotected | **OS blocked** |

Anthropic reports an 84% reduction in prompts internally.

### Where it breaks

Go-based CLI tools — `gh`, `gcloud`, `terraform` — fail with TLS certificate errors inside the sandbox. The network proxy intercepts HTTPS, and Go's TLS stack rejects the proxy's certificate because it can't reach macOS `trustd`.

```
tls: failed to verify certificate: x509: OSStatus -26276
```

`enableWeakerNetworkIsolation` is supposed to fix this, but as of early 2026 [the setting is broken](https://github.com/anthropics/claude-code/issues/28954) — it's stripped by the settings validator and never reaches the sandbox runtime. `excludedCommands` only exempts from the filesystem sandbox, not network.

The workaround is [rebuilding `gh` with embedded root certificates](https://zencoder.ai/blog/fixing-gh-cli-in-claude-codes-sandbox). It works, but maintaining a custom `gh` build isn't something I wanted to take on.

### The `gh` problem

Even if TLS worked, the sandbox can't tell read from write on the same domain. `gh pr list` and `gh gist create -f secrets.txt` both hit `api.github.com`. Allowing the domain means allowing both. For tools with read and write capabilities, permission rules are safer — they let you allowlist specific subcommands.

## What I ended up with

No sandbox. Permission rules only. The built-in safe list handles most read-only commands. Explicit rules handle the rest:

```json
{
  "permissions": {
    "allow": [
      "Bash(gh pr list*)",
      "Bash(gh pr view*)",
      "Bash(gh issue list*)",
      "Bash(gh issue view*)",
      "Bash(gh repo view*)",
      "...",

      "Bash(git add*)",
      "Bash(git stash*)",
      "Bash(git fetch*)",

      "WebFetch(domain:docs.github.com)",
      "WebSearch"
    ],
    "ask": [
      "Bash(gh pr create*)",
      "Bash(gh pr merge*)",
      "Bash(gh gist create*)",
      "...",

      "Bash(git commit*)",
      "Bash(git push*)",
      "Bash(git reset --hard*)",
      "Bash(git branch -D*)",
      "..."
    ]
  }
}
```

`allow` auto-approves read-only `gh` subcommands, safe git operations, and trusted documentation domains. `ask` always prompts for git writes and `gh` mutations, even if you accidentally click "don't ask again" on a similar command. `ask` rules override saved approvals because `ask > allow` in the evaluation order.

Everything not in either list prompts by default. It's an allowlist approach: safe until explicitly permitted.

## Key takeaways

**The safe list is your friend.** Most commands Claude runs day-to-day are already auto-approved. You don't need rules for `git status`, `ls`, `cat`, or pipes between safe commands.

**`ask` rules protect against misclicks.** Without them, accidentally clicking "Yes, don't ask again" on `git push` permanently auto-approves it. `ask` rules can't be bypassed.

**Pattern matching is fragile for arguments.** `Bash(curl http://github.com/ *)` fails against reordered flags, variables, and protocol variants. For network tools, deny the command and use `WebFetch(domain:...)` instead.

**The sandbox isn't ready for Go toolchains.** If you use `gh`, `gcloud`, or `terraform`, you'll hit TLS errors. Revisit when Anthropic fixes the `enableWeakerNetworkIsolation` wiring.

**No space before `*` is simpler.** `Bash(gh pr list*)` matches both `gh pr list` and `gh pr list --limit 3`. With a space, you rely on a word-boundary rule that also matches end-of-string. Works, but it's one more thing to remember.

## References

**Official docs:**
- [Configure permissions](https://code.claude.com/docs/en/permissions)
- [Sandboxing](https://code.claude.com/docs/en/sandboxing)

**Anthropic engineering blog:**
- [Making Claude Code more secure and autonomous](https://www.anthropic.com/engineering/claude-code-sandboxing) — design rationale for the built-in safe list and user sandbox
- [Claude Code auto mode: a safer way to skip permissions](https://www.anthropic.com/engineering/claude-code-auto-mode) — the AI classifier behind layer 4

**Open-source implementation:**
- [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime) — the Seatbelt/bubblewrap wrapper Claude Code uses

**Independent testing:**
- [Understanding Claude Code Permissions — Pete Freitag](https://www.petefreitag.com/blog/claude-code-permissions/)

**Known issues and workarounds:**
- [enableWeakerNetworkIsolation setting not wired through](https://github.com/anthropics/claude-code/issues/28954)
- [Go TLS issue causing gh/gcloud/terraform failures](https://github.com/anthropics/claude-code/issues/26466)
- [Fixing gh CLI in Claude Code's sandbox — Zencoder](https://zencoder.ai/blog/fixing-gh-cli-in-claude-codes-sandbox) — embedded certificates workaround
