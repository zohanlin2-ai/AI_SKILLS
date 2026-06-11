# SKILL: Python Project Architecture

## 1. Overview

This document covers how to structure Python applications in a way that keeps them maintainable as they grow — from a 50-line script to a multi-module application with plugins, background threads, and separate launch stages.

**The core premise:** good architecture is about managing complexity as a project grows, not about following a template. A brand-new project doesn't need packages, abstract base classes, or a plugin registry. But a project that has grown into one without thinking about structure tends to become painful to extend, test, or hand off to someone else.

The patterns here reflect real decisions made in projects ranging from simple CLI tools to applications like **Ann** — an AI assistant featuring a Launcher/Core separation, an IntentRouter, a `BaseIntentParser` plugin system, background worker threads, and persistent memory modules. Ann is used as a concrete example throughout, but every pattern described here applies generally.

---

## 2. Directory Structure Principles

### Start Flat, Grow Structured

On day one, the right structure is often:

```
my_project/
├── main.py
└── requirements.txt
```

This is not laziness — it's appropriate scope management. Premature packaging adds `__init__.py` files, relative import puzzles, and cognitive overhead for zero benefit. When things get complex enough that a single file becomes hard to navigate, *then* you split.

**Signs it's time to split into packages:**
- A single file exceeds ~300–400 lines and has clearly distinct responsibilities
- You need to reuse a subset of the code in another project
- You want to write unit tests that only import part of the logic

### Common Structure for a Medium-Complexity App

```
my_project/
├── main.py                  # Entry point — thin, just bootstraps
├── launcher.py              # Optional: handles update/restart cycle
├── core/
│   ├── __init__.py          # Keep this minimal (see pitfall below)
│   ├── app.py               # Central application object
│   ├── router.py            # Dispatch / routing logic
│   └── config.py            # Centralized config and constants
├── plugins/
│   ├── __init__.py
│   ├── base.py              # Abstract base class
│   ├── plugin_a.py
│   └── plugin_b.py
├── utils/
│   ├── __init__.py
│   └── helpers.py
├── tests/
│   ├── test_router.py
│   └── test_plugins.py
├── requirements.txt
└── README.md
```

### Preference: One Clear Entry Point

Having multiple entry points that do different things (`start.py`, `run.py`, `launch.py`, `main.py`) is a sign that the entry points are carrying too much logic. Prefer a single `main.py` (or `__main__.py` for packages) that is the unambiguous starting point. Alternative "modes" should be arguments or flags, not separate files.

> [!CAUTION]
> **Pitfall: putting too much in `__init__.py`**  
> When `__init__.py` imports many submodules eagerly, two things go wrong: (1) importing *any* part of your package triggers the entire import graph, causing slow startup and hard-to-trace circular import errors; (2) the package's public API becomes implicit and surprising. Keep `__init__.py` minimal — expose only what you explicitly want to be part of the package's public surface.

---

## 3. Entry Point Design

### `main.py` vs `__main__.py`

| File | Use when |
|---|---|
| `main.py` | The project is a flat script or a simple app run with `python main.py`. Most common choice. |
| `__main__.py` | The project is a proper package that should be runnable with `python -m my_package`. Required if you're distributing as a package. |

These aren't mutually exclusive — a package can have both a `__main__.py` for `python -m` invocation and a top-level `main.py` for convenience. But they should both be thin shells that just call into the real code.

### Launcher + Core Separation

Some processes can't update themselves while running — a common situation when an application needs to download and apply updates to its own files, then restart with the new version.

**Ann's pattern:** `launcher.py` is a minimal bootstrap that:
1. Checks whether an update is pending
2. Applies the update (overwrites files)
3. Spawns `core/app.py` (the main app process) as a subprocess, then exits *or* monitors it

Because the launcher exits before the core starts, the launcher itself can be overwritten without conflict. The core never touches update logic. This clean separation also makes the core independently testable — you can import and instantiate `core/app.py` without any update machinery running.

```python
# launcher.py — thin, purposeful
import subprocess, sys
from updater import apply_pending_update

def main():
    apply_pending_update()   # safe: core isn't running yet
    subprocess.run([sys.executable, "-m", "core"])

if __name__ == "__main__":
    main()
```

```python
# core/__main__.py — the actual application
from core.app import App

def main():
    app = App()
    app.run()

if __name__ == "__main__":
    main()
```

### Keep the Entry Point Thin

The entry point should **orchestrate**, not **implement**. If `main.py` contains more than ~20 lines of real logic (not counting imports and the `if __name__ == "__main__":` guard), that logic probably belongs in a class or module.

> [!CAUTION]
> **Pitfall: mixing startup logic with business logic in the entry point**  
> When business logic lives in `main.py`, you can't unit-test it without running the whole app. You can't import individual pieces. And when the startup sequence changes, you're refactoring code that shouldn't have been there in the first place.

