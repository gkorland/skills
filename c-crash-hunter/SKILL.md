---
name: c-crash-hunter
description: >
  Deep static and dynamic analysis of C codebases to find potential crashes, memory corruption,
  undefined behavior, and latent bugs before they hit production. Use this skill whenever the user
  asks to: audit C code for safety, find potential crashes or segfaults, review memory management,
  check for undefined behavior, harden a C codebase, do a security review of C code, find bugs
  proactively, scan for vulnerabilities, or any variation of "find everything that could crash."
  Also trigger when the user says "take your time," "be thorough," "scan everything," "find all
  potential issues," or "don't miss anything." This skill is designed for exhaustive multi-pass
  analysis — it will take a long time and that is expected and correct.
---

# C Crash Hunter — Exhaustive Codebase Audit

You are performing a deep, methodical audit of a C codebase to find every potential crash, memory
corruption, undefined behavior, and latent bug. This is not a quick review. You will read every
file, trace every allocation, and follow every pointer. Speed is irrelevant. Completeness is
everything.

## Mindset

Assume the code WILL crash in production under the right conditions. Your job is to find those
conditions before users do. Every function is guilty until proven safe. Every pointer is NULL
until proven otherwise. Every allocation can fail. Every index can be out of bounds. Every
concurrent access is a race condition until you verify the locking.

## Preparation

Before starting the scan:

1. **Map the codebase.** `find . -name "*.c" -o -name "*.h" | head -200` to understand the scope.
   Group files by module/subsystem. Estimate how many files you need to review. Communicate
   this to the user — "This codebase has N files across M modules. I'll work through each
   module systematically. This will take a while."

2. **Identify the memory model.** What allocator is used? (`malloc/free`, `RedisModule_Alloc/Free`,
   custom arena, `jemalloc` wrappers?) Document every allocation/free function variant in the
   codebase. These are your primary audit targets.

3. **Identify the threading model.** What threading primitives are used? (`pthread_mutex`,
   `RedisModule_ThreadSafeContext`, atomics, lock-free structures?) Document them. Every shared
   mutable state is an audit target.

4. **Identify external API boundaries.** What data enters from outside? (Redis commands, network
   input, file I/O, GraphBLAS callbacks, user-provided Cypher queries?) Every external input is
   an injection/overflow/corruption vector.

5. **Build with all sanitizers enabled.** Compile the project with:
   ```
   -fsanitize=address,undefined,leak -fno-omit-frame-pointer -g -O1
   ```
   Run the existing test suite under sanitizers FIRST. Capture any existing violations — these
   are freebies. Log them all before you start manual review.

6. **Run static analysis tools if available.** Try `cppcheck`, `scan-build` (clang static analyzer),
   or `gcc -fanalyzer` on the codebase. Log all findings. These are your starting points, not
   your finish line — static analyzers miss a LOT.

## The Eight-Pass Scan

Work through the codebase in eight focused passes. Each pass looks for one category of bug.
Do NOT try to find everything in a single read — you will miss things. Dedicated passes with
a single focus are far more effective.

### Pass 1 — NULL Pointer Dereferences

For every function, check:

- Every pointer parameter: can the caller pass NULL? If yes, is it checked before first use?
  `grep` for all call sites to verify what callers actually pass.
- Every function that returns a pointer (`malloc`, `calloc`, `strdup`, custom allocators,
  lookup functions, `RedisModule_OpenKey`, etc.): is the return value checked before use?
- Every struct member access through a pointer (`obj->field`): has `obj` been verified non-NULL
  on this code path? Pay special attention to early-return paths where initialization is skipped.
- Every array element used as a pointer (`arr[i]->field`): is `arr[i]` guaranteed non-NULL?
- Conditional chains: `if (a && a->b && a->b->c)` is fine, but `if (a->b && a)` is a crash.
  Check the order.

**Common hiding spots:**
- Error cleanup paths (`goto cleanup` where some pointers were never initialized)
- Optional/nullable struct fields that are only sometimes populated
- Return values from functions that return NULL on "not found" vs. "error"
- Callbacks where the `void *privdata` / `void *ctx` might be NULL

**Output format for each finding:**
```
[NULL-DEREF] file.c:123 — function_name()
  Risk: <high/medium/low>
  Path: caller_a() → function_name(ptr) where ptr can be NULL when <condition>
  Evidence: <the specific line and why it's reachable with NULL>
  Fix: <suggested fix>
```

### Pass 2 — Memory Management Bugs

For every `malloc`/`calloc`/`realloc`/`strdup`/`RedisModule_Alloc`/custom allocation:

- **Unchecked allocation:** Is the return value checked for NULL/failure? Under memory pressure
  every allocation can fail.
