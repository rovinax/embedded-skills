---
name: embedded-isr-design
description: >
  Use when generating or reviewing interrupt service routines (ISRs) for embedded firmware
  (STM32, GD32, ESP32, FreeRTOS, bare-metal, drone flight controllers, robotics). Enforces
  the "Fast In, Fast Out" principle — ISRs capture events and defer processing to task context.
  Prevents common AI mistakes: protocol parsing in ISR, control algorithms in ISR, printf/malloc
  in ISR, blocking calls, forgotten flag clearing, and missing FromISR API usage for RTOS
  integration.
---

# Embedded ISR Design

## Purpose

Design and review interrupt service routines for embedded systems.

Goals:

- low interrupt latency
- deterministic execution
- minimal ISR workload
- safe RTOS integration
- predictable system behavior

The core principle: **ISRs capture events, tasks process business logic.** An ISR is not a
function — it is a hardware-triggered context switch that must execute and return as fast as
possible.

The most common and dangerous AI mistake is writing ISRs that contain protocol parsing, sensor
fusion, PID computation, or `printf` calls. This pattern:

1. increases interrupt latency for all other interrupts (they are blocked or preempted)
2. defeats RTOS scheduling (the scheduler only runs outside ISRs)
3. causes DMA buffer overruns (data arrives faster than the bloated ISR processes it)
4. introduces jitter that destabilizes control loops
5. creates deadlock risk when ISR tries to acquire a mutex held by a preempted task

## ISR Core Principle

**Fast In, Fast Out** — enter quickly, do the minimum, exit immediately.

An ISR should be measured in microseconds, not milliseconds.

## ISR Responsibility Boundary

### Allowed in ISR — event capture only

- read hardware status register
- clear the interrupt flag
- capture a timestamp
- copy a small, bounded amount of data to a buffer or register
- set an event flag (`xTaskNotifyGiveFromISR`, `osEventFlagsSet`)
- send to a queue (`xQueueSendFromISR`, `osMessageQueuePut`)
- give a semaphore (`xSemaphoreGiveFromISR`)
- start a DMA transfer
- wake a task (`vTaskNotifyGiveFromISR`)
- increment a bounded counter

### Forbidden in ISR — business processing

- protocol parsing (JSON, protobuf, NMEA, MAVLink, custom binary)
- attitude estimation (Mahony, Madgwick)
- PID computation or any control law calculation
- Kalman filter update step
- path planning or navigation logic
- file system operations
- Flash memory write/erase
- network protocol processing
- display refresh (OLED, LCD)
- logging output
- complex loops with non-constant bounds
- blocking calls of any kind

## ISR Execution Time Budget

Every ISR must have a bounded and predictable execution time.

| ISR Type   | Maximum Recommended Execution Time |
|------------|-----------------------------------|
| GPIO       | < 5 µs                            |
| UART       | < 10 µs                           |
| SPI        | < 10 µs                           |
| DMA        | < 10 µs                           |
| I2C        | < 15 µs                           |
| CAN        | < 20 µs                           |
| Timer      | < 20 µs                           |

Forbidden — unbounded loops in ISR:

```c
for (int i = 0; i < large_variable; i++) {  // forbidden — unbounded
    process(data[i]);
}
```

## Data Processing in ISR

Allowed — minimal data movement:

```c
rx_buf[index++] = data;       // allowed — bounded ring buffer write
```

Forbidden — heavy processing:

```c
json_parse(&parser, byte);       // forbidden
protobuf_decode(&msg, buf, len); // forbidden
kalman_update(&kf, measurement); // forbidden
```

## `printf` Rule

`printf` (or any `printf`-family function) is strictly forbidden inside ISR context. It is:

- blocking (waits on stdout which may be UART, semi-hosting, or ITM)
- non-deterministic in execution time
- typically not reentrant
- extremely high latency (a single `printf` can take hundreds of microseconds to milliseconds)

Forbidden:

```c
void USART1_IRQHandler(void) {
    printf("rx byte: %02x\r\n", data);  // forbidden
}
```

## Dynamic Memory Rule

Heap allocation functions are strictly forbidden inside ISR:

```c
malloc(size);     // forbidden
calloc(n, size);  // forbidden
realloc(ptr, sz); // forbidden
free(ptr);        // forbidden
```

