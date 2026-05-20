# Experiment 7 — FreeRTOS Dual Tasks and Priority-Based Preemptive Scheduling

> Two tasks. Same delay interval. Different priority levels. Completely different observable outcomes.

---

## Objective

Configure two FreeRTOS tasks with programmable priority levels and analyze how the **preemptive scheduler** distributes CPU time between them — validated through LED blink behavior and SWV ITM trace console output.

---

## Components Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| External LED | 1 |
| 220 Ohm Resistor | 1 |
| Breadboard + connecting wires | — |
| USB Type-A to Mini-B cable | 1 |

### External LED Connection
```
PA6 ---- [220 Ohm] ---- LED(+) ---- LED(-) ---- GND
```

---

## FreeRTOS Priority Scheduling — How It Works

### Equal Priority — Round-Robin Time Sharing
```
Time:    0ms      500ms    1000ms   1500ms
         |         |         |         |
Task_1:  ##........##........##........   LED1 blinks at 1 Hz
Task_2:  ..##......  ..##......  ..##..   LED2 blinks at 1 Hz

Both tasks yield via osDelay(500) -- CPU time shared equally.
```

### Unequal Priority — Preemption in Action
```
Time:    0ms      500ms    1000ms
         |         |         |
Task_1:  ########################   (HIGH priority -- first in CPU queue)
         (during blocking, Task_2 gets occasional CPU access)
Task_2:  .....#.....#.....#.....#   LED2 blinks more slowly / starves
```

### Extreme Priority Gap — Task Starvation
```
Task_HIGH (Realtime): ################################
Task_LOW  (Low):      ................................ (may never execute!)
```

---

## CubeMX Configuration

### Step 1 — GPIO Setup

| Pin | Label | Mode |
|-----|-------|------|
| PA5 | LED_1 (onboard LD2) | GPIO Output |
| PA6 | LED_2 (external) | GPIO Output |

### Step 2 — System Settings

| Setting | Value |
|---------|-------|
| RCC | BYPASS Clock Source |
| HCLK | 84 MHz |
| SYS → Debug | Trace Asynchronous Sw |
| SYS → Timebase | **TIM6** |

### Step 3 — FreeRTOS (CMSIS_V2)
- **Tasks & Queues** tab → create **two tasks**:

| Field | Task 1 | Task 2 |
|-------|--------|--------|
| Task Name | `LED_1` | `LED_2` |
| Entry Function | `Task1_function` | `StartLED_2` |
| Priority | (set per observation row) | (set per observation row) |
| Stack Size | 128 words | 128 words |

### Step 4 — Printf and SWV Setup (same as Exp 6)
- Enable `printf`/`scanf` float formatting under C/C++ Build Settings
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

### Task Function Bodies
```c
/* Task 1 -- drives LED on PA5 */
void Task1_function(void *argument) {
    /* USER CODE BEGIN 5 */
    for (;;) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        printf("Task_1 Executing for LED Toggle \n");
        osDelay(500);
    }
    /* USER CODE END 5 */
}

/* Task 2 -- drives LED on PA6 */
void StartLED_2(void *argument) {
    /* USER CODE BEGIN StartLED_2 */
    for (;;) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6);
        printf("Task_2 Executing for LED Toggle \n");
        osDelay(500);
    }
    /* USER CODE END StartLED_2 */
}
```

---

## Observation Table — Repeat for Each Priority Combination

Adjust priorities from the CubeMX Tasks & Queues tab, regenerate code, and record the behavior:

| S.No | Task_1 Priority | Task_2 Priority | LED_1 Rate | LED_2 Rate | SWV Console Pattern | Remarks |
|------|----------------|----------------|-----------|-----------|---------------------|---------|
| 1 | `osPriorityLow` | `osPriorityLow` | | | | Equal share |
| 2 | `osPriorityNormal` | `osPriorityLow` | | | | Minor imbalance |
| 3 | `osPriorityHigh` | `osPriorityNormal` | | | | Noticeable difference |
| 4 | `osPriorityRealtime` | `osPriorityNormal` | | | | Risk of starvation |
| 5 | `osPriorityError` | `osPriorityLow` | | | | Extreme scenario |

### Expected SWV Console Patterns

```
Equal priorities:
  Task_1 Executing for LED Toggle
  Task_2 Executing for LED Toggle
  Task_1 Executing for LED Toggle
  Task_2 Executing for LED Toggle   <-- regular alternation

High vs Low:
  Task_1 Executing for LED Toggle
  Task_1 Executing for LED Toggle
  Task_1 Executing for LED Toggle
  Task_2 Executing for LED Toggle   <-- Task_2 appears infrequently
```

---

## Running the Program

```
1. Set task priorities in CubeMX --> Regenerate Code
2. Re-enter task function bodies (overwritten after regeneration)
3. Connect board --> Build --> Debug
4. Window --> Show View --> SWV ITM Data Console
5. Enable Port 0 --> Start Trace --> Resume
6. Watch LED blink rates and console message frequency
7. Repeat this process for every row in the observation table
```

---

## Suggested Modifications

| Experiment | How to Implement |
|-----------|-----------------|
| Make Task_2 dominate Task_1 | Assign Task_2 a higher priority level |
| Demonstrate full starvation | Set Task_1 to `osPriorityRealtime`, Task_2 to `osPriorityLow` — Task_2 LED should stop |
| Introduce a third task | Add `LED_3` on a new GPIO pin at an intermediate priority |
| Remove `osDelay` from Task_1 | Task_1 will loop endlessly and starve Task_2 completely |

---

## Reflection Questions

1. How does the scheduler split CPU time between two equal-priority tasks both using `osDelay(500)`?
2. What happens to the LED states when one task has no `osDelay()` call at all?
3. How does this dual-task RTOS design compare to a super loop managing two LEDs in terms of structure and flexibility?
4. How does SWV ITM tracing confirm correct RTOS scheduling without consuming GPIO pins or UART bandwidth?

---

## Result

Two FreeRTOS tasks were created with configurable priorities using the CMSIS-RTOS v2 interface. Priority-based preemptive scheduling behavior was clearly demonstrated through differing LED blink rates and SWV ITM console output, illustrating how a higher-priority task dominates CPU access and can lead to starvation of lower-priority tasks.
