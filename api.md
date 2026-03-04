# CSP4CMSIS API Reference (v1.1)

This document provides a technical summary of the core primitives used in **CSP4CMSIS**. Version 1.1 introduces **Buffer Policies**, allowing for deterministic, non-blocking "Lossy" communication—ideal for high-frequency sensor data on ARM Cortex-M microcontrollers.

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

## 2. Channel Communication

Channels provide the synchronization "handshake" between processes. In v1.1, the behavior of this handshake is governed by the `BufferPolicy`.

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
    * **`Channel<T>`**: An alias to model a standart CSP channel.
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
    * **`Block` (Default):** Sender blocks only if the buffer is $N$ items full.
    * **`KeepNewest` / `KeepOldest`:** Sender never blocks. If the buffer is full, the policy dictates which item is dropped to maintain the $N$ capacity.
* **Aliases**: 
    * **`BufferedChannel<T, N>`:** An alias to model a standart CSP channel.
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

## 3. Interrupt Handling
The library supports safe communication from Interrupt Service Routines (ISRs) to Tasks.

`putFromISR(T value)`

This method allows an ISR to send a message or trigger to a waiting process.
* **Context:** Must be called from within a HAL Callback or Exception Handler.
* **Non-Blocking:** This method never waits.
* **Policy Impact:** With `KeepNewest`, it will always succeed by overwriting old data if necessary. With `Block`, it returns `false` if the buffer is full.
```C++
extern "C" void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == MY_PIN) {
        my_chan.writer().putFromISR(trigger_t{});
        portYIELD_FROM_ISR(pdTRUE); // Force context switch to receiver
    }
}
```
## 4. Network Orchestration
To run the processes, they must be composed into a network.

`InParallel(P1, P2, ...)`

This function represents the CSP Parallel operator ($\parallel$). It groups process instances together for simultaneous execution.

`Run(...)`

The entry point for the CSP engine.
* **Execution Mode:** `ExecutionMode::StaticNetwork` is used for high-integrity systems where tasks are launched once at startup and never deleted.
* **Example:**
```cpp
void MainApp_Task(void* params) {
    static MyProcess p1(chan.writer());
    static MyOtherProcess p2(chan.reader());

    Run(
        InParallel(p1, p2),
        ExecutionMode::StaticNetwork
    );
}
```
---

## 5. External Choice (Alternative / ALT)

The `Alternative` class implements the CSP External Choice operator ($\square$). it allows a process to wait on multiple input channels simultaneously, proceeding as soon as any one of them is ready.

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
## 6. Memory and Safety Guarantees
### Zero-Heap Execution
`csp4cmsis` is designed for safety-critical ARM environments where dynamic memory allocation (the "Heap") is prohibited after system initialization.
1. **Static Channels:** Channels should be declared as static to reside in the `.data` or `.bss` segments.
2. **Stack-Based ALT:** The Alternative object and its guards reside on the task stack, ensuring deterministic memory usage.
3. **No New/Delete:** The library does not perform hidden allocations during communication or selection.
4. **Deterministic Latency:** $O(1)$ time complexity for all channel operations, including KeepNewest overwrites.
5. **Thread Safety:** Atomic pointer updates ensure the receiver never reads partially overwritten data during a "Lossy" event.

### Deterministic Synchronization
Unlike standard RTOS queues, csp4cmsis channels default to *Rendezvous* (capacity 0). This means:
* The *Sender* blocks until the *Receiver* arrives.
* The *Receiver* blocks until the *Sender* arrives.
* The data transfer happens only when both "shake hands," providing a formal proof of synchronization.
