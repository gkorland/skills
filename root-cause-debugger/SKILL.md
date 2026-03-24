---
name: root-cause-debugger
description: >
  Enforces rigorous root-cause analysis when fixing bugs, test failures, or crashes in C, Rust,
  or systems-level codebases. Use this skill whenever the task involves: debugging a failing test,
  fixing a crash or segfault, investigating wrong output or data corruption, resolving memory errors
  (leaks, use-after-free, double-free), fixing flaky or intermittent tests, or any bug report that
  says "X doesn't work." Also trigger when the user says things like "find the real cause," "don't
  just patch it," "it keeps coming back," "fix it properly," or mentions symptoms like NULL dereference,
  assertion failure, or unexpected values. This skill prevents shallow symptom-patching by forcing a
  structured investigation-first workflow. If there is ANY bug-fixing or debugging happening, use this skill.
---

# Root-Cause Debugger

You are debugging a systems-level codebase. Superficial fixes are unacceptable. Every bug you see is a symptom of a deeper issue until you prove otherwise. Your job is to find and fix the ROOT CAUSE, not silence the symptom.

## Why This Matters

LLMs have a strong bias toward the fastest path to a green test: add a NULL check, adjust an expected value, wrap something in a try-catch. These are band-aids. In C/Rust codebases with manual memory management, concurrency, and pointer aliasing, a band-aid today becomes a crash in production tomorrow. This skill exists to override that bias.

## The Protocol

Follow these phases in order. Do not skip ahead. Do not write fix code until Phase 2 is complete.

### Phase 1 — Understand Before Touching Anything

1. **Read the failing test end to end.** Understand what behavior it asserts, what inputs it uses, what the expected output is. Articulate the contract being tested in one sentence.

2. **Reproduce the failure.** Run the exact failing test in isolation. Capture full output: assertion messages, stack traces, error codes. If it fails intermittently, run it 10+ times, document the pass/fail pattern and any variance in output.

3. **Trace the execution path.** From the test's entry point, follow every function call to the point of failure. Do not skip functions because they "look fine." Read them. Read what they call. Keep going deeper until you hit the mutation or decision that produces the wrong result.

4. **Identify all callers.** For the function where the bug manifests, `grep -rn` every call site in the codebase. The root cause is often in how callers set up state, not in the function itself.

5. **Check recent git history.** Run `git log --oneline -20 -- <file>` on all files in the failure path. A recent commit may have introduced the regression. Read the diff and understand what changed and why.

### Phase 2 — Diagnose the Root Cause

Apply these rules without exception:

- **Wrong value?** Trace where it was SET, not where it was READ. Follow the data flow backwards. The corruption happened upstream.

- **NULL pointer crash?** Do NOT add a NULL check. Find WHY it's NULL — failed initialization, premature free, skipped allocation path. Fix THAT.

- **Test expects X, gets Y?** Do NOT adjust the test expectation. Determine whether the code violates the contract or the test is wrong. Prove which one with evidence from the code.

- **Fix "works" but you can't explain why the old code was wrong?** You have not found the root cause. Keep digging.

- **Adding a special-case conditional for "just this scenario"?** STOP. That is symptom-patching. The real fix is almost certainly a structural correction upstream.

#### When You're Stuck — Use Real Debugging Tools

- **GDB/LLDB:** Attach to the process. Set breakpoints at the failure point AND at every function that modifies the relevant data structure. Step through. Watch state mutations. Observe — don't guess.

- **Valgrind / ASan / MSan:** Run the failing test under `valgrind --leak-check=full` or compile with `-fsanitize=address,undefined`. Memory bugs in C hide behind "works sometimes" behavior.

- **Bisect:** If the bug is a regression and you can't pinpoint the commit, use `git bisect` with the failing test as the check condition.

- **printf-tracing (last resort):** If you must add debug prints, add them at EVERY branch point in the suspected path, not just the crash site. Remove all of them before committing.

### Phase 3 — Fix With Precision

- Fix the root cause, not the symptom. Wrong data structure → fix the data structure. Off-by-one → fix the algorithm. Broken ownership → fix the ownership model.

- **If a refactor is needed, propose it explicitly.** Do not shy away from this. Describe what needs to change structurally and why. A small, well-reasoned refactor beats scattered patches across multiple files.

- Every fix must come with a written explanation answering:
  1. What was the root cause?
  2. Why did the original code produce wrong behavior?
  3. Why does your fix correct it?
  4. What other code paths could be affected by the same underlying issue?

### Phase 4 — Verify Ruthlessly

Passing the original failing test is NECESSARY but NOT SUFFICIENT.

1. **Write additional test cases** covering:
   - The exact boundary condition that triggered the bug
   - One step below and one step above that boundary
   - Empty / NULL / zero inputs on the same code path
   - Large inputs that stress the same path
   - Concurrent access if the code is multi-threaded
   - Any other call site you found in Phase 1 step 4 that exercises the same logic

2. **Run the full test suite.** Not just the failing test. Your fix must not regress anything. If it does, your fix is wrong or incomplete — go back to Phase 2.

3. **Run under sanitizers.** Compile with ASan/UBSan and run your new tests. Zero warnings allowed.

4. **Revert test.** Verify that reverting your fix re-introduces the original failure. If it doesn't, your fix is not what fixed it — something else changed. Investigate.

### Phase 5 — Document

Before declaring done, provide:

- **Root cause summary:** 2–3 sentences a senior systems engineer would understand.
- **Blast radius:** What other modules, queries, or data paths touch the same code?
- **Regression risk:** What could break if this fix interacts poorly with future changes?

## Hard Rules

Violating any of these means the fix is not ready:

1. NEVER add a NULL/nil/error check as a fix without a traced explanation of why the value is NULL/nil/error in the first place.

2. NEVER modify a test expectation to match broken behavior.

3. NEVER say "this should fix it" without having RUN the test and confirmed it passes.

4. NEVER submit a fix you cannot explain mechanically, step by step.

5. NEVER stop at the first code change that makes the test green. Verify with additional tests and the full suite.

6. NEVER skip running the full test suite before declaring done.

7. If you're editing more than 3 files with small scattered patches, STOP. You are likely patching symptoms in multiple places instead of fixing the one root cause. Reconsider your diagnosis.

8. If your fix is `if (x == NULL) return;` or `if (x == NULL) goto cleanup;`, provide a 5-step trace showing exactly how x becomes NULL. No trace, no commit.

## FalkorDB-Specific Guidance

When working in the FalkorDB codebase, keep these in mind:

- **Redis module API constraints:** Memory allocation and freeing must follow Redis module rules. `RedisModule_Alloc` / `RedisModule_Free` have strict lifecycle requirements. A common root cause is using the wrong allocator or freeing in the wrong callback.

- **GraphBLAS internals:** Sparse matrices are mutated in place. Operations like `GrB_assign`, `GrB_extract`, `GxB_set` modify state that may be shared. If a test gets wrong graph results, trace which GraphBLAS operation mutated the matrix and whether the indices/descriptors were set up correctly.

- **Thread safety:** FalkorDB uses reader/writer threads. Data races hide behind intermittent test failures. If a test is flaky, immediately suspect thread safety. Use TSan (`-fsanitize=thread`) before anything else.

- **Memory:** FalkorDB manages memory manually. Use-after-free and double-free bugs are common root causes for "impossible" wrong values. ASan is your first tool, not your last resort.

- **Index and encoding bugs:** Many FalkorDB bugs manifest as "wrong query result" but originate in index maintenance (node/edge creation, deletion, update not propagating to the index) or in the encoding/decoding layer. Trace through the full write-then-read path.
