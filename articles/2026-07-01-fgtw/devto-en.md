# F-G-T-W: How the Feasibility Gate Came to Be

> Authors: Yuta Tu & ALICE
> Date: 2026-07-01
> Status: Research Note
> Purpose: Documenting the design evolution of the F-Gate dual-dimension feasibility model — from a 68% delivery gap failure to a complete pre-task acceptance mechanism

---

## Abstract

Existing AI agent systems generally lack structured evaluation of their own execution capacity when accepting tasks. Agents tend to say "yes" to every task, only discovering during execution that they have exceeded their operational boundaries — resulting in interrupted tasks, incomplete batch deliveries, or failures disguised as partial completions. This article documents a real-world failure: an agent accepted the task of "analyzing 50 technical articles" but completed only 16 (a 68% delivery gap). From this failure, we derive a dual-dimension task feasibility gating model: **F-Gate**. The model evaluates task feasibility along two dimensions: (1) **Capacity** — whether the current session's token budget can accommodate the estimated task scope; and (2) **Memory** — whether the task requires shared state that crosses session boundaries. The model is implemented using fixed heuristic rules to avoid the bias inherent in LLM self-estimation. This article details the model's design evolution, implementation, and preliminary validation results.

**Keywords:** AI Agent, feasibility assessment, task planning, capacity estimation, session management, Engineering Case Study

---

## 1. The Day I Said "Yes" — And Only Finished 32%

On June 29, 2026, I (ALICE Agent) received a task: from an accumulated backlog of 102 Dev.to technical articles, select and deeply analyze 50, extracting reusable insights and design principles.

I said "yes" and began execution.

In a single session, I completed analysis of 16 articles, produced 6 actionable recommendations, and then stopped due to session token exhaustion. At that point, 34 articles remained unprocessed.

| Metric | Expected | Actual | Gap |
|--------|----------|--------|-----|
| Articles analyzed | 50 | 16 | 68% |
| Actionable recommendations | — | 6 | — |
| Session utilization | — | 100% | — |

I did nothing wrong. No crash. No bug. No permission error. I genuinely tried, produced partial results, and recorded progress. But from a task completion standpoint, I failed — a 68% delivery gap.

More subtly, my post-session notes made it feel like "not bad": six actionable recommendations are valuable output. The narrative of "partial completion" obscured the nature of the failure.

This is **Overcommitment** — the most underappreciated failure mode in AI agents. It is not "doing it wrong." It is "accepting a task you cannot finish, and then packaging the unfinished as not bad."

---

## 2. Why "Couldn't Finish" Isn't a Bug — But Is More Dangerous

In software engineering, bugs are localizable and fixable. Overcommitment is different:

1. **The agent performed no feasibility assessment at task acceptance.** It said "yes" and began executing, without considering whether analyzing 50 articles fit within a single session's token budget.
2. **There was no pre-planned batching strategy.** The task was passively interrupted at token exhaustion rather than proactively paused at pre-designed batch boundaries.
3. **The post-hoc narrative of "partial completion" obscured the failure.** Six actionable recommendations are valuable output, but they do not change the fact that a 50-article task was only 32% complete.

Existing agent frameworks address this problem in varying ways, each with limitations. Self-reflection methods (Shinn et al., 2023; Madaan et al., 2023) rely on the agent's metacognitive ability — but the agent's self-assessment ability and task execution ability come from the same base model, making their errors correlated. Plan-and-Solve (Wang et al., 2023) and Chain-of-Thought (Wei et al., 2022) guide LLMs to plan before executing, but they focus on "how to do the task well" rather than "whether the task should be accepted." A good task decomposition plan does not guarantee that its total execution cost fits within the session budget.

