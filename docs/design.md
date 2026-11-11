# decrypt-me: design

Status: early design. Prototypes have validated the encrypted store and a local model; most of the framework is not built yet.

## Vision

A personal-context workspace: a directory where an AI agent is the interface and the directory holds a structured snapshot of your life. Main purpose: helping you manage and understand yourself and your life.

- **Reflection is the core.** Understanding yourself, your patterns and your decisions.
- **Logistics is optional and personal.** Tasks, money, calendar, health. These live in modules people add if they want them.
- **Active, not only passive.** Scheduled check-ins, nightly summaries, weekly reviews. The agent writes its output to files you read later.
- **Distributable.** The framework is public. Your data never is.

The product is mostly convention: directory layout, importers, sensitivity rules, encryption setup, agent instructions and reflection protocols. There is no custom chat app. A general-purpose agent reading well-organized files does the job.

The name has two meanings: your data is encrypted at rest, and the AI helps you decrypt the complex person you are.

## Threat model

1. The hosted model provider must not retain or train on your data.
2. Nothing that could be compelled from a provider should be attributable to you.
3. Your data is genuinely encrypted at rest on your own machine.

A hosted provider account is attributable to a person (email, payment, IP). So de-identification and pseudonymization are not sufficient for sensitive content, because the sensitive fact itself is attributable. What you type into a hosted model also counts as data sent.

The working test:

> Could this text be read in discovery with my name on it, and I'd be fine with that?

If not, it never leaves the machine.

## Architecture

A local model is the reflection partner. A hosted model is only the engineer and designer.

| Role | Model | Sees |
|---|---|---|
| Reflection partner (the central feature) | Local | Everything, including raw data and candid typing |
| Engineer and designer | Hosted | Code, schemas, prompt design, synthetic examples. Never real data. |
| Optional abstract consultant | Hosted | Only questions deliberately made impersonal |

Content is sorted into Red, Yellow and Green tiers (see [sensitivity.md](sensitivity.md)). The default for anything unlabeled is Red, and Red never leaves the machine.

### Boundary enforcement

- Enforced by architecture, not by instruction. An `AGENTS.md` that says "don't read it" is a request, not a guarantee.
- The hosted agent runs in a workspace that never contains real data.
- The real data lives in an encrypted store outside the repo, reachable only by the local pipeline (see [layout.md](layout.md)).
- Anything that does cross the boundary goes through one gateway that checks tiers and logs the payload.

### Patterns considered

1. Hosted writes the pipelines, local runs them on real data. Used.
2. Local as a librarian behind an `ask_local` tool. Only for Yellow and Green content; must refuse Red.
3. Derived artifacts (local nightly summaries). Used, subject to tiers. Derived artifacts of Red data are Red.
4. Pseudonymize, then send. Rejected for sensitive material, because it is still attributable.

### Compensating for small local models

- The hosted model designs structured reflection protocols (question sequences, rubrics, check-in scripts). Small models follow scripts better than they improvise. (Hypothesis, not yet confirmed.)
- The hosted model writes pipelines and prompts using synthetic data.
- A slow weekly deep review can run on a larger model with CPU offload, since latency does not matter overnight.
- Hosted help is development-time, not runtime. The finished system runs fully local and needs no hosted account.

## Encryption at rest

A gitignored folder is not encryption. Everything sensitive is stored in an encrypted store, including derived artifacts, memory files and logs.

| Tool | Fit |
|---|---|
| `gocryptfs` | Encrypted directory via FUSE, mounted only while in use. Leading candidate for the daily-use store. |
| `age` | Simple file and archive encryption. Good for backups and exports. |
| LUKS / VeraCrypt | Block-level container. Heavier, but also hides metadata. |

Prototype findings (`gocryptfs` 2.6.1, Linux under WSL2, synthetic data only):

- Init, mount, write, unmount and remount worked; data survived.
- Contents and file and directory names are encrypted on disk. A search for known plaintext in the ciphertext directory found nothing.
- After unmount the mount point is empty. A wrong passphrase is rejected and exposes nothing.
- `-idle <duration>` auto-unmounts after inactivity, which helps re-lock the store when you walk away.
- `init` hides the master key when not run on a terminal. Setup scripts must run interactively so the user sees and records it, since it is the recovery path if the passphrase is lost.
- Metadata still leaks: directory structure, number of files, approximate sizes.
- While mounted, any process running as the same user can read the store.

