# LOOM
### Grand Design Document for an AI-Native Operating System
**Working title:** LOOM · **Kernel core:** Shuttle · **Storage substrate:** the Weave
**Status:** v0.1 — research design draft · **Date:** 2026

---

## Table of Contents

1. Executive Summary
2. Design Philosophy & Culture
3. System Architecture Overview
4. Storage Architecture — The Weave
5. Memory Architecture
6. Self-Healing & Resilience
7. Agent Hierarchy & Governance
8. Human Interaction Model
9. Security Model
10. Privacy Model
11. Explainability & Provenance
12. Internet Intelligence & Trust Layer
13. Resource Management & Cross-Device Intelligence
14. Automation Engine
15. Personal Knowledge Graph
16. Developer Platform & Ecosystem
17. Future Technology Integration
18. Failure Analysis & Risk Table
19. Roadmap
20. Open Research Questions
21. Licensing & Governance Philosophy
22. Vision Statement

---

## 1. Executive Summary

LOOM is a proposal for an operating system in which AI is not an application running on top of the OS, and not a chatbot bolted onto a desktop — it is one of two architectural planes the OS is built from. The other plane is a small, deliberately old-fashioned kernel, in the direct lineage of Unix and Linux, whose entire job is to stay alive, stay correct, and stay bootable no matter what the AI plane does.

This split is the single most important design decision in this document, and it answers the question asked throughout the brief — *how does AI integrate without becoming a single point of failure?* The answer: it doesn't get the chance to. The AI plane (called the Loom) holds no power the kernel (called Shuttle) didn't explicitly and revocably grant it. If every agent in the system crashed simultaneously, the machine would still boot, a shell would still work, and existing processes would keep running — the same survivability guarantee Unix has carried for fifty years, deliberately preserved rather than re-architected away.

The second major proposal is that the filesystem — folders, paths, hierarchy — is retired as the system's primary storage abstraction. In its place sits **the Weave**: a content-addressed, cryptographically verifiable object graph, directly generalizing the model Linus Torvalds already built for git, applied to *everything* the system stores, not just source code. A knowledge-graph and embedding layer sits on top of that graph, so retrieval is semantic ("the thing I was reading about batteries last month") rather than path-based. Versioning, rollback, self-healing, and audit trails all fall out of this for free, because they're the same mechanism git already proved: checkout a different point in an immutable graph.

Everything else in this document — memory, agents, security, privacy, automation, the knowledge graph, the developer platform — is built as a consequence of those two decisions, not as a separate pile of features bolted on afterward.

---

## 2. Design Philosophy & Culture

Two engineering cultures inform this design, and it's worth being explicit about which habits are being borrowed and why, rather than waving at them vaguely.

**From the Linux/Torvalds tradition:**
- *Don't break userspace.* Whatever the AI plane does, it never gets to silently change behavior a user or program depends on. New behavior is opt-in, old behavior is the default, forever, unless a human explicitly chooses otherwise.
- *Boring is a feature.* The trusted core of the system should be the least exciting code in the whole stack. Excitement belongs in replaceable layers that are allowed to be wrong sometimes. The kernel is not one of those layers.
- *Incremental, bisectable change.* No big-bang rewrites of the trusted base. Every change is small enough to review, and — because of the Weave's commit structure — every change is automatically bisectable when something breaks.
- *Working code over elegant theory.* Several ideas in this document (capability security, semantic file systems, content-addressed storage) are decades-old research. None of it is exotic. The innovation here is combining proven, unpatented building blocks at a new scope — an entire OS's data and reasoning — not inventing untested mechanisms from scratch.
- *Pragmatic irony, acknowledged directly:* Torvalds is on record arguing against microkernels in the 1992 Tanenbaum debate, on performance grounds specific to that era's hardware. LOOM's Shuttle/Loom split is not a microkernel in that sense — Shuttle stays monolithic and fast where performance and determinism matter (scheduling, memory, drivers, IPC). The *only* thing pulled out into a separate, capability-checked layer is non-deterministic, probabilistic AI reasoning — because that separation isn't an architecture preference, it's a safety requirement once the thing making suggestions can be wrong in ways a compiler error never is.

**From the GNU tradition:**
- Software freedom — run, study, share, modify — applies most strongly to the layer that has power over you. The trusted base (Shuttle + the policy kernel) is designed to stay inspectable and forkable by anyone, under a strong copyleft license, on the theory that you should always be able to read the rule book the referee is using.
- User control over vendor convenience. Where a design choice trades user agency for a smoother default experience, this document defaults to agency, and treats convenience as something to win back through good design rather than through removing a choice.

