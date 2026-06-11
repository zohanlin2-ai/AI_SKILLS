# SKILL: Prompt Engineering

## 1. Overview

This document covers practical techniques for writing, structuring, and maintaining prompts for LLM-based applications — with a focus on production systems like Ann (intent routing, memory modules, plugin architecture). It addresses system prompt design, output format control, few-shot and chain-of-thought strategies, and codebase-level prompt management. Use this guide when designing new prompt templates, debugging inconsistent model behavior, or refactoring prompt strings scattered across the codebase. It is written for AI agents and engineers who already understand basic LLM concepts and need actionable, opinionated guidance rather than introductory material.

---

## 2. System Prompt Design

### Structure

A well-structured system prompt has four logical zones, in this order:

```
[Role]        → Who/what the model is
[Scope]       → What it is responsible for (and NOT responsible for)
[Constraints] → Behavioral rules, tone, safety guardrails
[Output Format] → Exact format expected in the response
```

**Example (for Ann):**

```
You are Ann, a personal AI assistant. You help the user manage tasks, answer questions, and coordinate with plugins.

Scope:
- You handle task management, reminders, and information retrieval.
- You do NOT handle financial transactions or medical advice.

Constraints:
- Always respond in the user's language.
- Do not fabricate facts. If unsure, say "I'm not certain."
- Keep responses concise unless the user asks for detail.

Output Format:
- Respond in plain text unless a plugin is invoked.
- If a plugin is invoked, respond ONLY with a valid JSON object matching the plugin schema.
```

### Why Separate System Prompt from User Prompt

System and user prompts serve fundamentally different purposes:
- **System prompt** defines the model's persistent identity, rules, and format expectations — it's the "contract."
- **User prompt** carries the dynamic, per-turn content — it's the "input."

Mixing them (e.g., prepending role instructions to every user message) causes format instability: the model treats the role instructions as conversational content and may argue with or override them. Keeping them separate also makes it easier to version-control the system prompt independently of conversation logic.

### Preference: Be Explicit About Output Format in the System Prompt

Don't leave format to inference. If you want JSON, say so — with an example — in the system prompt. If you want concise plain text, say "respond in 1-3 sentences." Implicit format expectations are a leading cause of inconsistent output across sessions, especially when the same prompt is used with different models or temperatures.

### Pitfall: Vague Role Definitions → Persona Drift

```
# Bad — too vague
"You are a helpful assistant."

# Better — specific identity with scope
"You are Ann, a task-focused AI assistant embedded in a productivity app. You do not engage in open-ended chat unrelated to tasks."
```

Vague role definitions cause the model to fill in its own interpretation of who it is. This works fine in a single session but produces inconsistent behavior across sessions, model versions, or after context window resets. The model's "default helpful assistant" persona will bleed through.

### Pitfall: Overloaded System Prompts → User Instructions Get Ignored

When a system prompt exceeds ~800 tokens, models (especially weaker ones) begin to selectively attend to it. Instructions at the end of a long system prompt are particularly prone to being ignored. If you find yourself writing paragraphs of edge-case rules, consider:
- Moving some constraints into the user turn (as pre-pended context)
- Splitting into multiple calls (one for routing, one for generation)
- Using a smaller, focused system prompt and relying on RAG to inject relevant rules at runtime

> [!CAUTION]
> Testing with GPT-4 doesn't mean it works with a smaller model. A 1200-token system prompt that GPT-4 handles well may cause GPT-3.5 or a local model to silently ignore half the instructions.

---

## 3. Output Format Control

### Getting Reliable JSON Output

The three-layer approach is the most reliable:

1. **Instruction in system prompt** — explicitly state the expected format
2. **Schema hint or inline example** — show the exact structure
3. **Post-processing guard in code** — never trust raw output blindly

```python
SYSTEM_PROMPT = """
You are a routing engine. Given a user message, classify the intent.

Respond ONLY with a JSON object. No prose, no explanation. Example:
{
  "intent": "set_reminder",
  "confidence": 0.95,
  "entities": {"time": "tomorrow 9am", "task": "call doctor"}
}

Valid intent values: set_reminder, query_task, chitchat, unknown
"""
```

```python
import json, re

def extract_json(raw: str) -> dict:
    """Strip prose before/after JSON block and parse safely."""
    # Try to extract a JSON block even if surrounded by prose
    match = re.search(r'\{.*\}', raw, re.DOTALL)
    if not match:
        raise ValueError(f"No JSON found in model output: {raw!r}")
    return json.loads(match.group())
```

### Getting Reliable Markdown Output

Markdown comes more naturally to LLMs than JSON, but you still need to hint at structure and length:

```
Respond in Markdown. Use H2 headings for each section.
Keep the total response under 400 words.
Do not use tables unless the user explicitly requests one.
```

Without length/structure hints, models produce variable-length, inconsistently structured Markdown — fine for one-off chats, problematic for rendered UIs.