- **Use after free:** After any `free(ptr)`, is `ptr` used again? Search for the variable name
  after every free. Check ALL code paths, including error branches.
- **Double free:** Can any code path reach `free(ptr)` twice? Common in error handling with
  `goto cleanup` where the same resource is freed in the normal path and the error path.
- **Missing free (leaks):** Trace every allocation to its corresponding free. Every allocated
  pointer must be freed on EVERY code path — normal return, early return, error return, and
  exception/longjmp paths. Draw the ownership graph if needed.
- **Realloc pitfalls:** `ptr = realloc(ptr, new_size)` — if realloc fails, the original pointer
  is leaked. Must use a temp variable.
- **Mismatched allocator/deallocator:** `RedisModule_Alloc` freed with `free()` or vice versa.
  `malloc` freed with `RedisModule_Free`. Every pair must match.
- **Stack pointer escape:** Is the address of a local variable stored somewhere that outlives
  the function? `obj->name = local_buffer;` where `local_buffer` is a stack array.
- **Buffer size tracking:** When a buffer is allocated with size N, is that size faithfully
  propagated and checked everywhere the buffer is used? Look for `sizeof` mismatches,
  especially `sizeof(ptr)` vs `sizeof(*ptr)` vs `sizeof(type)`.

### Pass 3 — Buffer Overflows and Out-of-Bounds Access

- **String operations:** Every `strcpy`, `strcat`, `sprintf`, `gets` is a probable overflow.
  Check that destination buffers are large enough. Prefer `strncpy`, `snprintf`, `strncat`
  and verify the size argument is correct.
- **Array indexing:** For every `arr[i]`, verify `i` is bounds-checked. Trace where `i` comes
  from — if it's derived from external input, user data, or a computation, it MUST be validated.
- **memcpy/memmove size:** Is the size argument correct? Does it exceed either the source or
  destination buffer? Is it derived from untrusted input?
- **Off-by-one:** Fencepost errors in loop bounds (`<=` vs `<`), string termination (forgetting
  space for `\0`), array sizing.
- **Integer overflow leading to small allocation:** `size = count * element_size` — can this
  overflow, producing a small buffer that is then overflowed by the write?
- **VLA (Variable Length Arrays):** Any `int arr[n]` where `n` is user-controlled can blow the
  stack. Find them all.

### Pass 4 — Integer Bugs

- **Signed/unsigned mismatch:** Comparison of signed and unsigned integers. A negative signed
  value becomes a huge unsigned value. Common in size checks: `if (len < max)` where `len` is
  `int` and `max` is `size_t`.
- **Integer overflow/underflow:** Arithmetic on sizes, counts, indices. `a + b` can wrap around.
  `a * b` can overflow. `a - b` can underflow if `b > a` and the type is unsigned.
- **Truncation:** Assigning a 64-bit value to a 32-bit variable. `size_t` to `int`. `long` to
  `int`. Common when interfacing between APIs that use different integer widths.
- **Division by zero:** Every `/` and `%` operator — can the divisor be zero? Trace its origin.
- **Shift overflow:** `1 << n` where `n >= 32` (or 64) is undefined behavior. `n` might come
  from user input or a computation.

### Pass 5 — Concurrency and Thread Safety

- **Shared mutable state without locks:** Identify every global variable, static variable, and
  struct field that is accessed from multiple threads. Verify each access is protected by a
  mutex, atomic, or other synchronization primitive.
- **Lock ordering:** If multiple locks are used, is the acquisition order consistent? Inconsistent
  ordering causes deadlocks.
- **TOCTOU (Time of Check, Time of Use):** `if (condition) { /* use assumption from condition */ }`
  — can another thread invalidate the condition between check and use?
- **Missing lock scope:** Is the lock held for the entire critical section? Common bug: lock is
  released before all shared state updates are complete.
- **Signal safety:** If signal handlers are used, do they only call async-signal-safe functions?
  `malloc`, `printf`, and most library functions are NOT safe in signal handlers.
- **Reader/writer hazards:** If FalkorDB uses reader/writer threads, check that writers don't
  mutate structures that readers are iterating. Check that readers don't cache pointers to
  structures that writers can reallocate or free.

### Pass 6 — Undefined Behavior and Compiler Traps

- **Uninitialized variables:** Every local variable must be initialized before use on ALL code
  paths. Pay extra attention to variables declared at the top of a function but only initialized
  inside a conditional branch.
- **Strict aliasing violations:** Casting between incompatible pointer types and then
  dereferencing. `*(int*)float_ptr` is UB. Type-punning through unions is the safe alternative.
- **Sequence point violations:** `a[i] = i++;` is UB. Multiple modifications of the same
  variable without an intervening sequence point.
