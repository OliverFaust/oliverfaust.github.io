# CSP4CMSIS: Bridging Formal Methods and AI at the Edge

Welcome to the official portal for the **CSP4CMSIS** ecosystem. Based in Cambridge, this project focuses on the implementation of **Communicating Sequential Processes (CSP)** for ARM-based embedded systems, providing a mathematically sound foundation for high-integrity AI deployment.

## The Vision: Formal-AI-Edge (F.A.E.)

In the era of "AI at the Edge," the industry faces a **Safety Wall**. Traditional "Super-loop" architectures cannot guarantee the deterministic behavior required for safety-critical systems when integrated with non-deterministic AI inference.

**CSP4CMSIS** solves this by providing a zero-heap, static-memory mapping of CSP primitives directly onto **ARM CMSIS-RTOS** and **FreeRTOS**.

### Core Formalisms
We leverage the power of process algebra to model system behavior:
* **External Choice ($\square$):** For truly reactive, event-driven sensor handling.
* **Parallel Composition ($\parallel$):** For verified, deadlock-free process networks.
* **Internal Choice ($\sqcap$):** To formally wrap and isolate AI decision-making.

---

## Documentation
For a detailed technical breakdown of the library primitives, including Channels, Processes, and External Choice, please see our:

👉 **[CSP4CMSIS API Reference](./api.md)**

--- 

## Technical Demos & Repositories

Explore our implementation across the ARM ecosystem:

| Target Hardware | Complexity | Repository Link |
|:---|:---|:---|
| **Nucleo-F401RE** | Entry Level | [View Repo](https://github.com/OliverFaust/CSP4CMSIS-Nucleo) |
| **Nucleo-G474RE** | Intermediate | [View Repo](https://github.com/OliverFaust/CSP4CMSIS_for_NUCLEO-G474RE) |
| **Cortex M55 / Ethos NPU** | Flagship / AI | [View Repo](https://github.com/OliverFaust/CSP4CMSIS) |

---

## 2026 Funding Call: TechLocal AI Professional Accelerator

We are currently seeking partners for the **DSIT TechLocal AI Professional Degree and Traineeship Accelerator**. 

**The Objective:** To develop a Level 7 (Masters equivalent) curriculum focused on "Certifiable AI." We aim to provide graduates with the skills to deploy AI on ARM Ethos NPUs using formally verified process networks.

**Calling ARM & Cambridge-based Industry Partners:** If you are interested in supporting this traineeship model or integrating these formal methods into your safety-critical workflow, please get in touch.

[Contact Oliver Faust via GitHub](https://github.com/OliverFaust)