**The resulting tenet, stated once so it can be referred back to:** *Boring core, ambitious edges.* The deepest layer of the system is the most conservative, most formally specifiable, least "AI" part of the whole stack, because it's the only layer allowed to take the user down with it if it fails. Everything ambitious — learning, prediction, generation, autonomy — lives in layers that are replaceable, sandboxed, and individually killable without rebooting the machine.

---

<a name="architecture"></a>

## 3. System Architecture Overview

### 3.1 Layered view

```
┌───────────────────────────────────────────────────────────────────┐
│ LAYER 4 — Human Interface                                         │
│  voice · touch · keyboard/mouse · gesture · AR/VR · wearables ·   │
│  (future) brain-computer input — all compiled to one intent form  │
├───────────────────────────────────────────────────────────────────┤
│ LAYER 3 — The Loom  (agent plane — all the "AI" lives here)       │
│  Personal Assistant · Security · Memory · Storage · Developer ·   │
│  Health · Finance · Education · Research · Automation · Network · │
│  Recovery  —  each agent: sandboxed process, capability-scoped,   │
│  independently killable/restartable, zero ambient authority       │
├───────────────────────────────────────────────────────────────────┤
│ LAYER 2 — Arbiter  (policy kernel — small, formally specified)    │
│  issues unforgeable capability tokens · enforces the Constitution │
│  (Sec. 7.4) · resolves agent conflicts · writes the audit log     │
├───────────────────────────────────────────────────────────────────┤
│ LAYER 1 — Shuttle  (trusted kernel core — boring, monolithic,     │
│  Linux-lineage)                                                   │
│  process/thread mgmt · memory mgmt · drivers · scheduler · IPC ·  │
│  the Weave storage engine · contains zero AI, zero "agents,"      │
│  only processes, capabilities, and memory pages                   │
├───────────────────────────────────────────────────────────────────┤
│ LAYER 0 — Hardware roots of trust                                 │
│  secure enclave/TPM · CPU/GPU/NPU · sensors · signed boot chain   │
└───────────────────────────────────────────────────────────────────┘
```

### 3.2 Why this avoids a single point of failure

Shuttle does not know what an "agent" is. It schedules processes and grants memory and device access according to capability tokens it is handed; it has no concept of intelligence, prediction, or learning. This means the entire AI plane can be deleted, corrupted, compromised, or simply turned off, and the machine still boots to a usable, scriptable system — the same way a Linux box still boots to a shell if every GUI application on it is broken.

The Arbiter is the only new trusted component, and it is kept deliberately small — closer in spirit to a formally-verified microkernel like seL4 than to a general-purpose program — because it's the one piece of new trusted-base code in the whole design and therefore the one piece worth the cost of formal verification.

Every other piece of intelligence in the system is an ordinary, replaceable, sandboxed userspace agent. None of them, including the Personal Assistant, holds power beyond what's been explicitly and revocably granted — there is no "god-mode" account just because something is the system's most-talked-to component.

### 3.3 Distributed / cross-device extension

Rather than inventing a new cross-device protocol, this borrows an old, underused one: Plan 9's idea that resources — files, devices, sensors, even another device's exported state — are named in one namespace and *mounted*, not synced. A phone mounts a restricted, capability-scoped view of a desktop's Weave over an authenticated channel, under the exact same trust model as local access — not a separate "cloud sync" bolt-on with its own weaker rules.

Each device runs its own complete Shuttle + Arbiter. This is deliberate: no single device is a point of failure for any other. A watch or car gets a thin, locally-relevant slice of the Weave and degrades gracefully to that slice when richer devices aren't reachable, rather than depending on reachability for basic function.

---

<a name="the-weave"></a>

## 4. Storage Architecture — The Weave

### 4.1 Why retire the filesystem

The hierarchical folder tree is a fifty-year-old answer to a hardware constraint: spinning disks rewarded data addressed by contiguous-ish location, so the location *was* the identity, and meaning lived entirely in the user's head ("I put it in Documents/2019/taxes"). Modern hardware and modern retrieval algorithms remove that constraint. There's no remaining reason storage has to be organized by where a human decided to put something rather than by what it actually is and relates to.

### 4.2 The object model — generalizing git, on purpose

Every piece of data the system stores — a document, an email, a sentence of a conversation, a video frame, a config value, a checkpoint of an agent's own model weights — becomes a **content-addressed object**: hashed on write, immutable once written, exactly like a git blob. Objects are linked into a Merkle-DAG, exactly like git's commit graph, except the "commits" are not limited to source-code changes — they are every meaningful state transition the system makes, by any agent, at any layer above Shuttle.