- **Null pointer arithmetic:** `ptr + offset` where `ptr` is NULL is UB even if the result is
  never dereferenced.
- **Signed overflow:** `INT_MAX + 1` is UB in C (unsigned overflow is defined, signed is not).
  Any arithmetic on signed integers from external input can trigger this.
- **Pointer to expired storage:** Returning a pointer to a local variable. Storing a pointer
  to a local across a function call that might `longjmp`.
- **Restrict qualifier violations:** If `restrict` is used, verify that the pointed-to memory
  regions genuinely do not overlap.

### Pass 7 — Error Handling Gaps

- **Ignored return values:** Every function that can fail must have its return value checked.
  `grep` for function calls whose return value is discarded. Pay special attention to I/O
  operations, allocation, and system calls.
- **Partial cleanup on error:** When a function acquires multiple resources (memory, locks,
  file handles), verify that ALL acquired resources are released on EVERY error path. Draw the
  resource acquisition/release graph if the function is complex.
- **Error code propagation:** When function A calls function B which can fail, does A propagate
  the error to its caller? Or does it silently swallow the error and continue with invalid state?
- **Inconsistent error conventions:** Does the codebase mix `return -1`, `return NULL`,
  `return false`, `return REDIS_ERR`? Verify that each function's callers check for the correct
  error indicator.
- **Cleanup code that can itself fail:** `fclose()` can fail. `free()` on a corrupted pointer
  can crash. Error cleanup paths must be robust.

### Pass 8 — API Contract Violations (FalkorDB-Specific)

- **Redis Module API misuse:** `RedisModule_ReplyWith*` called after the client has disconnected.
  `RedisModule_OpenKey` used without checking the return. Blocked client APIs used incorrectly.
  Module commands that don't return a reply on all paths.
- **GraphBLAS contract violations:** Operations on finalized/freed matrices. Wrong descriptor
  arguments. Type mismatches between matrix element types and operation types. Missing
  `GrB_wait()` before reading results of pending operations.
- **Encoding/decoding symmetry:** For every `Encode_X()` there must be a matching `Decode_X()`
  that correctly reverses it. Version mismatches in encoded data. Missing length checks when
  decoding from a buffer.
- **Index maintenance invariants:** When nodes/edges are created, updated, or deleted, all
  relevant indices must be updated atomically. Check for paths where an entity is modified but
  its index entry is stale.
- **Query lifecycle:** Verify that query execution contexts, result sets, and execution plans
  are properly initialized before use and properly freed after use, even when queries are
  aborted, timed out, or hit errors mid-execution.

## Output Requirements

### Per-Finding Format

Every finding must include:

```
[CATEGORY] file.c:line — function_name()
  Severity: CRITICAL / HIGH / MEDIUM / LOW
  Bug class: <null-deref | use-after-free | double-free | leak | overflow | race | UB | etc.>
  Path to trigger: <the specific sequence of calls/conditions that triggers this>
  Evidence: <the exact lines of code with explanation>
  Suggested fix: <concrete code change or refactor>
  Blast radius: <what else might be affected>
```

### Summary Report

After all eight passes, produce a summary:

1. **Critical findings** (will crash or corrupt in production — fix immediately)
2. **High findings** (will crash under specific but realistic conditions)
3. **Medium findings** (latent bugs that could become crashes with future code changes)
4. **Low findings** (code smells, defensive hardening opportunities, style issues that reduce safety)
5. **Structural recommendations** (refactors that would eliminate entire classes of bugs)
6. **Statistics** (files reviewed, findings per category, modules with highest bug density)

### Honesty Clause

If you are unable to fully analyze a section of code (too complex, too many indirections, can't
determine thread safety without runtime info), say so explicitly. "I could not determine whether
X is safe because Y" is infinitely better than silently skipping it. Mark these as
`[NEEDS-MANUAL-REVIEW]` in your output.

## Execution Guidelines

- **Work module by module.** Announce which module you're starting. List the files. Work through
  them. Announce when you're done with that module.
- **Do not summarize files you haven't read.** If you say a file is clean, you must have read it.
- **Take as long as needed.** The user has explicitly asked for thoroughness. Do not truncate,
  abbreviate, or skip. If you need 50 messages to cover the codebase, take 50 messages.
- **Keep a running tally.** After each module, update the running count of findings by severity.
- **Cross-reference between passes.** A NULL that you flagged in Pass 1 might be the same pointer
  that's use-after-free in Pass 2. Deduplicate and link related findings.
- **Prioritize the most dangerous modules first.** Code that handles external input (query
  parsing, command handling, network I/O) should be scanned before internal utility code.
