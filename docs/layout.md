# Directory layout

Two separate places: the **repo** (public framework, no personal data) and the **store** (encrypted personal data, outside the repo).

None of the directories below exist yet except `docs/`. This is the target layout.

## Why the store lives outside the repo

The sensitivity rules say hosted agent sessions must never be able to read real data. The simplest guarantee is physical: the repo does not contain the store, and a hosted session is run in the repo only. The store is mounted only for local sessions, and only at a path outside the repo.

The repo `.gitignore` still ignores `data/`, `store/`, `*.age`, `*.enc` and `.env` as a backstop, so a misplaced file cannot be committed by accident.

## Repo (public)

```
decrypt-me/
  README.md
  AGENTS.md                rules for hosted engineering sessions (Green only, synthetic data)
  docs/
    design.md              design, threat model, prototype findings
    sensitivity.md         tier rubric
    layout.md              this file
  agents/
    local/AGENTS.md        instructions for the local reflection agent
  core/
    protocols/             structured reflection scripts (check-ins, weekly review, ...)
    prompts/               prompt templates used by pipelines
    schemas/               formats for store files and config
  modules/                 optional, one directory per module
    finance/  calendar/  tasks/  health/
  importers/               source-specific import scripts (raw export -> store/raw/)
  gateway/                 the single path to hosted models: tier check + outbound log
  scripts/
    init-store             create encrypted store, tiers.toml, directory skeleton
    mount / unmount        decrypt/encrypt the store around a session
    hooks/pre-commit       refuse commits touching personal-data paths
  examples/
    store/                 fully synthetic store, used for development and hosted sessions
  .gitignore
```

Everything in the repo is Green.

## Store (encrypted, outside the repo)

Two locations on disk:

| Path (default) | Contents |
|---|---|
| `~/.decrypt-me/cipher/` | Ciphertext. This is what exists at rest. |
| `~/decrypt-me-store/` | Mount point. Plaintext is visible here only while mounted. |

Layout inside the mounted store:

```
decrypt-me-store/
  config.toml              model endpoint, enabled modules, active-mode settings
  tiers.toml               path-glob -> tier rules (default: everything Red)
  raw/                     imported source material, left untouched
    journal/  messages/  health/  finance/  calendar/
  notes/                   things the user writes directly
  derived/                 local-model output: summaries, themes, timelines
  memory/                  agent memory across sessions
    profile.md             long-lived picture of the user
    threads/               ongoing topics
    decisions/             decisions and the reasoning at the time
  reflections/             weekly/monthly reviews, check-in results
  inbox/                   active-mode output waiting for the user to read
  outbound/                log of every payload sent to a hosted model, with approval
```

Default tier of every top-level directory is Red. Only `outbound/` content is, by construction, text that was already approved to leave, but the log itself stays in the store and is treated as Red because it records the user's activity.

Everything the agent writes, including memory, derived files and logs, goes inside the store so it is encrypted along with the rest.

## Session flow

1. `mount` prompts for the key and decrypts the store to the mount point.
2. Local model and local agent run with the store as their working data.
3. Optional: the gateway forwards approved Yellow/Green payloads to a hosted model and logs them.
4. `unmount` re-locks the store. Nothing sensitive remains on disk in plaintext.

Hosted engineering sessions skip steps 1 and 2 entirely and use `examples/store/`.

## Active mode and encryption (unresolved)

Scheduled jobs need the store mounted. Options still open: run only while the user has the store unlocked; cache the key in an OS keyring or hardware token (weaker); give unattended jobs a separate lower-sensitivity subset. See `docs/design.md`.

## Plaintext leaks outside the store

The store being encrypted does not cover: swap, the model server's logs and request history, shell history, editor caches, temp files and the agent harness's captured output. These need explicit handling in `scripts/mount` and in setup docs.
