# Changelog

All notable changes to this repository will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.5.0] - 2026-06-11

### Added
- **prompt_engineering**: New `SKILL.md` covering LLM prompt engineering patterns — system prompt design, output format control (JSON/Markdown), few-shot and Chain-of-Thought usage decision tables, prompt version management, and a 6-row common pitfalls table.
- **python_architecture**: New `SKILL.md` covering general-purpose Python project architecture — directory structure principles, entry point design (Launcher+Core pattern), plugin/extension architecture (BaseClass+Registry), inter-module communication patterns, dependency management, and a 7-row design pitfalls table.
- **local_ai_model**: New `SKILL.md` covering local AI model management — framework selection (Diffusers, llama.cpp, Ollama, Transformers), VRAM management strategy, model loading patterns (singleton), precision tradeoffs (fp32/fp16/int4), long-running inference stability, and a 9-row pitfalls table.
- **logging_debugging**: New `SKILL.md` covering logging and debugging for complex Python apps — log design principles, loguru vs standard logging comparison, log level semantics, multithreaded logging setup, structured output, debugging strategies, and a 9-row pitfalls table.

### Changed
- **github_AI_SKILLS**: Added Section 4 (Conventional Commits message convention), Section 5 (pre-upload `.gitignore` checklist), and a Workflow Model note explaining why PR conventions are intentionally omitted.
- **andrej-karpathy-skills**: Fixed misleading `# CLAUDE.md` title to `# SKILL: LLM Coding & Interaction Guidelines`; added Section 11.5 Pre-Completion Self-Check table.
- **UML_seq_act**: Extended Section 3 with a 4-type Diagram Quick Reference table (Sequence, Activity, Class, State); added `[!TIP]` cross-reference in Section 8 pointing to the full interactive editor.
- **README.md**: Updated Project Index and detailed overviews to include all 7 SKILL directories.

## [1.4.0] - 2026-06-11


### Changed
- **UML_seq_act**: Updated `SKILL.md` to document the layout overflow top-clipping fix, zoom controls implementation, and sequence vs. activity diagram selection rationales.

## [1.3.0] - 2026-06-11

### Added
- **andrej-karpathy-skills**: Added `Section 11: Testing and Verification Standards` establishing strict rules for Unit/Integration tests, Edge/Boundary cases, Fault Tolerance, and Regression/Manual verification.
- **README.md**: Added `Testing & Verification Standards` to the key features list.

## [1.2.0] - 2026-06-11

### Changed
- **andrej-karpathy-skills**: Added `Section 10: Security-First Check` to ensure AI agents avoid plaintext keys and verify input security (SQL injection/XSS).
- **README.md**: Updated key features under the `andrej-karpathy-skills` section.

## [1.1.0] - 2026-06-11

### Changed
- **github_AI_SKILLS**: Updated `SKILL.md` to incorporate root `README.md` and `CHANGELOG.md` verification, generation, description comparison, and update logic during the upload workflow.

## [1.0.0] - 2026-06-11

### Added
- **UML_seq_act**: A comprehensive guide to Mermaid.js sequence and activity diagrams with a 2x PNG rendering HTML export solution.
- **andrej-karpathy-skills**: Behavioral guidelines based on Andrej Karpathy's LLM coding philosophy to prevent common AI coding errors.
- **github_AI_SKILLS**: An SOP document outlining git operations and conflict resolution workflows for uploading local files.
- **README.md**: Repository landing page introducing all three skills.
- **.gitignore**: Basic ignore rules for IDEs and temporary directories.