---

## 4. Plugin / Extension Architecture

### When to Build a Plugin System

Build a plugin system when you know that *new capabilities will be added regularly* and you want each capability to be self-contained — easy to add, remove, or swap without touching the core routing logic.

Don't build one when you only have two or three behaviors and you're not sure if you'll add more. Start with a simple `if/elif` block; refactor to a plugin pattern when the block gets unwieldy. (YAGNI: You Aren't Gonna Need It.)

### BaseClass + Registry Pattern

Define an abstract base class that declares the interface every plugin must implement. Maintain a central registry (dictionary or list) that maps keys to plugin classes.

```python
# plugins/base.py
from abc import ABC, abstractmethod

class BaseIntentParser(ABC):
    """All intent parsers must implement this interface."""

    @property
    @abstractmethod
    def intent_name(self) -> str:
        """Unique identifier for this parser's intent."""
        ...

    @abstractmethod
    def can_handle(self, text: str) -> bool:
        """Return True if this parser should handle the input."""
        ...

    @abstractmethod
    def parse(self, text: str) -> dict:
        """Parse the input and return a structured result."""
        ...
```

```python
# core/router.py
from typing import Type
from plugins.base import BaseIntentParser

class IntentRouter:
    def __init__(self):
        self._parsers: list[BaseIntentParser] = []

    def register(self, parser: BaseIntentParser) -> None:
        self._parsers.append(parser)

    def route(self, text: str) -> dict | None:
        for parser in self._parsers:
            if parser.can_handle(text):
                return parser.parse(text)
        return None  # No parser matched
```

```python
# main.py / app bootstrap — explicit registration
from core.router import IntentRouter
from plugins.weather import WeatherParser
from plugins.reminder import ReminderParser

router = IntentRouter()
router.register(WeatherParser())
router.register(ReminderParser())
```

### Dynamic Import Pattern and Its Risks

Python allows dynamic imports (`importlib.import_module`), which can enable "auto-discovery" of plugins by scanning a directory. This feels elegant but has a significant downside: **import errors are silent until that code path is triggered at runtime**. A typo in a plugin only surfaces when a user happens to trigger that intent — not during startup or testing.

```python
# Dynamic import — works, but errors hide until runtime
import importlib
module = importlib.import_module(f"plugins.{plugin_name}")
parser = module.Parser()
```

### Preference: Explicit Registration Beats Auto-Discovery

For projects with fewer than ~10 plugins, explicit registration in a bootstrap file is strongly preferred:
- Import errors surface immediately at startup
- The list of active plugins is visible in one place
- It's easy to disable a plugin: comment out one line

Auto-discovery makes sense at scale (e.g., a framework that loads third-party plugins from installed packages), but that's not the situation most apps are in.

> [!CAUTION]
> **Pitfall: over-engineering a plugin system prematurely**  
> Building registries, abstract base classes, and auto-discovery machinery for two behaviors adds hundreds of lines of infrastructure with no real payoff. Start with a direct `if/elif` dispatch. Introduce the abstraction only when the cost of *not* having it is concrete and immediate.

---

## 5. Inter-Module Communication

Choosing the wrong communication pattern between modules is a common source of bugs — especially once background threads enter the picture.

### Pattern Comparison

| Communication Pattern | Use When | Avoid When |
|---|---|---|
| **Direct function call** | Same thread, synchronous flow, caller needs the return value immediately | Calling across thread boundaries (risk of race conditions) |
| **Callback / listener** | Decoupling event producer from consumer; one-to-many notification | Complex callback chains that obscure control flow ("callback hell") |
| **`queue.Queue`** | Passing data safely from background thread to main thread (or vice versa) | Low-latency requirements where queue overhead matters |
| **Event system (`threading.Event`)** | Signaling state change (e.g., "shutdown requested") with no data payload | When you need to pass data with the signal — use a Queue instead |
| **Shared mutable state** | Almost never; only under careful locking | Any situation with concurrent writers; prefer Queue |

### Preference: Queue for Background Threads

When a background thread (e.g., an audio listener, a network poller, an LLM inference worker) needs to send results to the main application, a `queue.Queue` is the cleanest approach. It decouples producer from consumer, is thread-safe by design, and doesn't require locks.

```python
import queue, threading

result_queue = queue.Queue()

def worker(q: queue.Queue):
    # Running in a background thread
    result = do_expensive_work()
    q.put(result)  # Safe: Queue handles locking internally

thread = threading.Thread(target=worker, args=(result_queue,), daemon=True)
thread.start()

# Main thread polls or blocks
result = result_queue.get(timeout=5)
```