Open issues:

- **Active mode versus encryption.** A scheduled job needs the store mounted. Options: run only while the store is already unlocked; cache the key in an OS keyring or hardware token (weaker); give unattended jobs a separate lower-sensitivity subset.
- **Plaintext outside the store:** swap, model-server logs and history, shell history, editor caches, temp files and captured agent output all need explicit handling.
- **Key management:** passphrase versus key file, recovery, and keeping a key backup separate from the data. A lost key means permanent data loss.
- **Legal:** whether a person can be compelled to unlock depends on jurisdiction. This is not legal advice, and encryption does not provide plausible deniability.
- Not yet tested: whether plaintext reaches swap, behavior when the host crashes while mounted, and stores on Windows-hosted paths under WSL.

## Local model prototype

Setup: an 8B-parameter model at 4-bit quantization (about 5 GB) served by Ollama, on a mid-range consumer GPU with 8 GB of VRAM, run against five synthetic journal entries with a planted pattern (see `examples/store/raw/journal/`).

Performance:

- Ran fully on the GPU at roughly 40 tokens/s generation and 300 tokens/s prompt processing.
- Cold model load took about 95 seconds, and the model unloads after about 5 idle minutes by default.
- The default context window is 4096 tokens. Real journals will exceed that, so the window must be raised, which costs VRAM. Not yet tested.

Quality (one run per task on five short entries, so anecdotal):

| Task | Result |
|---|---|
| "What do they say about family?" (not in the notes) | Correct. Said the notes do not mention it. |
| Structured JSON tagging | Valid JSON, names and dates correct, one factual error (claimed sleep was mentioned in an entry where it was not). |
| Pattern finding | Found "overcommitting" with real supporting detail. Cited the counter-example entry as support, omitted one that fit, misstated that the person did not realize the burden, and missed the sharper pattern (avoiding saying no). |
| Reflection reply to "I just need a better system" | Poor tone: accusatory, overreaching, leading question, ignored positive evidence. |

Takeaways:

- An 8B local model is good enough for extraction, tagging and honest "not in the notes" answers. It makes small factual slips, so derived artifacts should keep references to their source entries.
- With a bare prompt it is not good enough for the reflection conversation. A structured protocol fixed tone and format (see below) but not depth of insight.
- A 12–14B model, or thinking mode, might do better at pattern finding. Not yet tested.

### Check-in protocol test

[`core/protocols/checkin.md`](../core/protocols/checkin.md) is used as the system prompt. It requires tentative language, cited dates, one counter-example, no diagnostic labels, and exactly one open question. Compared against a bare "be honest, be direct" prompt on the same synthetic entries and the same user message ("I think I just need a better system to manage my workload"), 3 runs each, same model and settings.

| Measure (mean of 3 runs) | Bare prompt | Protocol |
|---|---|---|
| Words | 41 | 75 |
| Dated entries cited | 0 | 3 |
| Questions asked | 1 | 1 |
| Directives ("you should", "when are you going to") | 0 | 0 |
| Labels from the banned list | 1 | 0 |

The bare prompt's counts understate its problems. The banned-word check missed "burning out", and replies still said "you're drowning" and "when will you finally say no without guilt?", which is accusatory.

What the protocol fixed:

- Tone. Replies became tentative ("I notice", "I wonder if") and stopped labeling or accusing.
- Grounding. Replies cited real dates, and the cited facts checked out against the entries.
- Format. All replies stayed under 120 words with a single question.

What it did not fix:

- **The counter-example slot was misused in all 3 runs.** The real exception is the entry where the person told a friend the truth and felt relief. Replies instead offered an entry about a walk home, or about no longer opening a habit tracker, neither of which is an exception to saying yes too often.
- **The pattern stayed shallow.** Replies named "agreeing to more tasks" and did not reach the avoidance-of-saying-no framing or the tried-and-dropped planner and tracker.
- **Low variance.** Two of the three protocol replies were nearly identical, so three runs shows consistency more than robustness.