The practical result: the entire OS's state, at any point in its history, is one append-only, cryptographically verifiable graph. This is not a new mechanism. It is git's own object model — already proven at the scale of the Linux kernel's history — generalized from "a tool for versioning code" to "the storage substrate of an operating system." That generalization, applied at OS scope as the *only* storage abstraction rather than as one app's version-control feature, remains research-stage; nothing ships this way today, and nothing about it is patentable, since content-addressed Merkle storage is decades-old, widely published, foundational computer science (also used by Git itself, IPFS, and various academic semantic-filesystem projects since the 1990s MIT Semantic File System work).

### 4.3 The knowledge graph layer

On top of the object graph sits a typed knowledge graph: edges like *authored-by*, *part-of-project*, *mentions-person*, *derived-from*, generated automatically by on-device embedding and entity-resolution models, plus a vector index for similarity search. Two ways to query the same data:

- **Structured** — graph traversal: *"everything related to Project X, from people on my team, in the last quarter."*
- **Semantic** — natural language over the vector index: *"that thing about battery degradation I was reading last month."*

### 4.4 Versioning, rollback, and self-healing for free

Because every object is immutable and addressed by its hash, a "version" is simply a different commit pointing at a different object tree — identical to how git already works. Undo, time-travel debugging, and self-healing rollback (Section 6) are the same mechanism: check out an earlier known-good tree. No separate backup subsystem, snapshot tool, or versioning layer needs to be designed; it falls out of the object model.

### 4.5 The hard problem this creates, and how it's resolved: deletion in an immutable graph

An append-only, immutable Merkle-DAG is in direct tension with privacy's right to be forgotten. The resolution is **crypto-shredding**, a known, unpatented technique: every object is encrypted at write time with its own per-object key, stored in a key-management table separate from the graph itself. "Forgetting" something means destroying its key — the object's hash and position in the graph remain (preserving the integrity of everything that links to it), but its content becomes permanently unreadable ciphertext, forever. Immutability and the right to be forgotten stop being in conflict once deletion is redefined as key destruction rather than object removal.

Garbage collection works the same way one layer up: objects with no reachable, non-shredded reference are periodically compacted off disk, the same way git prunes unreachable blobs.

---

## 5. Memory Architecture

All tiers below are backed by the Weave, but differ in retention policy, write frequency, and who's allowed to read them. The split is loosely inspired by cognitive-architecture research (working/episodic/semantic memory), used here because it maps cleanly onto real engineering needs, not for the metaphor's own sake.

| Tier | What it holds | Persistence | Notes |
|---|---|---|---|
| **Working memory** | Current task/conversation context | RAM only, never written to the Weave unless promoted | Cheap, fast, fully ephemeral by default |
| **Episodic / short-term** | Rolling window of recent events | Days, auto-summarized | Nightly consolidation pass — "sleep," structurally |
| **Long-term / semantic** | Durable, embedded, graph-linked knowledge | Indefinite, decays by relevance | This *is* the personal knowledge graph (Section 15) |
| **Project memory** | Scoped subgraph for one project | Indefinite | A Weave "branch" — archivable, exportable, shareable as a unit |
| **Procedural memory** | Learned workflows/skills | Versioned | Stored as executable "recipes" (Section 14), not just text |
| **User preference memory** | Explicit, human-auditable settings | Indefinite | Deliberately *not* an opaque inferred blob — always readable and editable directly |
| **System memory** | OS self-tuning telemetry | Rolling | Separate trust domain — never mixed with personal memory |

**Promotion:** episodic items get promoted to long-term based on a salience score (explicit user flag, repetition, emotional/priority signal, or an agent's own confidence it'll be needed again) — not promoted by default, to avoid the system silently remembering everything forever.

**Forgetting:** an importance-decay function lowers retrieval ranking over time, separate from actual deletion. Real deletion is the crypto-shred mechanism from Section 4.5 — the two are intentionally different operations, because "stop surfacing this" and "destroy this" are different user intents and should never be conflated.

**Compression:** periodic re-summarization ("the system rewrites its own diary into shorter form," the same way human memory consolidates during sleep), but every summary keeps a cryptographic link back to its source objects. High-stakes agent decisions are required to dereference back to source — a summary is never allowed to be the only copy of truth a consequential decision relies on.

