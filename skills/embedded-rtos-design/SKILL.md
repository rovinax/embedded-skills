---
name: embedded-rtos-design
description: >
  Use when designing, generating, or reviewing RTOS-based embedded firmware (FreeRTOS, CMSIS-RTOS2,
  RT-Thread, Zephyr, ThreadX) for STM32, GD32, ESP32, drone flight controllers, and robotics
  projects. Enforces real-time system architecture — task decomposition, scheduling, priority
  assignment, inter-task communication, ISR cooperation, resource ownership, and watchdog
  monitoring. Prevents the common AI anti-pattern of creating monolithic super-tasks with mixed
  responsibilities and jitter-prone control loops coupled to communication timing.
---

# Embedded RTOS Design

## Purpose

This skill defines architecture rules for RTOS-based embedded systems.

Goals:

- deterministic scheduling
- low latency
- task isolation
- maintainability
- scalability
- predictable timing behavior

This skill applies to: FreeRTOS, CMSIS-RTOS2, RT-Thread, Zephyr, ThreadX, and similar RTOS targets.

The most common AI mistake is generating a single monolithic task that reads sensors, runs PID,
drives UART, updates OLED, and writes flash — all in one loop. This creates a system that is:

- impossible to analyze for real-time behavior
- impossible to extend without breaking existing functions
- tightly coupled across all concerns
- subject to jitter from unrelated operations

This skill decomposes that pattern into proper task architecture.

## RTOS Design Principles

### One Task, One Responsibility

Each task owns exactly one functional domain.

Correct task decomposition:

```text
sensor_task      — read sensors, publish data
control_task     — compute control outputs
telemetry_task   — handle communication
logger_task      — background logging
```

Incorrect — the monolithic super-task:

```text
main_task:
    read IMU
    run PID
    update OLED
    send UART
    write flash
    osDelay(1)
```

This pattern is forbidden. It couples unrelated concerns, prevents real-time analysis, and
cannot scale.

### Control Loop Timing Isolation

Control loops must never depend on communication timing.

Allowed:

```text
sensor_task  (1 kHz) → queue → control_task (1 kHz) → queue → motor_task (1 kHz)
telemetry_task (50 Hz, independent)
```

Forbidden — control loop triggered by communication events:

```text
UART Receive → Trigger Control Loop
```

Coupling control to communication introduces jitter and destabilizes the control period.
This is critical for flight controllers, balance bots, and无人车 (autonomous vehicles).

## Task Classification

Every task the agent creates must be classified into one of the categories below.

### Acquisition Task

Examples: `imu_task`, `adc_task`, `encoder_task`, `gps_task`

Responsibilities:

- acquire raw data from hardware
- publish data via queue or event

Must not contain:

- control algorithms
- logging calls
- network or protocol processing

### Control Task

Examples: `attitude_task`, `motor_control_task`, `balance_task`

Responsibilities:

- compute control outputs from sensor inputs
- enforce control loop timing with `osDelayUntil` / `vTaskDelayUntil`

Must not contain:

- SPI or I2C register access
- display refresh
- file I/O

### Communication Task

Examples: `uart_task`, `can_task`, `usb_task`, `ethernet_task`

Responsibilities:

- handle protocol encoding/decoding
- manage communication buffers
- forward received data to appropriate queues

Must not contain:

- control law calculations
- application business logic

### Service Task

Examples: `logger_task`, `storage_task`, `monitor_task`, `health_check_task`

Responsibilities:

- background housekeeping
- health monitoring
- data logging to nonvolatile storage

Must not contain:

- real-time control operations
- sensor acquisition

## Scheduling Rules

### Fixed-Rate Scheduling

Periodic tasks must use fixed-rate scheduling to avoid drift.

Forbidden — drifting loop:

```c
while (1) {
    work();
    osDelay(1);
}
```

`osDelay(1)` waits 1 ms *from the end of work()*, so the actual period drifts by the
execution time of `work()`.

Required — drift-free scheduling:

```c
uint32_t tick = osKernelGetTickCount();

while (1) {
    work();
    osDelayUntil(tick, 1);
}
```

or:

```c
vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(1));
```

## Priority Rules

Assign priority based on real-time criticality:

```text
Higher Priority
    ↑
    Control Task           (highest — hard real-time)
    ↑
    Sensor Task            (next — data freshness)
    ↑
    Communication Task     (soft real-time)
    ↑
    Service / Logging Task (background)
    ↑
    Idle                   (lowest)
```

Example assignment for a flight controller:

```text
control_task       priority 6
imu_task           priority 5
uart_task          priority 4
logger_task        priority 2
```

## Inter-Task Communication Rules

Shared data between tasks must **never** use bare global variables.