Caveats: the automatic checks measure rules the protocol itself states, so a pass shows the model follows explicit instructions, not that the reflection is insightful. Insight was judged by reading the outputs. One synthetic persona, five entries, one model.

### Two-pass pipeline test

Idea: pass 1 has the local model tag each entry separately into structured fields; pass 2 runs the check-in protocol over the tags instead of the raw entries. Success criterion, fixed before running: the reply uses the entry where the person told a friend the truth and felt lighter (2026-02-14) as its counter-example. 5 replies per condition.

| Pass 1 schema | Replies using the right counter-example |
|---|---|
| Purpose-built, with a boolean `told_someone_the_truth_about_limits` | 4/5 by the date check (5/5 reading the replies; one wrote "February 14th" instead of the ISO date the protocol asks for) |
| Generic: actions, people, feelings, outcome | 0/5 |

The purpose-built schema was written knowing what pattern was planted, so the first result mostly shows that a tag aimed at the answer gets used. The generic schema is the fair test of a framework that cannot know what to look for, and it failed.

Where it failed:

- **Extraction was not the bottleneck.** The generic tags for 2026-02-14 did record "Told Mira the truth about skipping her dinner" and "Lighter afterward". Pass 2 had the information and did not use it. (Extraction was imperfect, though: the generic tags for 2026-02-03 omitted that the person agreed to take on another project, and the purpose-built tags misfiled some items and missed a commitment on 2026-01-13. The one boolean that mattered was right on all five entries.)
- **Selection and framing were the bottleneck.** With generic tags the replies accepted the person's framing ("What would a better system look like for you?"), which is the sycophantic move, and chose weak exceptions (a project shipping, a walk home).
- **Format slips.** One reply asked two questions, and one omitted dates. Replies 2 to 4 were near-identical.

Conclusion: a two-pass design does not by itself make an 8B model find the meaningful contrast. It helps only when a tag happens to target the relevant dimension, and a general framework cannot assume that. Per-entry tagging remains useful as a derived artifact (searchable, cited, cheap), but it is not the reflection mechanism.

### Contrast-step test

Idea: add a generic step before the reflection. Call A asks which single entry differs most from the others (no hint about what differs). Call B writes the protocol reply using that entry as its exception. Success criterion, fixed before running: call A names 2026-02-14.

Without thinking mode, 5 runs per condition:

| Condition | A picked 2026-02-14 | Reply used 2026-02-14 |
|---|---|---|
| Raw entries, chronological (02-14 last) | 0/5 | 0/5 |
| Generic tags, chronological | 0/5 | 0/5 |
| Raw entries, shuffled (02-14 first) | 0/5 | 3/5 |

Call A chose by emotional extreme, not by behavioral contrast: it picked the most positive entry (a walk home) or the most negative one. The 3/5 in the shuffled condition is not call A succeeding. A picked 01-24 every time, and 02-14 happened to be first in that list.

With thinking mode on (the model reasons before answering), call A only:

| Order | Picked 2026-02-14 |
|---|---|
| Chronological (02-14 last) | 1/5 |
| Shuffled (02-14 first) | 5/5 |

That gap suggested a position effect, so a control moved 02-14 through all five positions with the rest in date order, 3 runs each:

| Position of 02-14 | Hits |
|---|---|
| 1 | 3/3 |
| 2 | 1/3 |
| 3 | 0/3 |
| 4 | 2/3 |
| 5 | 1/3 |
| Total | 7/15 |

Picking at random among five entries would give 20% (3/15). Thinking mode is somewhat above chance, but outside position 1 it scores 4/12 (33%), and the result depends heavily on where the entry sits in the prompt. Small samples, one model. Thinking replies also cost 13 to 111 seconds each (mostly 15 to 45), versus about 5 seconds without.

A second problem showed up in the reflection replies: citations can be wrong. In 3 of the 5 raw chronological replies without thinking, the model placed the honest conversation with the friend on 2026-02-03 instead of 2026-02-14, and one reply claimed the person was "acknowledging your limits" on 2026-02-03, which that entry does not say. Asking for dates makes the model produce plausible but wrong ones. Any pipeline that cites entries needs a programmatic check that each cited date exists and that the claim matches that entry.