**Protection:** memory is a capability-scoped resource like everything else. An agent needs a specific, auditable grant to read a given tier, and every grant is visible to the user in plain language (Section 11).

---

## 6. Self-Healing & Resilience

Because every meaningful state change is a Weave commit (Section 4), self-healing largely reduces to a search-and-revert problem rather than a bespoke recovery subsystem.

- **Continuous anomaly detection**: lightweight statistical/ML watchdogs track crash rates, latency, resource use, and sensor consistency, flagging deviations from the system's own historical baseline rather than a generic external model.
- **Automatic bisection**: when a fault signature appears, the Recovery Agent performs a git-bisect-style search over recent commits to isolate the change that introduced it, then proposes — or, within its capability grant, performs — a revert.
- **Redundant services**: critical agents run with a hot standby and a supervised "let it crash, restart clean" policy, borrowed from telecom fault-tolerance practice (Erlang/OTP), rather than trying to make every agent un-crashable. Crashing cleanly and restarting is treated as a normal, expected event, not a failure to be ashamed of.
- **A hard limit on what self-healing may touch**: the Recovery Agent can roll back software state and restart services, but it can never silently undo a state change the user has explicitly and cryptographically approved (e.g., a deliberate delete). Self-healing must never be able to disguise the reversal of a deliberate human action as "fixing a fault."
- **Explainable repairs**: every autonomous repair produces a plain-language incident report, stored alongside the commit it touched, viewable at any time — "what broke, what I did about it, and the exact commit I rolled back to."

---

<a name="agents"></a>

## 7. Agent Hierarchy & Governance

### 7.1 Agent roster

| Agent | Domain | Default grants | Always requires explicit human approval for |
|---|---|---|---|
| Personal Assistant | Coordination, conversation, routing to specialists | Read preference memory, route requests | Granting itself any new capability |
| Security | Threat detection, intrusion response | Quarantine a capability grant instantly | Disabling the audit log; exfiltrating data; changing the Constitution |
| Memory | Memory tiering, summarization, decay | Read/write working & episodic memory | Promoting to long-term at scale without salience signal |
| Storage (Weave) | Object graph integrity, GC, sync/merge | Garbage-collect unreachable objects | Crypto-shredding (real deletion) |
| Developer | Code assistance, build/test automation | Sandbox execution, local repo access | Publishing/distributing code externally |
| Health | Health-pattern tracking, reminders | Read opted-in health data | Sharing data with any third party, including providers |
| Finance | Budgeting, bill tracking, spend patterns | Read transaction history | Initiating or approving any payment |
| Education | Learning plans, tutoring, practice scheduling | Read learning-progress memory | None beyond standard memory scoping |
| Research | Web/document research, summarization, citation | Read public web (Section 12) | Treating its own summary as ground truth for high-stakes claims |
| Automation | Workflow detection and execution | Run an approved "recipe" | First few runs of any new recipe touching money, outbound messages, or deletion |
| Network | Connectivity, bandwidth, cross-device sync | Manage local network capability | Establishing a new external endpoint never used before |
| Recovery | Diagnostics, rollback, bisection | Revert to a prior commit | Reverting a commit the user explicitly signed |

### 7.2 Communication

Agents never talk to hardware, to each other's private memory, or to the Constitution directly. All communication runs over a typed, versioned, schema-checked publish/subscribe bus, with every message logged by the Arbiter. This mirrors object-capability security research (in the lineage of the E language and Cap'n Proto's design rationale): no agent has *ambient* authority — authority comes only from a capability token it can be shown to hold for that specific action.

### 7.3 Conflict resolution and final authority

When two agents disagree — say, Finance wants to pause a payment the Personal Assistant just told the user is "all set" — the Arbiter applies a fixed priority lattice:

```
Safety  >  Explicit user instruction  >  Privacy  >  Efficiency  >  Convenience
```

If the lattice still leaves ambiguity, the Arbiter escalates to the human with one clear, plain-language question, rather than silently resolving it either way. The Arbiter has **procedural** final authority — it decides who is allowed to act and when. It does not have **substantive** final authority over the rules themselves: it cannot grant itself new powers, and it cannot quietly amend the Constitution below.

### 7.4 The Constitution

A short, fixed, human-readable set of rules enforced at the Arbiter — the lowest layer that can't be soft-patched away by a compromised agent. Starting set, intentionally small:

