# STM32F401RE firmware memory and communication investigation

Date: 2026-10-11. Scope: analysis only, following the user's correction not to
change the firmware. No optimization, timing, QoS, controller, interface,
peripheral or build-configuration edits were made. Only this requested report
was added. No board was connected, flashed or driven; no target program ran.

## Evidence, scope and reproducibility

- Active repository: `Embedded_UniPD-DriveOps`, branch `main`, commit
  `94c3dfb5a2f9e9e4107002c501ba8c71e504b358`.
- Actual existing artifacts: `build/microros.elf` and `build/microros.map`;
  the Makefile target is `microros`, not `firmware`.
- ELF SHA-256: `3b88f3a215b6e9431e7783f288c73b1dddadc867e5a887f5c3146f862ec6960d`.
- MAP SHA-256: `363d0fdee883924be4e77998d34eb4f8fa86c069f5fe08c5e83a628368c152d4`.
- Used `arm-none-eabi-size`, `nm`, `readelf`, `objdump`, and offline
  `gdb -nx -batch` with `ptype /o` and `sizeof`. No GDB target connection or
  function invocation was used. Source, annotated assembly listings, linker
  cross-references and all available Git branch histories were inspected.
- The ELF identifies application compilation with GCC 13.3.1, ARM Cortex-M4,
  hard-float single-precision ABI, `-Og`, from `/home/hisham/pieces/microros/`.
  Local GCC is 13.2.1. Exact layout below describes the actual existing ELF,
  not a host x86 `sizeof` guess or a claim that it was freshly rebuilt.
- The current micro-ROS directory is empty. Git records it as gitlink
  `a5b2127495ae0ab53d7a1360beaf17822309a3cc`, without a checked-in `.gitmodules`.
  `make -n all` fails on missing
  `micro_ros_stm32cubemx_utils/extra_sources/custom_memory_manager.c`.
  No real build was attempted after the no-change instruction.
- Other Git refs contain dependencies: `a2928be` converts the utility directory
  into tracked files and `c3becf6` adds a prebuilt library. Files read using
  `git show origin/encoder-tof:<path>` include the allocator, DMA transport,
  generated `rmw_microxrcedds_c/config.h`, `uxr/client/config.h` and
  `library_generation/colcon.meta`. Their pool capacities agree with the ELF.
  This corroboration is not proof that every byte of that branch's library is
  the exact original build input. No files or branches were restored/switched.
- Existing untracked `.vscode/` was left untouched. No application test suite
  was found in the active checkout. No numerical equivalence test or hardware
  performance measurement has been claimed.

## Original and final memory breakdown: unchanged

| Measurement | Original | Final (no optimization applied) |
|---|---:|---:|
| Berkeley `text` | 102,740 B | 102,740 B |
| `.data` | 1,928 B | 1,928 B |
| Berkeley `bss` | 96,024 B | 96,024 B |
| Flash contents, `text + data` | 104,668 B | 104,668 B |
| Allocated/reserved SRAM, `data + bss` | 97,952 B | 97,952 B |
| SRAM available | 98,304 B | 98,304 B |
| Unreserved linker margin | 352 B (0.358%) | 352 B |
| Verified recovered SRAM | — | **0 B** |

The section-level result is more informative than Berkeley `bss`:

| SRAM component | Bytes |
|---|---:|
| Initialized `.data` | 1,928 |
| Actual `.bss` variables, including both heaps/task stacks | 94,484 |
| Alignment before linker heap reserve | 4 |
| `_Min_Heap_Size` newlib minimum reservation | 512 |
| `_Min_Stack_Size` main/interrupt-stack minimum reservation | 1,024 |
| Total occupied/reserved | **97,952** |

`size` includes the 1,540-byte `._user_heap_stack` section in its `bss` total.
Do not add that reservation to 97,952 again. Conversely, this is **not measured
live RAM use**: the 14,000 B inside the two allocator arrays and unused portions
of task stacks are already counted and may have runtime headroom internally.
The 352 B is additional unreserved address space, not the remaining FreeRTOS
heap or task-stack space.

`STM32F401XX_FLASH.ld:53–66` places SRAM at `0x20000000`, length 96 KiB;
flash at `0x08000000`, length 512 KiB. `_end = 0x200178a0`,
`_sstack = 0x20017c00`, `_estack = 0x20018000`. The existing `_sbrk`
implementation (`Core/Src/sysmem.c:54`) therefore has at most **864 B** between
`_end` and the MSP limit: 512 B minimum plus 352 B spare. That is separate from
both 7,000-byte allocator arrays. The flash load span is **104,684 B**, from `0x08000000` through
`0x080198eb` inclusive: 104,668 B section contents plus 16 B alignment gaps.
The ELF file also contains debug sections which are not programmed SRAM/flash.

## Detailed Memory Analysis of SpeedController and PositionController

### Exact types, definitions and call sites

Both names are individual global **objects of the same non-polymorphic class
`PID`**, defined in `Core/Inc/pid.h:190–280`. There is no inheritance, virtual
function table, nested owning object or hidden allocation. Fixed arrays are
embedded directly into each instance. Methods occupy flash once, not per object.

