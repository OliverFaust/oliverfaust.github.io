---
layout: page
title: API Reference (v1.0)
---

# CSP4CMSIS API Reference

This document provides a technical summary of the core primitives used in **CSP4CMSIS**. The library is designed to map the mathematical rigor of *Communicating Sequential Processes* (CSP) onto the deterministic requirements of ARM Cortex-M microcontrollers.

---

## 1. Process Management

### `CSProcess`
The fundamental unit of execution. Every task in your network must inherit from this class.

* **Usage**: Inherit from `CSProcess` and implement the `run()` method.
* **Lifecycle**: The `run()` method contains the infinite loop of the process. When using `ExecutionMode::StaticNetwork`, these processes are mapped to underlying RTOS tasks (e.g., FreeRTOS tasks).

```cpp
class MyProcess : public CSProcess {
public:
    void run() override {
        while(true) {
            // Process Logic
        }
    }
};
```

## 2. Channel Communication
Channels provide the synchronization "handshake" (Rendezvous) between processes.

### Channel<T> (Rendezvous)
The default point-to-point synchronization primitive (Capacity 0).
* **Behavior:** Strict Rendezvous. The sender blocks until the receiver arrives, and vice versa.
* **Declaration:** `static Channel<Message> msg_chan;`

### `BufferedOne2OneChannel<T, N>` (Asynchronous)
A buffered version of the point-to-point channel that decouples the timing of the sender and receiver.
* **Usage:** Ideal for "Pipeline" topologies where execution times vary (e.g., SD Card I/O vs. NPU Inference).
* **Behavior:**
    * **Sender:** Only blocks if the buffer is full.
    * **Receiver:** Only blocks if the buffer is empty.
* **Declaration:**
```C++
// Static allocation of a channel with a 16-slot buffer
static BufferedOne2OneChannel<work_packet_t, 16> c1;
```

### `Chanin<T>` and `Chanout<T>`
These represent the input and output ports of a channel. Processes should store these as members to interact with the network.
*Read (Synchronous): Use the >> operator. This blocks the calling process until data is available.
```cpp
in >> msg;
```
* Write (Synchronous): Use the `<<` operator. This blocks the calling process until a receiver is ready.
```cpp
out << msg;
```

## 3. Interrupt Handling
One of the most critical features for embedded systems is the ability to signal processes from an Interrupt Service Routine (ISR).

`putFromISR(T value)`

This method allows an ISR to send a message or trigger to a waiting process.
* **Context:** Must be called from within a HAL Callback or Exception Handler.
* **Behavior:** Non-blocking. It places the value into the channel synchronization register and alerts the scheduler.Example (HAL Callback):
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
```C++
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

### Deterministic Synchronization
Unlike standard RTOS queues, csp4cmsis channels default to *Rendezvous* (capacity 0). This means:
* The *Sender* blocks until the *Receiver* arrives.
* The *Receiver* blocks until the *Sender* arrives.
* The data transfer happens only when both "shake hands," providing a formal proof of synchronization.