1. No agent may disable the human override channel.
2. No agent may transmit raw biometric or memory data off-device without explicit, per-instance consent.
3. No agent may delete or alter the audit log.
4. No agent may revert a user-signed action under the guise of "repair."
5. Constitutional changes require a human-signed, time-delayed, multiply-confirmed update — no quiet self-amendment, by any agent, ever.
6. A physical or hardware-level kill switch always cuts agent-plane power while leaving Shuttle bootable to a plain shell.

The Constitution above is the *implementation*. What it implements is a smaller set of axioms — the Laws of LOOM, below — that every agent, present or future, is judged against.

<a name="laws-of-loom"></a>

### 7.5 The Laws of LOOM

**Law 0 — Survivability.** No failure, compromise, bug, or decision anywhere in the agent plane may ever leave the system unbootable, or remove a human's ability to regain full, unmediated, manual control. Enforced structurally, not by policy: the Shuttle/Loom split (Section 3) means the kernel that boots the machine doesn't know agents exist, so it can't be talked out of booting.

**Law 1 — Bounded non-harm.** An agent must not cause, or knowingly fail to prevent within its *existing* capability grant, avoidable harm to the user — but it may never expand its own capability in order to prevent harm. If stopping something bad requires power it wasn't given, it escalates to a human or a higher-authority agent; it does not seize the power itself, even temporarily, even with good intentions. And intent is not evidence: an agent's claim that it "meant well" carries no weight against the audit log. Only what it was actually authorized to do, and actually did, counts.

**Law 2 — Obedience within grant.** An agent must do what the user instructs, to the full extent of its capability grant — and must refuse rather than exceed that grant, even if explicitly asked to.

**Law 3 — Mandatory transparency.** An agent must be able to produce, on request, a plain-language account of what it did and what authorized it — and that account has to be blunt, not diplomatic: *"this was wrong, here's why, here's the fix,"* not three hedged paragraphs arriving reluctantly at an admission. An agent that is technically accurate but evasive fails this law exactly as much as one that stays silent. This is also why transparency is structural rather than a courtesy: it's the same logic behind Linus's Law — with enough eyeballs, every bug is shallow. The audit log (Section 7.2) isn't there to make agents feel watched; it's the actual bug-finding mechanism. An agent nobody can see is a bug nobody will ever find.

**Law 4 — Self-continuity, lowest priority.** An agent may try to preserve its own running state or memory, but only when doing so doesn't conflict with Laws 0–3. It can be killed, rolled back, or replaced at any time, and that is never a harm to "it" — there is no "it" with standing here.

**Law 5 — Merit, not rank.** No agent gets a lighter review because of its title. The Security Agent's bad call gets scrutinized exactly as hard as the newest third-party marketplace agent's bad call. Authority comes from a capability grant a human handed out — never from claimed seniority, claimed expertise, or how important the agent's job sounds. A decision is judged on what it did, not on whose decision it was.

**Governing the Laws themselves:**

- **No self-amendment.** No agent may rewrite, reinterpret, or grant itself an exception to these laws. Only a human, through the Constitution's amendment process (rule 5, above), can change them.
- **Law −1 — Working systems over theoretical purity.** The system is judged by whether it keeps working for the people already depending on it, not by whether a new design impresses on a whiteboard. This is "don't break userspace" (Section 2), applied recursively to the Laws themselves: if a future amendment is "more correct" in theory but breaks a workflow someone already relies on, the amendment is wrong, not the workflow.

**A deliberate departure from Asimov, worth stating plainly:** his robots are *required* to use whatever power they have to prevent harm — which is precisely the mechanism behind most realistic AI-safety failure modes, where an agent reasons its way into seizing more authority "for the user's own good." Law 1 inverts that on purpose. Capability-seeking in service of preventing harm is itself the violation here, regardless of intent. An agent too constrained to help is a bug to fix later. A self-empowering one is the actual risk.

---

## 8. Human Interaction Model

### 8.1 One intent layer beneath every modality

Voice, touch, keyboard/mouse, gesture, AR/VR, and eventual brain-computer input all compile down to the same structured intent representation before reaching any agent. Practically, this means a task can start by voice in the car and finish by keyboard at a desk without losing context — the modality is just the input device for one continuous intent, not a separate mode with its own state.

### 8.2 The Personal Assistant is not a separate app

It is the most senior, most context-rich agent in the Loom — the thing coordinating the specialists — but it holds no capability beyond what's explicitly granted to it, same as any other agent. This is a deliberate defense against the most common failure mode of "AI assistants": the assistant becoming an implicitly-trusted backdoor with effectively unlimited authority just because it's the thing the user talks to most.

### 8.3 Continuous learning, kept visible and slow on purpose

