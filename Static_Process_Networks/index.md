# Static Process Networks in Embedded Systems
### A Rigorous Approach to Concurrency

Superloops and RTOS-centric designs make concurrency bugs easy to write and hard to find. This book takes a different approach, built on **Communicating Sequential Processes (CSP)**: the set of processes and channels in an embedded system is fixed at compile time, so the whole network can be reasoned about — and verified — before it ever runs on hardware.

Every chapter follows the same shape: a minimal, complete example; a CSP specification and network diagram; and an "under the hood" look at how the [**CSP4CMSIS**](https://github.com/OliverFaust/CSP4CMSIS) library maps that specification onto FreeRTOS primitives.

### Core Formalisms
* **External Choice (☐):** a process reacting to whichever of several channels fires first, without polling.
* **Parallel Composition (∥):** independent processes composed into a single, deadlock-checkable network.
* **Internal Choice (⊓):** used in the closing chapter to model — and expose the risks of — autonomous decision-making in AI agents.

---

## Chapters & Code {#chapters}

Chapters 3–8 build the core model on any Arm Cortex-M board (examples use an **STM32 NUCLEO-G474RE**). Chapter 9 extends it to NPU-accelerated hardware.

### Part I — Core Concepts

| Chapter | Focus | Hardware | Repository |
|:---|:---|:---|:---|
| **3. The Process** | A single `CSProcess` mapped to a FreeRTOS task | NUCLEO-G474RE | [View Repo](https://github.com/OliverFaust/nucleo-g474re_The_Process) |
| **4. Processes and Channels** | Rendezvous communication between processes | NUCLEO-G474RE | [View Repo](https://github.com/OliverFaust/nucleo-g474re_Processes_and_Channels) |
| **5. Interrupts** | ISR-safe injection of external events via `putFromISR` | NUCLEO-G474RE | [View Repo](https://github.com/OliverFaust/nucleo-g474re_Interrupts) |
| **6. Sensor Data Processing Network** | A multi-stage, high-frequency acquisition pipeline | NUCLEO-G474RE + L3G4200D gyroscope | [View Repo](https://github.com/OliverFaust/nucleo-g474re_Sensor_Data_Processing_Network) |
| **7. Alternation** | External choice across multiple senders via `Alternative` | NUCLEO-G474RE | [View Repo](https://github.com/OliverFaust/nucleo-g474re_Alternation) |

### Part II — Advanced Topics

| Chapter | Focus | Hardware | Repository |
|:---|:---|:---|:---|
| **8. Neuropathways** | A Capture → Inference → Output vision pipeline across heterogeneous CPU/NPU hardware | Himax WE2 (Cortex-M55 + Ethos-U55) | [View Repo](https://github.com/OliverFaust/CSP4CMSIS/tree/main/EPII_CM55M_APP_S/app/scenario_app/csp4cmsis_allon_sensor_tflm) |

### Theory: Agency

Chapter 9, **Agency**, is a theoretical closing chapter with no accompanying repository — it uses internal choice (⊓) to model how independent, autonomous decisions in AI-agent-style systems can lead to deadlock, as a cautionary note for embedding learning components in a static network.

---

## Documentation

* 👉 **[CSP4CMSIS API Reference](https://oliverfaust.github.io/CSP4CMSIS/api)** — library primitives used throughout the book.
* 📄 **[Download Quick Reference Cheatsheet (PDF)](https://oliverfaust.github.io/CSP4CMSIS/CSP4CMSIS_cheatcheet.pdf)** — CSP-to-C++ mappings at a glance.

---

## About

Written for embedded software engineers, students, and researchers who want to move beyond ad-hoc concurrency toward a rigorous, verifiable design discipline. All core examples run on Arm Cortex-M microcontrollers; the advanced chapters target NPU-enabled boards such as the Himax WE2. CSP4CMSIS is open source and depends only on FreeRTOS.

[View the CSP4CMSIS library](https://github.com/OliverFaust/CSP4CMSIS) · [Contact via GitHub](https://github.com/OliverFaust)

<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.7/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
