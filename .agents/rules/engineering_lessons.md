# Engineering Lessons & Design Guidelines (PR Review Feedback)

## 1. AI Decision Budget & Loop Safety
- **Strict Bounds on Sub-Queries**: When paging or fanning out decisions (e.g. candidate pagination), enforce a hard cap (e.g. `MAX_TARGET_PAGES = 3`) to prevent exceeding upstream request budgets.
- **Dual Loop Protection**: Bound loops using both execution counts (history items) and snapshot/iteration counts (e.g. `max_iterations = max(max_steps * 2, 10)`) so stale-page retries cannot spin indefinitely.

## 2. Preservation of Model Probability Distributions
- **Never Flatten Probabilities**: Do not artificially distribute or average out model probabilities across unchosen candidates. Preserve raw per-candidate probabilities and signals (e.g., runner-up confidence).
- **Zero Padding for Unobserved Candidates**: If candidates were not evaluated on the final decision cycle, pad them with `0.0`, never fabricated non-zero probabilities.

## 3. Agent Protocol & MCP Reliability
- **Outcome Verification**: Never treat a model's self-reported `DONE` choice as verified success without an independent contract assertion. Explicitly flag unverified results (`verified: false`).
- **Surface Tool Errors**: Set `result.isError = true` on any execution failure so clients do not misinterpret errors as success.
- **Protocol Error Compliance**: On malformed inputs (invalid JSON), immediately respond with standard JSON-RPC 2.0 error codes (`-32700 Parse error`).
- **Input Guarding & Early Return**: Cast and clamp parameters upfront. Return immediately before starting browsers or heavy agents if `max_steps <= 0`.

## 4. Multi-Platform & Enterprise Windows Compatibility
- **Windows vs POSIX Layout**: Always provide Windows-specific virtualenv paths (`.venv\Scripts\python.exe`) and explain backslash escaping in JSON configurations alongside POSIX examples.
- **Non-Interactive Batch Files**: Never include `pause` in batch files designed for automated execution or Task Scheduler.
- **Standard User Privilege**: Avoid instructing users to register scheduled tasks as administrator if standard corporate user logons are targeted.

## 5. Repository & Documentation Hygiene
- **Root-Anchor Ignore Rules**: Anchor directory ignore rules (e.g. `/tasks/`) to prevent unintended exclusion of nested packages.
- **Placeholder Paths**: Avoid committing personal environment paths or personal fork URLs in public documentation.