`Core/Src/freertos.cpp:145–146` defines:

```cpp
PID SpeedController = PID(&carSpeedfiltered, &motorPWM, &speedRef,
                         0.020001, 0.0250001, 0,
                         _PID_P_ON_E, _PID_CD_DIRECT);
PID PositionController = PID(&localDistance, &speedTargetPos, &localDistanceRef,
                            6, 2, 0, _PID_P_ON_E, _PID_CD_DIRECT);
```

The pointer parameters reference six external `double` variables; those variables
are not contained in the 24,960-byte objects. The three pointers consume 12 bytes
per object. Do not count the pointees as embedded members.

Constructors: `Core/Src/pid.cpp:31` (default), `:39` (full parameter constructor
used here), and `:69` (delegating overload). `Init()` at `:72` and parameterized
`Init()` at `:101/:131` reset history/accumulators. `Compute()` starts at `:137`;
`UseIFilter()` at `:437`; inline `EnableProportionalFilter()` at
`Core/Inc/pid.h:278` only changes a flag.

All active object uses:

| Location in `Core/Src/freertos.cpp` | SpeedController | PositionController |
|---|---|---|
| 145–146 | Global construction | Global construction |
| 386–397 | Automatic; limits -0.009 to 0.0119; sample time `samplingPID`; P filter **true**; I filter **true** | Automatic; limits -0.2 to 0.2; same sample time; P filter **false**; I filter remains **false** after Init |
| 715 | — | `Compute()` in active position maneuver |
| 724 | `Compute()` when speed control is enabled | — |
| 649–658 | Commented tuning/debug calls only | Commented tuning/debug calls only |

No active later call enables either position-controller filter. Both controllers
can execute in the same TIM11 interrupt during position control. They cannot
share general mutable control state merely because they execute sequentially.

ELF verification:

```text
(gdb) whatis SpeedController     -> PID
(gdb) whatis PositionController  -> PID
(gdb) p sizeof(SpeedController)  -> 24960
(gdb) p sizeof(PositionController) -> 24960
SpeedController address:    0x2000c8b0
PositionController address: 0x20006730
```

`readelf -A build/{pid,freertos}.o` reports small enums and 8-byte alignment.
DWARF confirms the three PID enum types are 1 byte each, `double` is 8 bytes,
a pointer is 4 bytes, and `bool` is 1 byte. Assuming desktop enum/pointer sizes
would produce the wrong layout.

### SpeedController: complete compiled member layout

| Offset | Member | Data type | Elements | Bytes | Purpose |
|---:|---|---|---:|---:|---|
| 0 | `_pOnE` | `PIDPON_TypeDef` | 1 | 1 | Select proportional-on-error branch |
| 1 | `_inAuto` | `PIDMode_TypeDef` | 1 | 1 | Manual/automatic mode |
| 2 | `_pOn` | `PIDPON_TypeDef` | 1 | 1 | Requested proportional mode for retuning |
| 3 | `_controllerDirection` | `PIDCD_TypeDef` | 1 | 1 | Direct/reverse gain sign |
| 4 | `_lastTime` | `uint32_t` | 1 | 4 | Last accepted computation tick |
| 8 | `_sampleTime` | `uint32_t` | 1 | 4 | Computation interval in milliseconds |
| 16 | `_dispKp` | `double` | 1 | 8 | User-facing proportional tuning |
| 24 | `_dispKi` | `double` | 1 | 8 | User-facing integral tuning |
| 32 | `_dispKd` | `double` | 1 | 8 | User-facing derivative tuning |
| 40 | `_kp` | `double` | 1 | 8 | Internal proportional coefficient |
| 48 | `_ki` | `double` | 1 | 8 | Integral coefficient scaled by sample time |
| 56 | `_kd` | `double` | 1 | 8 | Derivative coefficient scaled by sample time |
| 64 | `_myInput` | `double *` | 1 | 4 | Pointer to external measured input; pointee is not embedded |
| 68 | `_myOutput` | `double *` | 1 | 4 | Pointer to external output; pointee is not embedded |
| 72 | `_mySetpoint` | `double *` | 1 | 4 | Pointer to external setpoint; pointee is not embedded |
| 80 | `_outputSum` | `double` | 1 | 8 | Integral/proportional-on-measurement accumulator |
| 88 | `_lastInput` | `double` | 1 | 8 | Previous input for derivative/change calculation |
| 96 | `_outMin` | `double` | 1 | 8 | Lower output clamp |
| 104 | `_outMax` | `double` | 1 | 8 | Upper output clamp |
| 112 | `_pBuffer` | `double` | 3000 | 24,000 | Rolling proportional error history |
| 24112 | `_pBufferIndex` | `uint16_t` | 1 | 2 | Proportional history ring index |
| 24120 | `_pSum` | `double` | 1 | 8 | Running sum of proportional history |
| 24128 | `_usePFilter` | `bool` | 1 | 1 | Proportional filter enable flag |
| 24136 | `_iBuffer` | `double` | 100 | 800 | Rolling integral-input error history |
| 24936 | `_iBufferIndex` | `uint8_t` | 1 | 1 | Integral history ring index |
| 24944 | `_iSum` | `double` | 1 | 8 | Running sum of integral error history |
| 24952 | `_useIFilter` | `bool` | 1 | 1 | Integral-input filter enable flag |
| — | **Member storage** | | | **24,925** | |
| — | **Padding/alignment** | | | **35** | Detailed below |
| — | **Object total** | `PID` | 1 | **24,960** | `sizeof` verified from ELF DWARF |