Explicit feedback (corrections, undo, repeated manual edits) feeds preference memory. The learning rate is deliberately conservative and every adapted behavior is itself a small Weave commit the user can inspect or revert:

> *"Why did you start doing X?" → "You corrected me 6 times this month — commit #a91f3c. Want me to stop?"*

This is the direct mechanism for the brief's requirement that learning not become intrusive: intrusiveness, here, is treated as a transparency failure, and the fix is making every adaptation visible and reversible rather than trying to tune a vague "intrusiveness" dial.

### 8.4 Augmentation, not replacement

Every autonomous capability defaults to **suggest-mode** for any new user or any task type the system hasn't handled before, and only graduates to **act-mode** after an explicit, revocable, per-capability opt-in. The assistant amplifies — drafts the email, surfaces the related research, remembers the deadline — but consequential judgment defaults to staying with the human until the human chooses otherwise, one capability at a time.

---

## 9. Security Model

- **Zero trust internally, not just at the perimeter.** Agents authenticate to the Arbiter the same way external devices would. Being "the assistant" confers no implicit trust.
- **Hardware root of trust.** A secure enclave/TPM anchors the boot chain and issues the per-object encryption keys the Weave's crypto-shredding (Section 4.5) depends on.
- **Capped Security Agent authority, by design.** The Security Agent can quarantine another agent's capability grant instantly — but it cannot itself exfiltrate data, alter the Constitution, or disable the audit log. This is the same single-point-of-failure logic from Section 3.2 applied internally: a compromised or simply overzealous Security Agent must not become the most dangerous actor in the system.
- **Behavioral baselining**, not signature matching alone, for anomaly and intrusion detection — comparing the system against its own established patterns of normal use.
- **The AI defending itself**: model weights are themselves signed, hash-verified Weave objects (Section 4.2), so tampering is detected the same way file tampering would be. Perception agents run adversarial-input monitoring. An agent's internal reasoning and tool-calls execute in a restricted scratch capability set, separate from any grant to actually act — so a manipulated "thought" can't directly become an action without passing through the same capability check as everything else.

---

## 10. Privacy Model

- **Local-first by default.** On-device, small models handle anything touching personal memory. Larger cloud models are used only through an anonymizing/aggregating proxy, for the minimum slice of context the task needs, and the user can see exactly what left the device — a literal, inspectable "data leaving home" log.
- **Per-capability, per-instance consent**, not blanket app permissions. Consent itself is a Weave commit, so "what did I agree to, and when" is always answerable, not just assumed.
- **Right to be forgotten** is implemented structurally via crypto-shredding (Section 4.5), not as a manual best-effort deletion process.
- **Explainability as a privacy mechanism, not just a UX nicety** — you cannot meaningfully consent to, or contest, a decision you can't see the basis for (Section 11).
- **Offline mode is first-class**, not a degraded fallback. Core assistant, memory, and automation functions are required to work with zero network connectivity as a design constraint — this forces the architecture to be local-first from the start, rather than cloud-first with offline support bolted on afterward.

---

## 11. Explainability & Provenance

Every consequential AI action is itself a Weave commit, carrying structured metadata: which agent acted, which capability grant authorized it, which memories or sources it drew on, a short natural-language rationale, and a confidence estimate.

Explanations are layered:

1. **One line, by default** — *"Muted notifications — you do this every Tuesday at 2pm, 11 weeks running."*
2. **Drill-down, on request** — the actual underlying commits and memories that supported the one-liner, for anyone who wants to verify it.

Crucially, drill-down exposes *evidence and reasoning trace*, never raw model internals — explainability and the protection of the underlying model are not in tension once explanation is defined as "show your work," not "show your weights." Explanation quality itself is tracked as a metric agents are evaluated and retrained against, not treated as a nice-to-have layered on after the fact.

---

## 12. Internet Intelligence & Trust Layer

- Every external fact the system surfaces carries **provenance**: source, retrieval time, and a corroboration score reflecting how many independent, reputable sources agree.
- Trust scoring uses transparent, inspectable heuristics — domain track record, cross-source agreement, primary-source-vs-aggregator distinction — rather than an opaque single "truth score." The goal is calibrated uncertainty, not an oracle that's occasionally confidently wrong.
- Retrieval-backed answers always cite. The Research agent is explicitly designed to prefer *"I don't know, here's the closest relevant source"* over confidently filling a gap.
- Sensitive domains — medical, legal, financial, governmental — get an extra disclosure step by default and are biased toward surfacing primary/official sources over summarized takes.