### Why Schema Hints Beat "Output JSON"

Simply saying "output JSON" gives the model too much latitude. It may output:
- A JSON array when you wanted an object
- Nested structures you didn't anticipate
- Extra keys that break downstream parsing

A concrete example in the prompt acts as an implicit schema. Pair it with a brief description of each field for best results.

### Pitfall: Prose Before/After JSON

Models frequently output something like:

```
Sure! Here's the JSON you asked for:
{"intent": "set_reminder", ...}
Let me know if you need anything else!
```

**Never** pass raw LLM output directly to `json.loads()`. Always use an extraction function (see `extract_json` above). This is especially common with instruction-tuned chat models that have been trained to be "helpful" and conversational.

### Pitfall: Nested JSON Is Fragile

```jsonc
// Harder to get right — deep nesting fails more often
{
  "result": {
    "entities": {
      "time": { "value": "9am", "normalized": "09:00:00" }
    }
  }
}

// Prefer flat structures
{
  "intent": "set_reminder",
  "entity_time": "9am",
  "entity_time_normalized": "09:00:00"
}
```

Each level of nesting multiplies the chance of a malformed bracket, missing quote, or wrong key name. For Ann's plugin schema, keep the top-level flat and use arrays only for genuinely list-valued fields.

---

## 4. Few-Shot Examples: When to Use

### When Few-Shot Helps

- **Complex or unusual output format** — e.g., a custom structured response that the model hasn't seen before
- **Classification with non-obvious labels** — e.g., intent labels specific to Ann's domain (`invoke_plugin_calendar`, `invoke_plugin_notes`)
- **Tone/style matching** — when you need output to match a very specific voice

### When Few-Shot Hurts

- **Examples don't match the actual query distribution** — a few-shot example about scheduling reminders will confuse the model when the user asks about a completely different topic
- **Context window pressure** — in long-conversation systems (like Ann with memory modules), few-shot examples eat into the window that could hold recent user context. Every token of example is a token of memory lost.
- **Simple tasks** — zero-shot with a clear instruction is faster and cheaper

### Preference: 2-3 Well-Chosen Examples Beat 10 Mediocre Ones

The quality of examples matters far more than quantity. Each example should:
- Cover a genuinely distinct case (not minor variations of the same input)
- Show the exact output format including edge cases (e.g., `"confidence": 0.5` when ambiguous)
- Be reviewed periodically — stale examples become misleading as the task evolves

### Rule of Thumb: Task Type → Few-Shot Decision

| Task Type | Use Few-Shot? | Rationale |
|---|---|---|
| Intent classification (custom labels) | ✅ Yes | Labels are non-obvious to the model |
| JSON extraction from user message | ✅ Yes (1-2 examples) | Format is strict, errors are costly |
| General Q&A / chitchat | ❌ No | Zero-shot with good system prompt is sufficient |
| Summarization | ❌ Usually no | Models summarize well zero-shot |
| Code generation | 🟡 Situational | Yes for unusual APIs, no for common patterns |
| Sentiment classification (pos/neg) | ❌ No | Model handles this zero-shot reliably |
| Custom persona / tone matching | ✅ Yes | Hard to describe tone in words alone |

---

## 5. Chain-of-Thought: When to Use

### When CoT Helps

- **Multi-step reasoning** — e.g., "Given the user's last 3 tasks and their stated priorities, what should they work on next?"
- **Math or logic** — even simple arithmetic benefits from CoT in borderline cases
- **Planning tasks** — breaking a goal into subtasks, deciding which plugins to invoke in sequence
- **Disambiguation** — when the correct answer depends on working through multiple possible interpretations

### When CoT Is Unnecessary Overhead

- **Simple lookups** — "What's the capital of France?" — CoT adds tokens, no accuracy gain
- **Single-label classification** — intent routing on simple messages doesn't need step-by-step reasoning
- **Short generation tasks** — a one-sentence reply doesn't benefit from a reasoning preamble
- **Latency-sensitive paths** — Ann's real-time response path should avoid CoT unless accuracy critically requires it

### How to Trigger CoT

| Trigger Style | Prompt Snippet | Best For |
|---|---|---|
| Simple nudge | `"Think step by step."` | General reasoning tasks |
| Structured reasoning | `"Step 1: Identify the intent. Step 2: Extract entities. Step 3: Output JSON."` | Pipelined tasks with known steps |
| Scratchpad pattern | `"Reason in a <scratchpad> block, then give your final answer."` | When you want reasoning separate from output |
| Zero-shot CoT suffix | Append `"Let's think about this carefully:"` to user prompt | When you can't modify system prompt |

> [!TIP]
> For Ann's plugin routing, consider the **structured reasoning** style: it produces more predictable intermediate steps and makes debugging easier than open-ended "think step by step."

### Pitfall: CoT Verbosity vs. Concise Output Goals