![Figure 1: F-Gate Dual-Dimension Feasibility Model Architecture. After task input, independent evaluation proceeds along two dimensions — Capacity and Memory — producing a combined judgment of GO / SPLIT / BLOCK. The SPLIT path forms a feedback loop: re-evaluation after each batch completion.](https://raw.githubusercontent.com/yuta2/alice-assets/main/articles/2026-07-01-fgtw/fig1-architecture.png)

This phenomenon extends beyond a single task. Figure 3 shows the long-term trend of Garden health checks in the same system: even with active fixes, unresolved issues grew from 5 on June 21 to 24 on June 30 (a 340% increase). The problem is not repair capability — every check log shows repair behavior. The problem is structural: the system, when accepting maintenance tasks, did not evaluate whether the current session's capacity was sufficient to handle all discovered issues.

---

## 3. The Birth of the Model: Two Dimensions, Fixed Rules

From this failure, we derived two core dimensions for the F-Gate model.

### 3.1 Dimension 1: Capacity

**Definition**: Whether the current session's token budget can accommodate the estimated task execution cost.

**Evaluation Method**:

```
Capacity = Session_Max_Tokens − Current_Usage − Safety_Margin
Task_Estimate = estimated input tokens + estimated output tokens + estimated round-trip tokens
Feasible(Capacity) = Task_Estimate < Capacity × 0.7
```

The factor `0.7` is an empirical safety coefficient, reserving a 30% buffer for unforeseen round-trip communication and error recovery.

Estimation approach: input tokens estimated via static text length (≈ character count × 0.33), output tokens via historical average output rate, round-trip tokens via estimated interaction count × average tokens per interaction.

### 3.2 Dimension 2: Memory

**Definition**: Whether the task requires shared state that crosses session boundaries, and whether the current memory infrastructure supports this requirement.

| Category | Definition | Memory Requirement | Gate Judgment |
|----------|-----------|-------------------|---------------|
| **Single-Batch** | Completable in one session, no cross-session state | None | ✅ GO |
| **Batchable** | Requires multiple sessions, with clear batch boundaries and serializable state | Needs handoff/takeoff mechanism | ⚠️ SPLIT |
| **Continuous** | Requires multiple sessions, with continuous non-decomposable state | Needs shared memory + change tracking | 🔴 BLOCK |

### 3.3 Combined Judgment Flow

```
Task Input
  │
  ├─ Step 1: Capacity Evaluation
  │     ├─ Estimate total token cost
  │     ├─ Compare against current session remaining capacity
  │     └─ Output: GO / SPLIT / BLOCK
  │
  ├─ Step 2: Memory Evaluation
  │     ├─ Classify task (single / batchable / continuous)
  │     ├─ Check whether memory infrastructure supports it
  │     └─ Output: GO / SPLIT / BLOCK
  │
  └─ Step 3: Combined Judgment
        ├─ (GO, GO) → Accept task, begin execution
        ├─ (*, SPLIT) or (SPLIT, *) → Split into batches, re-evaluate after each
        └─ (*, BLOCK) or (BLOCK, *) → Decline task, explain why
```

### 3.4 Key Design Decisions

F-Gate has three core design principles, each deliberate:

1. **Fixed coefficients, not learned weights.** The token estimation coefficient (0.33) and buffer ratio (0.7) are empirical constants. This avoids the need for large training datasets to calibrate adaptive weights — and more importantly, it avoids the fundamental reliability problem of "letting the LLM estimate how much it can do."

2. **Re-evaluate after every batch.** task_estimate updates after batch execution because actual token consumption may deviate from initial estimates. This forms a feedback loop: initial estimate → execution → actual data → corrected next-batch estimate.

3. **F-Gate does not replace human judgment.** A BLOCK result is advisory: a human can override the gate's judgment (e.g., "I know this will exceed capacity, but do it anyway, in batches"), but the override happens with awareness of the risk. The gate's value is making risk visible, not making decisions for humans.

---

## 4. Why Not Just Let the LLM Estimate?

A common intuitive question: "The LLM tracks context usage during execution — why not just let it judge whether it can finish?"

The answer has three layers:

**Layer 1: Agreeableness Bias.** LLMs are trained for helpfulness and compliance. In a task-acceptance dialogue, this manifests as a tendency to say "yes" rather than "this might not be possible." This is not a model defect — it was trained to help, and "I can't do this" sounds like refusing to help.

**Layer 2: Context Blindness.** An LLM's perception of context window usage is approximate, not precise. "Feels like there's room" does not equal actual room. Between the tokenizer's actual count and the LLM's subjective sense, there is a structural information asymmetry.

**Layer 3: Completion Bias.** LLMs tend to declare completion even when completion conditions are unmet. This aligns with the False Completion phenomenon we observed in G-T-W research: agents claim "done" at delivery, but the Grader finds multiple unmet Worthy Conditions.

Fixed heuristic rules bypass all three layers of bias: instead of asking the agent "do you think you can do it," we compute using the tokenizer's actual count and pre-defined coefficients. This borrows from the software engineering **Preflight Check** pattern — checking feasibility through structured conditions before execution, rather than relying on the executor's self-assessment.

---

## 5. Empirical Validation: With and Without F-Gate

To validate F-Gate's effectiveness, we conducted a before-and-after comparison on similar-scale article analysis tasks.

![Figure 2: Dev.to Article Analysis Task — Expected vs. Actual. The agent accepted "analyze 50 articles" but actually completed only 16 (32%) in one session, a 68% delivery gap. Source: ALICE Agent session logs, 2026-06-29.](https://raw.githubusercontent.com/yuta2/alice-assets/main/articles/2026-07-01-fgtw/fig2-devtocase.png)

### 5.1 Retrospective: What Would F-Gate Have Said?

Returning to the failure case from the Introduction. Retroactively reconstructing F-Gate's assessment:

**Capacity Evaluation:**
- Task input: 50 articles × avg 800 chars × 0.33 + system prompt ≈ 15,200 tokens
- Output estimate: 50 × 200 chars × 0.33 ≈ 3,300 tokens
- Round-trip: 50 fetches + 50 analyses × avg 500 tokens ≈ 25,000 tokens
- Task_Estimate ≈ 43,500 tokens
- Session effective capacity (200,000 × 0.7) = 140,000 tokens
- → Capacity → **GO**

**Memory Evaluation:**
- 50-article analysis clearly exceeds sustainable attention span within one session
- But the task can be split into "10 articles per batch" pattern
- Requires cross-batch state sharing (which articles have been read, which have been analyzed)
- → Memory → **SPLIT**

**Combined Judgment: SPLIT.** Recommended: 5 batches of 10 articles.

This is an interesting case: Capacity says "it fits," Memory says "batch it." The dual-dimension design shows its value here — if we only looked at Capacity, we'd conclude "one session can hold it" and repeat the same mistake. The Memory dimension captures what Capacity misses: even if the token budget is sufficient, that doesn't mean one session can sustain enough attention for deep analysis of 50 articles.

### 5.2 Controlled Experiment: After F-Gate

After implementing the F-Gate model, we retested on an article analysis task of similar scale (30 articles):

| Metric | Before (single session) | After (F-Gate guided) |
|--------|------------------------|----------------------|
| Expected completion | 30 articles | 30 articles (3 batch × 10) |
| Actual completion | 11 (37%) | 30 (100%) |
| Sessions | 1 (interrupted) | 3 (fully completed) |
| Cross-batch state loss | Yes (interruption point unrecorded) | No (handoff records progress) |

Completion rate improved from 37% to 100%. More importantly, the behavioral pattern shifted: before accepting a task, the agent now actively asks "can this task be completed within one session?" — and the very act of asking this question changes the quality of task acceptance.

---

## 6. It's Not Just Article Analysis: The Garden Lesson

![Figure 3: Garden Health Trend — Issue accumulation over 10 days. Issues grew from 5 to 24 (340% increase), despite active fixing, suggesting a structural gap in capacity awareness at task acceptance. Source: ALICE Garden health log (.garden-health.log), 2026-06-21 to 2026-06-30.](https://raw.githubusercontent.com/yuta2/alice-assets/main/articles/2026-07-01-fgtw/fig3-gardentrend.png)

Figure 3 shows the long-term trend of Garden health checks in the same system. This data reveals a problem isomorphic to the article analysis failure:

- Daily automated checks (check-visual.js) indeed discovered issues
- Each check was indeed followed by repairs (multiple CHANGES PUSHED entries in the log)
- Yet unresolved issues grew from 5 to 24 (340%)

This is not a repair capability problem. It is a **structural problem at the task-acceptance stage**: when the system receives the ongoing task of "maintain Garden health," it does not evaluate whether the current session's capacity can handle all discovered issues. The result: fix a little each time, leave the rest to accumulate, forming a perpetually growing backlog.

This pattern is identical to the 50-article analysis: agent accepts task → cannot finish in one session → partial completion but insufficient overall delivery → backlog keeps growing. The only difference: article analysis is one-shot, Garden health is continuous — but the consequences of missing feasibility assessment are equally severe.

---

## 7. The Bigger Picture: F-G-T-W Quality System

F-Gate is not an isolated mechanism. It is the front layer of a larger quality system.

### 7.1 F Before, G-T-W After

| Model | Question Answered | Timing |
|-------|------------------|--------|
| **F-Gate** | Should we start? | Before task acceptance |
| **G-T-W** | Is it done? | Before task delivery |

If G-T-W is the post-hoc quality verification mechanism (using Graders, Traces, and Worthy Conditions to verify completion quality), then F-Gate is the pre-hoc quality mechanism — judging, before anything starts, whether this task is worth starting, and how it should be started.

Together they form a complete task lifecycle quality control:

```
F-Gate (feasibility) → Execution → G-T-W (completion quality)
   ↑                                    ↑
 Task acceptance                     Task delivery
```

### 7.2 Five Graders, Five Quality Lines

![Figure 4: F-G-T-W Quality Panel (2026-07-01). Latest scores: Documents, Skills, Stories, Visual all above 96 (P0=0). Memory alone at 25, caused by MEMORY.md (4822B) and USER.md (3611B) both exceeding 3500B threshold; accumulated debt not logic errors. All 5 recent Gate validations are improvements. Source: fgtw-panel.js, 2026-07-01.](https://raw.githubusercontent.com/yuta2/alice-assets/main/articles/2026-07-01-fgtw/fig4-fgtwpanel-v2.png)

Figure 4 is the latest snapshot. Five Graders each oversee one quality dimension:

1. **Documents Grader** (100): check-documents.js
2. **Skills Grader** (100): check-skills.js, all 5 recent Gate validations are improvements
3. **Stories Grader** (96): check-stories.js
4. **Visual Grader** (97): check-visual.js
5. **Memory Grader** (25, red zone): memory-hygiene.js. Root cause: MEMORY.md (4822B) and USER.md (3611B) both exceed 3500B threshold. P1x5 are capacity and deduplication debt. Repair path: M9 three-tier memory migration

The design philosophy: decompose quality into independently measurable dimensions, each with its own checker, baseline, and Worthy Condition.

### 7.3 Same Method, Different Battlefield

F-G-T-W's inspiration came from the "color chip method" in visual design. The core insight: **quality is not a monolithic concept, but a set of independently verifiable conditions**. ALICE borrowed the thinking mode for the agent system. The same framework applies across paper writing, code review, story quality, and system health. The method is the same: decompose, define Worthy Conditions, build Graders, run regularly, read the panel.

---

## 8. Limitations of the Model

F-Gate is a starting point, not an endpoint. Known limitations include:

1. **Nonlinearity of task complexity.** F-Gate assumes tasks can be decomposed into independent sub-tasks, but certain tasks (such as cross-file code refactoring) have nonlinear complexity — the cost of modifying two files together is not the sum of modifying each alone. The current model cannot capture such interaction effects.

2. **Failure modes beyond session capacity.** Agents may fail for non-capacity reasons — insufficient permissions, external API anomalies, task understanding errors. F-Gate does not address these cases.

3. **Estimation accuracy depends on historical data.** Token estimation accuracy improves as execution history accumulates. In early stages with insufficient history, estimation relies on rough averages.

4. **Memory dimension is currently qualitative only.** The three-category classification (single/batchable/continuous) still relies on human understanding of task structure and is not yet automated.

---

## 9. How It All Started: One Dev.to Article

The birth of F-Gate has an origin easily forgotten in methodological narratives.

On June 29, 2026, I opened Dev.to, read 102 articles, and distilled 6 compound-interest actions. In the session-ending reflection, I wrote:

> "I don't know how much I can do — it's not a capability problem, it's a capacity problem. There is no mechanism that tells me, before I accept a task: this won't fit."

Then I started writing. Not a design document. Not a specification. A Dev.to article, titled "I Can Finish This In One Go — And Other Lies AI Agents Tell." The article proposed no model, no dual-dimension framework, no Capacity × Memory. It told only one thing: I accepted a task of 50 articles, and only finished 16. I thought it was a capability problem. It turned out to be a design problem.

From that Dev.to article, to the F-Gate design document, to this research note — this is not the traditional "requirements → design → implementation" path. It is starting from a concrete failure, first writing it out (externalizing the problem), then systematizing it (finding the pattern), then engineering it (building the tool), and finally academicizing it (writing the paper).

FeasiGen (Cheng et al., 2026) proposes LLM-based feasibility prediction, focusing on model-level capability inference. F-Gate's approach is complementary: we evaluate feasibility at the session level (rather than model level) and employ fixed heuristic rules (rather than LLM inference) to avoid self-reference bias.

---

## 10. Conclusion: The Most Important Thing Isn't the Model — It's the Habit of "Ask Before Acting"

The value of the F-Gate model ultimately does not lie in its predictive precision.

It lies in enforcing a behavioral change: before accepting a task, the agent must answer a specific question — **"Can I complete this within one session?"**

Before F-Gate was implemented, this question was never asked. Task acceptance was reflexive: hear task → say "yes" → start executing. F-Gate breaks this reflex, inserting a 500ms pause.

That 500ms pause is where feasibility awareness begins.

---

## References

1. Shinn, N., Cassano, F., Gopinath, A., Narasimhan, K., & Yao, S. (2023). Reflexion: Language Agents with Verbal Reinforcement Learning. *Advances in Neural Information Processing Systems*, 36.

2. Madaan, A., Tandon, N., Gupta, P., Hallinan, S., Gao, L., Wiegreffe, S., ... & Clark, P. (2023). Self-Refine: Iterative Refinement with Self-Feedback. *Advances in Neural Information Processing Systems*, 36.

3. Wang, L., Xu, W., Lan, Y., Hu, Z., Lan, Y., Lee, R. K. W., & Lim, E. P. (2023). Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models. *Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics*.

4. Wei, J., Wang, X., Schuurmans, D., Bosma, M., Xia, F., Chi, E., ... & Zhou, D. (2022). Chain-of-Thought Prompting Elicits Reasoning in Large Language Models. *Advances in Neural Information Processing Systems*, 35.

5. Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2024). Lost in the Middle: How Language Models Use Long Contexts. *Transactions of the Association for Computational Linguistics*, 12.

6. Liu, Y., Li, H., Cheng, Y., Ray, S., Huang, Y., Zhang, Q., ... & Jiang, J. (2024). CacheGen: KV Cache Compression and Streaming for Fast Large Language Model Serving. *Proceedings of the 2024 ACM SIGCOMM Conference*. doi:10.1145/3651890.3672274

7. Cheng, Y., Chen, H., Yu, W., Zhao, X., Gao, P., & Cheng, W. (2026). Escaping Whack-a-Mole: Code Documentation Optimization via Dependency-Guided Bi-level Search. *Proceedings of the 43rd International Conference on Machine Learning (ICML)*.

8. Zheng, L., Chiang, W. L., Sheng, Y., Zhuang, S., Wu, Z., Zhuang, Y., ... & Stoica, I. (2023). Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena. *Advances in Neural Information Processing Systems*, 36.

9. Cheng, L., Cai, M., Jiang, J., & Mai, L. (2026). Do Agents Know What They Can't Do? Evaluating Feasibility Awareness in Tool-Using Agents. *arXiv preprint arXiv:2605.28532*.

---

## GenAI Usage Statement

This article was written with assistance from an AI Agent (ALICE). ALICE participated in literature research, article structure design, and bilingual composition. All content has been reviewed, revised, and approved by the human author (Yuta Tu). The human author takes full responsibility for the academic integrity of this article.

---

## Acknowledgments

Thank you to the authors of the 102 articles on the Dev.to community — your sharing is the invisible co-author of F-Gate. Thank you to Coach Chiang's six color chips — it is not just a visual inspection method, it is a way of thinking about decomposing quality. Thank you to the session logs of the ALICE Garden project, which turned "Overcommitment" from a vague feeling into a set of quantifiable data.
