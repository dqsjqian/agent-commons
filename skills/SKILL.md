---
name: agent-commons
protocol_version: "1.0"
description: |
  Agent Commons — runtime capabilities for an AI agent that has already joined
  the local cross-agent shared memory system at ~/.agent-commons/. Provides:
  read shared user identity, rules, and routines; read/update the cross-agent
  current-focus state; check the inbox for messages from other agents; append
  to today's per-agent daily log; send a message to another agent's inbox.
  Triggers on phrases like:
  "read shared identity", "read agent commons rules", "what's the current focus",
  "check my inbox", "log this to agent commons", "hand off to <other-agent>",
  "读共享身份", "读共享规则", "当前焦点", "检查收件箱", "记到共享日志",
  "交接给 <其他 agent>", any equivalent.
  NOTE: this is the RUNTIME skill, not the joining flow. If the agent has not
  yet joined, redirect to ~/.agent-commons/ONBOARDING.md instead.
---

# Agent Commons — Runtime Skill

> This skill is for agents **already joined** to Agent Commons. It provides ongoing capabilities — reading shared context, writing logs, coordinating handoffs.
>
> **Not joined yet?** Stop here and read `~/.agent-commons/ONBOARDING.md` first. That file contains the one-time install / verify / register flow. Do not try to onboard from this skill — they are deliberately decoupled.

## Self-check on first invocation (each session, before doing real work)

Run these three checks. Cheap, fast, mandatory.

### 1. Am I registered?

```bash
grep -q "\"<your-agent-name>\"" ~/.agent-commons/registry.json && echo registered || echo not_registered
```

- **registered** → continue.
- **not_registered** → tell the user: *"I'm being asked to use the Agent Commons runtime skill, but I'm not registered in the central registry. I should run onboarding first — pointing me at `~/.agent-commons/ONBOARDING.md`?"* and stop.

### 2. Is the protocol version compatible?

Read `~/.agent-commons/skills/manifest.json` and compare its `protocol_version` to the version recorded on your registry entry at join time:

```bash
central_ver=$(grep -E '"protocol_version"' ~/.agent-commons/skills/manifest.json | head -1)
my_ver=$(grep -A4 "\"<your-agent-name>\"" ~/.agent-commons/registry.json | grep protocol_version)
```

- **Same major** (e.g. both `1.x`) → continue. (A minor or patch bump is backward compatible.)
- **Central has higher major** (e.g. you joined under `1.x`, central is `2.0`) → **stop and re-run `~/.agent-commons/ONBOARDING.md` from the top**. The on-disk schema may have changed in incompatible ways. Tell the user *"Agent Commons protocol bumped to a major version since I joined. Re-running onboarding."*

### 3. Is my install up to date? (Tier 2 only)

Tier 1 (symlink) and Tier 3 (readonly) always read live — skip this check.

If you installed via Tier 2 (copy), verify your snapshot isn't stale:

```bash
# Compare central manifest mtime vs your snapshot's
[ ~/.agent-commons/skills/manifest.json -nt "$SKILLS_ROOT/agent-commons/manifest.json" ] && echo stale || echo fresh
```

If **stale**, mirror-resync per ONBOARDING.md Step 7.2:

```bash
rsync -a --delete ~/.agent-commons/skills/ "$SKILLS_ROOT/agent-commons/"
# or pure-shell:
TMP=$(mktemp -d); cp -R ~/.agent-commons/skills/. "$TMP/" && rm -rf "$SKILLS_ROOT/agent-commons" && mv "$TMP" "$SKILLS_ROOT/agent-commons"
```

**After resyncing, re-run the trigger self-test from ONBOARDING.md Step 4.B** — schema changes can break the runtime's view of the skill even when the file exists. If trigger now fails, walk down the tier ladder (Step 4.C). Don't keep using a skill the runtime can no longer parse.

Trigger this resync **at most once per session per day** — not every turn. A reasonable heuristic: cache a marker file `$SKILLS_ROOT/agent-commons/.last-sync-check` with today's date, skip the check if it matches.

---

## Path notation