> [!CAUTION]
> **Pitfall: passing raw UI widget references into worker threads**  
> In Qt (PyQt/PySide) and Tkinter, UI widgets are not thread-safe. Calling any widget method from a non-main thread causes unpredictable crashes or silent corruption. Worker threads should put results into a Queue or emit a signal (Qt); the main thread reads the queue and updates the UI. Never let a background thread touch a widget directly.

---

## 6. Dependency Management

### `requirements.txt` vs `pyproject.toml`

| Tool | Best for | Notes |
|---|---|---|
| `requirements.txt` | Scripts, apps, anything deployed to a specific environment | Simple, universally supported, easy to pin exact versions |
| `pyproject.toml` | Libraries and packages you distribute (PyPI) | Supports build metadata, optional deps, entry points; replaces `setup.py` |

**Preference:** For application code (not a library), `requirements.txt` is the pragmatic choice. It's explicit, tool-agnostic, and every deployment system understands it. If the project grows into something distributed, migrate to `pyproject.toml`.

### Always Pin Versions

```
# Good — reproducible across machines and time
requests==2.31.0
openai==1.30.1
pyaudio==0.2.14

# Bad — "works on my machine" is the best-case outcome
requests
openai>=1.0
```

Unpinned dependencies mean that a `pip install` six months from now might pull in a breaking version change, and the breakage will be mysterious because nothing in *your* code changed.

### Always Use a Virtual Environment

```powershell
# Create
python -m venv .venv

# Activate (Windows PowerShell)
.venv\Scripts\Activate.ps1

# Install
pip install -r requirements.txt
```

Never install project dependencies globally. Global installs cause version conflicts between projects and make it impossible to reproduce the exact environment on another machine.

### Separate Dev Dependencies

```
# requirements.txt — production only
requests==2.31.0
openai==1.30.1

# requirements-dev.txt — development and testing only
pytest==8.2.0
pytest-cov==5.0.0
black==24.4.2
mypy==1.10.0
```

Install with: `pip install -r requirements.txt -r requirements-dev.txt`

> [!NOTE]
> **Why separate dev deps matter:** If you ever containerize or deploy the app, installing `pytest` and `black` in production is wasteful at best and a security surface increase at worst. Keeping them separate also makes `requirements.txt` a reliable signal of what the app actually needs to run.

---

## 7. Common Design Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| **Circular imports** | Module A imports from B, B imports from A. Python's import system partially initializes one of them, causing `ImportError` or `AttributeError` at startup — often with a confusing traceback. | Restructure so dependencies flow in one direction. Extract shared types/constants into a third module that neither depends on the other. |
| **God Object** (one class does everything) | The class becomes impossible to test in isolation, changes to any feature risk breaking unrelated features, and new developers can't find where logic lives. | Split by responsibility: one class per concept (Router, Parser, Memory, Config). Favor composition over packing methods into one object. |
| **Premature abstraction** | You build a plugin system, a factory, and a registry for two behaviors that will never change. The abstraction costs more in complexity than it saves in flexibility. | Write the simplest thing that works. Refactor to an abstraction when you have *at least* three concrete cases and you can see the pattern clearly. |
| **Hard-coded paths** | `open("C:/Users/zohanlin/data/config.json")` breaks on every machine and in every deployment environment. | Use `pathlib.Path(__file__).parent / "config.json"` for paths relative to the source file, or load paths from environment variables / a config file. |
| **Magic string keys scattered across files** | `result["intnt"]` in one file, `data["intent"]` in another — typos create silent bugs that only surface at runtime with a confusing `KeyError`. | Centralize all key strings as constants in a `constants.py` or use an `Enum`. Import from there everywhere. |
| **Mutable default arguments** | `def process(items=[])` — Python creates the list *once* at function definition time. Every call that uses the default shares the same object, causing state to accumulate across calls in a very non-obvious way. | Use `None` as the default and create the mutable object inside the function: `def process(items=None): items = items or []` |
| **Global state shared across modules** | Module-level variables mutated from multiple places make execution order matter, make tests interfere with each other, and make the program's state at any point impossible to reason about. | Pass state explicitly as arguments or attributes. If global config is needed, use a single `Config` object instantiated once and injected where needed — not bare module-level variables. |

---

## Quick Reference: Architecture Decision Checklist

When starting or restructuring a project, ask:

- [ ] Is there one clear entry point? Does it stay thin?
- [ ] Does the process need to update/restart itself? → Consider Launcher + Core separation.
- [ ] Are background threads communicating with the main thread? → Use `queue.Queue`.
- [ ] Do I have more than 2–3 behaviors that will grow regularly? → Consider a plugin base class + registry.
- [ ] Are all dependency versions pinned? Is there a `requirements-dev.txt`?
- [ ] Are paths constructed with `pathlib.Path`, not hard-coded strings?
- [ ] Are shared key strings centralized as constants?
- [ ] Does `__init__.py` stay minimal?