### PositionController: complete compiled member layout

| Offset | Member | Data type | Elements | Bytes | Purpose |
|---:|---|---|---:|---:|---|
| 0 | `_pOnE` | `PIDPON_TypeDef` | 1 | 1 | Select proportional-on-error branch |
| 1 | `_inAuto` | `PIDMode_TypeDef` | 1 | 1 | Manual/automatic mode |
| 2 | `_pOn` | `PIDPON_TypeDef` | 1 | 1 | Requested proportional mode for retuning |
| 3 | `_controllerDirection` | `PIDCD_TypeDef` | 1 | 1 | Direct/reverse gain sign |
| 4 | `_lastTime` | `uint32_t` | 1 | 4 | Last accepted computation tick |
| 8 | `_sampleTime` | `uint32_t` | 1 | 4 | Computation interval in milliseconds |
| 16 | `_dispKp` | `double` | 1 | 8 | User-facing proportional tuning |
| 24 | `_dispKi` | `double` | 1 | 8 | User-facing integral tuning |
| 32 | `_dispKd` | `double` | 1 | 8 | User-facing derivative tuning |
| 40 | `_kp` | `double` | 1 | 8 | Internal proportional coefficient |
| 48 | `_ki` | `double` | 1 | 8 | Integral coefficient scaled by sample time |
| 56 | `_kd` | `double` | 1 | 8 | Derivative coefficient scaled by sample time |
| 64 | `_myInput` | `double *` | 1 | 4 | Pointer to external measured input; pointee is not embedded |
| 68 | `_myOutput` | `double *` | 1 | 4 | Pointer to external output; pointee is not embedded |
| 72 | `_mySetpoint` | `double *` | 1 | 4 | Pointer to external setpoint; pointee is not embedded |
| 80 | `_outputSum` | `double` | 1 | 8 | Integral/proportional-on-measurement accumulator |
| 88 | `_lastInput` | `double` | 1 | 8 | Previous input for derivative/change calculation |
| 96 | `_outMin` | `double` | 1 | 8 | Lower output clamp |
| 104 | `_outMax` | `double` | 1 | 8 | Upper output clamp |
| 112 | `_pBuffer` | `double` | 3000 | 24,000 | Rolling proportional error history; initialized/reset but not read by the enabled numerical path |
| 24112 | `_pBufferIndex` | `uint16_t` | 1 | 2 | Proportional history ring index |
| 24120 | `_pSum` | `double` | 1 | 8 | Running sum of proportional history |
| 24128 | `_usePFilter` | `bool` | 1 | 1 | Proportional filter enable flag |
| 24136 | `_iBuffer` | `double` | 100 | 800 | Rolling integral-input error history; initialized/reset but not read by the enabled numerical path |
| 24936 | `_iBufferIndex` | `uint8_t` | 1 | 1 | Integral history ring index |
| 24944 | `_iSum` | `double` | 1 | 8 | Running sum of integral error history |
| 24952 | `_useIFilter` | `bool` | 1 | 1 | Integral-input filter enable flag |
| — | **Member storage** | | | **24,925** | |
| — | **Padding/alignment** | | | **35** | Detailed below |
| — | **Object total** | `PID` | 1 | **24,960** | `sizeof` verified from ELF DWARF |

### Padding, constants and totals

Both objects have the following identical padding ranges (inclusive offsets):

| Offsets | Bytes | Reason |
|---|---:|---|
| 12–15 | 4 | Align `_dispKp` to 8 bytes |
| 76–79 | 4 | Align `_outputSum` to 8 bytes |
| 24114–24119 | 6 | Align `_pSum` after 16-bit index |
| 24129–24135 | 7 | Align `_iBuffer` after bool |
| 24937–24943 | 7 | Align `_iSum` after 8-bit index |
| 24953–24959 | 7 | Tail padding to 8-byte object alignment |
| **Total** | **35** | |

`static const uint16_t _P_FILTER_WINDOW = 3000` is a class constant and occupies
**zero bytes within each instance**. `_I_FILTER_WINDOW` is a preprocessor macro
(100), also zero instance bytes. `sizeof(_P_FILTER_WINDOW) == 2` describes its
type; it does not mean two bytes are embedded in each object.

| Category | SpeedController | PositionController | Combined |
|---|---:|---:|---:|
| `_pBuffer[3000]` | 24,000 | 24,000 | 48,000 |
| `_iBuffer[100]` | 800 | 800 | 1,600 |
| All other actual member storage | 125 | 125 | 250 |
| Padding | 35 | 35 | 70 |
| **Total** | **24,960** | **24,960** | **49,920** |