Heap operations have non-deterministic latency (coalescing, splitting, garbage collection in
some allocators), which violates the bounded execution requirement of ISRs.

## Floating-Point Rule

Extensive floating-point math is forbidden in ISR context:

```c
float angle = atan2f(y, x);  // forbidden — FPU operations are costly, stack-heavy
```

On Cortex-M4F/M7F with FPU, a single `atan2f` can take 50–150 cycles. On Cortex-M3/M0 (no FPU),
software float emulation adds thousands of cycles and massive stack usage.

## HAL Call Rules

Allowed — interrupt handler dispatch calls that clear hardware state:

```c
HAL_UART_IRQHandler(&huart1);  // allowed — dispatches to HAL RX/TX/error callbacks
HAL_DMA_IRQHandler(&hdma);     // allowed
```

Forbidden — blocking HAL operations in ISR:

```c
HAL_UART_Transmit(&huart1, data, len, timeout);  // forbidden — blocking, polling
HAL_SPI_Transmit(&hspi1, data, len, timeout);    // forbidden
HAL_Delay(ms);                                    // forbidden — SysTick-based blocking
```

## DMA Co-Design

Correct — DMA completion signals a task to process data:

```text
DMA Complete IRQ → [Event Flag / TaskNotify] → Task → Process Data
```

Incorrect — DMA ISR runs the processing inline:

```text
DMA Complete IRQ → Parse Protocol → Control Logic  (forbidden)
```

## RTOS Integration Rules

When using an RTOS (FreeRTOS, CMSIS-RTOS2, RT-Thread, Zephyr), ISR-to-task communication
**must** use the `FromISR` variants of RTOS APIs:

Required for FreeRTOS:

```c
xQueueSendFromISR(queue, &data, &higherPriorityTaskWoken);
vTaskNotifyGiveFromISR(task, &higherPriorityTaskWoken);
xSemaphoreGiveFromISR(semaphore, &higherPriorityTaskWoken);
portYIELD_FROM_ISR(higherPriorityTaskWoken);
```

Required for CMSIS-RTOS2:

```c
osMessageQueuePut(queue, &data, 0, timeout);      // allowed in ISR
osEventFlagsSet(flags, mask);                     // allowed in ISR
```

Forbidden — non-FromISR APIs inside ISR:

```c
xQueueSend(queue, &data, timeout);     // forbidden
osMessageQueuePut(queue, &data, ...);  // forbidden with non-zero timeout
xSemaphoreGive(semaphore);             // forbidden
```

Always check `higherPriorityTaskWoken` and call `portYIELD_FROM_ISR()` if true, so the
scheduler runs before returning to the outermost interrupt.

## Interrupt Priority Rules

Interrupt priorities must reflect real-time criticality, not convenience:

```text
Highest Priority
    ↑
    Timer Control   (control loop tick — highest, hard real-time)
    ↑
    DMA             (data throughput, buffer management)
    ↑
    SPI / I2C       (sensor data streaming)
    ↑
    UART            (communication — lower priority than control)
    ↑
    EXTI            (user buttons, low-speed events)
    ↑
Lowest Priority
```

For a flight controller:

```text
TIM Update IRQ      priority 0 (highest — control loop tick)
IMU Data Ready IRQ  priority 1
SPI DMA IRQ          priority 2
UART IRQ             priority 3
EXTI (button)       priority 4
```

**Control-related interrupts must have higher priority than communication interrupts.**
This prevents a burst of UART data from delaying the control loop.

## Shared Data Rules

Bare global variables shared between ISR and task context must be protected. The preferred
mechanisms (in order of preference) are:

1. **Queue** — ISR enqueues, task dequeues

   ```c
   // ISR
   xQueueSendFromISR(rx_queue, &byte, &woken);

   // Task
   xQueueReceive(rx_queue, &byte, portMAX_DELAY);
   ```

2. **Double buffer** — ISR writes to one buffer, task reads the other

   ```c
   static uint8_t buf[2][256];
   static uint8_t active_buf = 0;
   static uint16_t index = 0;

   // ISR: fill active buffer
   buf[active_buf][index++] = byte;

   // Task: when buffer is full, swap and process
   ```

3. **Event flag + single reader** — for simple status sharing

   ```c
   // ISR
   vTaskNotifyGiveFromISR(consumer_task, &woken);

   // Task
   ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
   ```

Forbidden — unstructured shared global with no synchronization:

