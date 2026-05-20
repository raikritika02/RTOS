# Experiment 6 — FreeRTOS: Creating Your First Task with SWV ITM Tracing

> Your entry point into the RTOS world. One task. One LED. One debug message. But an entirely different programming paradigm.

---

## Objective

Create a single **FreeRTOS task** on the STM32F446RE using the CMSIS-RTOS v2 interface, toggle the onboard LED every 500 ms, and observe live task execution through the **SWV ITM Data Console** in STM32CubeIDE.

---

## Components Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |

---

## How FreeRTOS Transforms the Execution Model

### The Old Way (Super Loop)
```
main() --> while(1) { toggle; HAL_Delay(500); toggle; ... }
           CPU is STUCK during HAL_Delay -- no other work is possible
```

### The New Way (FreeRTOS)
```
Scheduler starts
    |
    +--> Task_1 (RUNNING)
            |
            +-- HAL_GPIO_TogglePin()
            +-- printf("Task_1 Executing...")
            +-- osDelay(500)  <-- Task enters BLOCKED state
                                   CPU is FREE for other tasks
                    |
                   500 ms later...
                    |
            Task_1 --> READY --> RUNNING again
```

### Task State Machine
```
              osThreadNew()
                   |
                   v
              +---------+
              |  READY  |<--------------------------+
              +----+----+                           |
                   |  Scheduler selects task        |
                   v                                |
              +---------+                           |
              | RUNNING |---- osDelay(500) -------> |
              +---------+                      +----+----+
                                               | BLOCKED |
                                               +---------+
```

---

## CubeMX Configuration

### Step 1 — GPIO
- **PA5** → GPIO Output (onboard LED LD2)

### Step 2 — System Settings

| Setting | Value |
|---------|-------|
| RCC | BYPASS Clock Source |
| HCLK | 84 MHz |
| SYS → Debug | Trace Asynchronous Sw |
| SYS → Timebase Source | **TIM6** (SysTick is reserved for FreeRTOS) |

### Step 3 — FreeRTOS Setup
- **Middleware & Software Packs** → **FreeRTOS** → Interface: **CMSIS_V2**
- In the **Tasks & Queues** tab, configure the default task as follows:

| Field | Value |
|-------|-------|
| Task Name | `Task_1` |
| Priority | `osPriorityNormal` |
| Entry Function | `Task1_function` |
| Stack Size | `128` words (default) |

### Step 4 — Enable Printf Float Support
- **Project Properties** → **C/C++ Build** → **Settings** → enable `printf` and `scanf` float formatting

### Step 5 — Code Generation
- Enter project name, select **STM32CubeIDE**, click **Generate Code**, and open the project

---

## Debugger and SWV Configuration

```
Open Debug icon --> Edit Debug Configurations
    |
    +-- Debugger tab:
    |     +-- ST-LINK --> click SCAN --> board serial number detected automatically
    |
    +-- Serial Wire Viewer (SWV):
          +-- Enable (checked)
          +-- Core Clock: 84.0 MHz   <-- must match your HCLK exactly
          +-- Apply --> Close
```

---

## Source Code — `main.c`

### Include Header
```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
/* USER CODE END Includes */
```

### Redirect `printf` to ITM Port 0
```c
/* USER CODE BEGIN 0 */
int _write(int file, char *ptr, int len) {
    for (int i = 0; i < len; i++) {
        ITM_SendChar(*ptr++);
    }
    return len;
}
/* USER CODE END 0 */
```

> `ITM_SendChar()` transmits characters one at a time through the **Instrumentation Trace Macrocell** — a built-in Cortex-M4 debug channel that requires **no UART pins**.

### Task Function Definition
```c
/* USER CODE END Header_Task1_function */
void Task1_function(void *argument) {
    /* USER CODE BEGIN 5 */
    for (;;) {
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
        printf("Task_1 Executing for LED Toggle \n");
        osDelay(500);   // Yield CPU for 500 ms; scheduler is free to run other tasks
    }
    /* USER CODE END 5 */
}
```

---

## Launching the SWV ITM Console

```
Step 1:  Window --> Show View --> Other --> SWV --> SWV ITM Data Console
Step 2:  Click Debug --> Switch to Debug perspective
Step 3:  In the SWV ITM Data Console:
           Enable PC Sampling (checked)
           Port 0 (checked)  <-- output channel for ITM_SendChar
Step 4:  Click Start Trace (red circle button)
Step 5:  Click Resume

Expected output in Port 0:
   Task_1 Executing for LED Toggle
   Task_1 Executing for LED Toggle
   Task_1 Executing for LED Toggle
   ...  (repeating every 500 ms)
```

---

## Observation Template

Record these values during the lab session:

| Parameter | Observed Value |
|-----------|---------------|
| LED blink period (ms) | |
| Console messages per second | |
| Compilation errors | |
| Compilation warnings | |
| Observation duration (min) | |
| Any timing drift noticed? | |

---

## FreeRTOS `osDelay` vs. Super Loop `HAL_Delay`

| Feature | `HAL_Delay(500)` | `osDelay(500)` |
|---------|-----------------|----------------|
| CPU during delay | Spinning (busy-wait) | Returned to scheduler |
| Other tasks can execute | No | Yes |
| Timing management | SysTick dependent | Managed by RTOS kernel |
| Safe inside ISR | Allowed | Not permitted |

---

## Reflection Questions

1. How does `osDelay(500)` differ conceptually from `HAL_Delay(500)` in a super loop context?
2. What advantages are apparent now with a single task, and what additional benefits emerge when a second task is added?
3. Did the SWV ITM output appear in sync with the LED toggling?
4. If no output appeared or the LED remained off, which configuration steps would you review first?

---

## Result

A single FreeRTOS task (`Task_1`) was created using the CMSIS-RTOS v2 API. The LED toggled at an accurate 500 ms interval driven by `osDelay()`, and correct execution was non-intrusively verified through the SWV ITM Data Console on Port 0 — marking the first successful step into RTOS-based embedded development.
