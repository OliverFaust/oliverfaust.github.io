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
