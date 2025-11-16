# SDK API

## Overview
`AXCL API` is divided into two parts: the `Runtime API` and the `Native API`.
The `Runtime API` is a self-contained set of APIs that currently include `Memory` APIs for memory management and `Engine` APIs to drive the Axera NPU. For compute-only card deployments without encoding/decoding features, the `Runtime API` is sufficient. If you need encoding/decoding, also refer to the `Native API` and the `FFMPEG` integration.

## Runtime
The `Runtime API` allows the host system to invoke the NPU for compute tasks. The `Memory` APIs cover allocation/deallocation on both host and device, while the `Engine` APIs cover model initialization, IO setup, and execution of inference on the NPU.

### runtime
(axclinit)=
#### axclInit

```c
axclError axclInit(const char *config);
```

**Description**:

System initialization; synchronous call.

**Parameters**:

- `config [IN]`: Path to a JSON configuration file.
  - The JSON file may specify runtime options such as log level; see [FAQ](https://github.com/AXERA-TECH/axcl-docs/wiki/0.FAQ#how-to-configure-runtime-log--level) for details.
  - Passing NULL or a missing JSON file will cause the runtime to use default configuration.


**Restrictions**:

- Must be paired with [`axclFinalize`](#axclfinalize) to properly clean up resources.
- Must be called before using any other AXCL APIs in an application.
- Should only be called once per process.

---
(axclfinalize)=
#### axclFinailze

```c
axclError axclFinalize();
```

**Description**:

De-initialize the runtime and free AXCL resources associated with the process; synchronous call.

**Restrictions**:

- Must be paired with [`axclInit`](#axclinit).
- Applications should explicitly call this before exiting.
- For C++ programs, avoid calling this in destructors because the undefined order of global destructors may cause crashes on process exit.

---
(axclrtgetversion)=
#### axclrtGetVersion

```c
axclError axclrtGetVersion(int32_t *major, int32_t *minor, int32_t *patch);
```

**Description**:

Query the runtime version information; synchronous call.

**Parameters**:

- `major [OUT]`: Major version number.
- `minor[OUT]`: Minor version number.
- `patch [OUT]`: Patch version number.

**Restrictions**:

No special restrictions.

---
(axclrtgetsocname)=
#### axclrtGetSocName

```c
const char *axclrtGetSocName();
```

**Description**:

Return the SoC name string associated with the active device; synchronous call.

**Restrictions**:

No special restrictions.

---
(axclrtsetdevice)=
#### axclrtSetDevice

```c
axclError axclrtSetDevice(int32_t deviceId);
```

**Description**:

Bind the current process or thread to a specific device and implicitly create a default Context; synchronous call.

**Parameters**:

- `deviceId [IN]`: Device ID to bind.

**Restrictions**:

- This API implicitly creates a default Context that will be automatically reclaimed by the runtime when `axclrtResetDevice` is called; do not call `axclrtDestroyContext` on implicit contexts.
- If the same device is set by multiple threads in the same process, they will share the same implicit Context.
- This API must be paired with `axclrtResetDevice`. The runtime tracks references and only releases resources when the reference count reaches zero.
- To switch devices in multi-device scenarios, use this API or `axclrtSetCurrentContext`.

---
(axclrtresetdevice)=
#### axclrtResetDevice

```c
axclError axclrtResetDevice(int32_t deviceId);
```

**Description**:

Reset the device and free device resources, including contexts created implicitly or explicitly; synchronous call.

**Parameters**:

- `deviceId [IN]`: Device ID.

**Restrictions**:

- For Contexts explicitly created with [`axclrtCreateContext`](#axclrtcreatecontext), it is recommended to call [`axclrtDestroyContext`](#axclrtdestroycontext) to destroy the Context before calling this API to release device resources.
- Pair with [`axclrtSetDevice`](#axclrtsetdevice); the runtime will automatically reclaim the default Context resource.
- Multiple calls are allowed; the runtime uses reference counting and will release device resources only when the reference count reaches zero.
- **Ensure `axclrtResetDevice` is called before the application exits, especially after handling signals; otherwise a C++ "terminated abort" exception may occur.**

---
(axclrtgetdevice)=
#### axclrtGetDevice

```c
axclError axclrtGetDevice(int32_t *deviceId);
```

**Description**:

Get the current device ID in use; synchronous call.

**Parameters**:

- `deviceId [OUT]`: Device ID.

**Restrictions**:

- The API returns an error if no device has been set using [`axclrtSetDevice`](#axclrtsetdevice) or [`axclrtCreateContext`](#axclrtcreatecontext).

---
(axclrtgetdevicecount)=
#### axclrtGetDeviceCount

```c
axclError axclrtGetDeviceCount(uint32_t *count);
```

**Description**:

Get the total number of connected devices; synchronous call.

**Parameters**:

- `count [OUT]`: Number of devices connected.

**Restrictions**:

No special restrictions.

---
(axclrtgetdevicelist)=
#### axclrtGetDeviceList

```c
axclError axclrtGetDeviceList(axclrtDeviceList *deviceList);
```

**Description**:

Get all connected device IDs; synchronous call.

**Parameters**:

- `deviceList[OUT]`: Structure with information about all connected device IDs.

**Restrictions**:

No special restrictions.

---
(axclrtsynchronizedevice)=
#### axclrtSynchronizeDevice

```c
axclError axclrtSynchronizeDevice();
```

**Description**:

Synchronize and complete all tasks on the current device; synchronous call.

**Restrictions**:

At least one device must be active.

---
(axclrtGetDeviceProperties)=
#### axclrtGetDeviceProperties

```c
axclError axclrtGetDeviceProperties(int32_t deviceId, axclrtDeviceProperties *properties);
```

**Description**:

Retrieve device UID, CPU utilization, NPU utilization, memory details, and similar properties; synchronous call.

**Restrictions**:

---
(axclrtcreatecontext)=
#### axclrtCreateContext

```c
axclError axclrtCreateContext(axclrtContext *context, int32_t deviceId);
```

**Description**:

Explicitly create a Context in the current thread; synchronous call.

**Parameters**:

- `context [OUT]`: Created Context handle.
- `deviceId [IN]`: Device ID.

**Restrictions**:

- If a worker thread needs to call AXCL APIs, it must create a Context with this API or bind an existing Context using [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext).
- If the specified device has not been activated, this API will internally activate the device.
- Call [`axclrtDestroyContext`](#axclrtdestroycontext) to explicitly free Context resources.
- Multiple threads may share a Context (bound using [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext)), but task execution depends on system thread scheduling; the user must manage synchronization between threads. For multi-threaded programs, we recommend creating a dedicated Context per thread to improve maintainability.

---
(axclrtdestroycontext)=
#### axclrtDestroyContext

```c
axclError axclrtDestroyContext(axclrtContext context);
```

**Description**:

Explicitly destroy a Context; synchronous call.

- **Parameters**:

- `context [IN]`: The Context handle to destroy.

**Restrictions**:

- Only Contexts created by [`axclrtCreateContext`](#axclrtcreatecontext) may be destroyed using this API.

---
(axclrtsetcurrentcontext)=
#### axclrtSetCurrentContext

```c
axclError axclrtSetCurrentContext(axclrtContext context);
```

**Description**:

Bind a Context to the current thread; synchronous call.

**Parameters**:

- `context [IN]`: Context handle.

**Restrictions**:

- If the thread binds Context multiple times with this API, the last set Context will be used.
- If the device associated with the bound Context has been reset by [`axclrtResetDevice`](#axclrtresetdevice), do not set that Context as the thread's Context; attempting to do so may cause an exception.
- It is recommended that a Context is created and used on the same thread. If a Context is created in thread A and used in thread B, the application must ensure proper execution ordering of tasks that use that Context.

---
(axclrtgetcurrentcontext)=
#### axclrtGetCurrentContext

```c
axclError axclrtGetCurrentContext(axclrtContext *context);
```

**Description**:

Retrieve the Context handle bound to the current thread; synchronous call.

**Parameters**:

- `context [OUT]`: Current context handle.

**Restrictions**:

- The calling thread must have bound a Context using [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext) or created one with [`axclrtCreateContext`](#axclrtcreatecontext) in order to retrieve it.
- If the thread binds Context multiple times with [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext), the last bound Context will be returned.



### memory

(axclrtmalloc)=
#### axclrtMalloc

```c
axclError axclrtMalloc(void **devPtr, size_t size, axclrtMemMallocPolicy policy);
```

**Description**:

Allocate non-cached physical memory on the device; returns the allocated pointer via `devPtr`; synchronous call.

**Parameters**:

- `devPtr [OUT]`: Returns the pointer to allocated device physical memory.
- `size [IN]`: Size of the allocation in bytes.
- `policy[IN]`: Memory allocation policy; currently unused.

**Restrictions**:

- This API allocates contiguous physical memory from the device's CMM pool.
- Allocated memory is non-cached; cache coherence handling is not required.
- Free the memory via [`axclrtFree`](#axclrtfree).
- Frequent allocations and deallocations may degrade performance. Consider preallocation or reuse strategies.

---
(axclrtmalloccached)=
#### axclrtMallocCached

```c
axclError axclrtMallocCached(void **devPtr, size_t size, axclrtMemMallocPolicy policy);
```

**Description**:

Allocate cached physical memory on the device; return the allocated pointer via `devPtr`; synchronous call.

**Parameters**:

- `devPtr [OUT]`: Returns the pointer to the allocated device-side physical memory.
- `size [IN]`: Size of the allocation in bytes.
- `policy [IN]`: Memory allocation policy; currently unused.

**Restrictions**:

- This API allocates contiguous physical memory from the device's CMM pool.
- Allocated memory is cached; the application must handle cache coherence.
- Free the memory via [`axclrtFree`](#axclrtfree).
- Frequent allocations and deallocations may degrade performance; prefer preallocation or pooling.

---
(axclrtfree)=
#### axclrtFree

```c
axclError axclrtFree(void *devPtr);
```

**Description**:

Free device-side allocated memory; synchronous call.

**Parameters**:

- `devPtr [IN]`: Device memory to be freed.

**Restrictions**：

- Only device-side memory allocated by [`axclrtMalloc`](#axclrtmalloc) or [`axclrtMallocCached`](#axclrtmalloccached) may be freed.

---
(axclrtmemflush)=
#### axclrtMemFlush

```c
axclError axclrtMemFlush(void *devPtr, size_t size);
```

**Description**:

Flush cached data to DDR and mark cache lines as invalid; synchronous call.

**Parameters**:

- `devPtr [IN]`: Pointer to the start of the DDR memory region to flush.
- `size [IN]`: Size in bytes to flush.

**Restrictions**：

No special restrictions.

---
(axclrtmeminvalidate)=
#### axclrtMemInvalidate

```c
axclError axclrtMemInvalidate(void *devPtr, size_t size);
```

**Description**:

Invalidate cached data for the specified DDR memory region; synchronous call.

**Parameters**:

- `devPtr [IN]`: Pointer to the start of the DDR memory region to invalidate.
- `size [IN]`: Size in bytes to invalidate.

**Restrictions**：

- No special restrictions.

---
(axclrtmallochost)=
#### axclrtMallocHost

```c
axclError axclrtMallocHost(void **hostPtr, size_t size);
```

**Description**:

Allocate virtual memory on the host; synchronous call.

**Parameters**:

- `hostPtr [OUT]`: Address of the allocated host memory.
- `size [IN]`: Size of the allocation in bytes.

**Restrictions**:

- Memory allocated with [`axclrtMallocHost`](#axclrtmallochost) must be freed with [`axclrtFreeHost`](#axclrtfreehost).
- Frequent allocate/free operations may degrade performance; consider pooling or preallocation.
- You can also use standard `malloc` on host memory, but `axclrtMallocHost` is recommended.

---
(axclrtfreehost)=
#### axclrtFreeHost

```c
axclError axclrtFreeHost(void *hostPtr);
```

**Description**:

Free memory allocated with [`axclrtMallocHost`](#axclrtmallochost); synchronous call.

**Parameters**:

- `hostPtr [IN]`: Address of the host memory to free.

**Restrictions**:

- Only host memory allocated with `axclrtMallocHost` should be freed with `axclrtFreeHost`.

---
(axclrtmemset)=
#### axclrtMemset

```c
axclError axclrtMemset(void *devPtr, uint8_t value, size_t count);
```

**Description**:

Initialize device memory allocated by `axclrtMalloc` or `axclrtMallocCached`; synchronous call.

**Parameters**:

- `devPtr [IN]`: Pointer to the device memory to initialize.
- `value [IN]`: Value to set.
- `count [IN]`: Number of bytes to initialize.

**Restrictions**:

- Only device memory allocated by `axclrtMalloc` or `axclrtMallocCached` can be initialized.
- For memory allocated with `axclrtMallocCached`, call `axclrtMemInvalidate` to maintain coherence if necessary.
- Host memory allocated by `axclrtMallocHost` should be initialized with standard `memset`.

---
(axclrtmemcpy)=
#### axclrtMemcpy

```c
axclError axclrtMemcpy(void *dstPtr, const void *srcPtr, size_t count, axclrtMemcpyKind kind);
```

**Description**:

Perform synchronous memory copy operations for host-host, host-device, and device-device transfers.

- `dstPtr [IN]`: Destination memory pointer.
-- `srcPtr [IN]`: Source memory pointer.
-- `count [IN]`: Number of bytes to copy.
-- `kind [IN]`: Type of copy.
  - [`AXCL_MEMCPY_HOST_TO_HOST`]: Host-to-host copy.
  - [`AXCL_MEMCPY_HOST_TO_DEVICE`]: Host (virtual) to device copy.
  - [`AXCL_MEMCPY_DEVICE_TO_HOST`]: Device to host (virtual) copy.
  - [`AXCL_MEMCPY_DEVICE_TO_DEVICE`]: Device-to-device copy.
  - [`AXCL_MEMCPY_HOST_PHY_TO_DEVICE`]: Host physically-contiguous memory to device copy.
  - [`AXCL_MEMCPY_DEVICE_TO_HOST_PHY`]: Device to host physically-contiguous memory copy.


**Restrictions**：

- The source and destination memory for copying must meet the requirements of `kind`.

---
(axclrtmemcmp)=
#### axclrtMemcmp

```c
axclError axclrtMemcmp(const void *devPtr1, const void *devPtr2, size_t count);
```

**Description**:

Performs device-side memory comparison; synchronous call.

**Parameters**:

- `devPtr1 [IN]`: Device-side pointer 1.
- `devPtr2 [IN]`: Device-side pointer 2.
- `count [IN]`: Number of bytes to compare.

**Restrictions**：

- Only supports device-side memory comparisons; AXCL_SUCC(0) is returned only if the contents are equal.



### engine

(axclrtengineinit)=
#### axclrtEngineInit
```c
axclError axclrtEngineInit(axclrtEngineVNpuKind npuKind);
```
**Description**:

This function initializes the `Runtime Engine`. Users must call this before using any `Runtime Engine` features.

**Parameters**:

- `npuKind [IN]`: Specify the VNPU type to initialize.

**Restrictions**：

After using the `Runtime Engine`, users should call [`axclrtEngineFinalize`](#axclrtenginefinalize) to clean up and release `Runtime Engine` resources.

---

(axclrtenginegetvnpukind)=
#### axclrtEngineGetVNpuKind
```c
axclError axclrtEngineGetVNpuKind(axclrtEngineVNpuKind *npuKind);
```
**Description**:

This function returns the VNPU type used to initialize the `Runtime Engine`.

**Parameters**:
- `npuKind [OUT]`: VNPU type returned.

**Restrictions**：

Users must call [`axclrtEngineInit`](#axclrtengineinit) to initialize the `Runtime Engine` before calling this function.

---

(axclrtenginefinalize)=
#### axclrtEngineFinalize
```c
axclError axclrtEngineFinalize();
```
**Description**:

This function finalizes and cleans up the `Runtime Engine`. Users should call it after all Engine operations are finished.

**Restrictions**：

Users must call [`axclrtEngineInit`](#axclrtengineinit) before invoking this function.

---

(axclrtengineloadfromfile)=
#### axclrtEngineLoadFromFile
```c
axclError axclrtEngineLoadFromFile(const char *modelPath, uint64_t *modelId);
```
**Description**:

This function loads model data from a file and creates a model `ID`.

**Parameters**:
- `modelPath [IN]`: Path to the offline model file to load.
- `modelId [OUT]`: The generated model ID used for subsequent operations.

**Restrictions**:

Users must call [`axclrtEngineInit`](#axclrtengineinit) before invoking this function.

---

(axclrtengineloadfrommem)=
#### axclrtEngineLoadFromMem
```c
axclError axclrtEngineLoadFromMem(const void *model, uint64_t modelSize, uint64_t *modelId);
```
**Description**：

This function loads an offline model from memory and lets the system manage runtime memory for the model.

**Parameters**:
- `model [IN]`: Model data stored in memory.
- `modelSize [IN]`: Size of the model data.
- `modelId [OUT]`: The generated model ID used for subsequent operations.

**Restrictions**:

Model memory must be device memory; the user is responsible for allocation and freeing.

---

(axclrtengineunload)=
#### axclrtEngineUnload
```c
axclError axclrtEngineUnload(uint64_t modelId);
```
**Description**：

This function unloads the model identified by the specified model `ID`.

**Parameters**:
- `modelId [IN]`: Model ID to unload.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetmodelcompilerversion)=
#### axclrtEngineGetModelCompilerVersion
```c
const char* axclrtEngineGetModelCompilerVersion(uint64_t modelId);
```
**Description**：

This function returns the version string of the model compiler toolchain.

**Parameters**:
- `modelId [IN]`: Model ID.

**Restrictions**：

No special restrictions.

---
(axclrtenginesetaffinity)=
#### axclrtEngineSetAffinity
```c
axclError axclrtEngineSetAffinity(uint64_t modelId, axclrtEngineSet set);
```
**Description**：

This function sets the model's NPU affinity.

**Parameters**:
-- `modelId [IN]`: Model ID.
-- `set [IN]`: Affinity bitset to apply.

**Restrictions**：

The affinity set must not be zero; mask bits cannot exceed the available NPU affinity range.

---
(axclrtenginegetaffinity)=
#### axclrtEngineGetAffinity
```c
axclError axclrtEngineGetAffinity(uint64_t modelId, axclrtEngineSet *set);
```
**Description**：

This function obtains the model's NPU affinity.

**Parameters**:
-- `modelId [IN]`: Model ID.
-- `set [OUT]`: Returned affinity bitset.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetusage)=
#### axclrtEngineGetUsage
```c
axclError axclrtEngineGetUsage(const char *modelPath, int64_t *sysSize, int64_t *cmmSize);
```
**Description**：

This function returns the required system memory size and CMM memory size for executing the model from a file.

**Parameters**:
- `modelPath [IN]`: Path to the model file used to fetch memory usage information.
- `sysSize [OUT]`: System memory required for runtime execution.
- `cmmSize [OUT]`: CMM memory required for runtime execution.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetusagefrommem)=
#### axclrtEngineGetUsageFromMem
```c
axclError axclrtEngineGetUsageFromMem(const void *model, uint64_t modelSize, int64_t *sysSize, int64_t *cmmSize);
```
**Description**：

This function returns the required system memory size and CMM memory size for executing the model represented in memory.

**Parameters**:
- `model [IN]`: The model memory managed by the user.
- `modelSize [IN]`: Model data size.
- `sysSize [OUT]`: Size of the working system memory required to execute the model.
- `cmmSize [OUT]`: Size of the CMM memory required to execute the model.

**Restrictions**:

Model memory must be device memory; the user is responsible for allocation and freeing.

---
(axclrtenginegetusagefrommodelid)=
#### axclrtEngineGetUsageFromModelId
```c
axclError axclrtEngineGetUsageFromModelId(uint64_t modelId, int64_t *sysSize, int64_t *cmmSize);
```
**Description**：

This function returns the required system memory size and CMM memory size for executing the model identified by the model `ID`.

**Parameters**:
- `modelId [IN]`: Model ID.
- `sysSize [OUT]`: Size of the working system memory required to execute the model.
- `cmmSize [OUT]`: Size of the CMM memory required to execute the model.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetmodeltype)=
#### axclrtEngineGetModelType
```c
axclError axclrtEngineGetModelType(const char *modelPath, axclrtEngineModelKind *modelType);
```
**Description**：

This function retrieves the model type based on the model file.

**Parameters**：
- `modelPath [IN]`: Path to the model file used to get the model type.
- `modelType [OUT]`: Returned model type.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetmodeltypefrommem)=
#### axclrtEngineGetModelTypeFromMem
```c
axclError axclrtEngineGetModelTypeFromMem(const void *model, uint64_t modelSize, axclrtEngineModelKind *modelType);
```
**Description**：

This function retrieves the model type based on model data in memory.

**Parameters**：
- `model [IN]`: User-managed model memory.
- `modelSize [IN]`: Model data size.
- `modelType [OUT]`: Returned model type.

**Restrictions**:

Model memory must be device memory; the user is responsible for allocation and freeing.

---
(axclrtenginegetmodeltypefrommodelid)=
#### axclrtEngineGetModelTypeFromModelId
```c
axclError axclrtEngineGetModelTypeFromModelId(uint64_t modelId, axclrtEngineModelKind *modelType);
```
**Description**：

This function retrieves the model type for the specified model `ID`.

**Parameters**：
- `modelId [IN]`: Model ID.
- `modelType [OUT]`: Returned model type.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetioinfo)=
#### axclrtEngineGetIOInfo
```c
axclError axclrtEngineGetIOInfo(uint64_t modelId, axclrtEngineIOInfo *ioInfo);
```
**Description**：

This function obtains the IO information for the given model `ID`.

**Parameters**：
- `modelId [IN]`: Model ID.
- `ioInfo [OUT]`: Pointer to returned `axclrtEngineIOInfo`.

**Restrictions**：

The user should call `axclrtEngineDestroyIOInfo` to free the `axclrtEngineIOInfo` before the model ID is destroyed.

---
(axclrtenginedestroyioinfo)=
#### axclrtEngineDestroyIOInfo
```c
axclError axclrtEngineDestroyIOInfo(axclrtEngineIOInfo ioInfo);
```
**Description**：

This function destroys an `axclrtEngineIOInfo` object.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetshapegroupscount)=
#### axclrtEngineGetShapeGroupsCount
```c
axclError axclrtEngineGetShapeGroupsCount(axclrtEngineIOInfo ioInfo, int32_t *count);
```
**Description**：

This function returns the number of IO shape groups available.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `count [OUT]`: Number of shape groups.

**Restrictions**：

The Pulsar2 toolchain can specify multiple shapes during model conversion. A standard model typically has only one shape; therefore, for normally converted models, calling this function is unnecessary.

---
(axclrtenginegetnuminputs)=
#### axclrtEngineGetNumInputs
```c
uint32_t axclrtEngineGetNumInputs(axclrtEngineIOInfo ioInfo);
```
**Description**：

This function returns the number of inputs according to the `axclrtEngineIOInfo` data.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetnumoutputs)=
#### axclrtEngineGetNumOutputs
```c
uint32_t axclrtEngineGetNumOutputs(axclrtEngineIOInfo ioInfo);
```
**Description**：

This function returns the number of outputs according to the `axclrtEngineIOInfo` data.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetinputsizebyindex)=
#### axclrtEngineGetInputSizeByIndex
```c
uint64_t axclrtEngineGetInputSizeByIndex(axclrtEngineIOInfo ioInfo, uint32_t group, uint32_t index);
```
**Description**：

This function returns the size of a specified input from `axclrtEngineIOInfo` data.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `group [IN]`: Input shape group index.
- `index [IN]`: Index for the input size to retrieve (0-based).

**Restrictions**：

No special restrictions.

---
(axclrtenginegetoutputsizebyindex)=
#### axclrtEngineGetOutputSizeByIndex
```c
uint64_t axclrtEngineGetOutputSizeByIndex(axclrtEngineIOInfo ioInfo, uint32_t group, uint32_t index);
```
**Description**：

This function returns the size of a specified output from `axclrtEngineIOInfo` data.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `group [IN]`: Output shape group index.
- `index [IN]`: Index for the output size to retrieve (0-based).

**Restrictions**：

No special restrictions.

---
(axclrtenginegetinputnamebyindex)=
#### axclrtEngineGetInputNameByIndex
```c
const char *axclrtEngineGetInputNameByIndex(axclrtEngineIOInfo ioInfo, uint32_t index);
```
**Description**：

This function retrieves the name of the specified input.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `index [IN]`: Input IO index.

**Restrictions**：

Returned input tensor names share the same lifecycle as the `ioInfo` object.

---
(axclrtenginegetoutputnamebyindex)=
#### axclrtEngineGetOutputNameByIndex
```c
const char *axclrtEngineGetOutputNameByIndex(axclrtEngineIOInfo ioInfo, uint32_t index);
```
**Description**：

This function retrieves the name of the specified output.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `index [IN]`: Output IO index.

**Restrictions**：

Returned output tensor names share the same lifecycle as the `ioInfo` object.

---
(axclrtenginegetinputindexbyname)=
#### axclrtEngineGetInputIndexByName
```c
int32_t axclrtEngineGetInputIndexByName(axclrtEngineIOInfo ioInfo, const char *name);
```
**Description**：

This function obtains the input index based on the name of the input tensor.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `name [IN]`: Input tensor name.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetoutputindexbyname)=
#### axclrtEngineGetOutputIndexByName
```c
int32_t axclrtEngineGetOutputIndexByName(axclrtEngineIOInfo ioInfo, const char *name);
```
**Description**：

This function obtains the output index based on the name of the output tensor.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `name [IN]`: Output tensor name.

**Restrictions**：

No special restrictions.

---
(axclrtenginegetinputdims)=
#### axclrtEngineGetInputDims
```c
axclError axclrtEngineGetInputDims(axclrtEngineIOInfo ioInfo, uint32_t group, uint32_t index, axclrtEngineIODims *dims);
```
**Description**：

This function obtains the dimension information for the specified input.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `group [IN]`: Input shape group index.
- `index [IN]`: Input tensor index.
- `dims [OUT]`: Returned dimensional information.

**Restrictions**：

Storage for `axclrtEngineIODims` must be allocated by the user; free it before destroying the `axclrtEngineIOInfo` object.

---
(axclrtenginegetoutputdims)=
#### axclrtEngineGetOutputDims
```c
axclError axclrtEngineGetOutputDims(axclrtEngineIOInfo ioInfo, uint32_t group, uint32_t index, axclrtEngineIODims *dims);
```
**Description**：

This function obtains the dimension information for the specified output.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `group [IN]`: Output shape group index.
- `index [IN]`: Output tensor index.
- `dims [OUT]`: Returned dimensional information.

**Restrictions**：

Storage for `axclrtEngineIODims` must be allocated by the user; free it before destroying the `axclrtEngineIOInfo` object.

---
(axclrtenginecreateio)=
#### axclrtEngineCreateIO
```c
axclError axclrtEngineCreateIO(axclrtEngineIOInfo ioInfo, axclrtEngineIO *io);
```
**Description**：

This function creates data of type `axclrtEngineIO`.

**Parameters**：
- `ioInfo [IN]`: Pointer to `axclrtEngineIOInfo`.
- `io [OUT]`: Pointer to the created `axclrtEngineIO`.

**Restrictions**：

The user should call `axclrtEngineDestroyIO` to free the `axclrtEngineIO` before the model ID is destroyed.

---
(axclrtenginedestroyio)=
#### axclrtEngineDestroyIO
```c
axclError axclrtEngineDestroyIO(axclrtEngineIO io);
```
**Description**：

This function destroys data of type `axclrtEngineIO`.

**Parameters**：
- `io [IN]`: The `axclrtEngineIO` pointer to destroy.

**Restrictions**：

No special restrictions.

---
(axclrtenginesetinputbufferbyindex)=
#### axclrtEngineSetInputBufferByIndex
```c
axclError axclrtEngineSetInputBufferByIndex(axclrtEngineIO io, uint32_t index, const void *dataBuffer, uint64_t size);
```
**Description**：

This function sets an input data buffer by IO index.

**Parameters**：
- `io [IN]`: Address of the `axclrtEngineIO` data buffer.
- `index [IN]`: Input tensor index.
- `dataBuffer [IN]`: Address of the data buffer to set.
- `size [IN]`: Size of the data buffer.

**Restrictions**：

Data buffers must be device memory; the user is responsible for managing and freeing them.

---
(axclrtenginesetoutputbufferbyindex)=
#### axclrtEngineSetOutputBufferByIndex
```c
axclError axclrtEngineSetOutputBufferByIndex(axclrtEngineIO io, uint32_t index, const void *dataBuffer, uint64_t size);
```
**Description**：

This function sets an output data buffer by IO index.

**Parameters**：
- `io [IN]`: Address of the `axclrtEngineIO` data buffer.
- `index [IN]`: Output tensor index.
- `dataBuffer [IN]`: Address of the data buffer to set.
- `size [IN]`: Size of the data buffer.

**Restrictions**：

Data buffers must be device memory; the user is responsible for managing and freeing them.

---
(axclrtenginesetinputbufferbyname)=
#### axclrtEngineSetInputBufferByName
```c
axclError axclrtEngineSetInputBufferByName(axclrtEngineIO io, const char *name, const void *dataBuffer, uint64_t size);
```
**Description**：

This function sets an input data buffer by IO name.

**Parameters**：
- `io [IN]`: Address of the `axclrtEngineIO` data buffer.
- `name [IN]`: Input tensor name.
- `dataBuffer [IN]`: Address of the data buffer to set.
- `size [IN]`: Size of the data buffer.

**Restrictions**：

Data buffers must be device memory; the user is responsible for managing and freeing them.

---
(axclrtenginesetoutputbufferbyname)=
#### axclrtEngineSetOutputBufferByName
```c
axclError axclrtEngineSetOutputBufferByName(axclrtEngineIO io, const char *name, const void *dataBuffer, uint64_t size);
```
**Description**：

This function sets an output data buffer by IO name.

**Parameters**：
- `io [IN]`: Address of the `axclrtEngineIO` data buffer.
- `name [IN]`: Output tensor name.
- `dataBuffer [IN]`: Address of the data buffer to set.
- `size [IN]`: Size of the data buffer.

**Restrictions**：

Data buffers must be device memory; the user is responsible for managing and freeing them.

---
(axclrtenginegetinputbufferbyindex)=
#### axclrtEngineGetInputBufferByIndex
```c
axclError axclrtEngineGetInputBufferByIndex(axclrtEngineIO io, uint32_t index, void **dataBuffer, uint64_t *size);
```
**Description**：

This function retrieves the input data buffer by IO index.

**Parameters**：
- `io [IN]`: Address of the `axclrtEngineIO` data buffer.
- `index [IN]`: Input tensor index.
- `dataBuffer [OUT]`: Address of the data buffer.
- `size [IN]`: Size of the data buffer.

**Restrictions**：

Data buffers must be device memory; the user is responsible for managing and freeing them.

---
(axclrtenginegetoutputbufferbyindex)=
#### axclrtEngineGetOutputBufferByIndex
```c
axclError axclrtEngineGetOutputBufferByIndex(axclrtEngineIO io, uint32_t index, void **dataBuffer, uint64_t *size);
```
**Description**：

This function retrieves the output data buffer by IO index.

**Parameters**：
- `io [IN]`: Address of the `axclrtEngineIO` data buffer.
- `index [IN]`: Output tensor index.
- `dataBuffer [OUT]`: Address of the data buffer.
- `size [IN]`: Size of the data buffer.

**Restrictions**：

Data buffers must be device memory; the user is responsible for managing and freeing them.

---
(axclrtenginegetinputbufferbyname)=
#### axclrtEngineGetInputBufferByName
```c
axclError axclrtEngineGetInputBufferByName(axclrtEngineIO io, const char *name, void **dataBuffer, uint64_t *size);
```
**Description**：

This function retrieves the input data buffer by IO name.

**Parameters**：
- `io [IN]`: Address of the `axclrtEngineIO` data buffer.
- `name [IN]`: Input tensor name.
- `dataBuffer [OUT]`: Address of the data buffer.

**Restrictions**：

Data buffers must be device memory; the user is responsible for managing and freeing them.

---
(axclrtenginegetoutputbufferbyname)=
#### axclrtEngineGetOutputBufferByName
```c
axclError axclrtEngineGetOutputBufferByName(axclrtEngineIO io, const char *name, void **dataBuffer, uint64_t *size);
```
**Description**：

This function retrieves the output data buffer by IO name.

**Parameters**：
- `io [IN]`: Address of the `axclrtEngineIO` data buffer.
- `name [IN]`: Output tensor name.
- `dataBuffer [OUT]`: Address of the data buffer.

**Restrictions**：

Data buffers must be device memory; the user is responsible for managing and freeing them.

---
(axclrtenginesetdynamicbatchsize)=
#### axclrtEngineSetDynamicBatchSize
```c
axclError axclrtEngineSetDynamicBatchSize(axclrtEngineIO io, uint32_t batchSize);
```
**Description**：

This function sets the number of images processed in one pass in dynamic batching scenarios.

**Parameters**：
- `io [IN]`: The engine IO used for inference.
- `batchSize [IN]`: Number of images to process at once.

**Restrictions**：

No special restrictions.

---
(axclrtenginecreatecontext)=
#### axclrtEngineCreateContext
```c
axclError axclrtEngineCreateContext(uint64_t modelId, uint64_t *contextId);
```
**Description**：

This function creates a runtime context for the specified model `ID`.

**Parameters**：
- `modelId [IN]`: Model `ID`.
- `contextId [OUT]`: The created context ID.

**Restrictions**：

A model `ID` can create multiple runtime contexts, each running only within its own settings and memory space.

---
(axclrtengineexecute)=
#### axclrtEngineExecute
```c
axclError axclrtEngineExecute(uint64_t modelId, uint64_t contextId, uint32_t group, axclrtEngineIO io);
```
**Description**：

This function performs synchronous inference for the model until the inference result is returned.

**Parameters**：
- `modelId [IN]`: Model `ID`.
- `contextId [IN]`: Model inference context.
- `group [IN]`: Model shape group index.
- `io [IN]`: IO for model inference.

**Restrictions**：

No special restrictions.

---
(axclrtengineexecuteasync)=
#### axclrtEngineExecuteAsync
```c
axclError axclrtEngineExecuteAsync(uint64_t modelId, uint64_t contextId, uint32_t group, axclrtEngineIO io, axclrtStream stream);
```
**Description**：

This function performs asynchronous inference for the model and returns immediately; use the provided `stream` to synchronize.

**Parameters**：
- `modelId [IN]`: Model `ID`.
- `contextId [IN]`: Model inference context.
- `group [IN]`: Model shape group index.
- `io [IN]`: IO for model inference.
- `stream [IN]`: The stream used for synchronization.

**Restrictions**：

No special restrictions.



## native

- The AXCL NATIVE module supports SYS, VDEC, VENC, IVPS, DMADIM, ENGINE, and IVE modules.

- The AXCL NATIVE APIs have identical parameters to the AX SDK APIs; the main difference is the function name prefix changed from `AX` to `AXCL`, as shown in the example below:

  ```c
  AX_S32 AXCL_SYS_Init(AX_VOID);
  AX_S32 AXCL_SYS_Deinit(AX_VOID);

  /* CMM API */
  AX_S32 AXCL_SYS_MemAlloc(AX_U64 *phyaddr, AX_VOID **pviraddr, AX_U32 size, AX_U32 align, const AX_S8 *token);
  AX_S32 AXCL_SYS_MemAllocCached(AX_U64 *phyaddr, AX_VOID **pviraddr, AX_U32 size, AX_U32 align, const AX_S8 *token);
  AX_S32 AXCL_SYS_MemFree(AX_U64 phyaddr, AX_VOID *pviraddr);

  ...

  AX_S32 AXCL_VDEC_Init(const AX_VDEC_MOD_ATTR_T *pstModAttr);
  AX_S32 AXCL_VDEC_Deinit(AX_VOID);

  AX_S32 AXCL_VDEC_ExtractStreamHeaderInfo(const AX_VDEC_STREAM_T *pstStreamBuf, AX_PAYLOAD_TYPE_E enVideoType,
                                           AX_VDEC_BITSTREAM_INFO_T *pstBitStreamInfo);

  AX_S32 AXCL_VDEC_CreateGrp(AX_VDEC_GRP VdGrp, const AX_VDEC_GRP_ATTR_T *pstGrpAttr);
  AX_S32 AXCL_VDEC_CreateGrpEx(AX_VDEC_GRP *VdGrp, const AX_VDEC_GRP_ATTR_T *pstGrpAttr);
  AX_S32 AXCL_VDEC_DestroyGrp(AX_VDEC_GRP VdGrp);

  ...
  ```

- For more details refer to the AX SDK API documentation, such as "AX SYS API Documentation.docx" and "AX VDEC API Documentation.docx".

- Shared library names have changed from `libax_xxx.so` to `libaxcl_xxx.so`. Mapping table:

  | Module | AX SDK          | AXCL NATIVE SDK   |
  | ------ | --------------- | ----------------- |
  | SYS    | libax_sys.so    | libaxcl_sys.so    |
  | VDEC   | libax_vdec.so   | libaxcl_vdec.so   |
  | VENC   | libax_venc.so   | libaxcl_venc.so   |
  | IVPS   | libax_ivps.so   | libaxcl_ivps.so   |
  | DMADIM | libax_dmadim.so | libaxcl_dmadim.so |
  | ENGINE | libax_engine.so | libaxcl_engine.so |
  | IVE    | libax_ive.so    | libaxcl_ive.so    |

- Some AX SDK APIs are not available in AXCL NATIVE; see the list below for details:

  | Module | AXCL NATIVE API                 | Description                                        |
  | ------ | ------------------------------- | ------------------------------------------------- |
  | SYS    | AXCL_SYS_EnableTimestamp        |                                                   |
  |        | AXCL_SYS_Sleep                  |                                                   |
  |        | AXCL_SYS_WakeLock               |                                                   |
  |        | AXCL_SYS_WakeUnlock             |                                                   |
  |        | AXCL_SYS_RegisterEventCb        |                                                   |
  |        | AXCL_SYS_UnregisterEventCb      |                                                   |
  | VENC   | AXCL_VENC_GetFd                 |                                                   |
  |        | AXCL_VENC_JpegGetThumbnail      |                                                   |
  | IVPS   | AXCL_IVPS_GetChnFd              |                                                   |
  |        | AXCL_IVPS_CloseAllFd            |                                                   |
  | DMADIM | AXCL_DMADIM_Cfg                 | Does not support setting callback functions, i.e., AX_DMADIM_MSG_T.pfnCallBack |
  | IVE    | AXCL_IVE_NPU_CreateMatMulHandle |                                                   |
  |        | AX_IVE_NPU_DestroyMatMulHandle  |                                                   |
  |        | AX_IVE_NPU_MatMul               |                                                   |

## PPL

### architecture

**libaxcl_ppl.so** is a highly integrated module that implements typical pipeline (PPL). The architecture diagram is as follows:

```bash
|-----------------------------|
|        application          |
|-----------------------------|
|      libaxcl_ppl.so         |
|-----------------------------|
|      libaxcl_lite.so        |
|-----------------------------|
|         axcl sdk            |
|-----------------------------|
|         pcie driver         |
|-----------------------------|
```

:::{Note}
`libaxcl_lite.so` and `libaxcl_ppl.so` are open-source libraries which located in `axcl/sample/axclit` and `axcl/sample/ppl` directories.
:::



#### transcode

![](../res/transcode_ppl.png)

Refer to the source code of **axcl_transcode_sample**  which located in path: `axcl/sample/ppl/transode`.



### API

#### axcl_ppl_init

axcl runtime system and ppl initialization.

##### prototype

axclError axcl_ppl_init(const axcl_ppl_init_param* param);

##### parameter

| parameter | description | in/out |
| --------- | ----------- | ------ |
| param     |             | in     |

```c
typedef enum {
    AXCL_LITE_NONE = 0,
    AXCL_LITE_VDEC = (1 << 0),
    AXCL_LITE_VENC = (1 << 1),
    AXCL_LITE_IVPS = (1 << 2),
    AXCL_LITE_JDEC = (1 << 3),
    AXCL_LITE_JENC = (1 << 4),
    AXCL_LITE_DEFAULT = (AXCL_LITE_VDEC | AXCL_LITE_VENC | AXCL_LITE_IVPS),
    AXCL_LITE_MODULE_BUTT
} AXCLITE_MODULE_E;

typedef struct {
    const char *json; /* axcl.json path */
    AX_S32 device;
    AX_U32 modules;
    AX_U32 max_vdec_grp;
    AX_U32 max_venc_thd;
} axcl_ppl_init_param;
```

| parameter     | description                                  | in/out |
| ------------- | -------------------------------------------- | ------ |
| json          | json file path pass to **axclInit** API.     | in     |
| device        | device id                                    | in     |
| modules       | bitmask of AXCLITE_MODULE_E according to PPL | in     |
| max_vdec_grp  | total VDEC groups of all processes           | in     |
| max_venc_thrd | max VENC threads of each process             | in     |

> [!IMPORTANT]
>
> - AXCL_PPL_TRANSCODE:  modules = AXCL_LITE_DEFAULT
> - **max_vdec_grp** should be equal to or greater than the total number of VDEC groups across all processes. For example, if 16 processes are launched, with each process having one decoder, **max_vdec_grp** should be set to at least 16.
> - **max_venc_thrd** usually be configured as same as the total VENC channels of each process.

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_deinit

axcl runtime system and ppl deinitialization.

##### prototype

axclError axcl_ppl_deinit();

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_create

create ppl.

##### prototype

axclError axcl_ppl_create(axcl_ppl* ppl, const axcl_ppl_param* param);

##### parameter

| parameter | description                         | in/out |
| --------- | ----------------------------------- | ------ |
| ppl       | ppl handle created                  | out    |
| param     | parameters related to specified ppl | in     |

```c
typedef enum {
    AXCL_PPL_TRANSCODE = 0, /* VDEC -> IVPS ->VENC */
    AXCL_PPL_BUTT
} axcl_ppl_type;

typedef struct {
    axcl_ppl_type ppl;
    void *param;
} axcl_ppl_param;
```

**AXCL_PPL_TRANSCODE** parameters as shown as follows:

```C
typedef AX_VENC_STREAM_T axcl_ppl_encoded_stream;
typedef void (*axcl_ppl_encoded_stream_callback_func)(axcl_ppl ppl, const axcl_ppl_encoded_stream *stream, AX_U64 userdata);

typedef struct {
    axcl_ppl_transcode_vdec_attr vdec;
    axcl_ppl_transcode_venc_attr venc;
    axcl_ppl_encoded_stream_callback_func cb;
    AX_U64 userdata;
} axcl_ppl_transcode_param;
```

| parameter | description                                               | in/out |
| --------- | --------------------------------------------------------- | ------ |
| vdec      | decoder attribute                                         | in     |
| venc      | encoder attribute                                         | in     |
| cb        | callback function to receive encoded nalu frame data      | in     |
| userdata  | userdata bypass to  axcl_ppl_encoded_stream_callback_func | in     |

:::{Warning}
Avoid high-latency processing inside the *axcl_ppl_encoded_stream_callback_func* function
:::


```c
typedef struct {
    AX_PAYLOAD_TYPE_E payload;
    AX_U32 width;
    AX_U32 height;
    AX_VDEC_OUTPUT_ORDER_E output_order;
    AX_VDEC_DISPLAY_MODE_E display_mode;
} axcl_ppl_transcode_vdec_attr;
```

| parameter    | description                                                  | in/out |
| ------------ | ------------------------------------------------------------ | ------ |
| payload      | PT_H264 \| PT_H265                                           | in     |
| width        | max. width of input stream                                   | in     |
| height       | max. height of input stream                                  | in     |
| output_order | AX_VDEC_OUTPUT_ORDER_DISP  \| AX_VDEC_OUTPUT_ORDER_DEC       | in     |
| display_mode | AX_VDEC_DISPLAY_MODE_PREVIEW \| AX_VDEC_DISPLAY_MODE_PLAYBACK | in     |

:::{Important}
- **output_order**:
  - If decode sequence is same as display sequence such as IP stream, recommend to AX_VDEC_OUTPUT_ORDER_DEC to save memory.
  - If decode sequence is different to display sequence such as IPB stream, set AX_VDEC_OUTPUT_ORDER_DISP.
- **display_mode**
  - AX_VDEC_DISPLAY_MODE_PREVIEW:  preview mode which frame dropping is allowed typically for RTSP stream... etc.
  - AX_VDEC_DISPLAY_MODE_PLAYBACK: playback mode which frame dropping is not forbidden typically for local stream file.
  :::

```c
typedef struct {
    AX_PAYLOAD_TYPE_E payload;
    AX_U32 width;
    AX_U32 height;
    AX_VENC_PROFILE_E profile;
    AX_VENC_LEVEL_E level;
    AX_VENC_TIER_E tile;
    AX_VENC_RC_ATTR_T rc;
    AX_VENC_GOP_ATTR_T gop;
} axcl_ppl_transcode_venc_attr;
```

| parameter | description                         | in/out |
| --------- | ----------------------------------- | ------ |
| payload   | PT_H264 \| PT_H265                  | in     |
| width     | output width of encoded nalu frame  | in     |
| height    | output height of encoded nalu frame | in     |
| profile   | h264 or h265 profile                | in     |
| level     | h264 or h265 level                  | in     |
| tile      | tile                                | in     |
| rc        | rate control settings               | in     |
| gop       | gop settings                        | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_destroy

destroy ppl.

##### prototype

axclError axcl_ppl_destroy(axcl_ppl ppl)

##### parameter

| parameter | description        | in/out |
| --------- | ------------------ | ------ |
| ppl       | ppl handle created | in     |

#### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_start

start ppl.

##### prototype

axclError axcl_ppl_start(axcl_ppl ppl);

##### parameter

| parameter | description        | in/out |
| --------- | ------------------ | ------ |
| ppl       | ppl handle created | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_stop

stop ppl.

##### prototype

axclError axcl_ppl_stop(axcl_ppl ppl);

##### parameter

| parameter | description        | in/out |
| --------- | ------------------ | ------ |
| ppl       | ppl handle created | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_send_stream

send nalu frame to ppl.

##### prototype

axclError axcl_ppl_send_stream(axcl_ppl ppl, const axcl_ppl_input_stream* stream, AX_S32 timeout);

##### parameter

| parameter | description             | in/out |
| --------- | ----------------------- | ------ |
| ppl       | ppl handle created      | in     |
| stream    | nalu frame              | in     |
| timeout   | timeout in milliseconds | in     |

```c
typedef struct {
    AX_U8 *nalu;
    AX_U32 nalu_len;
    AX_U64 pts;
    AX_U64 userdata;
} axcl_ppl_input_stream;
```

| parameter | description                                                  | in/out |
| --------- | ------------------------------------------------------------ | ------ |
| nalu      | pointer to nalu frame data                                   | in     |
| nalu_len  | bytes of nalu frame data                                     | in     |
| pts       | timestamp of nalu frame                                      | in     |
| userdata  | userdata bypass to ***axcl_ppl_encoded_stream.stPack.u64UserData*** | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_get_attr

get attribute of ppl.

##### prototype

axclError axcl_ppl_get_attr(axcl_ppl ppl, const char* name, void* attr);

##### parameter

| parameter | description        | in/out |
| --------- | ------------------ | ------ |
| ppl       | ppl handle created | in     |
| name      | attribute name     | in     |
| attr      | attribute value    | out    |

```c
/**
 *            name                                     attr type        default
 *  axcl.ppl.id                             [R  ]       int32_t                            increment +1 for each axcl_ppl_create
 *
 *  axcl.ppl.transcode.vdec.grp             [R  ]       int32_t                            allocated by ax_vdec.ko
 *  axcl.ppl.transcode.ivps.grp             [R  ]       int32_t                            allocated by ax_ivps.ko
 *  axcl.ppl.transcode.venc.chn             [R  ]       int32_t                            allocated by ax_venc.ko
 *
 *  the following attributes take effect BEFORE the axcl_ppl_create function is called:
 *  axcl.ppl.transcode.vdec.blk.cnt         [R/W]       uint32_t          8                depend on stream DPB size and decode mode
 *  axcl.ppl.transcode.vdec.out.depth       [R/W]       uint32_t          4                out fifo depth
 *  axcl.ppl.transcode.ivps.in.depth        [R/W]       uint32_t          4                in fifo depth
 *  axcl.ppl.transcode.ivps.out.depth       [R  ]       uint32_t          0                out fifo depth
 *  axcl.ppl.transcode.ivps.blk.cnt         [R/W]       uint32_t          4
 *  axcl.ppl.transcode.ivps.engine          [R/W]       uint32_t   AX_IVPS_ENGINE_VPP      AX_IVPS_ENGINE_VPP|AX_IVPS_ENGINE_VGP|AX_IVPS_ENGINE_TDP
 *  axcl.ppl.transcode.venc.in.depth        [R/W]       uint32_t          4                in fifo depth
 *  axcl.ppl.transcode.venc.out.depth       [R/W]       uint32_t          4                out fifo depth
 */
```

:::{Important}
**axcl.ppl.transcode.vdec.blk.cnt**:  blk count is related to the DBP size of input stream, recommend to set dbp + 1.
:::

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_set_attr

set attribute of ppl.

##### prototype

axclError axcl_ppl_set_attr(axcl_ppl ppl, const char* name, const void* attr);

##### parameter

| parameter | description                                      | in/out |
| --------- | ------------------------------------------------ | ------ |
| ppl       | ppl handle created                               | in     |
| name      | attribute name, refer to ***axcl_ppl_get_attr*** | in     |
| attr      | attribute value                                  | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



## Error Codes

### Definition

```c
typedef int32_t axclError;

typedef enum {
    AXCL_SUCC                   = 0x00,
    AXCL_FAIL                   = 0x01,
    AXCL_ERR_UNKNOWN            = AXCL_FAIL,
    AXCL_ERR_NULL_POINTER       = 0x02,
    AXCL_ERR_ILLEGAL_PARAM      = 0x03,
    AXCL_ERR_UNSUPPORT          = 0x04,
    AXCL_ERR_TIMEOUT            = 0x05,
    AXCL_ERR_BUSY               = 0x06,
    AXCL_ERR_NO_MEMORY          = 0x07,
    AXCL_ERR_ENCODE             = 0x08,
    AXCL_ERR_DECODE             = 0x09,
    AXCL_ERR_UNEXPECT_RESPONSE  = 0x0A,
    AXCL_ERR_OPEN               = 0x0B,
    AXCL_ERR_EXECUTE_FAIL       = 0x0C,

    AXCL_ERR_BUTT               = 0x7F
} AXCL_ERROR_E;

#define AX_ID_AXCL           (0x30)

/* module */
#define AXCL_RUNTIME         (0x00)
#define AXCL_NATIVE          (0x01)
#define AXCL_LITE            (0x02)

/* runtime sub module */
#define AXCL_RUNTIME_DEVICE  (0x01)
#define AXCL_RUNTIME_CONTEXT (0x02)
#define AXCL_RUNTIME_STREAM  (0x03)
#define AXCL_RUNTIME_TASK    (0x04)
#define AXCL_RUNTIME_MEMORY  (0x05)
#define AXCL_RUNTIME_CONFIG  (0x06)
#define AXCL_RUNTIME_ENGINE  (0x07)
#define AXCL_RUNTIME_SYSTEM  (0x08)

/**
 * |---------------------------------------------------------|
 * | |   MODULE    |  AX_ID_AXCL | SUB_MODULE  |    ERR_ID   |
 * |1|--- 7bits ---|--- 8bits ---|--- 8bits ---|--- 8bits ---|
 **/
#define AXCL_DEF_ERR(module, sub, errid) \
    ((axclError)((0x80000000L) | (((module) & 0x7F) << 24) | ((AX_ID_AXCL) << 16 ) | ((sub) << 8) | (errid)))

#define AXCL_DEF_RUNTIME_ERR(sub, errid)    AXCL_DEF_ERR(AXCL_RUNTIME, (sub), (errid))
#define AXCL_DEF_NATIVE_ERR(sub,  errid)    AXCL_DEF_ERR(AXCL_NATIVE,  (sub), (errid))
#define AXCL_DEF_LITE_ERR(sub,    errid)    AXCL_DEF_ERR(AXCL_LITE,    (sub), (errid))
```

:::{Note}

Error codes are split into AXCL runtime library errors and AX NATIVE SDK device errors, distinguished by the third byte in `axclError`. If the third byte equals `AX_ID_AXCL` (0x30), the code denotes an AXCL runtime error; otherwise, it denotes a device-side AX NATIVE SDK error forwarded to the host.
- AXCL runtime errors: see the `axcl_rt_xxx.h` header files for details.
- Device NATIVE SDK errors: these are passed through to the host; refer to the "AX Software Error Codes" document for details.
:::

### Parsing

Visit the [AXCL Error Code Lookup](https://gakki2019.github.io/axcl/) webpage to decode an error code online.