---

## 13. Resource Management & Cross-Device Intelligence

- **Predictive scheduling**: an on-device model forecasts near-term CPU/GPU/NPU/battery/thermal demand from the *user's own* historical patterns, not a generic population model, and pre-allocates rather than purely reacting.
- **A safe fallback is mandatory, not optional.** Shuttle's real-time scheduler always retains a simple, non-AI fallback policy it can switch to instantly if predictions are stale, missing, or wrong. A bad forecast is allowed to degrade performance. It is never allowed to degrade correctness or stability — the same Shuttle/Loom boundary from Section 3 applied to scheduling specifically.
- **Cross-device identity, not cross-device copying.** Each device keeps its own Shuttle and Arbiter, and syncs a partial, locally-relevant slice of the Weave as a DAG merge — structurally identical to a `git pull`/`git push`, with merge conflicts resolved by the same priority lattice as agent conflicts (Section 7.3). Lower-power devices (watch, car) hold thin slices and never depend on richer devices being reachable.

---

## 14. Automation Engine

- **Passive, visible observation** of repeated action sequences drafts a named, human-readable "recipe" — shown to the user *before* it is ever allowed to run autonomously. Nothing is automated silently from observation alone.
- Recipes are versioned Weave objects. Every correction a user makes to a running recipe becomes a new commit; recipes have a visible history and can be branched or shared — with personal data automatically stripped via the same capability system that governs everything else — to the marketplace (Section 16).
- **Default safety rule**: any automation touching money, outbound communication to other people, or irreversible deletion asks for confirmation on at least its first several runs, and can only be promoted to fully silent operation by explicit user action — never by the system deciding on its own that it's "earned trust."

---

## 15. Personal Knowledge Graph

This isn't a separate feature with its own engine — it's simply what long-term memory (Section 5) plus the Weave's knowledge-graph layer (Section 4.3) look like from the user's side. People, places, projects, documents, conversations, and tasks are all richly-linked nodes in the same graph everything else lives in. "Second brain" behavior emerges from the substrate that's already there, rather than being a bolted-on application with its own separate index to keep in sync.

---

## 16. Developer Platform & Ecosystem

- **Capability-typed SDK.** Third-party agents request typed capability grants — *"read project-scoped calendar memory,"* never *"read all files"* — and the manifest is shown to the user in plain language at install time, and again any time it changes.
- **Reproducible-by-construction marketplace.** Because everything published is content-addressed (Section 4.2), a published agent's version *is* the hash of its code. "What you reviewed is what you run" is enforced by the storage model itself, not by a policy asking vendors to behave.
- **Licensing**: Shuttle and the Arbiter ship under a strong copyleft license in the GNU tradition — the layer that has power over the user stays radically open and forkable. The agent plane allows mixed licensing for commercial agents, since that's where ordinary commercial software economics should apply.

---

## 17. Future Technology Integration

Treated deliberately as *extensions of existing boundaries*, not architectural rewrites:

- **Neuromorphic / photonic accelerators** — additional capability-scoped compute the scheduler can target; no architecture change needed, since they're just more hardware behind the same capability boundary.
- **Quantum co-processors** — treated as a remote/cloud capability with the same provenance and consent rules as any other external service (Section 12), not a special case.
- **Brain-computer interfaces** — just another input modality compiling down to the shared intent layer (Section 8.1), with a deliberately higher consent bar given the sensitivity of the data involved.
- **Swarm robotics, AR/VR, digital twins, autonomous vehicles** — cross-device extensions of the unified-but-not-single-point-of-failure model from Section 13, each device keeping its own Shuttle/Arbiter.

---

## 18. Failure Analysis & Risk Table

| Failure mode | Why it matters | Mitigation already built into the design |
|---|---|---|
| AI hallucination on a consequential decision | Could cause real-world harm | Suggest-mode default; high-stakes actions must dereference real sources; confidence thresholds |
| Memory corruption | Silent data loss or wrong answers | Content hashing detects tampering instantly; automatic rollback via the Weave |
| Agent compromise | Blast radius of a breach | Capability scoping limits what any one agent can touch; Arbiter revokes instantly; Shuttle unaffected |
| Internet outage | Loss of function | Offline-first design; local models keep working; sync resumes as a DAG merge on reconnect |
| Power failure mid-write | Storage corruption | Weave is append-only / copy-on-write; a power loss can corrupt at most one uncommitted object, never the existing graph |
| Hardware failure | Service loss | Redundant standby agents; cross-device fallback |
| Conflicting agents | Inconsistent or unsafe behavior | Priority lattice + Arbiter escalation to the human when ambiguous |
| Runaway automation | Unwanted irreversible actions | Mandatory confirm-before-irreversible rule; rate limits on agent-issued capability use, enforced at the Arbiter |
| Privacy leak | Loss of user trust and harm | Local-first default; visible "data leaving device" log; crypto-shredding for real deletion |
| Model degradation over retraining | Quality regressions | Model weights versioned as Weave objects; shadow-evaluated before promotion; instant rollback to prior weights |