Arrays account for **24,800/24,960 = 99.36%** of each object. Equal object sizes
come from using the same class, not equal runtime needs. A false enable flag does
not release an embedded array. Position's arrays are written by constructors
and reset functions, but are not read by its configured numerical computation.

### Why the arrays exist and what must be preserved

The P filter is an exact rectangular moving average of **3,000 error samples**:
subtract the outgoing ring value from `_pSum`, replace it with the current
error, add that error, advance modulo 3,000, then divide by 3,000 and multiply
by `_kp`. The I filter performs an analogous **100-sample** error average before
applying `_ki` to `_outputSum`. Each ring contains `double` values, not timestamps,
lookup-table entries, debug logging or pointers. Running sums do not eliminate
the need to remember the outgoing sample.

At the configured nominal 1 ms accepted sample interval, these windows span
3 seconds and 100 ms. Their approximate steady-state moving-average group delays
are 1.4995 seconds and 49.5 ms respectively. Startup divides by the full window
while the remaining entries are zero. Changing window lengths, floating-point
precision, initialization, accumulation order or replacing the moving average
with an exponential filter changes behavior. Preserve them for speed control.
Actual sample acceptance also depends on `HAL_GetTick()` and ISR scheduling;
these times are configuration-derived, not measured.

For position control, both branches select the unfiltered path. Its histories
and filter-only indices/sums can be omitted in a purpose-specific storage/type
**if** that design permanently expresses the current no-filter configuration
and keeps the existing unfiltered operations and reset semantics. Do not simply
remove arrays from the shared class, because speed control needs them.

Required mutable gains, accumulators, previous input, timing state and external
measurement/output variables belong in RAM. Gains can be retuned by the public
API; treating all constructor arguments as immutable changes that contract.
Speed histories cannot reside in flash. Position histories should be eliminated
from its unused capability rather than relabeled `const`: current initialization
writes to them. Compile-time window constants already consume no per-object RAM.
The separate 13-element FIR coefficient array is immutable in this application:
`freertos.cpp:127` defines 52 B in `.data`, its only consumer is
`arm_fir_init_f32()` at `:400`, whose coefficient parameter is `const float32_t *`
(`Middlewares/ST/ARM/DSP/Inc/arm_math.h:1920`). Making this array const is a
small future flash-placement candidate, subject to layout/build/timing checks.

## FreeRTOS allocators, stacks and scheduler

### Two independent heap owners

| Heap | Address | Raw reservation | Owner |
|---|---|---:|---|
| `heap_4.c::ucHeap` | `0x20001640` | 7,000 B | `pvPortMalloc` / `vPortFree`, FreeRTOS dynamic object/task API |
| `custom_memory_manager.c::ucHeap` | `0x200033c4` | 7,000 B | `pvPortMallocMicroROS`, `vPortFreeMicroROS`, custom realloc/calloc |

The micro-ROS allocator callbacks installed in `StartDefaultTask:262–276` call
`microros_allocate/deallocate/reallocate/zero_allocate`, which dispatch to the
**MicroROS-suffixed allocator**, not the FreeRTOS heap. ELF disassembly and the
annotated allocator listings confirm these call targets. The custom allocator
is based on heap_4 and independently defines its own free list, counters and
`ucHeap[configTOTAL_HEAP_SIZE]`. Changing the one shared 7000 setting affects
both array capacities. They are separate ownership domains, not duplicate symbol
names referring to the same memory.

The present normal startup uses static default/idle/timer tasks and a static
software-timer command queue. The compiled calls to `pvPortMalloc` are in the
retained `xTaskCreate` path, reachable from the dynamic alternative in
`osThreadNew`. The sole application `osThreadNew` call supplies both stack and
control-block storage, selecting its static path. Therefore an allocation from
the RTOS heap is **not established for the normal application path**, although
the allocator and dynamic APIs are linked. Do not describe all 7000 B as measured
runtime consumption, or remove the heap merely because no current task needs it.

A future all-static FreeRTOS build could reclaim that reservation after a full
call-graph/configuration test. However, the custom micro-ROS allocator explicitly
rejects `configSUPPORT_DYNAMIC_ALLOCATION == 0`; its independent allocator guard
would have to be separated correctly. Its 7000-byte buffer remains necessary
for ROS allocations even if FreeRTOS task allocation becomes static-only.

Consolidating both allocators is another option, but is not a safe one-line
redirect: realloc needs the old payload size, calloc needs overflow/failure
handling, and every pointer must be freed by the allocator that owns it. Keeping
a unified 14,000-byte capacity saves essentially no buffer RAM. Reducing to a
single 7,000-byte heap needs measured joint peak usage, fragmentation and
realloc-copy overlap, plus failure-path and concurrency tests. Never feed a
FreeRTOS pointer to the existing custom `getBlockSize`/free routines.

Each block carries an 8-byte header plus alignment. For these linked addresses,
heap initialization would provide 6,992 B of initial free block space in heap_4
and 6,984 B in the custom heap, before per-allocation overhead. These are computed
capacity limits, **not observed runtime free-byte readings**.