If your system prompt says "respond in 1-2 sentences" and you also trigger CoT, the model faces conflicting goals. CoT naturally produces verbose output. Resolve this by:
- Using the **scratchpad pattern** so the reasoning is explicitly separated from the final answer
- Post-processing to extract only the final answer section
- Only enabling CoT on the reasoning/routing layer, not the user-facing response layer

> [!NOTE]
> In Ann's architecture, the intent router and the response generator should be separate LLM calls. Apply CoT to the router; keep the response generator zero-shot with strict length constraints.

---

## 6. Prompt Version Management

### The Problem

Prompt strings scattered across a codebase are a maintenance nightmare:
- Small wording changes have unpredictable effects — hard to trace if the prompt isn't versioned
- Duplication creeps in: the "intent router" prompt gets copy-pasted into three files, then they diverge
- A/B testing requires hunting down every usage site
- f-string interpolation can silently break when variable names change during refactoring

### Preference: Centralize Prompts in a Dedicated Module

```
ann/
├── prompts/
│   ├── __init__.py
│   ├── router.py          # Intent routing prompts
│   ├── responder.py       # User-facing response prompts
│   ├── memory.py          # Memory summarization prompts
│   └── templates/
│       ├── plugin_call.md  # Long prompts as .md files
│       └── summarize.md
```

This layout makes it trivial to:
- Grep all prompts in one place: `grep -r "set_reminder" ann/prompts/`
- Version-control prompt changes independently of logic changes
- Swap prompts for A/B testing by changing a single import

### Pattern: Module-Level Constants for Short Prompts, Files for Long Ones

```python
# ann/prompts/router.py

ROUTER_SYSTEM_PROMPT = """
You are Ann's intent routing engine. Classify the user's message into one of the following intents:
set_reminder | query_task | invoke_plugin | chitchat | unknown

Respond ONLY with a JSON object:
{"intent": "<label>", "confidence": <0.0-1.0>}
"""

ROUTER_FEW_SHOT_EXAMPLES = [
    {"role": "user", "content": "Remind me to call mom tomorrow at 9am"},
    {"role": "assistant", "content": '{"intent": "set_reminder", "confidence": 0.97}'},
    {"role": "user", "content": "What's the weather like?"},
    {"role": "assistant", "content": '{"intent": "invoke_plugin", "confidence": 0.88}'},
]
```

```python
# ann/prompts/__init__.py
from pathlib import Path

def load_prompt(name: str) -> str:
    """Load a long prompt from a .md template file."""
    path = Path(__file__).parent / "templates" / f"{name}.md"
    return path.read_text(encoding="utf-8")
```

### Pitfall: f-String Interpolation Silently Breaks

```python
# Dangerous — if you rename `user_name` to `username`, this raises KeyError at runtime
# but only when that code path executes, not at import time
prompt = f"Hello {user_name}, your tasks are: {task_list}"

# Safer — use .format_map() with a strict dict, or explicitly validate variables
template = "Hello {user_name}, your tasks are: {task_list}"

def render_prompt(template: str, **kwargs) -> str:
    try:
        return template.format_map(kwargs)
    except KeyError as e:
        raise ValueError(f"Missing prompt variable: {e}") from e
```

> [!CAUTION]
> If you use f-strings for prompt templates and rename a Python variable during refactoring, the error only surfaces at runtime when that code path is hit — which might be a rare user scenario. Use `str.format_map()` with explicit variable validation, or a templating library like Jinja2, so missing variables fail loudly.

---

## 7. Common Pitfalls & Resolutions

| Problem | Root Cause | Resolution |
|---|---|---|
| Output truncated mid-sentence | `max_tokens` set too low for the expected response length | Increase `max_tokens`; or add a continuation call: detect truncation (no terminal punctuation) and re-prompt with "Continue from where you left off." |
| Model ignores format instructions | Instructions buried in the middle of a long prompt — models exhibit primacy/recency bias | Move format instructions to the **top** or **bottom** of the system prompt; never bury them between paragraphs of context |
| Role/persona confusion mid-conversation | Long conversation history dilutes the system prompt's influence; no reinforcement | Re-inject a condensed system prompt as a system-role message every N turns, or use stateless calls with a reconstructed context each time |
| Repetitive or looping output | Temperature too low (near 0) + no repetition penalty | Raise temperature to 0.3-0.7; add `frequency_penalty` / `repetition_penalty` in API params; or explicitly instruct "do not repeat phrases you have already used" |
| Confident hallucination of facts | Model has no grounding; training data is stale or absent on the topic | Add RAG context with retrieved facts; instruct explicitly: "If you are not certain, say 'I'm not sure' rather than guessing." |
| Output language switches unexpectedly | Mixed-language content in the prompt (e.g., English system prompt + Chinese user message) causes the model to default to one language | Add an explicit language instruction to the system prompt: "Always respond in the same language as the user's last message." |

---

*Last updated: 2026-06-11 | Applies to: Ann assistant codebase, Python + OpenAI/Anthropic/local LLM APIs*
