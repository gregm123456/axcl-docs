# Introduction

## Overview

AXCL is a C and Python language API library for developing deep neural network inference, transcoding, and other applications on the [AXERA](https://www.axera-tech.com/) chip platform. It provides APIs for runtime resource management, memory management, model loading and execution, and media data processing. The logical architecture diagram is shown below:

![](../res/axcl_architecture.svg)



## Concepts

The AXCL runtime library has basic concepts such as Device, Context, Stream, and Task, and their relationships are shown in the following diagram:

![](../res/axcl_concept.svg)

- Device: Hardware device that provides computing and media processing, connected to the Host via PCIe interface.
  - The Device lifecycle starts with the first call to the `axclrtSetDevice` interface.
  - Device uses reference counting to manage lifecycle; `axclrtSetDevice` increments the reference count by 1, `axclrtResetDevice` decrements it by 1.
  - When the reference count decreases to zero, the Device resources in this process become unavailable.
  - **If multiple Devices exist, memory resources cannot be shared between multiple Devices**.
- Context: Execution context container that manages lifecycles including Stream and memory; Context is bound to application threads. Context must belong to a unique Device. Context is divided into implicit and explicit types:
  - Implicit Context (i.e., default Context):
    - When the `axclrtSetDevice` interface is called to specify a device, the system creates an implicit Context, which is automatically destroyed when the `axclrtResetDevice` interface reference count reaches zero.
    - A Device will have only one implicit Context, and the implicit Context cannot be destroyed via the `axclrtDestroyContext` interface.
  - Explicit Context:
    - Explicitly created by calling the `axclrtCreateContext` interface, destroyed by calling the `axclrtDestroyContext` interface.
    - Contexts within the process are shared and can be switched via `axclrtSetCurrentContext`. **It is recommended to create and destroy explicit Contexts for each thread in the application**.
- Stream: Execution task stream, belonging to Context; tasks in the same Stream are executed in order. Stream is divided into implicit and explicit types:
  - Implicit Stream (i.e., default Stream):
    - Each Context creates an implicit Stream, whose lifecycle belongs to the Context.
    - Native SDK modules (such as `AXCL_VDEC_XXX`, `AXCL_SYS_XXX`, etc.) use implicit Stream.
  - Explicit Stream:
    - Created by the `axclrtCreateStream` interface, destroyed by the `axclrtDestroyStream` interface.
    - When the Context to which the explicit Stream belongs is destroyed or its lifecycle ends, the Stream cannot be used even if it has not been destroyed.
- Task: Device task execution entity, invisible to applications.



## Application Threads and Context

- An application thread must be bound to a Context; all Device tasks must be based on Context.
- There will be a unique Context in a thread, with the Device for task execution in this thread bound to the Context.
- Application threads support creating multiple Contexts via `axclrCreateContext`, but a thread can only use one Context at a time. When multiple Contexts are created consecutively, the thread is bound to the last created Context.
- Within the process, the current thread's Context can be bound via `axclrtSetCurrentContext`; after multiple consecutive calls to `axclrtSetCurrentContext`, the final binding is the last one.
- For multi-threading, it is recommended to create an explicit Context for each thread.
- It is recommended to switch Devices via `axclrtSetCurrentContext`.

:::{tip}

   The SDK axcl/sample/runtime provides sample code on how to create and destroy Context in threads.

:::


## Reference Hardware

### AX-M1

[Radxa AX-M1](https://docs.radxa.com/aicore/ax-m1) is a high-performance M.2 acceleration module launched based on AXERA's AX8850 SoC, featuring high computing power, high energy efficiency, and other characteristics, specially designed for edge intelligent computing and AI inference applications.
Integrates multi-core high-performance CPU and high-computing NPU, with excellent multimedia processing capabilities, providing efficient and flexible hardware support for various edge AI scenarios.

**Product Image**

![](../res/AX-M1.png)

**Product Specifications**

|              | Description                                                    |
| ------------ | ------------------------------------------------------- |
| CPU       | Octa-core Cortex-A55, up to 1.5GHz main frequency                            |
| Memory         | 8GB LPDDR4x                                             |
| NPU         | 24TOPS@INT8; supports matrix operation units and intelligent vision engines                 |
|              | Supports CNN, Transformer model deployment, supports LLM, VLM deployment      |
| VPU        | Supports H.264/H.265 8K@30fps encoding/decoding and 16-channel 1080p@30fps decoding  |
| Hardware Adaptation	     | Supports host platforms such as Intel, AMD, Rockchip                    |
| System Adaptation	     | Supports mainstream Linux distributions such as Ubuntu, Debian, CentOS         |
| Form Factor     | M.2 2280, M Key                                         |
| Operating Voltage     | 3.3 V                                                   |
| System Power Consumption	    | ≤ 8W                                                   |

### LLM-8850

[M5Stack LLM‑8850 Card](https://docs.m5stack.com/zh_CN/guide/ai_accelerator/overview) is an M.2 M-Key 2242 AI acceleration card from M5Stack for edge devices.

**Product Image**

![](../res/LLM8850.png)

**Product Specifications**

|              | Description                                                    |
| ------------ | ------------------------------------------------------- |
| CPU       | Octa-core Cortex-A55, up to 1.5GHz main frequency                            |
| Memory         | 8GB LPDDR4x                                             |
| NPU         | 24TOPS@INT8; supports matrix operation units and intelligent vision engines                 |
|              | Supports CNN, Transformer model deployment, supports LLM, VLM deployment      |
| VPU        | Supports H.264/H.265 8K@30fps encoding/decoding and 16-channel 1080p@30fps decoding  |
| Hardware Adaptation	     | Supports host platforms such as Intel, AMD, Rockchip                    |
| System Adaptation	     | Supports mainstream Linux distributions such as Ubuntu, Debian, CentOS         |
| Form Factor     | M.2 2242, M Key                                         |
| Operating Voltage     | 3.3 V                                                   |
| System Power Consumption	    | ≤ 8W                                                   |
