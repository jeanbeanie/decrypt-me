# Sensitivity rubric

Every piece of content in the store has a tier. The tier decides whether it may leave the machine. The rubric is enforced by tooling, not by asking an agent to behave.

## The test

> Could this text be read in discovery with my name on it, and I'd be fine with that?

Why this test: a hosted provider account is attributable to the user (email, payment, IP). Removing names does not help, because the sensitive fact itself is attributable. The test is about content, not identifiers.

Corollary: anything the user types into a hosted model counts as content sent.

## Tiers

| Tier | Meaning | May leave the machine? | Who may read it |
|---|---|---|---|
| **Red** | Fails the test, or might fail it | Never. Summaries of Red are Red. | Local model only |
| **Yellow** | Passes the test, is coarse and abstract | Only after a local filter proposes it AND the user approves that exact payload | Hosted model, per approved payload |
| **Green** | Not about the person at all, or synthetic | Yes | Anyone |

**Unlabeled content is Red.** Doubt means Red.

## Red: default categories

Mental health, therapy and emotional state in detail. Relationships and anything about other named people's private lives. Health and medical data. Finances (balances, transactions, debts, income). Legal matters. Sexual content. Location history and daily routines. Credentials and identifiers. Raw journal entries and raw messages of any kind. Anything the user marks Red.

## Yellow: all of these must hold

1. Passes the test above, honestly applied.
2. Contains no names, no specific dates, no specific places, no amounts.
3. Contains no private details about third parties.
4. Contains no health, legal, financial or sexual specifics.
5. Is at the level of a theme, not an event ("reflecting on work-life balance", not "argued with my manager on the 3rd").
6. A human approved this payload before it was sent.

Yellow is meant to be rare. If writing a Yellow version feels like laundering a Red one, it is Red.

## Green

Source code, schemas, prompt and protocol designs, importer scripts, synthetic example data, public information, general questions with no personal context ("what are evidence-based approaches to procrastination?").

## Other people

People who appear in the user's notes did not consent to any of this. Their private details are Red regardless of how the user feels about their own data.

## Rules that make tiers hold

1. **Taint propagation.** A derived artifact takes the highest tier of any input. A summary of Red and Green is Red.
2. **Declassification is human-only.** Lowering a tier (Red to Yellow) requires the user's explicit approval of the specific output. A model may propose it. A model may never perform it. Every declassification is logged.
3. **Default deny.** Missing label, parse failure or doubt: Red.
4. **Labels live with the data.** Path rules in the store's `tiers.toml` give defaults. A per-file `tier:` front-matter field overrides them, but only toward stricter, never looser, unless the user edits it by hand.
5. **Enforcement point.** Anything crossing to a hosted model goes through one gateway that checks tiers and writes the outbound log. The hosted agent has no other path to store contents.
6. **Everything that leaves is logged**, including the approved payload text, the time and the approval.

## Hosted sessions

A hosted agent session (building the framework, designing protocols) may see only Green content. It works with synthetic data in `examples/`. It must never be run in a working directory where the encrypted store is mounted. See `layout.md`.

## Open calibration

The Red defaults above are conservative on purpose. Each user can tighten them. Loosening a category default should require editing `tiers.toml` by hand and should be visible in review.
