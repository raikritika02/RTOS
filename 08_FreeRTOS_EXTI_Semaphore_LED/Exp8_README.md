# Experiment 8 — External Interrupt Handling with Binary Semaphore

> Never perform heavy processing inside an ISR. Use it to signal a task, and let the task do the work.

---

## Objective

Configure an **EXTI interrupt** on the onboard user button (PC13) and use a **Binary Semaphore** to synchronize an LED control task — implementing the industry-standard **Deferred Interrupt Processing** pattern.

---

## Components Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |

---

## Core Concept — Deferred Interrupt Processing

### Incorrect Method (Heavy Work Inside ISR)
```
Button pressed
    |
    v
ISR fires --> blinks LED 5 times (2.5 seconds of HAL_Delay inside ISR!)
              ^
              BLOCKS ALL OTHER INTERRUPTS throughout this entire period!
```

### Correct RTOS Method (This Experiment)
```
Button pressed
    |
    v
ISR fires --> osSemaphoreRelease()   <-- ISR performs ONE fast operation
    |                                    (completes in microseconds)
    |
    v
Scheduler detects LED_Control task is now READY
    |
    v
LED_Control task UNBLOCKS --> acquires semaphore --> blinks LED 5 times
    (runs in normal task context -- HAL_Delay is safe to use here)
```

---

## Complete Sequence Diagram

```
USER        BUTTON HW      EXTI/NVIC      SEMAPHORE      LED_Control TASK
 |               |               |               |               |
 |-- press -->   |               |               |               |
 |           falling             |               |               |
 |           edge  ----------->  |               |               |
 |                           IRQ fires           |               |
 |                               |-- Release --> |  count: 0->1  |
 |                               |               |               |
 |                               |               |  task unblocks|
 |                               |               |<-- Acquire -- |  count: 1->0
 |                               |               |               |
 |                               |               |          blink x5
 |                               |               |         (10 toggles
 |                               |               |          x 250 ms)
 |                               |               |               |
 |                               |               |          blocks again
 |                               |               |          (waits for next
 |                               |               |           semaphore signal)
```

---

## CubeMX Configuration

### Step 1 — GPIO Setup

| Pin | Label | Mode |
|-----|-------|------|
| PA5 | LED LD2 | GPIO Output |
| PC13 | USER Button | **GPIO_EXTI13** |

### Step 2 — EXTI and Interrupt Configuration
1. **GPIO → PC13** → Trigger detection: **Falling Edge**
2. **GPIO → NVIC tab** → enable: `EXTI line[15:10] interrupt` (checked)
3. **System Core → NVIC** → set EXTI[15:10] preemption priority to **7**

> FreeRTOS requires that any ISR calling FreeRTOS API functions must have its NVIC priority set at or above `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` (typically 5 or higher numerically). Priority 7 meets this requirement.

### Step 3 — System Settings

| Setting | Value |
|---------|-------|
| RCC | BYPASS |
| HCLK | 84 MHz |
| SYS → Debug | Trace Asynchronous Sw |
| SYS → Timebase | **TIM6** |

### Step 4 — FreeRTOS (CMSIS_V2)
- **Tasks & Queues** → rename the default task to `LED_Control`
- **Timers & Semaphores** → **Add** a Binary Semaphore:

| Field | Value |
|-------|-------|
| Semaphore Name | `myBinarySem01` |
| Initial Count | `1` (manually overridden to 0 in code — see below) |

### Step 5 — Printf and SWV Setup
- Enable `printf`/`scanf` formatting under C/C++ Build Settings
- Debugger → SWV enabled, Core Clock = 84 MHz

---

## Source Code — `main.c`

### Header Include + ITM Output Redirect
```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
/* USER CODE END Includes */

/* USER CODE BEGIN 0 */
int _write(int file, char *ptr, int len) {
    for (int i = 0; i < len; i++) {
        ITM_SendChar(*ptr++);
    }
    return len;
}
/* USER CODE END 0 */
```

### LED Control Task Body
```c
void StartDefaultTask(void *argument) {
    uint8_t i;
    for (;;) {
        /* Block here until the ISR releases the semaphore */
        if (osSemaphoreAcquire(myBinarySem01Handle, 100) == osOK) {
            printf("Inside LEDControl Task\n");
            i = 0;
            while (i < 10) {                         // 10 toggles = 5 full blinks
                HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
                HAL_Delay(250);                      // 250 ms ON, 250 ms OFF
                i++;
            }
        }
        /* Return to loop top and block again -- dormant until next press */
    }
}
```

### Button ISR Callback
```c
/* Called automatically by HAL when EXTI triggers on PC13 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    osSemaphoreRelease(myBinarySem01Handle);  // Wake the LED task
}
```

### CRITICAL — Fix the Semaphore Initial Count

Locate this auto-generated line in `main.c` and change the **second argument from `1` to `0`**:

```c
// Auto-generated (incorrect -- task runs immediately at startup without a button press):
myBinarySem01Handle = osSemaphoreNew(1, 1, &myBinarySem01_attributes);

// Required fix (task starts BLOCKED, only wakes on a button press):
myBinarySem01Handle = osSemaphoreNew(1, 0, &myBinarySem01_attributes);
//                                      ^
//                               Initial count = 0 --> starts blocked
```

> This manual change must be reapplied every time CubeMX regenerates the code.

---

## Running and Observing

```
1. Build --> Debug --> Switch to debug perspective
2. Window --> Show View --> SWV ITM Data Console
3. Enable Port 0 --> Start Trace --> Resume
4. Application starts -- LED_Control is BLOCKED (console is silent)
5. Press the USER button on the board
6. LED blinks 5 times
7. "Inside LEDControl Task" appears in the console
8. Task returns to BLOCKED -- waiting for the next button press
```

---

## Observation Table

| S.No | Query | Response |
|------|-------|----------|
| 1 | State of LED_Control task at program start | BLOCKED |
| 2 | State immediately after button press | READY --> RUNNING |
| 3 | Does LED_Control keep running after the press? | No -- blocks again |
| 4 | Priority level of the EXTI interrupt line | |

---

## Bonus Experiment — Behavior Variant

Replace the task body with this version and compare the results:

```c
/* Variant: Unconditional acquire attempt */
for (;;) {
    osSemaphoreAcquire(myBinarySem01Handle, 100);  // no if() check
    i = 0;
    while (i < 10) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        osDelay(250);
        i++;
    }
    printf("Inside LEDControl Task\n");
}
```

**Question:** What changes when the 100 ms timeout expires without a button press?

---

## Reflection Questions

1. Why does `LED_Control` stop running after the sequence completes rather than continuing indefinitely?
2. What is a breakpoint and how can it be used in debug mode during this experiment?
3. Walk through the full `osSemaphoreAcquire` execution path — what happens at each stage?
4. How does behavior change if the `if (... == osOK)` guard is removed from the task?

---

## Result

The Binary Semaphore successfully synchronized the LED blinking task with the hardware button press event. The ISR performed only a single fast `osSemaphoreRelease()` call, while all time-consuming LED operations were safely deferred to the `LED_Control` task — a clean and correct implementation of the Deferred Interrupt Processing pattern.