Forbidden:

```c
// shared header
extern imu_data_t g_imu;   // forbidden
```

### Queue — for Data Streams

Use message queues for streaming data from producer to consumer:

```text
imu_task → [queue] → control_task
```

```c
osMessageQueuePut(imu_queue_id, &data, 0, timeout);
osMessageQueueGet(imu_queue_id, &data, NULL, wait);
```

### Event Flags — for State Notification

Use event flags for signaling state changes:

- DMA transfer complete
- data ready
- GPS lock acquired

```c
osEventFlagsSet(event_id, IMU_DATA_READY);
osEventFlagsWait(event_id, IMU_DATA_READY, flags, timeout);
```

### Mutex — for Shared Resources

Use mutexes when multiple tasks access a shared resource (flash, SD card, shared memory buffer):

```c
osMutexAcquire(flash_mutex_id, timeout);
// access shared flash
osMutexRelease(flash_mutex_id);
```

Do **not** use a mutex as a message-passing mechanism.

## ISR Cooperation Rules

ISRs must be minimal. Allowed operations only:

- set an event flag
- send to a queue (from ISR-safe API)
- release a semaphore (from ISR-safe API)
- start a DMA transfer

Example:

```c
void USART_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    xQueueSendFromISR(rx_queue_id, &rx_byte, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

Forbidden inside ISR:

```c
printf()                  // blocking, non-deterministic
malloc() / free()         // non-deterministic latency
floating-point math       // excessive stack and cycle cost
protocol parsing          // should be in task context
control algorithm math    // should be in task context
```

### DMA Deferred Processing Pattern

Correct:

```text
DMA ISR → [event] → communication_task → protocol_parser
```

Incorrect (all work done in ISR context):

```text
DMA ISR → protocol_parser → control_algorithm
```

## Resource Ownership Rules

Every hardware resource must have exactly one owning task.

Example:

```text
UART1 → owner: telemetry_task
SPI1  → owner: imu_task
I2C1  → owner: sensor_task
```

Forbidden — multiple tasks writing the same UART directly:

```text
imu_task       ──┐
logger_task    ──┤→ UART1 (conflict)
control_task   ──┘
```

Correct — single owner, shared via queue:

```text
all tasks → [queue] → telemetry_task → UART1
```

## Watchdog Rules

Every RTOS project must include a watchdog or health-monitor mechanism:

```c
health_monitor_task:
    - verify all task periodic heartbeats
    - monitor queue fill levels
    - track CPU load (via idle task hook)
    - report anomalies or trigger safe state
```

Minimum watchdog coverage:

- task liveness (each task toggles a "still alive" flag within its period)
- queue depth (check for approaching overflow)
- stack high-water mark (configASSERT or uxTaskGetStackHighWaterMark)

## Timing Documentation

When creating a periodic task, record the following parameters in the task design doc:

```text
+----------------+------+----------+-------+--------+
| Task           | Hz   | Priority | Stack | WCET   |
+----------------+------+----------+-------+--------+
| imu_task       | 1000 | 5        | 1024  | 150 µs |
| control_task   | 1000 | 6        | 1024  | 200 µs |
| telemetry_task | 50   | 3        | 512   | 50 µs  |
+----------------+------+----------+-------+--------+
```

WCET (Worst-Case Execution Time) should be estimated and documented for every hard real-time task.

## Validation Checklist

After generating RTOS code, verify:

- [ ] Each task has a single, clearly defined responsibility
- [ ] No super-tasks mixing acquisition, control, and communication
- [ ] Periodic tasks use `osDelayUntil` or `vTaskDelayUntil` (not `osDelay`)
- [ ] Priorities assigned by real-time criticality
- [ ] ISRs contain no `printf`, `malloc`, floating-point math, protocol parsing, or control logic
- [ ] ISR-to-task handoff via queue or event (not shared globals)
- [ ] Inter-task communication uses queues, events, or mutexes — not bare `extern` globals
- [ ] Control loops are not coupled to communication timing
- [ ] DMA post-processing runs in task context, not ISR
- [ ] Each hardware resource has a single owning task
- [ ] Watchdog or health monitor exists and covers task liveness
- [ ] Timing parameters (period, priority, stack, WCET) are documented

## Auto Fix Strategy

When violations are found:

1. **Identify** task responsibility overlap or ISR overreach
2. **Split** monolithic tasks into single-responsibility tasks
3. **Replace** bare global sharing with queues, events, or mutexes
4. **Move** heavy ISR work (protocol parsing, control math) into task context via deferred processing
5. **Introduce** a resource ownership model — one owner per peripheral
6. **Reassign** priorities based on real-time criticality
7. **Regenerate** the RTOS architecture and task documentation
