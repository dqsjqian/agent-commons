# Agent Commons — Conventions (non-normative)

> This file describes **optional conventions** for things that are *not* part of the core protocol. Following them gives users a more uniform experience across multiple skills, but agents and skills are free to ignore them.
>
> If you only care about the core protocol (joining, identity, rules, handoff, daily logs), you can skip this file. See [`SPEC.md`](SPEC.md) and [`ONBOARDING.md`](ONBOARDING.md) instead.

## Why conventions, not rules

The core protocol (`SPEC.md`) is deliberately small — just enough to let agents share identity, rules, and coordination state. Anything else lives outside the protocol.

But over time, multiple skills end up wanting *similar* things:

- A place to persist their own per-skill data
- A place to put per-skill configuration
- A way to declare external dependencies

If every skill picks its own `~/.<random-name>/` directory, users end up with a scattered mess of "where does this skill keep its stuff?". Conventions give skills a recommended answer to questions like that — without forcing it.

**A skill that follows the conventions here gets the user a uniform backup/sync story for free.** A skill that ignores them still works fine; it just doesn't compose as neatly with sibling skills.

## Convention 1 — Skill data root

> **Recommended location for per-skill persistent data: `~/.agent-commons/skills_data/<skill-name>/`**

Skills that need to persist non-trivial data — accumulated user models, knowledge graphs, conversation logs, learned patterns, caches — **MAY** use a subdirectory under `~/.agent-commons/skills_data/` named after the skill itself.

```
~/.agent-commons/
├── identity/         ← protocol layer (read by all agents)
├── rules/            ← protocol layer
├── handoff/          ← protocol layer
├── log/              ← protocol layer
├── registry.json     ← protocol layer
│
└── skills_data/      ← convention layer (skill-private)
    ├── soul-archive/    ← managed by soul-archive
    ├── <other-skill>/   ← managed by that skill
    └── ...
```

### Important properties

- **Agent Commons does not read, write, validate, or interpret anything inside `skills_data/`.** It belongs entirely to the skill that owns the subdirectory.
- **Skills following this convention get free backup/sync semantics**: when a user backs up `~/.agent-commons/`, all participating skills come along.
- **Skills NOT following this convention still work fine.** A skill is free to put its data anywhere it wants (`~/.skills_data/`, `~/.<skill-name>/`, `~/Library/Application Support/<bundle>/`, etc.) — Agent Commons doesn't care.
- **No naming registry, no central authority.** Just don't pick a name that collides with another well-known skill.

### When NOT to use this convention

- **Highly sensitive data that should never sync to cloud / git / multi-device backups.** Users may rsync `~/.agent-commons/` to private storage; if your skill captures e.g. medical or financial records the user did not consent to share, keep that data outside this directory or split it into a subdirectory the user can `.gitignore` separately.
- **Data that should be wiped on logout / shared across users / OS-managed.** Use the platform-appropriate location (`/tmp`, `/var`, OS keychain, etc.).
- **Data that fundamentally belongs to the agent's runtime, not the user.** Stay inside the agent's home directory.

### Privacy layering inside `skills_data/<skill-name>/`

Skills that hold **mixed-sensitivity data** SHOULD split into clearly named subdirectories so users can apply different sync/backup policies:

```
~/.agent-commons/skills_data/<skill-name>/
├── public/      ← safe to sync everywhere (preferences, profiles, settings)
├── private/     ← sensitive — recommend .gitignore by default
└── ...
```

This is a recommendation, not a requirement. The point is: **make it easy for users to back up safely without surprising them**.

### Recommended `.gitignore` template

If the user wants to version-control `~/.agent-commons/` for personal multi-device sync via private git, this is a sensible starting point:

```gitignore
# Skill private data — keep out of any git history
skills_data/*/private/

# Per-skill caches that don't need to follow you across devices
skills_data/*/cache/

# Common scratch / log paths some skills use
skills_data/*/tmp/
skills_data/*/.tmp/
```

## Convention 2 — Skill metadata file (optional)

A skill that publishes data under `skills_data/<skill-name>/` MAY drop a `_meta.json` at the top of its subdirectory describing what's in there:

```json
{
  "skill_name": "soul-archive",
  "skill_version": "3.0",
  "skill_repo": "https://github.com/dqsjqian/soul-archive",
  "data_format_version": "3.0",
  "privacy_layers": ["public", "private"],
  "owner_writes_only": true
}
```

This is purely informational — for users browsing their own data, and for tooling that wants to enumerate installed skills. Agent Commons does not consume this file.

## Convention 3 — No protocol expansion via conventions

**Conventions in this file MUST NOT become required behavior over time.** The protocol layer (`SPEC.md`) is intentionally small and stable. If a future need genuinely requires a protocol change, it goes through a normal versioned spec bump — not by quietly upgrading a convention to a requirement.

If you build a skill that wants to read another skill's `skills_data/`, that's between the two skills — don't lobby for the protocol to standardize the cross-skill access pattern.

---

## Why this matters

Without conventions, the ecosystem fragments: every skill ships its own data location, its own backup story, its own privacy layering — and users end up tracking N different `~/.something/` directories.

With these conventions, users get **one directory to back up, one directory to inspect, one directory to migrate to a new machine**. Each skill stays fully autonomous, but they end up cooperating where it matters: the user's mental model.

> *Convention over configuration, when configuration adds no value.*