`usedMemory` is not a trustworthy live-byte metric: allocation adds requested
bytes even before success is known, while deallocation subtracts `getBlockSize`,
which the ELF shows returns header-inclusive/aligned block size. Custom realloc
also creates a replacement allocation before releasing the old one. Restore and
review this allocator implementation and its failure paths before changing it.

### Tasks, queues, priorities and interrupts

| Item | Reserved storage / setting | Assessment |
|---|---|---|
| defaultTask | 1000 `uint32_t` words = 4,000 B; static TCB 92 B; priority 24 | Runs ROS setup/executor and sequential publishes; stack usage unmeasured |
| Idle task | 128 words = 512 B; static TCB 92 B; priority 0 | Static CMSIS callback |
| Timer task | 256 words = 1,024 B; static TCB 92 B; priority 2 | Started by enabled FreeRTOS software timer subsystem |
| Timer queue | 10 × 16-byte daemon messages = 160 B + 80-byte static queue object | Static when `configSUPPORT_STATIC_ALLOCATION == 1`; not taken from heap_4 |
| Ready lists | 56 × 20 B = 1,120 B | CMSIS wrapper explicitly requires `configMAX_PRIORITIES == 56` |
| RTOS tick | 1000 Hz; preemptive | Retained |
| Mutex/semaphore APIs | Mutexes, recursive mutexes, counting semaphores enabled | No application-created mutex/queue/semaphore found in MX_FREERTOS_Init |
| MSP minimum | 1,024 B | Separate from task stacks; nested interrupts/FP frames need hardware validation |

Do not reduce priority count to 8 or 16: the bundled
`freertos_os2.h:293–299` rejects anything other than 56, and normal priority is
24. Disabling unused software timers could reclaim at least the 1,356 B of
stack/TCB/queue above plus related globals, but requires coordinated CMSIS timer
flags and the retained timer API/pend-function paths to be audited. No such
change is applied.

`INCLUDE_uxTaskGetStackHighWaterMark` is enabled in configuration;
`osThreadGetStackSpace()` multiplies its result by `sizeof(StackType_t)` to
return bytes. Unused diagnostic functions can be garbage-collected from this
ELF. No runtime watermark, heap minimum or failed-allocation count is available
from the offline binary. Stack-overflow and malloc-failed hooks default to zero
in this configuration. Reserving 4,000 B does not prove it is oversized.

DMA1 streams 5/6 and USART2 interrupts have priority 5; TIM11 control has priority
5; TIM1/TIM10 shared IRQ is configured at priority 15 by timer setup; PendSV is
15. The syscall threshold is 5. The source also uses TIM1 for HAL tick on the
shared TIM1/TIM10 interrupt: blocking I2C timeouts inside TIM10's callback deserve
particular scrutiny because their HAL tick cannot preempt the same active IRQ.
`VL6180X_ReadReg` uses synchronous transmit/receive with 30 ms timeouts;
`VL6180X_WriteReg` uses 10 ms. These delays can stall the ROS task independently
of available RAM. Do not move calls or alter interrupt priorities without a
separate scheduling/behavior review.

## micro-ROS memory configuration

The ELF's DWARF gives these actual fixed pools (not estimates from defaults):

| Pool | Elements × element size | Total bytes | Application demand / caution |
|---|---:|---:|---|
| `custom_sessions` | 1 × 5,544 | 5,544 | One session required |
| `custom_static_buffers` | 4 × 2,112 | 8,448 | Shared receive history; five subscriptions can receive bursts |
| `custom_publishers` | 10 × 216 | 2,160 | Current main requests five; later sensor branches have more |
| `custom_subscriptions` | 5 × 216 | 1,080 | All five requested |
| `custom_topics` | 15 × 28 | 420 | Current main has ten named topics |
| `custom_nodes` | 1 × 164 | 164 | One node |
| `custom_services` | 1 × 200 | 200 | No service requested by main |
| `custom_clients` | 1 × 200 | 200 | No client requested by main |
| `custom_guard_condition` | 4 × 32 | 128 | Internal control use; do not trim blindly |
| `custom_init_options` | 3 × 44 | 132 | Internal initialization storage |
| `custom_wait_set` | 4 × 28 | 112 | Executor uses a wait set; internal limits need checking |
| **Total pools** | | **18,588** | Already included in `.bss` |

Each 2,112-byte receive-history entry contains a 16-byte pool link, a
2,048-byte payload buffer, 4-byte length, 4-byte owner, 8-byte timestamp,
1-byte entity type, 7 bytes padding and 24-byte related request/sample identity.
Four entries are not four samples per subscription. DDS QoS depth 10 does not
magically allocate ten local history buffers for each reader.

The single session contains:

| Component | Bytes |
|---|---:|
| Pool link | 16 |
| Custom transport object | 656 |
| XRCE session object | 200 |
| Graph guard-condition handle | 12 |
| Stream IDs/pointers and creation/destruction timeouts | 32 |
| Reliable input stream buffer | 2,048 |
| Reliable output stream buffer | 2,048 |
| Best-effort output stream buffer | 512 |
| Eight entity-ID counters | 16 |
| Flag and tail padding | 4 |
| **Total** | **5,544** |

