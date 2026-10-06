# CSP4CMSIS API Reference (v3.0.0)

CSP4CMSIS is a C++17 library of CSP processes, channels and alternation on top of any CMSIS-RTOS2
kernel. This page documents the API of **v3.0.0** (tag
[`v3.0.0`](https://github.com/OliverFaust/CSP4CMSIS/tree/v3.0.0)). The 3.x API is stable: code that
compiles with 3.0 compiles with every 3.x release. Everything is in namespace `csp`
and comes with one include:

```cpp
#include "csp/csp4cmsis.h"

using namespace csp;
```

The pre-2.0 API (1.x, up to v1.3) is archived on a [separate page](./api-1.x).

Every code example on this page is compiled against v3.0.0 by
[`tests/doc_examples/`](https://github.com/OliverFaust/CSP4CMSIS/tree/main/tests/doc_examples) in the
CSP4CMSIS repository.

---

## 1. Getting the library

### 1a. STM32CubeMX / STM32CubeIDE, without packs

This is how the book's examples use the library. Step-by-step, tested guide:
[`Documentation/CSP4CMSIS_STM32CubeIDE.md`](https://github.com/OliverFaust/CSP4CMSIS/blob/v3.0.0/Documentation/CSP4CMSIS_STM32CubeIDE.md).

- **CubeMX:** Middleware **FREERTOS**, **Interface: CMSIS_V2** (ST's CMSIS-RTOS2 wrapper). Static
  allocation is enabled with that interface (Memory Allocation: Dynamic / Static).
- **Source:** take the folder `csp4cmsis/` and the file `LICENSE` from either the v3.0.0 release's
  **Source code (zip)** on GitHub ([release page](https://github.com/OliverFaust/CSP4CMSIS/releases/tag/v3.0.0))
  or the unpacked `OliverFaust.CSP4CMSIS.3.0.0.pack` (a zip archive; same release page). Copy them to
  `<project>/lib/csp4cmsis/`, outside `Core/` (CubeMX regenerates `Core/`). Add `lib/csp4cmsis/src` as a
  source folder.
- **Include path:** `lib/csp4cmsis/inc` only.
- **G++ language standard:** GNU++17 (CubeIDE's default GNU++14 does not compile the library).
- **Two G++ defines:** `CSP4CMSIS_MAX_SYSCALL_INTERRUPT_PRIORITY=5` and
  `CSP4CMSIS_DEVICE_HEADER="stm32g4xx.h"` (section 2). Static allocation is the default, and FreeRTOS is
  found from `FreeRTOS.h`.

### 1b. CMSIS pack

```bash
cpackget add -a https://github.com/OliverFaust/CSP4CMSIS/releases/download/v3.0.0/OliverFaust.CSP4CMSIS.3.0.0.pack
```

- `-a` accepts the pack's MIT licence non-interactively; without it, `cpackget` asks, and in a script
  or CI job it installs nothing.
- Install from the versioned `.pack` URL, which pins the exact version. To update, change the version
  in the URL. To find out whether there is a newer release, run `cpackget update-index` (it reports
  "can be upgraded from … to …").
- Component, in the `.cproject.yml`:

```yaml
components:
  - component: OliverFaust::CSP4CMSIS:Core
```

The component requires `CMSIS:RTOS2` (any implementation) and `CMSIS:CORE`. Pack builds get the
device header from `RTE_Components.h`, so `CSP4CMSIS_DEVICE_HEADER` is not needed.

---

## 2. Configuration

CSP4CMSIS needs a **CMSIS-RTOS2** implementation (`cmsis_os2.h`). Tested: Arm's CMSIS-FreeRTOS
adapter, Keil RTX5, and ST's STM32Cube wrapper (FreeRTOS, CubeMX "CMSIS_V2").

| Define | Required | Meaning |
|---|---|---|
| `CSP4CMSIS_MAX_SYSCALL_INTERRUPT_PRIORITY=<n>` | always (`#error` otherwise) | The NVIC priority, **unshifted**, at and below which the library's critical sections mask interrupts (BASEPRI). Use your RTOS's threshold, e.g. FreeRTOS's `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`. 0 or a shifted value (0x50, 0xA0) does not compile (`static_assert`). See section 5 for what it means for your ISRs. |
| `CSP4CMSIS_DEVICE_HEADER="<header>"` | builds without packs (`#error` if there is neither this nor a pack's `RTE_Components.h`) | The device header, for CMSIS-Core (`__NVIC_PRIO_BITS`, BASEPRI access), e.g. `"stm32g4xx.h"`. |
| `CSP4CMSIS_DYNAMIC_ALLOCATION` | optional | Opts out of the default, static allocation: the RTOS objects the library creates then come from the RTOS's allocator. |
| `CSP4CMSIS_RTOS2_BACKEND_FREERTOS` or `CSP4CMSIS_RTOS2_BACKEND_RTX5` | only if detection fails (`#error` then) | The backend, whose control-block types static allocation uses. Detected from `RTE_Components.h` (pack builds), else from `FreeRTOS.h` or `rtx_os.h` on the include path. Any FreeRTOS-based CMSIS-RTOS2 layer, including ST's, is `_FREERTOS`. |
| `CSP4CMSIS_ISR_MAX_ELEMENT_SIZE=<bytes>` | optional, default 64 | Largest element an ISR may write (section 5). |
| `CSP_TYPICAL_STACK_WORDS=<words>` | optional, default 256 | A documented starting value for `CSProcessStatic<N>`; not used by the library itself. |

How to choose each value for your board:
[`Documentation/CSP4CMSIS_Configuration.md`](https://github.com/OliverFaust/CSP4CMSIS/blob/v3.0.0/Documentation/CSP4CMSIS_Configuration.md).

---

## 3. Processes

### `CSProcessStatic<N>`

A process is a class derived from `CSProcessStatic<N>` that implements `run()`:

- **`N` is the stack size in words** (`uint32_t`, 4 bytes each): `CSProcessStatic<256>` has 1 KB of
  stack; at least `CSP_MIN_STACK_WORDS` (64, `static_assert`). The stack (8-byte aligned) and, with
  static allocation (the default), the thread control block are members of the object.
- **Process objects must have static storage duration** (namespace scope or function-local `static`):
  the object's lifetime is the thread's storage.
- **`void run()`** is the process body. When it returns, the process's thread ends.
- **`const char* name() const`** (optional) names the thread; default `"csp_task"`.
- **`osPriority_t taskPriority() const`** (optional) is the CMSIS-RTOS2 priority of this process. If it
  is not overridden, the process runs at the priority passed to `Run()`.

```cpp
#include "csp/csp4cmsis.h"
#include <cstdio>

using namespace csp;

struct Reading {
    uint32_t sequence;
    int32_t  value;
};

class Sensor : public CSProcessStatic<256> {   // 256 words = 1 KB of stack
    Chanout<Reading> out;
public:
    explicit Sensor(Chanout<Reading> w) : out(w) {}
    const char* name() const override { return "Sensor"; }
    void run() override {
        for (uint32_t i = 0; ; ++i) {
            out << Reading{i, static_cast<int32_t>(i) * 10};
            SleepFor(Milliseconds(100));
        }
    }
};

class Logger : public CSProcessStatic<512> {
    Chanin<Reading> in;
public:
    explicit Logger(Chanin<Reading> r) : in(r) {}
    const char* name() const override { return "Logger"; }
    osPriority_t taskPriority() const override { return osPriorityBelowNormal; }
    void run() override {
        Reading r;
        while (true) {
            in >> r;
            printf("%lu: %ld\r\n", (unsigned long)r.sequence, (long)r.value);
        }
    }
};

static Channel<Reading> readings;
static Sensor sensor(readings.writer());
static Logger logger(readings.reader());

void start_network(void) {
    // Sensor runs at osPriorityLow (the composition priority), Logger at its own.
    Run(InParallel(sensor, logger), ExecutionMode::StaticNetwork, osPriorityLow);
}
```

### Starting processes: `InParallel`, `Run`, `ExecutionMode`

`InParallel(p1, p2, ...)` is the parallel composition; `Run()` starts it. Every process becomes its own
CMSIS-RTOS2 thread (`osThreadNew()` with the process's static stack and control block); position in
the argument list has no meaning.

<!-- synopsis: declarations from the v3.0.0 headers -->
```cpp
template <typename... Processes>
void Run(ParallelHelper<Processes...> helper, ExecutionMode mode,
         osPriority_t priority = CSP_DEFAULT_NETWORK_PRIORITY);
```

- **`ExecutionMode::StaticNetwork`:** starts all threads and returns at once. For networks that run
  for ever. Can be called from a thread, or from `main()` between `osKernelInitialize()` and
  `osKernelStart()`.
- **`ExecutionMode::TerminatingNetwork`:** starts all threads and blocks the calling thread until every
  process has returned from `run()`. Call it from a thread.
- **The mode is always written** (3.0): there is no default, so a blocking `Run()` is never a surprise.
- **`priority`:** the composition priority, for every process that does not override
  `taskPriority()`. Default `CSP_DEFAULT_NETWORK_PRIORITY` = `osPriorityLow`.
- **One process** is a composition of one: `Run(InParallel(p), ExecutionMode::StaticNetwork, priority)`.
- **If `osThreadNew()` fails** (out of RTOS memory, invalid attributes), `Run()` calls
  `csp4cmsis_fatal_error("CSP4CMSIS: Run(): osThreadNew() failed")` and does not return (section 6).

```cpp
#include "csp/csp4cmsis.h"
#include <cstdio>

using namespace csp;

class Worker : public CSProcessStatic<CSP_TYPICAL_STACK_WORDS> {
    int id;
public:
    explicit Worker(int i) : id(i) {}
    void run() override { printf("worker %d done\r\n", id); }   // returning ends the thread
};

static Worker first(1), second(2);

// Called from a thread: returns when both workers have returned from run().
void run_workers(void) {
    Run(InParallel(first, second), ExecutionMode::TerminatingNetwork, osPriorityNormal);
}
```

### Stack introspection and `forEachProcess`

- **`uint32_t stackHighWaterMarkWords() const`:** the smallest amount of free stack the process's
  thread has had so far, **in words** (the unit of `N`). Returns `CSP_STACK_HWM_UNAVAILABLE` before the
  process has been started. Call it only while the thread exists (not after `run()` has returned).
- **`size_t stackWords() const`:** the stack size `N`.
- **Requirement:** it uses `osThreadGetStackSpace()`. With FreeRTOS (Arm's adapter and ST's wrapper),
  that function calls `uxTaskGetStackHighWaterMark()`, so `FreeRTOSConfig.h` needs
  `INCLUDE_uxTaskGetStackHighWaterMark 1`; without it the build fails to link.
- **`forEachProcess(fn)`:** `InParallel(...)` returns a `ParallelHelper`; keep it in a variable to call
  `fn(CSProcess&)` for each process, in declaration order, at any time after `Run()`.

```cpp
#include "csp/csp4cmsis.h"
#include <cstdio>

using namespace csp;

class Blinker : public CSProcessStatic<256> {
public:
    const char* name() const override { return "Blinker"; }
    void run() override { while (true) { SleepFor(Milliseconds(500)); } }
};

class Heartbeat : public CSProcessStatic<256> {
public:
    const char* name() const override { return "Heartbeat"; }
    void run() override { while (true) { SleepFor(Seconds(1)); } }
};

static Blinker blinker;
static Heartbeat heartbeat;

// Body of a monitoring thread (e.g. CubeMX's defaultTask).
void monitor(void) {
    auto network = InParallel(blinker, heartbeat);
    Run(network, ExecutionMode::StaticNetwork);
    while (true) {
        SleepFor(Seconds(5));
        network.forEachProcess([](CSProcess& p) {
            uint32_t free_words = p.stackHighWaterMarkWords();
            if (free_words != CSP_STACK_HWM_UNAVAILABLE) {
                printf("%s: %lu of %lu words never used\r\n", p.name(),
                       (unsigned long)free_words, (unsigned long)p.stackWords());
            }
        });
    }
}
```

### Time: `SleepFor`, `Ticks`, `Milliseconds`, `Seconds`, `Forever`

- **`void SleepFor(Time duration)`** suspends the calling process for a duration (`osDelay()`):
  `SleepFor(Milliseconds(250))`, `SleepFor(Ticks(10))`, `SleepFor(Forever)`. A plain number does not
  compile: whether it meant ticks or milliseconds would be a guess.
- **`csp::Time`** holds a tick count: `Time(ticks)` (explicit), `to_ticks()`; `constexpr`.
  **`Ticks(n)`** is `n` RTOS ticks; **`Forever`** is `osWaitForever` (as a `RelTimeoutGuard` delay, the
  longest timeout, 0xFFFFFFFE ticks). `Seconds(s)` and `Milliseconds(ms)` convert with the kernel tick
  frequency (`osKernelGetTickFreq()`).
- **They round up**, computed in 64 bits: a non-zero duration is never shorter than requested
  (at most one tick longer) and at least 1 tick; `Milliseconds(5)` at a 100 Hz tick is `Time(1)`.
  `Milliseconds(0)` is `Time(0)`. Durations that are a whole number of ticks are exact. Very long
  durations saturate at `0xFFFFFFFE` ticks (`0xFFFFFFFF` is `osWaitForever`). (2.0.x rounded down, so
  `Milliseconds(5)` at 100 Hz was 0 ticks.)

---

## 4. Channels

A channel object owns the communication; processes use its ends: `writer()` returns a `Chanout<T>`,
`reader()` a `Chanin<T>`. **Each process keeps its own end object** (the end holds that process's ALT
state): do not share one `Chanin`/`Chanout` object between processes.

**Any channel may have several writers and several readers**, each with its own end, using plain `<<`
and `>>`: they are served one at a time. **In an ALT, at most one reader and one writer of a channel may
be ALTing**; a second ALTing reader or writer is a fatal error.

| Channel | Capacity | Policies | ISR writer |
|---|---|---|---|
| `Channel<T>` | 0 (rendezvous) | Block only | no |
| `SignalChannel` | 0 (rendezvous, no data: `csp::Signal`) | Block only | no |
| `BufferedChannel<T, SIZE, Policy = BufferPolicy::Block>` | `SIZE` | Block, KeepNewest, KeepOldest | `isrWriter()` |

Channels are not copyable; declare them at namespace scope or as function-local `static`s.

### Rendezvous channels: `Channel`, `SignalChannel`

- A write completes only when a reader takes the value, and a read only when a writer offers one:
  writer and reader synchronise, and the value is copied directly from the writer to the reader.
- **Any number of processes may read or write** an end with plain (blocking) operations; they are
  served one at a time.
- **`SignalChannel`** carries no data: `out << Signal{}`, `in >> s` with `Signal s;`.
- **Block only:** a rendezvous or signal channel has no policy; for "latest value" semantics use
  `BufferedChannel<T, 1, BufferPolicy::KeepNewest>`.

### Buffered channels: `BufferedChannel`

- `SIZE` elements are stored inside the channel object (no RTOS queue, no heap); `SIZE` must be > 0.
- A reader always takes the oldest element and blocks while the channel is empty.
- What a write does when the channel is **full**, by `BufferPolicy`:

| Policy | Write to a full channel |
|---|---|
| `BufferPolicy::Block` | The writer blocks until a reader has taken an element. |
| `BufferPolicy::KeepNewest` | The oldest element is overwritten (one atomic step). The writer never blocks. |
| `BufferPolicy::KeepOldest` | The new element is discarded. The writer never blocks. |

### Element types

`T` must be **trivially copyable** (`static_assert`): elements are copied with `memcpy`. Use plain
structs of scalars, arrays, enums or raw pointers; not `std::string` or classes with their own copy
constructor.

### Declaring, writing, reading

```cpp
#include "csp/csp4cmsis.h"

using namespace csp;

struct Command { uint8_t opcode; uint16_t argument; };    // trivially copyable
struct Sample  { uint32_t timestamp; float value; };

static Channel<Command>          commands;   // rendezvous
static Channel<Command>          requests;   // rendezvous, several writers (one end each)
static SignalChannel             ready;      // rendezvous without data
static BufferedChannel<Sample, 8> samples;   // 8 slots, Block
static BufferedChannel<Sample, 1, BufferPolicy::KeepNewest> latest;     // newest value
static BufferedChannel<Sample, 4, BufferPolicy::KeepOldest> first_four; // first four values

class Producer : public CSProcessStatic<256> {
    Chanout<Sample> out = samples.writer();   // this process's own end
    Chanout<Signal> go  = ready.writer();
public:
    void run() override {
        go << Signal{};                        // waits until the consumer reads it
        for (uint32_t t = 0; ; ++t) {
            out << Sample{t, 0.5f};            // blocks only while all 8 slots are full
            out.write(Sample{t, 1.5f});        // same as <<
        }
    }
};

class Consumer : public CSProcessStatic<256> {
    Chanin<Sample> in    = samples.reader();
    Chanin<Signal> start = ready.reader();
public:
    void run() override {
        Signal s;
        start >> s;
        Sample x;
        while (true) {
            in >> x;                           // blocks while empty
            in.read(x);                        // same as >>
        }
    }
};
```

### Where channels may be constructed

A channel creates its RTOS objects (semaphores) in its constructor, and a failed creation is fatal.
Construct channels at **namespace scope** or as **function-local `static`s**, never in an ISR, and
before the interrupt that writes to one is enabled. Details:
[configuration guide, section 5](https://github.com/OliverFaust/CSP4CMSIS/blob/v3.0.0/Documentation/CSP4CMSIS_Configuration.md).

---

## 5. Interrupts

### Why an ISR cannot use a rendezvous channel

A rendezvous completes only when writer and reader meet, so a writer must be able to wait for its
partner. An interrupt handler cannot wait. Rendezvous and signal channels therefore have **no ISR
write path**. An ISR writes to a **buffered** channel, which keeps the value until the reader takes it
(also when the reader is not yet waiting).

### `isrWriter()`

`BufferedChannel<T, SIZE, P>::isrWriter()` returns an
`IsrChanout<T>`, the only way to write from an interrupt:

<!-- synopsis: declarations from the v3.0.0 headers -->
```cpp
bool putFromISR(const T& data);   // never blocks
```

- **Block:** returns `false` if the channel is full (nothing is written).
- **KeepNewest / KeepOldest:** always returns `true`; the policy decides what is kept.
- **Element size:** `sizeof(T)` must be ≤ `CSP4CMSIS_ISR_MAX_ELEMENT_SIZE` (default 64 bytes;
  `static_assert`). The copy runs with interrupts at and below the threshold masked, so the limit bounds
  the added interrupt latency. For larger data, send an index into a static pool.
- **No manual yield:** `putFromISR()` wakes a waiting reader through CMSIS-RTOS2 calls that are valid in
  an ISR, and the RTOS switches to it when the handler returns. Do not add an RTOS-specific yield.
- Reading is unchanged: `in >> v` or an ALT guard.

### The interrupt-priority rule

**Every ISR that calls CSP4CMSIS or the RTOS must have an NVIC priority numerically greater than or
equal to `CSP4CMSIS_MAX_SYSCALL_INTERRUPT_PRIORITY`** (lower urgency). The library's critical sections
raise BASEPRI to that level; an ISR with a numerically lower priority is not masked and can interrupt
a channel update half-way.

- Example: with 3 priority bits (0…7) and a threshold of 5, such ISRs must be at 5, 6 or 7. The
  STM32G4 has 4 bits (0…15), so with 5 it is 5…15.
- **After reset, every interrupt has priority 0, the highest.** Many vendor drivers do not change it.
  Set the priority explicitly before the interrupt is enabled or its driver is started: with
  `NVIC_SetPriority(irq, CSP4CMSIS_MAX_SYSCALL_INTERRUPT_PRIORITY)` (or a larger value), or in STM32CubeMX
  under **System Core > NVIC** by ticking **"Uses FreeRTOS functions"** for that interrupt.
- ISRs above the threshold (numerically lower) must not call the library or the RTOS; the library's
  critical sections do not delay them.

### Pattern: a button ("latest press counts")

Capacity 1 and `KeepNewest`: a press that arrives while the previous one has not been read replaces it.
The ISR never blocks and never fails.

```cpp
#include "csp/csp4cmsis.h"
#include <cstdio>

using namespace csp;

static BufferedChannel<uint32_t, 1, BufferPolicy::KeepNewest> presses;

// Called from the button's interrupt handler (STM32: from the EXTI callback).
extern "C" void button_pressed_from_isr(void) {
    static uint32_t count = 0;
    (void)presses.isrWriter().putFromISR(++count);     // KeepNewest: always true
}

class ButtonHandler : public CSProcessStatic<256> {
    Chanin<uint32_t> in = presses.reader();
public:
    void run() override {
        uint32_t count;
        while (true) {
            in >> count;                               // the latest press
            printf("button: %lu presses so far\r\n", (unsigned long)count);
        }
    }
};

// Before the interrupt is enabled (NUCLEO-G474RE user button: EXTI15_10_IRQn).
void allow_csp4cmsis_calls(IRQn_Type irq) {
    NVIC_SetPriority(irq, CSP4CMSIS_MAX_SYSCALL_INTERRUPT_PRIORITY);
}
```

### Pattern: a completion event (none may be lost)

Capacity 1 and `Block`: the process starts one transfer and waits for its completion before starting
the next, so at most one completion can be pending. A failed write therefore means a second
completion arrived before the first was read (a driver or protocol error): stop, through the
library's fatal-error hook, instead of losing it.

```cpp
#include "csp/csp4cmsis.h"

using namespace csp;

static BufferedChannel<bool, 1> transfer_done;

extern "C" void start_transfer(void);   // application: starts one interrupt-driven transfer

// Called from the transfer-complete interrupt (STM32 HAL: e.g. HAL_UART_TxCpltCallback).
extern "C" void transfer_complete_from_isr(void) {
    if (!transfer_done.isrWriter().putFromISR(true)) {
        // A second completion before the first was read: stop (section 6, fatal-error hook).
        csp4cmsis_fatal_error("transfer completion lost");
    }
}

class Transmitter : public CSProcessStatic<256> {
    Chanin<bool> done = transfer_done.reader();
public:
    void run() override {
        bool ok;
        while (true) {
            start_transfer();
            done >> ok;                 // kept even if it arrived before this read
            SleepFor(Seconds(1));
        }
    }
};
```

---

## 6. Alternation

`Alternative` waits for whichever of several **guards** becomes ready first, and performs exactly
that one.

- **Guards:** `in | var` (input guard: when selected, the value is read into `var`), `out | value`
  (output guard: when selected, `value` is written) and `RelTimeoutGuard` (a named object). Pass them to
  the constructor, `Alternative alt(g0, g1, ...)`, or add them with `addBinding(...)`. At most
  **16 guards** (`CSP4CMSIS_ALT_MAX_GUARDS`); a 17th is a fatal error.
- **`int priSelect()`:** blocks until a guard is ready and performs the **first** ready guard in the
  order given; returns its index (0 = first guard).
- **`int fairSelect()`:** the same, but starts the search after the previous winner, so a guard that is
  always ready cannot starve the others.
- An `Alternative` keeps references to its guards' variables; keep them alive while it is used. It is
  not copyable.

### `RelTimeoutGuard`

`RelTimeoutGuard(csp::Time delay)` is selected once `delay` ticks have passed since the `select()` it
takes part in started, unless another guard is selected first.

- **Timer-free:** no RTOS timer, no timer service task, no RTOS memory. The deadline is fixed when
  `select()` starts; nothing (e.g. a wakeup that turns out to be stale) postpones it.
- Precision: selected after `delay` ticks, at most `delay + 1`. `Time(0)` is ready at once, so the
  `select()` polls. With several timeout guards, the earliest deadline wins.
- A `RelTimeoutGuard` is not copyable; declare it as a named variable and pass it by name.

```cpp
#include "csp/csp4cmsis.h"
#include <cstdio>

using namespace csp;

static Channel<int> left, right, merged;

class Merger : public CSProcessStatic<512> {
    Chanin<int>  a   = left.reader();
    Chanin<int>  b   = right.reader();
    Chanout<int> out = merged.writer();
public:
    void run() override {
        int x = 0, y = 0;
        RelTimeoutGuard quiet(Milliseconds(100));      // measured from each select()
        Alternative alt(a | x, b | y, quiet);
        while (true) {
            switch (alt.fairSelect()) {
                case 0: out << x; break;
                case 1: out << y; break;
                case 2: printf("no input for 100 ms\r\n"); break;
            }
        }
    }
};
```

An output guard offers a value while the process also listens elsewhere:

```cpp
#include "csp/csp4cmsis.h"

using namespace csp;

static Channel<int>    numbers;
static SignalChannel stop;

class Counter : public CSProcessStatic<256> {
    Chanout<int>   out  = numbers.writer();
    Chanin<Signal> quit = stop.reader();
public:
    void run() override {
        int next = 0;
        Signal s;
        Alternative alt(quit | s, out | next);         // output guard: out | value
        while (alt.priSelect() == 1) {                 // stop wins if both are ready
            ++next;                                    // `next` was written; offer the next one
        }
    }
};
```

### Rules

- **Symmetric alternation communicates:** an ALT with an output guard on one end of a rendezvous
  channel and an ALT with an input guard on the other end pair up (since 2.0).
- **At most one ALTing process per channel end:** one ALTing reader and one ALTing writer per channel
  (rendezvous and buffered). Any number of processes may use plain blocking reads and writes.
- **Thread flags:** the library uses thread flags 0 and 8–23 of every process thread; 1–7 and 24–30 are
  free. With FreeRTOS, CMSIS-RTOS2 thread flags are the task notification at index 0, so native FreeRTOS
  code that uses index-0 notifications on a CSP process's thread collides with the library.

### Misuse ends in the fatal-error hook

A second ALTing process on a channel end, a failed RTOS object creation, a thread `Run()` cannot create,
a 17th guard and similar errors call
`csp4cmsis_fatal_error(const char* message)` instead of continuing with a broken channel. The default
(weak) stores the message in `csp4cmsis_last_fatal_error` for the debugger and spins. Applications may
call it too, for their own unrecoverable errors.

**It may be called from an ISR** (the library calls it from threads, but an application may call it from
an interrupt handler, as the completion pattern in section 5 does). A replacement must therefore be
ISR-safe: no `printf` or other RTOS or stdio call; for example, store the message and halt. It must not
return:

```cpp
#include "csp/csp4cmsis.h"

// For the debugger.
const char* volatile app_fatal_message = nullptr;

extern "C" void csp4cmsis_fatal_error(const char* message) {
    __disable_irq();
    app_fatal_message = message;      // e.g. "CSP4CMSIS: rendezvous channel: second ALTing reader ..."
    for (;;) { }
}
```

---

## 7. Barrier

`Barrier(N)` (N ≥ 1) is a reusable synchronisation point: `sync()` blocks until N processes have
called it, then releases all of them, and the next phase begins. A process that runs ahead into the
next phase waits there; it cannot overtake a slow process of the previous phase.

```cpp
#include "csp/csp4cmsis.h"

using namespace csp;

static Barrier step(3);                 // three processes per step

class Stage : public CSProcessStatic<256> {
public:
    void run() override {
        while (true) {
            // ... this stage's share of the step ...
            step.sync();                // wait for the other two stages
        }
    }
};

static Stage s1, s2, s3;

void start_stages(void) {
    Run(InParallel(s1, s2, s3), ExecutionMode::StaticNetwork);
}
```

---

## 8. Memory guarantees

- **The library allocates nothing.** No `malloc`, `operator new`, `pvPortMalloc()` or other allocator
  call anywhere in the library, during construction or at run time.
- **Static RTOS objects:** by default (unless `CSP4CMSIS_DYNAMIC_ALLOCATION`), every RTOS object the library creates
  (process threads, `Run()`'s completion semaphore, the semaphores of channels and `Barrier`) has a
  statically allocated control block. Process stacks are always static (`CSProcessStatic<N>`).
  `RelTimeoutGuard` creates no RTOS object at all. `Alternative`s and guards are ordinary objects on
  the process's stack.
- **Channels** store their elements inside the channel object; declare them static so they live in
  `.bss`/`.data`.
- **Not covered:** the RTOS's own configuration, objects your application creates, and the **C
  library**. newlib's `printf` allocates its `stdout` buffer with `malloc`: 1032 bytes measured on the
  NUCLEO-G474RE (book examples); `setvbuf(stdout, NULL, _IONBF, 0)` before the first `printf` removes
  that allocation.
- What a completely heap-free system additionally needs (RTOS settings and two adapter workarounds):
  [configuration guide, section 6](https://github.com/OliverFaust/CSP4CMSIS/blob/v3.0.0/Documentation/CSP4CMSIS_Configuration.md).

**What is verified:**

- **Protocols, model-checked with ProB** (CSP-M models in
  [`docs/formal/`](https://github.com/OliverFaust/CSP4CMSIS/tree/v3.0.0/docs/formal)): the rendezvous
  ALT protocol (`alt_owrv_extended.csp`), the buffered channel with its ALT wakeup
  (`buffered_channel_v2.csp`) and the timer-free ALT timeout (`alt_timeout_deadline.csp`). These are
  models of the protocols, not of the C++ code.
- **The code, by the regression suite:**
  [`tests/fvp_sse300/`](https://github.com/OliverFaust/CSP4CMSIS/tree/v3.0.0/tests/fvp_sse300) on the
  Corstone-300 FVP (FreeRTOS and Keil RTX5, Arm Compiler 6 and GCC, including builds with RTOS dynamic
  allocation disabled), and with ST's wrapper on the MPS2 Cortex-M4 FVP;
  [`tests/hw_nucleo_g474/`](https://github.com/OliverFaust/CSP4CMSIS/tree/v3.0.0/tests/hw_nucleo_g474)
  on a NUCLEO-G474RE; and compile-time checks in
  [`tests/compile_checks/`](https://github.com/OliverFaust/CSP4CMSIS/tree/v3.0.0/tests/compile_checks).

---

## 9. What changed in 2.0, 2.1 and 3.0

- **Channels:** rendezvous channels are Block only; sampling policies (KeepNewest/KeepOldest) only on
  buffered channels. Buffered channels are static ring buffers. Elements must be trivially copyable.
- **Interrupts:** ISRs write only to buffered channels, through `isrWriter()`;
  `Chanout<T>::putFromISR()` is gone.
- **ALT:** both ends of a rendezvous channel may alternate and communicate; misuse (a second ALTing
  process on an end) is reported through `csp4cmsis_fatal_error()`. `Barrier` is reusable.
- **RTOS:** CMSIS-RTOS2 throughout (`osPriority_t`, thread flags), static allocation of every RTOS
  object, no allocation in the library.
- **2.0.1:** `RelTimeoutGuard` without an RTOS timer (fixes the timer-task priority problem of 2.0.0);
  builds without packs (`CSP4CMSIS_DEVICE_HEADER`, include path `inc/` only); the STM32CubeIDE guide.
- **2.1.0:** a failed thread creation in `Run()` and a 17th ALT guard are fatal errors; `Seconds()` and
  `Milliseconds()` round up; `SleepFor(Time)`; `CSP_DEFAULT_NETWORK_PRIORITY`. Deprecated (removed in
  3.0): `Run(CSProcess&)`, `CSP_DEFAULT_TASK_PRIORITY`, `CSP_LEGACY_PARALLEL_PRIORITY`,
  `One2OneChannel`, `BufferedOne2OneChannel`. `CSProcess::endProcess()` (never called) is removed.
- **3.0.0 (the stable "book API"):** removed the 2.1.0 deprecations, `Any2OneChannel` and
  `SamplingChannel` (both now `Channel<T>`), `SamplingBufferedChannel` (now `BufferedChannel<T, N, P>`),
  `SleepFor(ticks)` (now
  `SleepFor(Ticks(n))`), `Run()` without a mode, and `SignalChannel<>` (now `SignalChannel`). Static
  allocation is the default and the backend is detected; a wrong interrupt-priority define or a stack
  below 64 words does not compile. The channel and ALT protocols are unchanged.

Details and migration notes:
[`docs/CHANGES_2.0.md`](https://github.com/OliverFaust/CSP4CMSIS/blob/v3.0.0/docs/CHANGES_2.0.md),
[`docs/CHANGES_2.0.1.md`](https://github.com/OliverFaust/CSP4CMSIS/blob/v3.0.0/docs/CHANGES_2.0.1.md),
[`docs/CHANGES_2.1.0.md`](https://github.com/OliverFaust/CSP4CMSIS/blob/v3.0.0/docs/CHANGES_2.1.0.md),
[`docs/CHANGES_3.0.md`](https://github.com/OliverFaust/CSP4CMSIS/blob/v3.0.0/docs/CHANGES_3.0.md).
