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

### `Channel<T>`
A point-to-point synchronization primitive. In the current version, these are typically declared as `static` to ensure Zero-Heap allocation.
* Declaration: static Channel<Message> msg_chan;
* Endpoints: Access the endpoints via .reader() and .writer().

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
4. Network OrchestrationTo run the processes, they must be composed into a network.InParallel(P1, P2, ...)This function represents the CSP Parallel operator ($\parallel$). It groups process instances together for simultaneous execution.Run(...)The entry point for the CSP engine.Execution Mode: ExecutionMode::StaticNetwork is used for high-integrity systems where tasks are launched once at startup and never deleted.Example:C++void MainApp_Task(void* params) {
    static MyProcess p1(chan.writer());
    static MyOtherProcess p2(chan.reader());

    Run(
        InParallel(p1, p2),
        ExecutionMode::StaticNetwork
    );
}
5. Best Practices: "Order and Method"Static Allocation: Always declare Channels and Processes as static within your task entry point. This guarantees that the memory is reserved at compile-time, satisfying safety-critical requirements.Rendezvous Safety: Remember that in >> msg and out << msg are blocking. If your system "hangs," examine your network for Deadlock—a situation where two processes are waiting for each other in a circle.POD Data: Messages should be "Plain Old Data" (structs with simple types) to ensure efficient copying across channel boundaries.