The 656-byte custom transport already includes its 512-byte packet buffer and
100-byte framing state (two 42-byte byte-stuffing buffers plus state), callbacks,
communication interface and padding. Do not count these again outside the pool.
The 200-byte session contains four single-stream arrays (one input/output stream
for each reliability mode), callbacks and timing state. The 2,048-byte UART DMA
ring is **additional** storage outside the session pool.

Historical generated configuration matches these measured dimensions: custom
transport MTU 512, reliable stream history 4 in each direction, RMW receive
history 4, node/session 1, publishers 10, subscriptions 5, service/client 1 each,
wait sets/guard conditions 4. The corresponding headers identify XRCE client
2.4.2 and disable dynamic RMW-pool expansion, graph support and multithreading.
The actual main build inputs must be recovered before relying on every flag as
reproducible configuration. Editing a header alone cannot resize a precompiled
`libmicroros.a`; rebuild library and application consistently. Upstream describes
these compile-time pools in its [RMW configuration documentation](https://github.com/micro-ROS/rmw_microxrcedds).

The executor requests **five handles** (`freertos.cpp:366`), each a DWARF-verified
64-byte `rclc_executor_handle_t`: 320 B payload within the custom allocator, plus
allocator header/alignment and additional RCL/wait-set allocations. It is not an
extra 320-byte `.bss` array. Five input and five output message structures are
stack locals: Float32 is 4 B; Bool and UInt8 are 1 B, with possible stack padding.
RCL handles/support structures also live on defaultTask's reserved stack; do not
double-count them against static SRAM.

## Communication rates, transport and bandwidth

### Actual configuration and historical evidence

- `Core/Src/usart.c:44–50`, confirmed in ELF machine code: USART2 at **115200,
  8 data bits, no parity, 1 stop bit**, no flow control. PA2 TX/PA3 RX; circular
  RX DMA1 stream 5 and normal TX DMA1 stream 6, channel 4. Micro-ROS uses custom
  framed UART transport through `huart2`, not an STM32 USB CDC implementation.
  A board USB virtual COM bridge does not remove the UART baud limit. The exact
  physical host cable/bridge and agent invocation cannot be proven from source.
- `RASPI_UART_BAUD=1000000` in `main.h` is unused by USART2 initialization.
  It is not evidence that the existing firmware transmits at 1 Mbaud.
- With the configured 84 MHz timer clock, TIM11 is `84 MHz/(84*1000)` = 1 kHz;
  TIM10 is `84 MHz/(84*15000)` = **66.667 Hz (15 ms)**. The comments saying
  10 ms are stale. ToF acquisition runs on an eight-TIM10-event cycle: nominally
  120 ms, while the latest values are published each serviced publish event.
- Normal periodic output is two Float32 messages (speed/distance) and two UInt8
  messages (front/left ToF). Bool position feedback is event-driven. Five input
  subscriptions are speed, steering, position, stop (Float32), and LED (Bool).
- All endpoints use default RCL QoS. Offline inspection of the linked
  `rmw_qos_profile_default` confirms reliable, volatile, keep-last depth 10.
- Disassembly of `rmw_create_publisher` sets `session_timeout` to **1000 ms**;
  the reliable `rmw_publish` branch calls
  `uxr_run_session_until_confirm_delivery`. Publishes can block the sole ROS task
  for confirmation/timeouts; four periodic publishes occur sequentially.
- Transport write uses TX DMA but waits with `osDelay(1)` until UART-ready.
  Transport read snapshots DMA position and unconditionally delays one tick
  before consuming available data. The latter delay occurs even if data was
  already buffered. Its timeout accounting counts loop iterations rather than
  elapsed wall-clock ticks under arbitrary scheduling delays.
- The RX ring has modulo head/tail only, with no observed overflow counter.
  A 2048-byte ring takes **177.8 ms** to fill at uninterrupted 115200/8N1 input.
  At 921600 it would fill in 22.2 ms. Faster baud therefore needs faster drainage
  and overflow validation, not a smaller unmeasured DMA ring.
- Executor timeout is 10 ms followed by `osDelay(1)`. These are not a guaranteed
  control/publication period. `pub_flag` is one boolean, so delayed service
  coalesces timer events; clearing it can also overwrite a concurrent set.
  The source publishes only after the executor returns `RCL_RET_OK`, and ignores
  individual publish return codes.

The main branch starts with a single migration commit already using 115200 and
15 ms publication. Other available branches add encoder/ToF/sensor changes;
`f6cac22` changes an encoder/control-period constant from 999 to 19999, but that
is not a recorded explanation for micro-ROS throughput reduction on current
main. No log, experiment or commit rationale establishes that RAM caused rate
reductions. UART occupancy, reliable round trips, task waits and blocking I2C
are independently evidenced candidates; their relative runtime contributions
remain unmeasured. Capacity reductions proposed for five publishers must not
be copied into the later multi-sensor branches without recounting endpoints.

### Theoretical link utilization (not a measured packet capture)

115200/8N1 carries **11,520 serial bytes/s per direction**. EProsima documents a
minimum 12-byte XRCE plus 7-byte stream-frame overhead for an individual sample,
before escaping and reliability traffic. This yields a simple unbatched baseline
of 23 bytes per Float32 and 20 bytes per UInt8/Bool, hence 86 bytes per periodic
four-message cycle. See [the transport framing specification](https://micro-xrce-dds.docs.eprosima.com/en/v2.4.1/transport.html).

| Requested four-message cycle rate | Baseline TX bytes/s | TX occupancy at 115200 |
|---:|---:|---:|
| 50 Hz | 4,300 | 37.33% |
| 66.667 Hz (configured timer) | 5,733 | 49.77% |
| 100 Hz | 8,600 | 74.65% |
| 200 Hz | 17,200 | 149.31% — exceeds capacity in this model |

The configured periodic cycle alone consumes about **7.47 ms wire time** out of
15 ms, excluding escaping, ACK/NACK/heartbeat traffic, retries and feedback.
Actual framing/coalescing/padding must be captured; the table is a stated
per-sample model, not a universal lower bound across arbitrary batching schemes.
Byte stuffing can approach doubling encoded bytes in adverse payload patterns.
Two incoming Float32 command streams at 50 Hz each would add about 2,300 B/s,
19.97% of the **opposite-direction** capacity before overhead beyond the model.
Do not sum TX and RX into one half-duplex utilization number. ACKs for incoming
commands also consume MCU TX time; ACKs for telemetry consume agent TX time.

At 460800 the same 100 Hz periodic baseline uses 18.66% TX; at 921600, 9.33%.
Those numbers establish wire capacity only, not achievable message rates or
safe callback latency. No desired target rate was specified beyond investigation.
A provisional recommendation is to retain current interfaces/reliable QoS and
validate existing 66.7 Hz delivery first. For a validated 100 Hz requirement,
consider a coordinated higher baud such as 460800 or 921600 on both ends, then
measure overflow, ACK delays and deadlines. A register divider, bridge and
host-agent compatibility check is required before selecting the final baud.
Do not switch commands to best-effort or shorten reliability timeouts to make
benchmarks look faster. Removing the RX delay/moving blocking I2C could help,
but changes scheduling and requires explicit approval and hardware tests.

## Proposed optimizations and before/after projections

**None are applied.** Present allocations are verified; all after values below
are design projections under the existing ARM ABI, not rebuilt firmware results.
Symbols may acquire different surrounding alignment at the final link. Estimates
for overlapping changes must not simply be added together.

| Proposal | Current allocation | Proposed allocation | Estimated saving | Conditions / decision |
|---|---:|---:|---:|---|
| Filter-free position-controller type/storage, retaining common PID fields in their current order | Position 24,960 B; speed 24,960 B | Position 112 B; speed unchanged 24,960 B | **24,848 B** | Strongest source-backed candidate: position uses neither filter. Preserve unfiltered arithmetic, limits, reset/sample semantics; numerical regression and rebuild required. |
| Remove only position arrays while retaining filter-only metadata (alternative design) | Position 24,960 B | About 152 B with current field order | **24,808 B** | 24,800 B arrays + 8 B coalesced padding; requires guarding/reset refactor and forbidding unavailable filter enablement. Less clear than a purpose-specific type. |
| Reorder PID members for alignment, keep both filters in both objects | 24,960 B each | 24,928 B each | **64 B combined** | 24,925 member bytes rounded to 8; no packing/unaligned doubles. Rebuild all users. Minor saving, not the main solution. |
| Make immutable FIR coefficients const | 52 B `.data` | 52 B flash-only data | **52 B symbol storage** | Same coefficient bits; CMSIS parameter accepts const. Verify final linker padding and ISR timing. |
| Publishers 10 → 5; derived topics 15 → 10 | 2,160 + 420 B | 1,080 + 280 B | **1,220 B pool storage** | Current main only; library rebuild and successful five-publisher initialization required. Preserve five subscriptions. |
| Services/clients 1 → 0 each | 200 + 200 B | 0 | **400 B pool storage** | No current-main service/client use; check supported zero-capacity configuration and rebuild library. |
| All-static FreeRTOS task/object API, retain independent ROS allocator | 7,000 B RTOS heap | 0 RTOS heap | **7,000 B array storage**, plus possible metadata/code | Separate custom allocator guard first; prove no required dynamic RTOS path. Do not remove the active micro-ROS heap. |
| Merge both allocator domains | 14,000 B raw heap arrays | Must cover measured joint peak; unknown | **Not established** | Ownership, realloc metadata/failure paths and fragmentation tests mandatory. A 14,000 B unified array does not itself save buffer RAM. |
| Remove unused software-timer subsystem | At least 1,356 B stack/TCB/queue | Those objects absent | **At least 1,356 B**, conditional | Audit pend-function/CMSIS flags and absence of runtime timer use. Scheduling change deferred. |
| MTU 512 → 128, keep both stream histories 4 and input-history slots 4 | Receive buffers 8,448 B + session 5,544 B | Receive buffers 2,304 B + session about 1,704 B | **9,984 B**, conditional | Illustration only; creation/control messages, fragmentation/reliability and future message sizes require validation. Not recommended blindly. |
| Shrink stacks, heaps or DMA ring by guesswork | Current reservations | Unknown | **Not justified** | Runtime watermarks/peaks/overflow measurements required. |
| Shrink speed filter windows or convert histories to float | 24,800 B active history | Smaller | **Rejected under behavior-preservation constraint** | Changes exact filter response or numerical results. |

For the preferred purpose-specific position design, its 112 B consists of the
existing first 19 members: 104 B actual common state plus 8 B alignment.
The 24,848 B reduction comprises 24,800 B arrays, 21 B filter bookkeeping and
27 B now-unneeded alignment. The speed object remains unchanged. Combined
controller storage would be **49,920 → 25,072 B**. Total reserved SRAM would
project to **97,952 → 73,104 B**, leaving **25,200 B (25.63%)** unreserved.
This is a conditional layout calculation, not a measured final ELF.

Filter histories cannot safely be shared between independently active filtered
controllers: their samples, sums and indexes differ. In the present firmware
only speed needs filter storage; a separate optional static storage block for
speed is another viable design, but introduces pointer/state-layout overhead
that must be measured. Avoid new runtime heap allocation or lazy allocation
inside the 1 kHz control interrupt.

Required regression coverage before implementing the controller proposal:
compare original/candidate output bits and Compute return values over more than
3000 accepted samples, ring wrap, initial zero fill, positive/negative inputs,
clamps, manual/automatic transitions, `Init` overloads, retuning, sample-time
changes and tick wraparound. Test position unfiltered and speed with both filters;
retain the order of floating-point operations. Then measure ISR execution time
and jitter on hardware. No such candidate was implemented or tested here.

## Risks, hardware-only measurements and recommendation

- Current main ends the TIM11 control branch with `drive_pwm(speedTarget)`
  (`freertos.cpp:740`), even though PID output is computed into
  `motorPWM`. This is a preexisting actuation choice/bypass requiring its own
  review. It is not authorization to delete speed-controller state or change
  the motor command. No motor behavior was altered.
- Record minimum free heap and largest free block **separately** for each owner,
  including ROS initialization, burst traffic, reconnect/failure paths and
  realloc overlap. Source counter snapshots are not runtime measurements.
- Measure default/idle/timer stack high-water marks and MSP worst-case nesting.
  Keep margin beyond the lowest observed watermark; add overflow/OOM diagnostics
  only in a separately approved change.
- Capture per-topic delivered rates/latency, UART error/overrun counts, DMA ring
  wrap/overflow, publish timeout/return codes, retransmissions and ISR duration.
  Missing transport errors, ignored RCL init/publish results, and blocking I2C
  are important follow-up issues. None was silently changed.
- Fix reproducibility before optimization: recover the intended main-branch
  utility/library inputs with known versions; do not mix a newer sensor branch
  into this firmware just because it contains the missing library.

**F401RE assessment:** the present image links, but 352 B of unreserved margin
is not a reasonable allowance for future static growth, and no stack/heap peak
measurements establish runtime safety. The position-storage opportunity alone
would plausibly leave about 24.6 KiB spare without changing configured control
math. Thus a hardware upgrade is not established as necessary for this main
firmware; targeted structural cleanup and real timing/heap measurements should
come first. RAM recovery alone will not fix serial/ACK latency.

If retaining the current storage or adding functionality requires more RAM, the
minimum close-family Nucleo-64 upgrade recommended here is **NUCLEO-F411RE**:
128 KiB SRAM, 512 KiB flash, Cortex-M4 up to 100 MHz. ST documents the
[STM32F411RE memory](https://www.st.com/en/microcontrollers-microprocessors/stm32f411re.html)
and the [Nucleo-64 board](https://www.st.com/en/evaluation-tools/nucleo-f411re.html).
With the same hypothetical 97,952-byte reservation, 128 KiB would leave
**33,120 B**. This is not a rebuilt F411 measurement or a drop-in firmware promise:
check the shield's actual pins, timer/DMA mappings, clock divisors and startup
configuration. Keep physical PWM/sample timings unchanged during any port.
More SRAM does not increase bandwidth at an unchanged 115200 baud.

## Verification performed

- Existing ELF/MAP sizes and hashes checked; firmware source and artifacts left
  unchanged. Actual compiled PID offsets, element counts, primitive types,
  member sizes and total matched between `ptype /o`, `sizeof`, `nm` and map.
- All 27 per-instance members sum to 24,925 B; six padding regions sum to 35 B.
- Pool element sizes/counts and 18,588 B aggregate independently checked through
  ELF debug types and symbols; default QoS and publisher timeout inspected.
- UART baud and allocator/transport call paths verified from machine code;
  source timer configuration and historical branch differences inspected.
- `make -n all`: **blocked by missing current-checkout dependency source**.
  No full build, optimization comparison or firmware test suite execution.
- No hardware watermark, timing, loss-rate or free-heap measurement available.
- Verified savings **0 B**; all proposed savings are explicitly conditional.

Only `docs/memory_optimization_report.md` was created. Firmware, ELF, MAP,
FreeRTOS configuration, linker script and peripheral outputs are unchanged.