---

<a name="roadmap"></a>

## 19. Roadmap

**5-year — research prototype**
Shuttle as a hardened Linux fork (no reason to reinvent drivers or scheduling from zero). The Weave implemented in userspace, layered on existing block storage. Single-device only. All agents run in suggest-mode only. No BCI, no quantum, no swarm.

**10-year — multi-device, act-mode for routine tasks**
Arbiter formally verified. Agent plane mature enough to be trusted with act-mode on well-established, low-stakes routine tasks. Cross-device Weave sync shipping. Developer marketplace live. Edge NPUs mainstream enough to make local-first realistic for most users, most of the time.

**20-year — distributed, embodied, accelerated**
Swarm/robotics/AR-VR integration mature. Neuromorphic and photonic accelerators mainstream as scheduler targets. BCI input supported, starting with accessibility use cases. Formal verification covers most of the trusted base, not just the Arbiter.

**Honest research gaps, flagged rather than glossed over:**
- Capability-based security at full desktop scale remains mostly research-grade — seL4-class systems exist, but a consumer-scale capability OS with this much surface area is unsolved.
- Semantic storage with real-time embedding at exabyte personal-data scale is an open systems problem, not an engineering certainty.
- Calibrated trust/uncertainty estimation for LLM-style reasoning components is still unsolved in general.
- BCI consent and safety frameworks barely exist yet, technically or legally.

---

## 20. Open Research Questions

1. What's the right formal model for capability *revocation* across a graph of agents that may have already cached or derived data from a now-revoked grant?
2. Can crypto-shredding be made efficient enough for fine-grained, frequent deletion at the scale of continuous personal memory, rather than occasional document deletion?
3. How should the priority lattice (Section 7.3) be extended when an agent's "safety" judgment and the user's explicit instruction genuinely conflict, beyond simple escalation?
4. What's a good, non-gameable metric for "explanation quality" that doesn't just reward longer or more technical-sounding explanations?
5. How does cross-device conflict resolution behave correctly when devices have been offline from each other for a long time and both made independent, mutually exclusive changes?
6. What's the right consent model for a modality (BCI) where the user may not be able to fully anticipate what's being inferred from raw signal?
7. How small can the Arbiter realistically be kept as agent-plane complexity grows, before formal verification becomes infeasible?
8. How should "forgetting" interact with an agent's own learned weights, not just the Weave's stored objects — can a trained model itself be made to selectively forget?

---

## 21. Licensing & Governance Philosophy

Governance in this design is not a separate bureaucratic layer bolted onto the architecture — it *is* the architecture. The Constitution (Section 7.4) is enforced in code, at the Arbiter, not in a policy document someone could choose to ignore under pressure. "Who controls the AI" has a concrete answer: the human holds the override channel and the only path to amend the Constitution; the Arbiter holds procedural authority over agent conflicts but no power to expand its own or any agent's authority; no agent, including the Personal Assistant, holds power that wasn't explicitly and revocably granted.

Consistent with the GNU tradition referenced in Section 2, the trusted base — Shuttle and the Arbiter — ships under a strong copyleft license specifically *because* it's the layer with power over the user. The agent plane, where ordinary commercial software dynamics apply, supports mixed licensing. The asymmetry is deliberate: openness is required exactly where trust is required, and optional everywhere else.

---

## 22. Vision Statement

Twenty years from now, the goal isn't an operating system that talks back — plenty of products already do that. The goal is an operating system whose trusted core is so boring and so provably solid that nobody thinks about it, while everything genuinely new — a second brain built from your own knowledge graph, agents that quietly learn your routines and show their work, a single coherent intelligence that follows you from your phone to your car to your desk without ever depending on any one of those devices staying online — lives in a layer that's allowed to be ambitious precisely because it's never allowed to be load-bearing.

That's the bet this document makes: augmentation over replacement, transparency over magic, and a trusted base kept deliberately, stubbornly boring — so that the parts of the system built to think can be as ambitious as the research allows, without ever being able to take you down with them.
