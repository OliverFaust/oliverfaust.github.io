# CSP4CMSIS API Reference (v1.3)

This document provides a technical summary of the core primitives used in **CSP4CMSIS**. Version 1.3 adds **process-level stack usage introspection** and a **network-wide reporting helper** on `ParallelHelper`, and **removes the single-process `Run(CSProcess&, priority)` overload** in favor of `Run(InParallel(process), mode)`. All v1.2 functionality — Buffer Policies, channels, ALT, ISR handling, per-process stack/priority declarations — is unchanged except where noted; this document folds in v1.2's content and adds the new material in Sections 1 and 4.

> **Migrating from v1.2?** Read [Section 8 — What Changed in v1.3](#8-what-changed-in-v13) first. Unlike the v1.2 migration, **this one is not fully source-compatible**: any code calling the single-process `Run(CSProcess&, priority)` overload will no longer compile. The fix is small and mechanical — see the migration checklist in Section 8.
>
> **Migrating from v1.1?** Read [Section 7 — What Changed in v1.2](#7-what-changed-in-v12) first, then Section 8.

---

## 1. Process Management

### CSProcess
The fundamental unit of execution. Every task in your network must inherit from this class.

* **Usage**: Inherit from `CSProcess` and implement the `run()` method.
* **Lifecycle**: The `run()` method contains the process logic. In `StaticNetwork` mode, these are mapped to persistent RTOS tasks.

```cpp
class MyProcess : public CSProcess {
public:
    void run() override {
        while(true) {
            // Process Logic (Reading from channels, etc.)
        }
    }
};
```

### Declaring Resource Requirements (Since v1.2)

`CSProcess` exposes two overridable methods so a process can declare what it needs, independent of where it appears in an `InParallel(...)` argument list:

```cpp
virtual size_t stackWords() const;      // FreeRTOS task stack size, in words
virtual UBaseType_t taskPriority() const; // FreeRTOS task priority
```

Both are optional. If you don't override them, the process falls back to the same default your v1.1 code already relied on — see [Section 7](#7-what-changed-in-v12) for exactly what those defaults are and why they're safe to leave alone.

Override them when a specific process has real requirements — most commonly, a process with a deep call chain (e.g. a neural-network inference stage calling through an interpreter into vendor NPU drivers) or one that needs to run above or below the rest of the composition:

```cpp
class InferenceProcess : public CSProcess {
public:
    // This process needs more stack than the composition default,
    // and should run one priority level above its peers.
    size_t stackWords() const override { return 4 * 2048; }
    UBaseType_t taskPriority() const override { return tskIDLE_PRIORITY + 3; }

    void run() override { /* ... */ }
};
```

**Position in `InParallel(...)` no longer matters for stack or priority.** Every process — regardless of where it's listed — is spawned with exactly the stack and priority it declares (or the composition default, if it declares none). See Section 4 for how these interact with `Run(...)`'s own priority argument.

### Stack Usage Introspection (New in v1.3)

`CSProcess` exposes two read-only accessors for inspecting how much stack a process's task has actually used, built on FreeRTOS's `uxTaskGetStackHighWaterMark()`:

```cpp
UBaseType_t stackHighWaterMarkWords() const;
UBaseType_t lastStackHighWaterMarkWords() const;
```

* **`stackHighWaterMarkWords()`** — a *live* query against the process's running task. Valid from the moment the task starts (`Run(...)` has spawned it) until it exits; safe to call at any time in between, including concurrently with a `StaticNetwork` composition whose processes run forever — this is the intended use case, since such processes never reach a "finished" point to report at.
* **`lastStackHighWaterMarkWords()`** — a snapshot taken automatically, immediately before a process's task exits (self-deletes). Unlike the live query, this remains safely readable indefinitely afterward, which matters for `TerminatingNetwork` compositions whose processes do return from `run()`. Processes that loop forever never populate this — use the live query for those instead.

Both return the high-water mark in **words** (`StackType_t` units), not bytes — multiply by `sizeof(StackType_t)` for a byte count. This mirrors `uxTaskGetStackHighWaterMark()`'s own convention, and matches the units `stackWords()` (above) already uses when declaring a stack size, so a returned value can be compared directly against what you asked for.

**Sentinel value.** Both accessors return `CSP_STACK_HWM_UNAVAILABLE` (`(UBaseType_t)-1`) rather than `0` when no real measurement is available — for example, before a task has started, or when `lastStackHighWaterMarkWords()`'s task hasn't exited yet. This is deliberately a *different* sentinel from `CSP_STACK_UNSPECIFIED` (used above for "no stack size declared"): a genuine high-water-mark reading of `0` words is a legitimate, if alarming, result — it means the task came within a hair of overflow — and must not be silently indistinguishable from "no data available."

**Build requirement.** These accessors depend on `INCLUDE_uxTaskGetStackHighWaterMark` being set to `1` in your project's `FreeRTOSConfig.h`. If it isn't, both accessors compile fine but always return `CSP_STACK_HWM_UNAVAILABLE` — there is no build error, only silently unavailable data, so check this first if every process reports unavailable.

```cpp
// Typical usage: a periodic health-check report on a StaticNetwork.
void MainApp_Task(void* params) {
    static Camera camera(/* ... */);
    static Inference inference(/* ... */);

    auto network = InParallel(camera, inference);
    Run(network, ExecutionMode::StaticNetwork);

    while (true) {
        vTaskDelay(pdMS_TO_TICKS(30000));
        network.forEachProcess([](CSProcess& p) {
            UBaseType_t hwm = p.stackHighWaterMarkWords();
            if (hwm != CSP_STACK_HWM_UNAVAILABLE) {
                printf("%s: %u words free at worst\n", p.name(), hwm);
            }
        });
    }
}
```

(`ParallelHelper::forEachProcess(...)`, used above, is documented in Section 4.)

---

## 2. Channel Communication

Channels provide the synchronization "handshake" between processes. As of v1.1, the behavior of this handshake is governed by the `BufferPolicy`. Unchanged in v1.2 and v1.3.

### Buffer Policies (`enum class BufferPolicy`)
Every channel can be configured with a policy that determines behavior when a producer and consumer are out of sync.

| Policy | Behavior | Use Case |
| :--- | :--- | :--- |
| **Block** (Default) | Standard CSP. Sender blocks until receiver is ready. | Critical control signals, command/response. |
| **KeepOldest** | Non-blocking. If the buffer is full, new data is discarded. | Error logging, capturing first-event triggers. |
| **KeepNewest** | Non-blocking. If the buffer is full, the oldest data is overwritten. | Sensor streams, IMU data, "Freshness" priority. |

### `SamplingChannel<T, Policy>` (Rendezvous)
The core synchronization primitive for point-to-point communication with zero internal capacity.

* **Behavior**:
    * **`Block` (Default)**: Implements strict Synchronous Rendezvous. Both sender and receiver must be present to exchange data.
    * **`KeepNewest` / `KeepOldest`**: The sender never blocks; data is captured only if a receiver is already waiting at the exact moment of the write. If no receiver is present, the data is discarded.
* **Aliases**:
    * **`Channel<T>`**: An alias to model a standard CSP channel.
    * **`Any2OneChannel<T, P>`**: An alias used to semantically indicate a shared input port, though the underlying implementation remains a point-to-point rendezvous.
    * **`One2OneChannel<T, Policy>`:** Legacy for API 1.0 compatibility.

* **Declaration**:

```cpp
// Sampling channels (Non-blocking, explicit policy)
static SamplingChannel<Message, BufferPolicy::KeepNewest> keepnewest_chan;
static SamplingChannel<Message, BufferPolicy::KeepOldest> keepoldest_chan;

// Standard blocking channel alias
static Channel<Message> chan;

// Semantic Any-to-One alias
static Any2OneChannel<Message> shared_input;
```

### `SamplingBufferedChannel<T, N, Policy>` (Asynchronous)
A buffered version of the point-to-point channel that decouples the timing of the sender and receiver.
* **Usage:** Ideal for "Pipeline" topologies where execution times vary (e.g., SD Card I/O vs. NPU Inference).
* **Behavior:**
    * **`Block` (Default):** Sender blocks only if the buffer is N items full.
    * **`KeepNewest` / `KeepOldest`:** Sender never blocks. If the buffer is full, the policy dictates which item is dropped to maintain the N capacity.
* **Aliases**:
    * **`BufferedChannel<T, N>`:** An alias to model a standard CSP channel.
    * **`BufferedOne2OneChannel<T, N, Policy>`:** Legacy for API 1.0 compatibility.

* **Declaration:**

```cpp
// Static allocation of a channel with a 16-slot "Lossy" buffer
static SamplingBufferedChannel<work_packet_t, 16, BufferPolicy::KeepNewest> keepnewest_b_chan;
static SamplingBufferedChannel<work_packet_t, 16, BufferPolicy::KeepOldest> keepoldest_b_chan;

// Standard blocking channel alias
static BufferedChannel<Message, 16> sync_b_chan;
```

### `Chanin<T>` and `Chanout<T>`
These represent the input and output ports of a channel. Processes should store these as members to interact with the network.
* Read (Synchronous): Use the `>>` operator. This blocks the calling process until data is available.
```cpp
in >> msg;
```
* Write (Synchronous): Use the `<<` operator. This blocks the calling process until a receiver is ready.
```cpp
out << msg;
```

---

## 3. Interrupt Handling
The library supports safe communication from Interrupt Service Routines (ISRs) to Tasks. Unchanged in v1.2 and v1.3.

`putFromISR(T value)`

This method allows an ISR to send a message or trigger to a waiting process.
* **Context:** Must be called from within a HAL Callback or Exception Handler.
* **Non-Blocking:** This method never waits.
* **Policy Impact:** With `KeepNewest`, it will always succeed by overwriting old data if necessary. With `Block`, it returns `false` if the buffer is full.

```cpp
extern "C" void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == MY_PIN) {
        my_chan.writer().putFromISR(trigger_t{});
        portYIELD_FROM_ISR(pdTRUE); // Force context switch to receiver
    }
}
```

---

## 4. Network Orchestration

To run the processes, they must be composed into a network.

### `InParallel(P1, P2, ...)`

This function represents the CSP Parallel operator (`||`). It groups process instances together for simultaneous execution.

> **v1.2 change:** In v1.1, the process listed *first* received different treatment than the rest (it ran inline on the caller's own task/stack instead of being spawned). As of v1.2 this is no longer true: `InParallel(...)` treats every argument identically, regardless of position. List processes in whatever order best communicates your system's topology — see [Section 7](#7-what-changed-in-v12).

### `ParallelHelper<Processes...>::forEachProcess(fn)` (New in v1.3)

`InParallel(...)` returns a `ParallelHelper<Processes...>` — normally passed straight into `Run(...)`, but it can also be kept as a named object and reused afterward:

```cpp
template <typename Fn>
void forEachProcess(Fn&& fn);
```

Applies `fn(process&)` to every process in the composition, in declaration order. `fn` receives each process by its own concrete reference (e.g. a `Camera&`, not sliced to `CSProcess&`), though for code that should treat every process uniformly, a lambda parameter typed `CSProcess&` works too — each concrete reference converts implicitly.

Safe to call at any time after `Run(...)` has spawned the composition's processes, including while a `StaticNetwork`'s processes are still running — it only reads each process's own state (e.g. via `stackHighWaterMarkWords()`, Section 1) and never touches anything a process owns internally. This is the primary tool for building a network-wide report — a stack-usage dump across every process, for instance — out of a composition that has no other natural point to report at.

```cpp
auto network = InParallel(camera, inference, console);
Run(network, ExecutionMode::StaticNetwork);
// ... later, from any task, any number of times:
network.forEachProcess([](CSProcess& p) { /* inspect p */ });
```

Keeping the `ParallelHelper` around like this (rather than passing an `InParallel(...)` temporary directly into `Run(...)`) costs nothing extra — `Run(...)` takes it by value, and the object itself is just a `std::tuple` of references.

### `Run(...)`

The entry point for the CSP engine.

* **Execution Mode:** `ExecutionMode::StaticNetwork` is used for high-integrity systems where tasks are launched once at startup and never deleted. `ExecutionMode::TerminatingNetwork` spawns all processes and blocks the calling task until every one of them completes.
* **Priority argument (Since v1.2):** Both `Run(...)` overloads for parallel compositions accept an optional trailing `priority` argument — the **composition-wide default** priority applied to any process that has not overridden `taskPriority()`. If omitted, it defaults to the same value v1.1 used internally, so existing call sites are unaffected.

```cpp
template <typename... Processes>
void Run(ParallelHelper<Processes...> helper,
         UBaseType_t priority = /* v1.1-compatible default */);

template <typename... Processes>
void Run(ParallelHelper<Processes...> helper, ExecutionMode mode,
         UBaseType_t priority = /* v1.1-compatible default */);
```

A process's own `taskPriority()` override, when present, always takes precedence over the composition-wide `priority` argument for that specific process.

* **Example:**
```cpp
void MainApp_Task(void* params) {
    static MyProcess p1(chan.writer());
    static MyOtherProcess p2(chan.reader());

    Run(
        InParallel(p1, p2),
        ExecutionMode::StaticNetwork
    );

    // As of v1.2, MainApp_Task no longer executes any process inline
    // (see Section 7). If it has nothing further to do, it may safely
    // end itself here:
    vTaskDelete(NULL);
}
```

* **Single-process `Run(CSProcess&, priority)` has been removed as of v1.3.** There is now exactly one spawn path, for compositions of any size: `Run(InParallel(process), mode)` covers everything the single-process overload used to do, plus a blocking mode it never offered. See [Section 8](#8-what-changed-in-v13) for why, and for a mechanical migration path.

---

## 5. External Choice (Alternative / ALT)

The `Alternative` class implements the CSP External Choice operator (☐). It allows a process to wait on multiple input channels simultaneously, proceeding as soon as any one of them is ready. Unchanged in v1.2 and v1.3.

### `Alternative`
The `Alternative` object is typically constructed on the process stack. It uses a "Resident-Guard" pattern that is entirely heap-free.

* **Syntax**: `Alternative alt(Guard1, Guard2, ...);`
* **Guard Binding (`|`)**: To bind a channel to a local variable, use the pipe operator: `chan_in | message_variable`. This ensures that when the channel is selected, the data is automatically copied into the variable.

### `fairSelect()`
This method blocks the process until at least one of the guarded channels is ready to synchronize.

* **Fairness**: It uses a "Fair" selection algorithm to prevent starvation, ensuring that if multiple channels are ready, one is not consistently ignored.
* **Return Value**: Returns an `int` representing the index of the selected channel (starting at 0).

**Example Usage**:
```cpp
void Receiver::run() {
    Message msgA, msgB;
    // Bind channels to local variables
    Alternative alt(inA | msgA, inB | msgB);

    while(true) {
        // Blocks until a message arrives on either inA or inB
        int selected = alt.fairSelect();

        switch(selected) {
            case 0:
                // msgA is already populated
                printf("Received from A: %d\n", msgA.sequence_num);
                break;
            case 1:
                // msgB is already populated
                printf("Received from B: %d\n", msgB.sequence_num);
                break;
        }
    }
}
```

---

## 6. Memory and Safety Guarantees

### Zero-Heap Execution *(channel/ALT operation only — see note below)*
`csp4cmsis` is designed for safety-critical ARM environments where dynamic memory allocation (the "Heap") is prohibited during steady-state operation.

1. **Static Channels:** Channels should be declared as static to reside in the `.data` or `.bss` segments.
2. **Stack-Based ALT:** The Alternative object and its guards reside on the task stack, ensuring deterministic memory usage.
3. **No New/Delete during communication or selection:** Channel operations, guard enable/disable/activate, and ALT selection perform no hidden allocations.
4. **Deterministic Latency:** O(1) time complexity for all channel operations, including KeepNewest overwrites.
5. **Thread Safety:** Atomic pointer updates ensure the receiver never reads partially overwritten data during a "Lossy" event.

> **Note:** `InParallel(...)` process spawning itself does perform one heap allocation per process (a small internal task-context object) at network-startup time, and per-process FreeRTOS task/stack allocation is naturally heap-derived unless your `heap_*.c` port and `xTaskCreateStatic` are used instead. This has always been true and is unrelated to the v1.2 or v1.3 changes. The "zero-heap" guarantee above refers specifically to *steady-state* channel and ALT operation after the network is running, not to network construction.
>
> The stack-introspection accessors added in v1.3 (Section 1) are pure reads of state FreeRTOS already maintains — they perform no allocation and add no runtime cost beyond the FreeRTOS call they wrap.

### Deterministic Synchronization
Unlike standard RTOS queues, csp4cmsis channels default to *Rendezvous* (capacity 0). This means:
* The *Sender* blocks until the *Receiver* arrives.
* The *Receiver* blocks until the *Sender* arrives.
* The data transfer happens only when both "shake hands," providing a formal proof of synchronization.

---

## 7. What Changed in v1.2

### The problem this release fixes

In v1.1, `InParallel(P1, P2, ..., Pn)` treated `P1` differently from every other argument:

* `P1` ran **inline**, on the stack and at the priority of whatever task called `Run(...)`.
* `P2 .. Pn` were each spawned as their own FreeRTOS task, with a **fixed, hardcoded 256-word stack** and a fixed priority, regardless of what that process actually needed.

This meant a process's required stack size determined where it had to be placed in the argument list — a constraint that was invisible from the public API and existed only as a source comment convention ("`X` must be listed first"). Reordering `InParallel(...)` arguments for readability, or adding a new process ahead of an existing one, could silently reduce a process's stack from several kilobytes to 256 words with no compiler warning and no runtime error until a stack overflow occurred.

### The fix

* Every process spawned via `InParallel(...)` — including what would have been position 0 — is now spawned as its own FreeRTOS task, sized and prioritized according to that process's own `stackWords()` / `taskPriority()` (or the composition default, if unset).
* The calling task (e.g. `MainApp_Task`) no longer executes any process logic inline. Once `Run(...)` returns (or, in `StaticNetwork` mode, once it has finished spawning), the calling task's own stack is free for it to do as it pleases — including ending itself via `vTaskDelete(NULL)` if it has no further role.

### Compatibility

* **Source compatibility:** Existing code compiles unchanged. `stackWords()` / `taskPriority()` are optional overrides; the extra `priority` argument on `Run(...)` is defaulted.
* **Default values are chosen to reproduce v1.1 behavior exactly** for any process that doesn't override the new methods: processes spawned via `InParallel(...)` default to the same 256-word stack and same priority v1.1 used for non-first processes. The one behavior that necessarily changes is that a process no longer inherits the *caller's* stack/priority by virtue of being listed first — if your v1.1 code relied on that (per the "must be listed first" convention), give that specific process an explicit `stackWords()` / `taskPriority()` override reproducing the values it used to inherit. This is a one-time, mechanical migration step: move the number from the comment into the class.
* **ABI note for prebuilt/vendored `libcsp4cmsis.a`:** `CSProcess` gained two new virtual methods, which changes its vtable layout. If your project links against a prebuilt static library rather than rebuilding CSP4CMSIS from source, ensure the library is rebuilt against the v1.2 headers before linking v1.2 application code against it — mixing an old prebuilt `.a` with new headers is not safe.
* **No channel, ALT, or ISR API changed.** Sections 2, 3, 5, and 6 above are identical to v1.1.

### Migration checklist

1. Identify any process that was relying on being listed first in `InParallel(...)` for extra stack or elevated priority (check for a comment like "`must be first`" or "`must remain argument 0`").
2. Add explicit `stackWords()` / `taskPriority()` overrides to that process's class, using the values it used to inherit from the caller.
3. Reorder `InParallel(...)` arguments freely — e.g. into physical/topological order — since position no longer carries any stack or priority meaning.
4. If the calling task (e.g. `MainApp_Task`) has nothing to do after `Run(...)` returns, consider ending it with `vTaskDelete(NULL)` to reclaim its stack, and confirm your project's heap scheme (`heap_2`/`heap_4`/`heap_5`) actually returns freed memory to the pool if this matters for your RAM budget.
5. If you link a prebuilt `libcsp4cmsis.a`, rebuild it against the v1.2 headers.

---

## 8. What Changed in v1.3

### New functionality

* **Stack usage introspection** (Section 1): `CSProcess::stackHighWaterMarkWords()` and `lastStackHighWaterMarkWords()`, backed by FreeRTOS's `uxTaskGetStackHighWaterMark()`. Requires `INCLUDE_uxTaskGetStackHighWaterMark` enabled in `FreeRTOSConfig.h` to return real data (silently returns `CSP_STACK_HWM_UNAVAILABLE` otherwise — no build error).
* **`ParallelHelper::forEachProcess(fn)`** (Section 4): apply a callback to every process in a composition, usable at any time after spawning — the mechanism for building a network-wide report (e.g. a periodic stack-usage dump) out of a `StaticNetwork` whose processes never naturally finish.

### The problem this release also fixes

Prior to v1.3, `public_task.h` exposed a second, independent spawn path — `Run(CSProcess&, priority)` — alongside the `InParallel(...)`-based path used for compositions. The two paths had diverged: the single-process path passed a raw `CSProcess*` as the FreeRTOS task's `pvParameters`, while `ThreadFuncWrapper` (the function every spawned task actually runs) unconditionally expected a `TaskCtx*` — the wrapper type the `InParallel(...)` path already constructs correctly. Any call to the single-process `Run(...)` overload was therefore reading application memory as if it were a different, incompatible struct: undefined behavior, not merely a latent risk.

`ParallelHelper<Processes...>` already handles a composition of exactly one process correctly — it's an ordinary variadic template, and nothing about it assumes two or more arguments — and every real call site in the existing codebase already used the `InParallel(...)` path. Rather than fix the single-process path's `pvParameters` mismatch and maintain two spawn implementations indefinitely, v1.3 removes it: `Run(InParallel(process), mode)` is now the one way to spawn any composition, from one process to many, and it's the path that was already correct.

### Compatibility

* **This is not a source-compatible change.** Unlike v1.2, any code that calls `Run(CSProcess&, priority)` directly will fail to compile under v1.3 — the overload no longer exists. This is intentional: the removed path was never safe to call in the first place (see above), so there is no working "old behavior" worth silently preserving.
* `TEST_STACK_SIZE_WORDS` and `CSP_DEFAULT_TASK_PRIORITY` — macros that existed solely to supply the removed overload's defaults — are also removed from `public_task.h`. Code referencing either directly will need its own replacement values.
* **New functionality is additive and non-breaking.** `stackHighWaterMarkWords()`, `lastStackHighWaterMarkWords()`, and `forEachProcess(...)` are new methods; no existing method signature changed.
* **ABI note for prebuilt/vendored `libcsp4cmsis.a`:** `CSProcess` gained two new *private data members* (backing the stack-introspection accessors above), which changes `sizeof(CSProcess)` and the offsets of anything a derived class adds after them. Unlike v1.2's change, the new accessors themselves are **not virtual**, so the vtable layout is unaffected this time — but the object layout is, which is exactly as ABI-breaking for a prebuilt static library. As in v1.2: rebuild `libcsp4cmsis.a` against the v1.3 headers before linking v1.3 application code against it.

### Migration checklist

1. Search for any call site of `Run(someProcess, priority)` — the single-argument-process form. (Grep for `Run(` and check each result's first argument type; the composition form always passes something built from `InParallel(...)`.)
2. Replace each with the `InParallel(...)`-based equivalent, choosing the mode that matches the old call's behavior:
   * The old single-process `Run(...)` spawned the task and returned immediately (fire-and-forget). The equivalent is:
     ```cpp
     Run(InParallel(process), ExecutionMode::StaticNetwork, priority);
     ```
   * If you'd actually prefer the caller to block until the process finishes — a mode the old overload never offered — use the default (`TerminatingNetwork`) mode instead:
     ```cpp
     Run(InParallel(process), priority);
     ```
3. If your code referenced `TEST_STACK_SIZE_WORDS` or `CSP_DEFAULT_TASK_PRIORITY` directly, replace with an explicit literal or your own named constant.
4. If you link a prebuilt `libcsp4cmsis.a`, rebuild it against the v1.3 headers (same requirement as v1.2, for a different underlying reason — see the ABI note above).
5. To use the new stack-introspection accessors meaningfully, confirm `INCLUDE_uxTaskGetStackHighWaterMark` is set to `1` in your project's `FreeRTOSConfig.h` — check this first if `stackHighWaterMarkWords()` always returns `CSP_STACK_HWM_UNAVAILABLE`.