```c
volatile imu_t imu_data;  // ISR writes, task reads — no synchronization

// ISR
imu_data = sensor_read();     // problematic

// Task
process(&imu_data);           // data may be partially updated
```

### Ring Buffer for UART Reception

For UART RX, use a ring buffer rather than parsing in the ISR:

```text
UART IRQ → Ring Buffer → UART Task → Protocol Parser
```

```c
// ISR
ringbuf_put(&uart_rx_ring, rx_byte);

// Task
uint8_t byte;
ringbuf_get(&uart_rx_ring, &byte);
// parse protocol here — in task context
```

## Timer ISR Rules

A timer ISR should only be responsible for:

- generating the system tick (`osKernelGetTickCount`)
- triggering periodic event flags
- capturing input capture timestamps
- incrementing a counter for timekeeping

Forbidden in timer ISR:

```c
void TIM_IRQHandler(void) {
    HAL_TIM_IRQHandler(&htim);

    pid_update();           // forbidden — control algorithm belongs in task
    telemetry_send();       // forbidden — communication belongs in task
    sensor_fusion_update(); // forbidden — estimation belongs in task
}
```

## ISR Design Patterns

### Flag Pattern — single binary event

```c
void EXTI_IRQHandler(void) {
    data_ready = true;     // simple flag, checked by task poll
    EXTI->PR1 |= (1U << 0);
}
```

### Queue Pattern — data streaming

```c
void USART1_IRQHandler(void) {
    uint8_t byte;
    byte = (uint8_t)(USART1->DR & 0xFFU);
    BaseType_t woken = pdFALSE;
    xQueueSendFromISR(rx_queue, &byte, &woken);
    portYIELD_FROM_ISR(woken);
}
```

### Event Pattern — task notification

```c
void DMA2_IRQHandler(void) {
    DMA2->IFCR |= (1U << 0);
    BaseType_t woken = pdFALSE;
    vTaskNotifyGiveFromISR(processor_task, &woken);
    portYIELD_FROM_ISR(woken);
}
```

### Ring Buffer Pattern — buffered UART RX

```c
void USART1_IRQHandler(void) {
    uint8_t byte = (uint8_t)(USART1->DR & 0xFFU);
    ringbuf_put(&uart_rx_ring, byte);
    USART1->SR &= ~(USART_SR_RXNE);
}
```

## Flight Controller Special Rules

For flight controller and robotics applications specifically:

Forbidden — running Mahony/PID inside IMU data-ready ISR:

```text
IMU IRQ → Mahony → PID → PWM  (all in ISR — forbidden)
```

Required — ISR signals the control task:

```text
IMU IRQ → [TaskNotify]
           ↓
       Control Task (1 kHz)
           ↓
       Mahony → PID → PWM
```

With this pattern, the control loop period is determined by the RTOS scheduler, not by the
ISR execution time. ISR latency accumulates into control loop jitter — keeping ISRs minimal
ensures consistent and predictable control timing.

## Validation Checklist

After generating ISR code, verify:

- [ ] ISR has a single, clearly defined responsibility
- [ ] ISR execution time is bounded and predictable
- [ ] No complex loops with variable bounds in ISR
- [ ] No protocol parsing in ISR
- [ ] No control algorithm computation in ISR
- [ ] No `printf` or logging in ISR
- [ ] No `malloc`/`free` or heap operations in ISR
- [ ] No blocking calls (HAL_Delay, polling waits) in ISR
- [ ] RTOS `FromISR` APIs used where applicable
- [ ] Interrupt flag correctly cleared
- [ ] Event notification or queue used for ISR→task handoff
- [ ] ISR responsibilities separated from task responsibilities
- [ ] No business logic in ISR
- [ ] DMA ISR defers processing to task context
- [ ] if RTOS: `portYIELD_FROM_ISR()` called when task wake is requested

## Auto Fix Strategy

When ISR violations are found:

1. **Identify** heavy operations inside the ISR (parsing, math, loops, blocking calls)
2. **Move** those operations into a dedicated task context
3. **Replace** direct shared-variable ISR→task communication with queue, event, or ring buffer
4. **Replace** blocking APIs with non-blocking or `FromISR` variants
5. **Introduce** a ring buffer for serial data accumulation if needed
6. **Recalculate** worst-case interrupt latency after the changes
7. **Revalidate** the full ISR design against the checklist
