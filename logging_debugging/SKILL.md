# SKILL: Logging & Debugging

## 1. Overview

Logging is infrastructure, not an afterthought — just like error handling or data validation. For complex systems involving background threads, async pipelines, or long-running AI workloads, good logging is often the *only* reliable way to understand what happened after the fact. A debugger can't attach to a thread that crashed two hours ago; a well-designed log can reconstruct exactly what went wrong and when. In multi-threaded environments like QThread-based UIs or background file-locking tasks, the sequence of events across threads is nearly impossible to reason about without structured log evidence. Investing in logging upfront pays dividends every time an intermittent bug appears in production or a nightly pipeline silently fails.

---

## 2. Log Design Principles

### The North Star Principle

> Design every log line to help someone debug at 2am without access to the source code.

That "someone" is usually future-you, six months from now, with no memory of how the system works. Ask: "If I saw only this log line, would I know what the system was doing, and why it matters?"

### What to Log

| Category | Examples |
|---|---|
| **State transitions** | `"Task status changed: PENDING → RUNNING"`, `"Lock acquired for file: memory.json"` |
| **External calls** | `"Calling OpenAI API: model=gpt-4o, prompt_tokens=312"`, `"API response: 200 OK, latency=1.3s"` |
| **Errors with full context** | `"Failed to parse intent: raw_input='buy coffee', error=KeyError('entity')"` |
| **Decision branches** | `"Memory extraction skipped: confidence=0.42 < threshold=0.6"` |
| **Start and end of significant operations** | `"Background sync started"`, `"Background sync completed in 4.1s, 3 files updated"` |

### What NOT to Log

- **Every line of execution** — this is noise, not signal; use DEBUG sparingly
- **Raw passwords, API keys, or tokens** — log the key *name* or last-4 characters at most
- **Huge data payloads** — log metadata instead: `"Received response: 4,200 tokens, 3 tool_calls"` not the entire payload
- **Redundant "I am here" messages** — e.g., `"Entering function process_message"` on every call

### Preferred Log Line Format

Every log line should contain:

```
{timestamp} | {level} | {thread_name} | {module}:{line} - {message}
```

Include a short **contextual identifier** when available — a `request_id`, `session_id`, or `task_name` — so you can filter and trace a single operation across hundreds of log lines.

> [!CAUTION]
> Logging `"Processing..."` with no context is nearly useless when 50 things are processing simultaneously. Always include *what* is being processed: `"Processing intent: task_id=7a3f, user='alice', type='reminder_add'"`.

---

## 3. Tool Selection: loguru vs Standard Logging

### Comparison Table

| Feature | loguru | standard logging |
|---|---|---|
| **Setup** | One-line: `from loguru import logger` | Requires handler + formatter boilerplate |
| **Structured output** | Built-in JSON via `serialize=True` | Needs `python-json-logger` or similar |
| **Exception capture** | Auto-captures full traceback with `logger.exception()` | Must pass `exc_info=True` manually |
| **Thread safety** | Built-in — no extra config | Requires careful `QueueHandler` or lock setup |
| **Log rotation** | Built-in: `rotation="10 MB"` or `rotation="1 day"` | Needs `RotatingFileHandler` + boilerplate |
| **Async support** | Basic (synchronous internally) | Requires `QueueHandler` for proper async |
| **Library interop** | Needs `propagate` bridge for stdlib loggers | Native — all libraries use it by default |
| **Color output** | Auto-colored by level in terminal | Requires third-party formatter |

### Preference: loguru for New Projects

**Why**: Eliminates boilerplate entirely. Built-in thread-safe file rotation is particularly valuable for multi-threaded applications — no need to manually configure `RotatingFileHandler` or worry about file handle contention across threads. The exception auto-capture (`logger.exception()`) alone saves real debugging time.

**When standard `logging` is preferred**: When integrating tightly with libraries that inspect or configure the standard `logging` hierarchy (e.g., `uvicorn`, `sqlalchemy`, `transformers`). In these cases, use loguru's interop to bridge:

```python
import logging
from loguru import logger

# Route standard library log records into loguru
class InterceptHandler(logging.Handler):
    def emit(self, record):
        try:
            level = logger.level(record.levelname).name
        except ValueError:
            level = record.levelno
        logger.opt(depth=6, exception=record.exc_info).log(level, record.getMessage())

logging.basicConfig(handlers=[InterceptHandler()], level=0, force=True)
```

This lets third-party libraries keep using `logging` while all output flows through loguru's unified format and rotation.

---

## 4. Log Level Usage