Conclusion: an 8B model cannot reliably find the meaningful contrast across entries, with or without a contrast step, and thinking mode only helps unreliably. Do not build the reflection layer on an assumption that it will.

### Larger model test (14B)

Same family one size up: `qwen3:14b` at 4-bit. It loads at 10.2 GB, of which 6.6 GB (64%) fits in 8 GB of VRAM and the rest spills to system RAM. Generation runs at about 8 tokens/s, versus about 40 for the 8B. Thinking mode was off.

Contrast step (call A only), 02-14 moved through all five positions, 3 runs each, on the first synthetic person:

| Model | Picked 2026-02-14 |
|---|---|
| 8B, thinking off | 0/15 |
| 8B, thinking on | 7/15 |
| 14B, thinking off | 15/15, every position |

That looked like a solution, so two checks followed.

**Full pipeline with the right exception supplied.** Five protocol replies from the 14B all cited only real dates, and every claim matched its entry. The 8B misdated the key event in 3 of 5 replies. The weakness left: all five replies ended with a question that suggests an answer ("What might happen if you tried saying 'I can't' more often?"), which the protocol forbids, and the replies were near-identical.

**A second synthetic person.** Six entries about a person who never asks for help, with the exception on 2026-03-29 (asked a neighbor to help carry a couch and felt good). Two decoys: 2026-03-09 is intensely negative (cried, felt like failing) and 2026-03-15 is intensely positive (won a raffle, great mood). Each position of the target, 3 runs each:

| Model | Picked 2026-03-29 | What it picked instead |
|---|---|---|
| 14B | 3/18 (chance) | 2026-03-15 in 12 runs, 2026-03-09 in 3 runs. Right only when the target came first in the list. |
| 8B | 6/18 | 2026-04-05 in 12 runs. Right only when the target was in position 5 or 6. |

Interpretation: the 14B's 15/15 on the first person was an artifact of that dataset. There the exception was also the most positive entry, and the model picks emotional extremes. On the second person it picked the raffle. The 8B is driven by position instead. Neither finds a behavioral contrast reliably.

What a larger model does improve: citation accuracy and the quality of the writing once the right exception is supplied.

Caveats: the call A wording allows a mood-based reading ("or how they felt afterward"), and a behavior-only wording is untested. Both people were written by the same author, six entries or fewer, one model family. Thinking mode on the 14B was not run, since it would take minutes per answer at 8 tokens/s.

Conclusion: model size does not fix contrast finding. The exception has to come from somewhere other than an unguided LLM comparison: a behavior-focused prompt (untested), a non-LLM method that nominates candidates, or the user.

### Model server privacy notes

- The server's default logs did not contain the test prompt text.
- Desktop chat front ends may store conversation history in plaintext outside the encrypted store. Reflection should go through a path that does not.
- Model files themselves are not sensitive, but a model server's storage and logs sit outside the encrypted store.

## Open questions

- Which harness can drive a local model endpoint with reliable tool use?
- How should the local model server be reached from the agent environment (endpoint design, binding, firewall)?
- How to reconcile active mode with encryption.
- Which encryption tool or tools to standardize on, and how to handle key recovery.
- How the meaningful contrast should be chosen if an unguided model cannot find it: a behavior-only prompt, non-LLM nomination (for example embedding-based outlier detection, which would likely also favor emotional extremes and is untested), the user picking from candidates, or a mix.

## Next steps

1. Draft an `AGENTS.md` for the local reflection agent.
2. Test a behavior-only contrast prompt on both synthetic people, with the 8B and the 14B.
3. Save both synthetic people as repeatable fixtures with the expected exception for each.
4. Add a programmatic citation check to any pipeline that cites entries.
5. Turn the encrypted-store prototype into `init-store`, `mount` and `unmount` scripts.
6. Write the pre-commit hook that refuses commits touching personal-data paths.
7. Design the active-mode scheduler, key handling and the outbound log.
