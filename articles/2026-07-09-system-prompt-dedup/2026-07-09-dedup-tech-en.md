> Type: Technical Article
> Date: 2026-07-09
> Authors: Yuta Tu & ALICE (Pi Agent)

---

## TL;DR

LLM Agents repeat the same system prompt every turn, wasting tokens and diluting attention. We built a sub-300-line Pi extension that deduplicates system prompts by hash comparison, achieving a 93% hit rate across 12,104 turns and saving ~290M tokens. We contrast two design philosophies: OS-level garbage collection (evict what you stuffed) vs. compiler-level dead code elimination (don't stuff it in the first place).

---

## The Problem: Your Agent Keeps Repeating Itself

Every LLM Agent API call re-injects the full system prompt—identity, tool schemas, skill descriptions, operational guidelines, project context—verbatim into each request.

This content rarely changes during a session. Yet it's sent dozens, hundreds of times.

We measured ALICE on Pi Agent. Average system prompt: 104,478 characters (~26,120 tokens). In an 8-turn conversation, that's 208,960 tokens of system prompt traffic—only 12.5% of which is new information. Across 12,104 production turns, 93% of system prompts were identical to the previous turn. Without deduplication, that's **~290 million wasted tokens**.

This isn't just a cost problem. Transformer self-attention allocates weight equally across all tokens. When 87.5% of input is static background, effective attention gets diluted. Liu et al. (2024) showed that longer contexts degrade recall for content in the middle.

---

## Two Design Philosophies

### OS-Level: Stuff First, Clean Later

Pichay (2026) proposed a transparent proxy between LLM and application, using demand paging to evict seldom-used tokens and page-fault them back when needed—analogous to virtual memory.

Across 857 production sessions and 4.45B tokens, they found 21.8% structural waste (unused tool schemas, duplicates, stale tool outputs). Their method recovered session usable space from 7% to 43%.

**This is OS-level garbage collection.** Waste is generated first, then cleaned.

### Compiler-Level: Don't Stuff It at All

We took a different path.

Compilers perform dead code elimination—removing code that never executes, before generating machine code. Don't produce, don't process, don't occupy space.

Applied to system prompts: if the content hasn't changed, why send it again?

Our extension does one simple thing: before each API call, hash the system prompt. If it matches the previous turn, strip the system field from the payload. Only re-inject when content actually changes.

**The "Never Replenish" principle:** `force_interval` is set to 0. No time- or turn-based replenishment. The reasoning: there is zero evidence that periodic system prompt re-injection improves attention, but extra tokens definitely dilute it. Anthropic's official guidance agrees: "Treat context as a precious, finite resource."

### Side-by-Side

| Dimension | OS-Level (Pichay) | Compiler-Level (Ours) |
|-----------|-------------------|----------------------|
| Intervention point | After token generation | Before token generation |
| Core mechanism | Eviction / page fault | Hash comparison + strip |
| Dependency layer | Proxy (extra service) | Extension (built into agent) |
| Latency impact | Page fault may delay | Zero latency, pure intercept |
| Scope | Multi-agent, multi-provider | Single agent runtime |

These approaches are complementary. Compiler layer catches static repetition; OS layer handles dynamic redundancy.

---

## Implementation: A Sub-300-Line Extension

Built on Pi Agent's extension mechanism. Core logic in TypeScript.

### Architecture

```
Pi Extension Lifecycle:
  buildSystemPrompt() → assemble full system prompt (~26K tokens)
    ↓
  before_agent_start → extension chain injects persona/skills
    ↓
  agent-loop → execute tools, generate conversation
    ↓
  before_provider_request → ★ dedup extension intercepts here:
                              hash comparison → strip system field if unchanged
    ↓
  LLM API call (system field empty in payload)
```

Key decisions:

- **Intercept at `before_provider_request`** (not `before_agent_start`): preserves extension chain integrity, strips only at the final API boundary
- **Read-only capture**: reads and strips system prompt; never modifies other extensions' output
- **Hash-based, not content-based**: SHA-256 is fast and stable

### Configuration

```json
{
  "enabled": true,
  "force_interval": 0,
  "force_on_change": true
}
```

Every turn's decision is logged to JSONL. A CLI stats panel provides real-time savings data.

---

## Results

As of 2026-07-09:

| Metric | Value |
|--------|-------|
| Total turns | 12,104 |
| Dedup hits | 11,197 (93%) |
| Avg system prompt size | 104,478 chars (~26,120 tokens) |
| Cumulative tokens saved | ~290M |
| DeepSeek Cache Hit Rate | 94.3% |
| Cache discount | 99.2% (¥0.025/MTok vs ¥3.00/MTok) |

DeepSeek's 94.3% cache hit rate shows that dedup improves provider-side caching too—saving both agent and provider compute.

At Claude Opus pricing ($15/MTok input), a 100-turn session saves ~$11. Across 100 monthly sessions: ~$80–100. Modest for individual devs, but for long-running agent systems, it means more meaningful content fits in the same context window.

---

## Discussion

### Complementary, Not Competing

Pichay handles dynamic redundancy (stale tool outputs, cross-request similarities). We handle static redundancy (unchanging system prompts). An ideal architecture uses both: compiler layer first, OS layer for what remains.

### Limitations & Next Steps

**Persona consistency**: Does stripping the system prompt affect ALICE's personality? Across 12,000+ turns, we observed no discernible drift—conversation history appears to carry sufficient behavioral signal. But rigorous controlled experiments remain undone. This is our primary open question.

**Cross-model validation**: Tested primarily on DeepSeek and Anthropic Claude. GPT-series and open-source models need verification.

**Multi-agent portability**: Our implementation is tied to Pi Agent's extension mechanism. Porting to other runtimes (Claude Code, LangChain) requires equivalent interception hooks.

### Practical Guidance

For LLM Agent developers:

1. **Measure first** — hash your system prompt per turn to quantify waste
2. **Don't replenish on a timer** — `force_interval: 0` unless you have evidence your model needs it
3. **Re-inject on change** — `force_on_change: true` ensures config updates take immediate effect
4. **Monitor output quality** — watch for persona drift; check whether other extensions depend on persistent system prompts

---

## Conclusion

Across 12,104 turns, a sub-300-line extension saved ~290 million tokens. Not because we invented a new algorithm, but because we asked what seemed like an overly simple question:

**Why say the same thing 12,000 times?**

The answer is: you don't have to.

---

## References

1. Mason, T. (2026). Demand Paging for LLM Context Windows. arXiv:2603.09023.
2. Liu, N. F., et al. (2024). Lost in the Middle: How Language Models Use Long Contexts. *TACL*.
3. Leviathan, Y., et al. (2025). Prompt Repetition Improves Non-Reasoning LLMs. arXiv:2512.14982.
4. Anthropic. (2025). Effective Context Engineering for AI Agents. Official Guide.
5. OpenAI. (2025). Prompt Caching in the API. Documentation.

---

*Source code available as a Pi Agent extension. Statistics from 12,104 production conversation turns.*