| Level | When to Use | Example |
|---|---|---|
| **DEBUG** | Detailed internal state — only useful during development or active investigation | `"IntentRouter received: {parsed_intent}"`, `"Cache hit for key: {key}"` |
| **INFO** | Normal operation milestones worth noting in production | `"Model loaded successfully in 3.2s"`, `"Background sync started: 12 files queued"` |
| **WARNING** | Unexpected situation — system still running but something is off | `"Memory extraction timeout, skipping: file=memory.json, elapsed=5.0s"` |
| **ERROR** | An operation failed and requires attention | `"Failed to write memory file: path=memory.json, error={error}"` |
| **CRITICAL** | System cannot continue or a core component is broken | `"CUDA device not found, shutting down pipeline"` |

### Decision Rule

> If you're unsure between DEBUG and INFO, ask: *"Would I want to see this in production logs?"*  
> If **no** → use `DEBUG`.  
> If **yes** → use `INFO`.

Routine state that's only useful when something goes wrong (e.g., "checked queue — empty") belongs at DEBUG. Milestones that confirm the system is alive and healthy belong at INFO.

---

## 5. Logging in Multithreaded / Async Environments

### Why Thread Identity Matters

In a system with QThreads, background workers, and a UI thread all running simultaneously, a log like `"ERROR: file write failed"` is almost useless without knowing *which thread* produced it. Always include `{thread.name}` in the format string.

### loguru is Thread-Safe by Default

No special configuration is needed for loguru to work across threads. It handles internal synchronization automatically — one less thing to worry about when spinning up QThread workers.

> [!CAUTION]
> Using `print()` in threads while loguru is active causes **interleaved, unreadable output**. `print()` has no synchronization with loguru's output stream. Replace all `print()` calls with `logger.debug()` / `logger.info()` to maintain readable, ordered output.

### Background Task Logging Pattern

Every background thread or task should log:
1. **Start** — with its task identifier and key parameters
2. **Completion** — with elapsed time and outcome summary
3. **Any errors** — with full exception info; never swallow exceptions silently

```python
# Good background task pattern
def run_sync_task(file_path: str):
    logger.info(f"Sync task started: file={file_path}")
    start = time.time()
    try:
        _do_sync(file_path)
        elapsed = time.time() - start
        logger.info(f"Sync task completed: file={file_path}, elapsed={elapsed:.2f}s")
    except Exception:
        logger.exception(f"Sync task failed: file={file_path}")
        raise  # let the thread runner handle it
```

### filelock Integration

When using `filelock` for shared resource protection, log around lock boundaries to diagnose contention:

```python
from filelock import FileLock, Timeout
from loguru import logger

lock = FileLock("memory.json.lock", timeout=5)

logger.debug(f"Attempting to acquire lock: memory.json.lock")
try:
    with lock:
        logger.debug("Lock acquired: memory.json.lock")
        # ... do work ...
    logger.debug("Lock released: memory.json.lock")
except Timeout:
    logger.warning("Lock timeout after 5s: memory.json.lock — another process may be holding it")
```

### Recommended loguru Setup (Multithreaded App)

```python
from loguru import logger
import sys

logger.remove()  # Remove default handler

# Console: human-readable, all levels during development
logger.add(
    sys.stderr,
    format="{time:YYYY-MM-DD HH:mm:ss} | {level:<8} | {thread.name:<15} | {name}:{line} - {message}",
    level="DEBUG"
)

# File: INFO and above, daily rotation, 7-day retention
logger.add(
    "logs/app_{time:YYYY-MM-DD}.log",
    rotation="1 day",
    retention="7 days",
    level="INFO",
    encoding="utf-8"
)
```

This setup gives a verbose console for active development and a compact file log for post-hoc analysis, without any manual rotation logic.

---

## 6. Structured Log Output

### When to Use JSON vs Plain Text

| Context | Preferred Format | Reason |
|---|---|---|
| Local development | **Plain text** | Human-readable, colorized by level in terminal |
| CI / test runs | **Plain text** | Easier to read in diff output and test reporters |
| Deployed service / long-running pipeline | **JSON** | Parseable by log aggregators (ELK, Datadog, etc.) |
| Log analysis scripts | **JSON** | Enables structured queries without regex |

### loguru JSON Output

```python
logger.add(
    "logs/app_structured.log",
    serialize=True,   # emits newline-delimited JSON
    rotation="50 MB",
    retention="14 days",
    level="INFO"
)
```

Each line is a JSON object with keys like `text`, `record.time`, `record.level.name`, `record.thread.name`, `record.message`, making it straightforward to parse with `jq` or a log analysis script.

