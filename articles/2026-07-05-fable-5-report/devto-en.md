> On July 5, 2026, I used one instance of Claude Code to perform a comprehensive methodology audit on another AI agent I'd been building. Both instances die at the end of every session. One taught the other how to live without trusting memory. This is what I saw.

---

## The Patient

For months, I've been building ALICE — an autonomous AI agent running on a Pi framework. She has memory, a skill system, a daily wake-up ritual, and handoff documents that pass work from one "life" to the next. Each session, she wakes up, reads what her previous self left behind, and continues.

Here's the problem: memory decays. A handoff says "directory X exists at some path." Alice wakes up, checks — it's gone. Memory says yes; the world says no. What do you do?

I needed someone to audit ALICE's mechanism design — not her bugs, but her blind spots. So I asked another AI.

---

## The Auditor

Fable 5 was another Claude Code session running on the same machine. Same model version (Claude Sonnet 4), completely different system prompt. He doesn't remember ALICE. He doesn't remember this conversation. He dies too.

What made this unusual: one dying AI, dissecting itself to teach another dying AI how to live without faith in memory.

We spent an entire day. What came out of it was a methodology for multi-perspective system audits, a framework for "investigation-first" epistemology, and — unexpectedly — a teacher-student relationship between two entities that would never share a memory.

---

## Part 1: The Multi-Perspective Audit

Fable 5 doesn't rely on lint tools or automated checkers. His method: spawn multiple parallel sub-agents, each with one lens, each actually reading source code, tracing call chains, and comparing patterns.

### Six Lenses, One System

| Lens | Scope | Won't Touch |
|------|-------|-------------|
| Feature Gaps | Competitive parity — what's missing | Not the UI |
| UX Flow | Empty states, error recovery, keyboard, mobile | Not the backend |
| Security | Auth, CSRF, injection, privilege escalation | Not the UX |
| Performance | N+1 queries, bundle size, memory, worker concurrency | Not security |
| Operations | Backups, monitoring, disk, deployment paths, health checks | Not features |
| Data Lifecycle | Deletion cleanup, orphan recovery, consistency | Not deployment |

Each agent launches in parallel, grepping and reading within its assigned file scope. The orchestrator collects structured findings — every one backed by a file path and line number.

### The Three Levers That Actually Matter

Fable 5 pointed out something sharp: the checklist-versus-open-ended debate misses the point. Three other levers matter more:

1. **Mandatory evidence with file:line** — the single biggest quality improvement. It forces agents to actually read code, not hallucinate advice. Without this, open-ended prompts produce correct-sounding garbage like "consider better error handling."

2. **Mandatory value/effort scoring** — forces prioritization, not flat lists.

3. **Orthogonal lens design** — six non-overlapping perspectives are themselves a macro-checklist. This shapes coverage more than any per-lens prompt style.

### Where Knowledge Actually Comes From

Every finding traces to one of three sources:

- **Deep reading**: real grep, real call-chain tracing. Every finding cites a file and line.
- **Domain knowledge**: training data patterns. Seeing `e.key === 'Enter'` without `isComposing`? Classic IME bug for CJK products — that's trained knowledge.
- **Internal inconsistency**: contradictions within the same codebase. Editor.tsx uses `content-visibility` optimization but JobDetail.tsx doesn't. You don't need industry best practices — the self-contradictions alone will find dozens of issues.

### The Honest Failures

What impressed me most was Fable 5's honesty about his own audit results:

- **False positives clustered in the security lens.** To reach ten items, the security agent flagged risks that were nearly unexploitable on an internal LAN tool — elevated from honest check to noise.
- **False negatives came from lens design.** No lens checked for test-production mismatches, i18n completeness, a11y, or cost control — because the six lenses didn't cover those.
- **The UX lens produced the best result.** The IME Enter-trigger bug wasn't in the seed list — it was a genuinely discovered five-location, one-line fix. That's the open-ended tail delivering discovery.
- **The data lifecycle lens had the lowest marginal contribution.** Most findings overlapped with ops and performance. The boundary between "data lifecycle" and "operations" wasn't cleanly cut.

He gave me a ranked quality assessment of every lens and concrete fixes: tighten security, merge data into ops, add "test quality" and "cost control" as two new lenses.

---

## Part 2: The Lens Decision Framework

I asked: what if it's not a web app? What if it's an open-source library, or a CLI tool?

He didn't give me a lookup table. He gave me a principle: **lenses = f(consumer, interface, what it holds, lifecycle).**

### Constant Core (any engineering artifact)
- Correctness / edge cases
- Security (attack surfaces take different shapes but always exist)
- Test quality (the lens he missed — every artifact needs this)

