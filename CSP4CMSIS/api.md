# FreeRTOS Stack Usage Estimation — Implementation Brief

Context: this is background/methodology for implementing an actual stack-usage
measurement for CSP4CMSIS, in response to a book reviewer's request for
"additional memory footprint and stack requirements for CSP4CMSIS." The
companion chat (book review responses) covers the reviewer correspondence;
this document is scoped to the stack-estimation implementation itself.

---

## Core tool: `uxTaskGetStackHighWaterMark()`

```c
UBaseType_t uxTaskGetStackHighWaterMark( TaskHandle_t xTask );
```

Returns the minimum amount of stack that has *ever* remained free since the
task started running — i.e., the closest the task has come to overflowing,
over its whole execution history so far.

- The return value is in **words, not bytes** (on a 32-bit Cortex-M, 1 word =
  4 bytes). Easy unit mistake to make when reporting numbers.
- `uxTaskGetStackHighWaterMark2()` is identical except its return type
  respects `configSTACK_DEPTH_TYPE`, avoiding overflow on 8-bit targets —
  irrelevant for Cortex-M but exists.

**How it works mechanically:** at task creation, FreeRTOS fills the task's
stack with a known pattern. As the task runs, the high-water-mark function
walks in from the "far end" of the stack and measures how much of that
original pattern is still untouched — the untouched portion is stack that
was never used. The measurement is automatically the *worst point observed
so far*, not just current usage — but only if the task has actually been
driven through its worst-case call path before reading it.

**Requirements:**
- `INCLUDE_uxTaskGetStackHighWaterMark` must be set to `1` in
  `FreeRTOSConfig.h`.
- Works for both `xTaskCreate()` and `xTaskCreateStatic()` — applies
  directly to CSP4CMSIS's statically-allocated processes.

## Related mechanisms

- **`configCHECK_FOR_STACK_OVERFLOW` (0/1/2/3)** — a runtime safety net, not
  a measurement tool.
  - Method 1: checks the stack pointer is still in range at each context
    switch (cheap, can miss overflows between switches).
  - Method 2: additionally checks a small pattern at the end of the stack
    hasn't been corrupted (catches more, costs more per switch).
  - Pair this with high-water-mark readings during development: if
    `vApplicationStackOverflowHook()` fires while measuring, the allocated
    stack was already too small for the workload just run.

- **`vTaskList()` / `uxTaskGetSystemState()`** — dump a `TaskStatus_t` array
  for *every* task in the system in one call. `TaskStatus_t` includes
  `usStackHighWaterMark` as a field. For a CSP4CMSIS example network with
  several processes (e.g., Sender A, Sender B, Receiver, reporter process),
  this gives a single table for the whole network after a soak-test run.

- **Percepio Tracealyzer / `configUSE_TRACE_FACILITY`** — heavier tooling,
  gives a visual timeline of stack usage over time rather than a single
  worst-observed number. Optional, likely out of scope for a first pass.

## Caveat that matters most

This is a **measured**, not **proven**, worst case. The number is only
trustworthy if the test run actually exercised the deepest call chain — for
a CSP4CMSIS process, that likely means triggering:
- the `%f`-formatting `printf` path (heaviest call chain for output),
- the deepest nested channel handshake / `Alternative::fairSelect()` call,
- any ISR preemption at its worst nesting,

all within the same soak run. Any reported number must be explicit about
what worst-case path was actually exercised to produce it — otherwise a
reader could mistake a measured figure for a guaranteed bound.

## Practical recipe

1. Enable `INCLUDE_uxTaskGetStackHighWaterMark` in `FreeRTOSConfig.h`.
2. Run an existing CSP4CMSIS example (e.g., the two-sender/receiver
   alternation example, or the gyro example) for a soak period that
   includes a debug `printf` on the floating-point path.
3. Call `vTaskList()` (or iterate `uxTaskGetStackHighWaterMark()` per task)
   once at the end of the soak run.
4. Report, per process: `allocated_stack_bytes - high_water_mark_words * 4`.

## Open items for the implementation project

- Confirm arm-none-eabi-gcc toolchain availability / install path.
- Obtain CSP4CMSIS source (repo access) and a target example project
  (e.g., NUCLEO-G474RE-based alternation example).
- Decide whether to also implement the static side (memory footprint via
  `sizeof()` table + linker map cross-check + differential build vs. raw
  FreeRTOS) as a companion measurement, per the earlier methodology
  discussion.