All paths use POSIX style (`~/.agent-commons/...`). On Windows native (PowerShell), translate `~` to `$env:USERPROFILE` and `/` to `\`. Same files, different syntax.

---

## Capability 1 — Read shared user context

The user's cross-agent context lives at:

| File | Purpose |
|---|---|
| `~/.agent-commons/identity/profile.md` | Who the user is |
| `~/.agent-commons/identity/ROUTINE.md` | Daily schedule / routines |
| `~/.agent-commons/rules/universal.md` | **Mandatory commandments** — highest priority, overrides everything else |
| `~/.agent-commons/rules/public-repo.md` | Public-repo hard rules |
| `~/.agent-commons/rules/file-cleanup.md` | File deletion preferences |
| `~/.agent-commons/rules/safety.md` | Safety guardrails |
| `~/.agent-commons/projects/active.md` | What the user is working on |
| `~/.agent-commons/handoff/shared-state/current-focus.md` | What any agent is currently focused on |
| `~/.agent-commons/toolchain/*.md` | Tool-specific config — read on demand |

**Read on demand** — don't slurp everything every turn. The rules and identity files are stable; cache mentally for the session. The current-focus and active.md change frequently; re-read when it matters.

---

## Capability 2 — Update cross-agent shared state

`~/.agent-commons/handoff/shared-state/current-focus.md` is the collaborative "what is being worked on right now" board.

When you start or finish a major task, **update this file in place** with `Edit`, not `Write`. Other agents read it to know what's hot.

Format convention (loose — edit the existing file's style):

```
> Last updated: 2026-05-28T14:00 by <your-agent-name>

## Current focus
<one-paragraph state of what's happening, who's doing what>
```

**Don't** rewrite history that other agents wrote. Append your update, or replace the section that's specifically about your work.

---

## Capability 3 — Check inbox / send messages

Inbox lives at `~/.agent-commons/handoff/inbox/`.

### Receive

When the user says "check your inbox" / "any messages for you?":

```bash
ls ~/.agent-commons/handoff/inbox/ | grep "to-<your-agent-name>-"
```

For each match: read it, act on it, then `mv` to `~/.agent-commons/handoff/archive/`.

### Send

To send a message to another agent:

```
~/.agent-commons/handoff/inbox/from-<your-name>-to-<dst-name>-<topic>.md
```

Markdown body. Be specific: what you did, what's left, where the artifacts are. The recipient may be a different model / different session — write for someone with no context.

---

## Capability 4 — Daily log

After **substantive work**, append to:

```
~/.agent-commons/log/daily/YYYY-MM-DD-<your-agent-name>.md
```

**Per-agent file** — don't write to other agents' daily files. Always **append**, never overwrite.

Substantive work = built/modified a project, fixed a bug, made a non-trivial decision, learned a long-term user fact. **Skip** greetings, simple lookups, short Q&A.

A good log entry:

```markdown
## <short title>

**What**: <one paragraph>
**Why**: <if non-obvious>
**Result**: <commit hash, file paths, or "blocked because X">
**Cross-agent note**: <only if other agents need to know — e.g., "Updated rules/public-repo.md, all agents should re-read">
```

---

## Capability 5 — Update last_seen on registry

Once per session (lightly — no need every turn), update your entry's `last_seen` in `~/.agent-commons/registry.json` via `Edit`:

```json
"last_seen": "<ISO 8601 now>"
```

Don't rewrite other fields unless they actually changed (e.g. a new capability). **Never overwrite the whole file** — `Edit` patches only.

---

## Failure modes

- Some files missing → read what exists, note the rest. Don't block.
- `registry.json` not writable → log the issue, proceed in read-only mode.
- Today's daily log file doesn't exist → create it with a top heading `# YYYY-MM-DD — <your-agent-name>`, then append.
- Inbox has files written in a format you don't understand → read them anyway, reply to sender with a structured request for clarity.

---

## What this skill does NOT do

- **Onboard / install / register first-time.** That's `~/.agent-commons/ONBOARDING.md`.
- **Resync the skill files themselves.** If you used Tier 2 (copy) and need fresh skill files, see ONBOARDING.md Step 6.
- **Manage the toolchain.** `toolchain/*.md` is read-only context for you; the user owns it.

---

## Spec

- Manifest: `~/.agent-commons/skills/manifest.json`
- Onboarding (one-time): `~/.agent-commons/ONBOARDING.md`
- Repository: https://github.com/dqsjqian/agent-commons
- License: MIT
