# AI SKILLS Repository

Welcome to the **AI_SKILLS** repository. This workspace serves as a structured collection of specialized technical skill guides, standard operating procedures (SOPs), and development guidelines curated and maintained for agentic workflows.

---

## Project Index

| Project Name | Path | Description | Key Focus Area |
| :--- | :--- | :--- | :--- |
| **UML_seq_act** | [`/UML_seq_act`](file:///C:/Users/zohanlin/Documents/AI_SKILLS/UML_seq_act) | Generating System Architecture Diagrams | Mermaid.js, HTML/Canvas 2x PNG Export, Sequence & Activity Diagrams |
| **andrej-karpathy-skills** | [`/andrej-karpathy-skills`](file:///C:/Users/zohanlin/Documents/AI_SKILLS/andrej-karpathy-skills) | Coding & LLM Interaction Guidelines | Simplicity, surgical modifications, goal-driven loops, safety |
| **github_AI_SKILLS** | [`/github_AI_SKILLS`](file:///C:/Users/zohanlin/Documents/AI_SKILLS/github_AI_SKILLS) | Git upload & Conflict Resolution SOP | Git workflow, index maintenance, remote repository detection |
| **prompt_engineering** | [`/prompt_engineering`](file:///C:/Users/zohanlin/Documents/AI_SKILLS/prompt_engineering) | LLM Prompt Engineering Patterns | System prompt design, output format control, few-shot & CoT usage |
| **python_architecture** | [`/python_architecture`](file:///C:/Users/zohanlin/Documents/AI_SKILLS/python_architecture) | Python Project Architecture Principles | Module structure, plugin patterns, inter-module communication |
| **local_ai_model** | [`/local_ai_model`](file:///C:/Users/zohanlin/Documents/AI_SKILLS/local_ai_model) | Local AI Model Management | VRAM management, precision tradeoffs, model loading patterns |
| **logging_debugging** | [`/logging_debugging`](file:///C:/Users/zohanlin/Documents/AI_SKILLS/logging_debugging) | Logging & Debugging for Python Apps | loguru setup, structured logs, multithreaded debugging |

---

## Detailed Skill Overviews

### 1. [UML_seq_act](file:///C:/Users/zohanlin/Documents/AI_SKILLS/UML_seq_act/SKILL.md)
A comprehensive guide on creating high-quality system architecture diagrams (Sequence and Activity diagrams) using Mermaid.js syntax. 
* **Key Features**:
  * Step-by-step instructions for Mermaid syntax.
  * Common pitfalls and resolutions (e.g., blank exports, transparent backgrounds, subgraph rendering, and layout overflow scaling fixes).
  * Client-side HTML wrapper template that renders and exports SVGs into high-resolution, crisp 2x supersampled PNGs with zoom controls using Canvas API.

### 2. [andrej-karpathy-skills](file:///C:/Users/zohanlin/Documents/AI_SKILLS/andrej-karpathy-skills/SKILL.md)
Practical rules and behavioral guidelines (inspired by Andrej Karpathy's LLM programming style) to minimize typical coding and alignment errors.
* **Key Features**:
  * **Think Before Coding**: Explicit assumptions and trade-off considerations.
  * **Surgical Changes**: Touch only what is necessary; match existing styling without arbitrary refactoring.
  * **Simplicity First**: Deliver minimal code to solve the core problem; avoid premature or speculative abstractions.
  * **Goal-Driven Execution**: Define clear success criteria and loop until verified.
  * **Security-First Check**: Prohibit plaintext keys and perform basic validation check for SQL injection/XSS vulnerabilities.
  * **Testing & Verification Standards**: Define strict standards for unit/integration testing, edge cases/boundaries, fault tolerance, and regression checks.

### 3. [github_AI_SKILLS](file:///C:/Users/zohanlin/Documents/AI_SKILLS/github_AI_SKILLS/SKILL.md)
The Standard Operating Procedure (SOP) guiding how developers and AI agents upload, update, and manage codebases within this specific repository.
* **Key Features**:
  * Branching scenario guidance (Initial Setup/Empty Repo vs Existing Content Repo).
  * Conflict resolution steps (interactive prompts to pull, push, or cancel).
  * Naming conventions, automatic root-level `README.md` index maintenance, and `CHANGELOG.md` verification rules.
  * Commit message convention (Conventional Commits format) and pre-upload `.gitignore` checklist.
  * Explicit note on why PR conventions are not required for this single-maintainer repository.

### 4. [prompt_engineering](file:///C:/Users/zohanlin/Documents/AI_SKILLS/prompt_engineering/SKILL.md)
Patterns and pitfalls for writing effective prompts for LLM-based systems, grounded in real usage with AI assistants and agentic pipelines.
* **Key Features**:
  * System prompt design: four-zone structure (role, scope, constraints, output format) with separation rationale.
  * Output format control: reliable JSON extraction, Markdown hinting, and the prose-wrapping pitfall.
  * Few-shot and Chain-of-Thought decision tables (when to use vs. when to skip).
  * Prompt version management: centralized `prompts/` module pattern to prevent scattered string maintenance.
  * Common pitfalls table: truncation, persona drift, hallucination, language switching, and more.

### 5. [python_architecture](file:///C:/Users/zohanlin/Documents/AI_SKILLS/python_architecture/SKILL.md)
General-purpose Python project architecture principles, illustrated with concrete examples from real projects.
* **Key Features**:
  * Start-flat-grow-structured philosophy with medium-complexity directory tree example.
  * Entry point design: `main.py` vs `__main__.py`, and the Launcher+Core separation pattern.
  * Plugin/extension architecture: BaseClass+Registry pattern with code examples (IntentRouter/BaseIntentParser).
  * Inter-module communication comparison (direct call / callback / Queue / Event).
  * Common pitfalls table: circular imports, God Object, premature abstraction, hard-coded paths, mutable defaults.

### 6. [local_ai_model](file:///C:/Users/zohanlin/Documents/AI_SKILLS/local_ai_model/SKILL.md)
Practical guide for managing local AI models (image generation, local LLMs) with a focus on VRAM constraints and stability.
* **Key Features**:
  * Framework selection table: Diffusers, llama.cpp/GGUF, Ollama, Transformers — with VRAM requirements.
  * VRAM management: correct `del` + `empty_cache()` order, CPU offload, multi-model sequential workflow.
  * Precision tradeoffs: fp32/fp16/bf16/int8/int4 comparison, and the VAE fp32 pitfall that causes NaN artifacts.
  * Singleton model loader pattern and long-running inference stability (OOM recovery, timeout handling).
  * Pitfalls table: 9 rows covering the most common local model failure modes.

### 7. [logging_debugging](file:///C:/Users/zohanlin/Documents/AI_SKILLS/logging_debugging/SKILL.md)
Structured logging and debugging strategies for complex Python applications, with emphasis on multithreaded environments.
* **Key Features**:
  * Log design principles: what to log, what not to log, and the contextual identifier preference.
  * Tool selection: loguru vs standard logging comparison table, with `InterceptHandler` bridge pattern.
  * Log level semantics with examples and a DEBUG-vs-INFO decision rule.
  * Multithreaded logging: thread-safe loguru setup, file rotation, and the `print()` interleaving pitfall.
  * Debugging strategies: silent failure anti-patterns, `logger.exception()` usage, timing-based bug reproduction.

---

*Repository maintained by Antigravity AI Assistant.*
