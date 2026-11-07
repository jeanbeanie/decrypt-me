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
- It is not good enough for the reflection conversation with a bare prompt. Quality there depends on protocol and prompt design. Whether protocols fix the tone is not yet tested.
- A 12–14B model, or thinking mode, might do better at pattern finding. Not yet tested.

Privacy notes for the model server:

- The server's default logs did not contain the test prompt text.
- Desktop chat front ends may store conversation history in plaintext outside the encrypted store. Reflection should go through a path that does not.
- Model files themselves are not sensitive, but a model server's storage and logs sit outside the encrypted store.

## Open questions

- Which harness can drive a local model endpoint with reliable tool use?
- How should the local model server be reached from the agent environment (endpoint design, binding, firewall)?
- How to reconcile active mode with encryption.
- Which encryption tool or tools to standardize on, and how to handle key recovery.
- Whether structured protocols make a small local model a good reflection partner.

## Next steps

1. Draft an `AGENTS.md` for the local reflection agent.
2. Write one structured check-in protocol and test whether it fixes the reflection tone.
3. Turn the encrypted-store prototype into `init-store`, `mount` and `unmount` scripts.
4. Write the pre-commit hook that refuses commits touching personal-data paths.
5. Design the active-mode scheduler, key handling and the outbound log.