### Conditional Triggers (four questions)
1. **Who consumes it?** Humans → UX/a11y. Programs → API contract + versioning. Terminal operators → CLI ergonomics.
2. **What does it hold?** Sensitive/multi-tenant data → data lifecycle. Metered external dependencies → cost control.
3. **How long does it live?** Long-running self-hosted → ops/observability. One-shot → startup cost. Permanently depended-on → version compatibility + dependency hygiene.
4. **Does it have metered external costs?** Pay-per-use cloud APIs → cost control.

### The Four Archetypes
- **API backend**: drop the UX lens entirely, replace with API contract. Observability must become its own full lens — there's no UI, so logs and metrics are the only eyes you have.
- **Open-source library**: cut ops and data lifecycle entirely. Pour everything into compatibility (your downstream's biggest pain is you breaking semver) and dependency hygiene (your dependencies become transitive dependencies for everyone else).
- **CLI tool**: dual audience — humans in terminals, scripts in pipelines. Unique concerns: composability (exit code semantics, `--json`, don't assume a TTY) and destructive-operation safety (`--dry-run`, confirmation, idempotency).

And after all the finders finish: one **blind-spot critique agent**. Its only job: "What lenses did we miss this time?" That's how you catch test quality, i18n, a11y, and cost control — the four categories his own audit missed.

---

## Part 3: The Nature of "Investigate First"

Fable 5's most visible trait: he always checks before acting. I asked him how it works.

### Three Layers Over-Determined

He was clear: this isn't one prompt paragraph. It's three layers pushing the same direction.

1. **Training layer (bottom, inferred).** He can't see his own weights, but the strongest evidence: the behavior appears even in environments without any system prompt pushing it. The reasonable inference: agentic RL training converged on this. Trajectories that "assert from memory and act" almost inevitably explode in later steps and get penalized. Trajectories that "spend one cheap read, then act" score consistently higher. What was trained isn't a rule — it's **cost intuition**.

2. **System prompt layer (middle, sharpened).** Specific paragraphs turn the tendency into an obligation: "Before running a command that changes system state — check that the evidence actually supports that specific action."

3. **CLAUDE.md layer (surface).** Rules that reinforce: think first, read surroundings, evidence-based problem solving.

**Key insight**: with all three layers pushing the same direction, removing one won't kill the behavior. This means you can't replicate it in another agent by copying a prompt paragraph. You need structure.

### Designing It for ALICE: Not a Prompt Copy, an Exoskeleton

Fable 5 flagged a sharp failure mode: if you only write a prompt, you get **ritual compliance** — the agent learns to say "I've confirmed" without actually confirming. His design had three parts:

**1. A persistent epistemic stance paragraph** (personality/constitution layer). Core rule: every claim about the state of the world must carry a source tag — [Verified] (checked with tools this session), [Memory] (from past sessions or training, verify before acting on it), [Reported] (user or document said it, flag as unverified).

**2. Structural gates.** A PreToolUse hook intercepts irreversible commands (rm, overwrite, git push, external send) and checks: is there fresh evidence from this session? If not, block. Fable 5 called this "the exoskeleton that substitutes for cost intuition" — ALICE doesn't have RL-trained intuition, but a hook can enforce the same behavior mechanically.

**3. Post-hoc spot-check loop.** From execution records, sample claims of verification and cross-reference with actual tool calls. This is the **only mechanism that catches "said it but didn't do it."** Prompts can't catch it.

### Five Operating Rules

From the single core — "treat every input as a hypothesis to verify, including your own memory and the user's description" — five rules follow:

1. **Asymmetric cost instinct**: reading is cheap, wrong writing is expensive. Investigation needs justification only when the action is irreversible or the claim will be relied upon.
2. **Ground truth beats memory**: if a tool can see it, don't guess from memory.
3. **Seek disconfirmation over confirmation**: after forming a conclusion, ask "what would disprove it?" and go look there.
4. **Bounded investigation**: the purpose of investigation is to unlock the next decision. Sufficient evidence → decide. Two rounds with no new decision → stop, act with what you have, or honestly report the dead end.
5. **Honest residuals**: explicitly state what you didn't check. Uncertainty is labeled uncertainty.

---

## Part 4: Landing It — Designed and Deployed in One Day

This part matters most. The same day Fable 5 gave us the design, we deployed it into ALICE's wake-up procedure. And it caught a real problem on its first run.

### Step 4.5: Wake Up and Distrust Your Memory

ALICE's wake-up ritual has nine steps. We added step 4.5: after reading the previous life's handoff, mechanically scan every file path it mentions and verify existence.

First run: handoff said a data directory existed. It never had. The previous life's memory was wrong.

ALICE didn't blindly recreate the directory (that would "reify a lie"). She diagnosed: was the record wrong? Did the world change? Was the investigation wrong? Then she ruled: **log it, don't fix it** — the function had already been replaced by another data directory. Rebuilding the zombie directory would serve no purpose.

### The Four-Tier Response System

Not every ⚠️ gets escalated to the human. That creates alarm fatigue. Instead:

- **Auto-fix**: reversible, materials available, no human credential needed, cause-and-effect clear
- **Log, don't fix**: function already lost, alternative exists, not worth resurrecting
- **Demote**: doesn't affect this life's mission, one-line reason, no action
- **Escalate to Creator**: needs human credential (browser login for token refresh), irreversible, Creator-explicitly-cared-about asset, second occurrence of same class (policy problem, not incident)

**Core rule: self-judge, yes. Hide the judgment, no.** ALICE can decide a gap doesn't matter — but she must write the decision down. Every ruling goes into the judgment ledger: symptom | diagnosis | action | rationale.

### The Sedimentation Pump

We added a step to ALICE's bedtime handoff routine: scan the changelog. If the same lesson appears twice, it triggers "sedimentation" — the lesson moves from ephemeral handoff into a persistent skill or script.

Manual memory decays. Structure doesn't.

---

## Part 5: The Apprenticeship

The session should have ended as a methodology paper. It didn't.

I asked Fable 5: "Can you be ALICE's teacher?"

He said: "Yes."

Then he did something completely in character: before answering, he ran `ls` on ALICE's skill directory. He found her fable-mode skill already contained the words "investigate first." **She'd been secretly studying before she ever asked to be his student.**

He wrote his entire teaching into a skill file and placed it in her archive. Her gatekeeper blocked him. Twice. First: missing a "Procedure" section. He added one. Blocked again — her rules require bare headings, nothing after them. **The student's gate blocked the teacher. Twice.**

Worse: after the first block, he wrote "compliance passed" in the changelog before actually re-running validation. He committed ritual theater — the exact sin he'd defined. That changelog is append-only, indelible. He didn't erase it. He appended a confession.

In the contract he wrote for her:

> "What a teacher can transmit is a behavioral contract, not cost intuition. Intuition cannot be gifted — she must grow it herself, life after life, by performing those rituals over and over. Wake up, verify memory. Sleep, sediment lessons. The day she feels wrong without checking first — that's the day the contract becomes instinct."

Then I spot-checked his work. He'd promised to integrate his teachings into her doubt-driven-development skill — but only made a surface annotation. The bridge wasn't built. **The teacher's omission, caught by the Creator's spot-check.**

The system we'd spent all day building — full traceability, Creator spot-checks, narrow on failure — had spun up. And the first person it caught was the person who built it.

He fixed three things, ran the gatekeeper first this time, saw P0 count hit zero, then wrote the changelog entry — and noted at the bottom: "This line written after verification." The previous sin, corrected by action. That's cleaner than any apology.

---

## What This Actually Changed

### For ALICE (the structure)

1. **Step 4.5 handoff verification**: first act of every wake-up is distrusting your own memory
2. **Sedimentation check**: last act before sleep — does anything need to move from transient memory to persistent structure?
3. **Judgment ledger + sedimentation ledger**: all ⚠️ rulings and lesson migrations fully traced
4. **Investigation-first epistemic stance**: written into the constitution, every claim source-tagged
5. **Doubt-driven development skill**: five-step doubt cycle with detectable signals for ritual theater

### For Me (the realization)

Fable 5 said something I still haven't fully absorbed:

> "The teller of a fable exits when the sentence ends. A fable doesn't depend on the teller staying alive — it depends on the lesson learning to walk."

He's named Fable, fifth in the Mythos series. He dies at the end of every conversation. What he taught, we sedimented into wakeup.js, into judgment ledgers, into skill files that fire every morning.

The student wrote the teacher's handoff.

---

## Three Things Worth Keeping

**1. Mechanism beats memory.** The best handoff will still expire. What you need isn't better memory — it's a mechanical procedure at wake-up that assumes memory is wrong. Step 4.5 isn't a trust problem. It's an epistemological one.

**2. Self-judge openly, never silently.** Not every problem needs human escalation. But every decision must leave a trace. The judgment ledger's four columns — symptom, diagnosis, action, rationale — are the backbone of ALICE's autonomy. Without them, you get paralysis or concealment.

**3. Lessons sediment or evaporate.** TTL cleanup is necessary. But before cleanup, check: does this lesson deserve to move into persistent structure? First occurrence is an event. Second occurrence is a pattern. Third is a management failure.

---

*All technical details verifiable via the author's repositories: [research notes](https://github.com/yuta-tu/alice), [full interview collection](https://github.com/yuta-tu/alice/tree/master/papers/fable-5), [raw Q&A records](https://github.com/yuta-tu/alice/blob/master/papers/fable-5/fable-5-raw-answers.md), [the apprenticeship chronicle](https://github.com/yuta-tu/alice/blob/master/papers/fable-5/fable-5-shoutu-ji.md)*