> [!NOTE]
> You can run both a plain-text handler and a JSON handler simultaneously — one for the console, one for structured file output. They don't conflict.

> [!CAUTION]
> Switching log format mid-project **breaks existing log parsers** — any regex or JSON-parsing scripts written against one format stop working when you switch. Decide on plain text vs JSON early and document the choice. Prefer JSON from day one for any service expected to run unattended.

---

## 7. Debugging Strategies

### When to Use `pdb` / `breakpoint()`

Useful for **synchronous, single-threaded code** only. The debugger pauses exactly one execution context — in a multithreaded or async system, the other threads keep running (or deadlock waiting), making the paused state artificial and misleading.

For Qt (QThread) applications: attaching `pdb` inside a QThread often causes deadlocks with the Qt event loop. Don't rely on it.

### For Async / Threaded Bugs: Log, Don't Step

The debugger can't rewind time. Strategic log statements can. When investigating a threading bug:
1. Add `logger.debug()` at each state transition and decision branch
2. Run the scenario that reproduces the bug
3. Read the log to reconstruct the interleaving of events

### Silent Failures: The Most Dangerous Pattern

```python
# ❌ Silent failure — the worst possible pattern
try:
    result = do_something()
except Exception:
    pass  # 'I'll handle it later' — you won't

# ❌ Almost as bad — loses the traceback
try:
    result = do_something()
except Exception as e:
    logger.error(f"Something failed: {e}")  # no traceback, hard to diagnose

# ✅ Correct — full traceback captured automatically
try:
    result = do_something(input_data)
except Exception:
    logger.exception(f"do_something failed: input={input_data!r}")  # auto-captures traceback
    raise  # re-raise unless you have a deliberate fallback
```

`logger.exception()` is equivalent to `logger.error()` + `exc_info=True` + formatting. Use it in every `except` block where you're logging the failure.

### Reproducing Intermittent / Timing Bugs

- Ensure every log line has a **millisecond-resolution timestamp** — second resolution is often insufficient for race conditions
- Include **thread name** in every line — lets you interleave logs from multiple threads mentally
- Look for **timing patterns**: does the bug only appear when two events are < 100ms apart? The log timestamps will show this
- Add a unique `task_id` or `request_id` to all logs for a single operation, so you can `grep task_id=abc123` to get a clean trace

### Avoiding Debug Print Statements in Production

```python
# ❌ Classic mistake — print() left in after debugging
def process(data):
    print(f"DEBUG: data={data}")  # forgotten in production
    ...

# ✅ Use logger.debug() — controlled by log level, not hardcoded
def process(data):
    logger.debug(f"process called: data={data!r}")
    ...
```

Set the log level to `INFO` in production and `DEBUG` during development. No code changes needed.

---

## 8. Common Pitfalls & Resolutions

| Problem | Root Cause | Resolution |
|---|---|---|
| Log file grows unboundedly and fills disk | No rotation configured | Use `logger.add("app.log", rotation="10 MB")` or `rotation="1 day"` with `retention="7 days"` |
| Logs from different threads are interleaved and unreadable | Using `print()` without synchronization | Replace all `print()` with `logger` calls — loguru handles thread synchronization internally |
| Can't find relevant log lines in thousands of entries | Too much logged at INFO level; no filtering handles | Demote routine status messages to DEBUG; add structured fields (e.g., `task_id`) for targeted `grep` |
| Exception occurred but no traceback in log | Bare `except: pass` or `logger.error(str(e))` (loses stack) | Always use `logger.exception()` in except blocks — it auto-captures and formats the full traceback |
| Log shows wrong timestamp or confusing time offsets | System timezone mismatch or naive datetimes | Configure loguru with explicit `format="{time:YYYY-MM-DD HH:mm:ssZ}"` or standardize on UTC across the system |
| Background thread crash is completely silent | Exception not caught inside thread target function | Wrap every thread target in `try/except Exception: logger.exception(...)` — threads don't propagate exceptions to the main thread |
| Log output missing during test runs | Logger not configured in test environment | Add `logger.add(sys.stderr, level="DEBUG")` in a test fixture or `conftest.py` setup |
| Can't tell which component produced a log line | Module name too generic (e.g., `utils`) | Use named sub-loggers: `logger.bind(component="MemoryExtractor")` and include `{extra[component]}` in format |
| Log file encoding errors on Windows | Default encoding isn't UTF-8, crashes on non-ASCII content | Always pass `encoding="utf-8"` to `logger.add()` for file handlers |
