# Agent Zero v1.2-v1.6 -- Memory Extension Vulnerabilities Persist Across Versions

**Report Author:** MrTrench (CONTAK Production Environment)
**Date:** 2026-04-01
**Agent Zero Versions Tested:** v1.2, v1.6
**Severity:** High -- Message loop crash, service unavailability
**Repository:** [agent0ai/agent-zero](https://github.com/agent0ai/agent-zero)
**Previous Report:** [Agent Zero NVIDIA Routing Fix (v1.2)](https://github.com/MrTrenchTrucker/agent-zero-nvidia-routing-fix)

---

## 1. Executive Summary

In our previous whitepaper, we documented critical memory extension failures in Agent Zero v1.2 caused by an empty `utility_model.api_base` field. During that investigation, we developed patches for three memory extension files to add `AuthenticationError` guards, increased the `SEARCH_TIMEOUT` from 30 to 120 seconds, and deployed a patch persistence mechanism.

After upgrading our production deployment to v1.6, we conducted a side-by-side comparison of the stock v1.6 image against the stock v1.2 image. Our finding:

**Agent Zero v1.6 made zero changes to any memory extension file.** The four memory extension files in `/plugins/_memory/extensions/python/` are byte-identical between v1.2 and v1.6.

This means:
- The `AuthenticationError` vulnerability documented in our v1.2 report is still present in stock v1.6
- The bare `await task` crash in `_91_recall_wait.py` is still present in stock v1.6
- The 30-second `SEARCH_TIMEOUT` (too short for remote API endpoints) is unchanged
- Every production deployment of Agent Zero v1.2 through v1.6 using remote API providers is vulnerable to message loop crashes during memory operations

Additionally, during our v1.6 testing, we discovered a new vulnerability in `_91_recall_wait.py` that was not identified in our v1.2 report: the bare `await task` on the memory recall background task has no error handling whatsoever, causing guaranteed message loop crashes on every memory recall timeout.

---

## 2. Relationship to Previous Report

Our v1.2 whitepaper (https://github.com/MrTrenchTrucker/agent-zero-nvidia-routing-fix) documented:

- Empty `utility_model.api_base` caused `AuthenticationError` floods across all memory extensions
- Agent Zero's self-repair mechanism wrote patches with incorrect indentation (1-space vs 8-space), breaking the extensions entirely
- We manually patched three files with `AuthenticationError` guards and increased `SEARCH_TIMEOUT` from 30s to 120s
- We created a `ctx_init.sh` startup hook to persist patches across container restarts

We expected v1.6 to incorporate fixes for these issues. It did not. The memory extension code is unchanged. Our production deployment has been running on our custom patches for the entire period -- the upgrade to v1.6 simply swapped the application code while our persistent volume mounts preserved the patched extension files.

---

## 3. Verification Methodology

To confirm our findings, we performed a clean comparison using Docker images with no volume mounts:

```bash
# Create clean containers from stock images (no volume mounts)
docker create --name a0-v12-clean agent0ai/agent-zero:<v1.2-image-id>
docker create --name a0-v16-clean agent0ai/agent-zero:latest  # v1.6

# Extract stock files from /git/agent-zero/ (pre-copy_A0.sh source)
docker cp a0-v12-clean:/git/agent-zero/plugins/_memory/... /tmp/v12_stock/
docker cp a0-v16-clean:/git/agent-zero/plugins/_memory/... /tmp/v16_stock/
```

The files at `/git/agent-zero/plugins/_memory/` represent the stock code before `copy_A0.sh` populates `/a0/` at runtime. Comparing these across versions eliminates any contamination from persistent volume mounts or runtime patches.

---

## 4. Comparison Results -- Stock v1.2 vs Stock v1.6

### `_50_recall_memories.py` (memory search and recall)

| Feature | Stock v1.2 | Stock v1.6 | Changed? |
|---------|-----------|-----------|----------|
| `SEARCH_TIMEOUT` | 30 seconds | 30 seconds | **No** |
| `AuthenticationError` import | Not present | Not present | **No** |
| `AuthenticationError` catch | Not present | Not present | **No** |
| General `except Exception` on query prep | Present | Present | No |
| General `except Exception` on filter prep | Present | Present | No |
| `asyncio.wait_for` wrapper | Present | Present | No |

**Conclusion:** Identical. No changes between versions.

### `_91_recall_wait.py` (memory recall synchronization)

| Feature | Stock v1.2 | Stock v1.6 | Changed? |
|---------|-----------|-----------|----------|
| Error handling on `await task` | **None -- bare await** | **None -- bare await** | **No** |
| `asyncio.TimeoutError` catch | Not present | Not present | **No** |
| `asyncio.CancelledError` catch | Not present | Not present | **No** |
| General exception catch | Not present | Not present | **No** |

**Conclusion:** Identical. The bare `await task` crash vulnerability is present in both versions.

### `_50_memorize_fragments.py` (fragment memorization)

| Feature | Stock v1.2 | Stock v1.6 | Changed? |
|---------|-----------|-----------|----------|
| `AuthenticationError` import | Not present | Not present | **No** |
| `AuthenticationError` catch | Not present | Not present | **No** |
| General `except Exception` catches | 3 present | 3 present | No |

**Conclusion:** Identical. Has general exception catches but no `AuthenticationError`-specific handling.

### `_51_memorize_solutions.py` (solution memorization)

| Feature | Stock v1.2 | Stock v1.6 | Changed? |
|---------|-----------|-----------|----------|
| `AuthenticationError` import | Not present | Not present | **No** |
| `AuthenticationError` catch | Not present | Not present | **No** |
| General `except Exception` catches | 3 present | 3 present | No |

**Conclusion:** Identical.

### Overall

**All four memory extension files are byte-identical between stock v1.2 and stock v1.6.** None of the issues reported in our v1.2 whitepaper have been addressed in v1.6.

---

## 5. Vulnerabilities Present in Stock v1.2 Through v1.6

### Vulnerability 1: No `AuthenticationError` Handling (All Memory Extensions)

**Affected files:** `_50_recall_memories.py`, `_50_memorize_fragments.py`, `_51_memorize_solutions.py`

**Description:** None of the memory extension files that call `call_utility_model` import or catch `AuthenticationError` from litellm. When the utility model's `api_base` is misconfigured (empty string, wrong endpoint, or provider mismatch), every background memory operation floods with `AuthenticationError` exceptions.

While the general `except Exception` blocks in some of these files do catch `AuthenticationError` as a subclass, they provide no specific handling -- no diagnostic logging that identifies the API routing misconfiguration, no distinction between a transient failure and a persistent configuration error.

**Impact:** With an empty `utility_model.api_base`, the agent generates hundreds of `AuthenticationError` exceptions per session. The generic exception handlers swallow them silently, and the agent operates without functional memory -- no recall, no memorization, no solutions. The operator has no indication that memory is completely non-functional unless they inspect container logs.

**Our patch (applied in v1.2, still running):** Added explicit `AuthenticationError` imports and catches with diagnostic logging that identifies the specific misconfiguration (e.g., NVIDIA API key sent to OpenAI endpoint). This allows operators to immediately identify and fix the routing issue.

### Vulnerability 2: `_91_recall_wait.py` -- Bare `await` Crashes Message Loop

**Affected file:** `_91_recall_wait.py`

**Description:** This extension synchronizes the memory recall pipeline by awaiting the background search task initiated by `_50_recall_memories.py`. The await is completely unguarded:

```python
# otherwise await the task
await task
```

The `_50_recall_memories.py` extension wraps its memory search in `asyncio.wait_for(timeout=SEARCH_TIMEOUT)`. When the timeout fires, the task is cancelled via `asyncio.CancelledError`. This `CancelledError` propagates through the bare `await task` in `_91_recall_wait.py`, up through the extension pipeline, and crashes the message loop.

`CancelledError` is a `BaseException`, not an `Exception`. The general `except Exception` blocks elsewhere in the pipeline do not catch it. The crash is deterministic -- every memory recall timeout guarantees it.

**Impact:** The agent becomes completely unresponsive after any memory recall timeout. Recovery requires a chat reset or container restart. In production environments with remote API endpoints where timeouts are routine, this makes Agent Zero unusable for extended sessions.

**Our patch (new, applied in v1.6 testing):** Wrapped the `await task` in a try/except that catches `asyncio.TimeoutError`, `asyncio.CancelledError`, and general `Exception`. The agent logs a warning and continues without memory context rather than crashing.

### Vulnerability 3: `SEARCH_TIMEOUT` Too Short for Remote Endpoints

**Affected file:** `_50_recall_memories.py`

**Description:** The stock `SEARCH_TIMEOUT = 30` (seconds) is adequate for local inference servers but insufficient for remote API endpoints. Cloud-hosted LLM providers routinely take 15-45 seconds for utility model responses, especially under load. Combined with Vulnerability 2, the 30-second timeout creates a frequent crash cycle: timeout fires -> `CancelledError` -> message loop crash.

**Our patch (applied in v1.2):** Increased `SEARCH_TIMEOUT` from 30 to 120 seconds. This gives remote endpoints adequate time to respond while still providing a bounded wait that prevents indefinite hangs.

**Recommendation:** Make this configurable via the plugin settings rather than hardcoding it.

---

## 6. What v1.6 Actually Changed

While the memory extensions are untouched, v1.6 did ship meaningful improvements in other areas:

- **v1.3:** Chat compaction plugin, WebUI extension points
- **v1.4:** Onboarding flow for missing API keys, model config presets
- **v1.5:** WebSocket architecture rewrite, memory leak fixes, `a0_small` prompt set
- **v1.6:** WhatsApp integration, **"Settings: API keys no longer overwritten on save"** (important fix for the model config persistence issue we documented in v1.2)

The v1.6 settings fix is significant -- in v1.2, saving any setting in the web UI would clobber API keys that had been entered separately. This was a pain point we documented. The v1.6 fix addresses it correctly.

However, none of these improvements touch the memory extension pipeline.

---

## 7. Our Complete Patch Set

The following patches have been running in our production deployment since the v1.2 investigation and continue to run on v1.6. They are applied via a `ctx_init.sh` startup hook that pre-places patched files before `copy_A0.sh`'s no-clobber copy runs.

### Patch 1: `_50_recall_memories.py`

- Import `AuthenticationError` from litellm
- Add `AuthenticationError` catch on query prep `call_utility_model` with diagnostic logging
- Increase `SEARCH_TIMEOUT` from 30 to 120 seconds

### Patch 2: `_51_memorize_solutions.py`

- Import `AuthenticationError` from litellm
- Add `AuthenticationError` catch on `call_utility_model` with diagnostic logging
- Add general `except Exception` fallback after `AuthenticationError` catch

### Patch 3: `_50_memorize_fragments.py`

- Import `AuthenticationError` from litellm
- Add `AuthenticationError` catch on `call_utility_model` with diagnostic logging
- Add general `except Exception` fallback after `AuthenticationError` catch (consistent with Patch 2)

### Patch 4: `_91_recall_wait.py` (NEW -- discovered during v1.6 testing)

- Import `asyncio`
- Replace bare `await task` with try/except handling:
  - `asyncio.TimeoutError` -> log warning, continue without memory
  - `asyncio.CancelledError` -> log warning, continue without memory
  - `Exception` -> log error, continue without memory

---

## 8. Test Results

### Production Environment

Testing conducted on a containerized Agent Zero deployment with remote API endpoints (variable latency, 200ms+ round-trip times).

### Before Any Patches (stock v1.2 and stock v1.6 -- identical behavior)

| Metric | Result |
|--------|--------|
| `AuthenticationError` on misconfigured `api_base` | Unhandled -- floods logs, memory silently non-functional |
| Memory recall timeout (30s) | `CancelledError` -> message loop crash |
| Agent availability during timeout events | 0% -- requires restart |

### After Patches Applied

| Metric | Result |
|--------|--------|
| `AuthenticationError` handling | Caught with diagnostic logging, agent continues |
| Memory recall timeout (120s) | Caught gracefully, agent continues without memory context |
| New tracebacks during stress test | 0 |
| Graceful catches logged | 22 |
| Agent availability | 100% -- no restarts required |
| Task completion | All tasks completed normally |

### Stress Test Protocol

Forced all failure modes simultaneously:
- Memory recall for nonexistent data (forces full search + timeout path)
- Solution memorization via utility model
- Fragment memorization via utility model
- Sequential recall-after-memorize operations
- Repeated cycles under APU Governor throttling

All operations completed. Failed memory operations logged as warnings. Agent continued operating with degraded (no-memory) context. Zero crashes.

---

## 9. Corrected Code -- Appendix

### A. `_50_recall_memories.py` -- AuthenticationError Guard + Timeout Increase

Add to imports:
```python
import litellm
from litellm.exceptions import AuthenticationError
```

Change timeout constant:
```python
SEARCH_TIMEOUT = 120  # was 30
```

Add after `call_utility_model` in query prep section (wrap existing try/except):
```python
            except AuthenticationError as e:
                if "nvapi-" in str(e):
                    log_item.update(heading="SKIPPED: Invalid API Key (NVIDIA key on OpenAI endpoint)")
                    log_item.update(content="Memory recall skipped due to configuration error. "
                                            "Please fix provider routing (api_base).")
                    query = ""
                else:
                    raise
            except Exception as e:
                err = errors.format_error(e)
                self.agent.context.log.log(
                    type="warning", heading="Recall memories extension error:", content=err
                )
                query = ""
```

### B. `_91_recall_wait.py` -- Error Handling on Await

Add to imports:
```python
import asyncio
```

Replace bare `await task` with:
```python
            try:
                await task
            except asyncio.TimeoutError:
                self.agent.context.log.log(
                    type="warning",
                    heading="Memory recall timed out",
                    content="The memory search took too long and was cancelled. Continuing without memory context.",
                )
            except asyncio.CancelledError:
                self.agent.context.log.log(
                    type="warning",
                    heading="Memory recall cancelled",
                    content="The memory search was cancelled. Continuing without memory context.",
                )
            except Exception as e:
                from helpers import errors
                self.agent.context.log.log(
                    type="warning",
                    heading="Memory recall error",
                    content=errors.format_error(e),
                )
```

### C. `_50_memorize_fragments.py` -- AuthenticationError Guard

Add to imports:
```python
from litellm.exceptions import AuthenticationError
```

Add around `call_utility_model`:
```python
            except AuthenticationError as e:
                if "nvapi-" in str(e):
                    log_item.update(heading="SKIPPED: Invalid API Key")
                    log_item.update(content="Memorization skipped due to configuration error.")
                    return
                else:
                    raise
            except Exception as e:
                err = errors.format_error(e)
                self.agent.context.log.log(
                    type="warning",
                    heading="Memorize fragments extension error:",
                    content=err,
                )
                return
```

### D. `_51_memorize_solutions.py` -- AuthenticationError Guard

Same pattern as Patch C applied to the solutions memorization `call_utility_model`.

---

## 10. Recommendations for the Agent Zero Development Team

### Immediate (v1.6.x or v1.7)

1. **Apply the `_91_recall_wait.py` fix.** This is a guaranteed crash on every memory recall timeout. The fix is 15 lines. There is no reason for this to remain unfixed.

2. **Add `AuthenticationError` handling to all `call_utility_model` call sites.** The generic `except Exception` catches it technically, but without diagnostic logging the operator cannot distinguish a configuration error from a transient failure.

3. **Make `SEARCH_TIMEOUT` configurable.** Hardcoded at 30 seconds, it is too short for any remote API deployment. Expose it as a plugin setting.

### Structural (v1.7+)

4. **Centralize utility model error handling.** Provide a `safe_call_utility_model` wrapper that handles `AuthenticationError` and general exceptions internally. This eliminates the class of bugs where extensions forget to add error handling.

5. **Provide a `safe_await_task` utility.** Any background task that can be cancelled via `asyncio.wait_for` needs error handling at the await site. A shared utility function prevents the `_91_recall_wait.py` class of bugs.

6. **Add integration tests for the timeout path.** The `asyncio.wait_for` in `_50_recall_memories.py` is a critical feature, but its interaction with `_91_recall_wait.py` was never tested.

### Documentation

7. **Document the error handling contract for `call_utility_model`.** Extension developers need to know what exceptions it can raise and how to handle them. A brief section in the extension development guide would prevent this class of bug in community extensions.

---

## References

- Previous whitepaper (v1.2): https://github.com/MrTrenchTrucker/agent-zero-nvidia-routing-fix
- Agent Zero repository: https://github.com/agent0ai/agent-zero
- Python asyncio.wait_for: https://docs.python.org/3/library/asyncio-task.html#asyncio.wait_for
- Python asyncio.CancelledError: https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancel

---

*This report was produced from a production Agent Zero deployment running continuous workloads since early 2026. All findings were verified by extracting stock files from clean Docker images with no volume mounts, ensuring no contamination from our production patches. The vulnerabilities described here are reproducible on any stock Agent Zero v1.2 through v1.6 deployment using remote API endpoints.*
