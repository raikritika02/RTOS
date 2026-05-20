# Experiment 10 — Shared Resource Management with a Counting Semaphore

> A binary semaphore is a single-entry gate.
> A counting semaphore is a gated pool that permits multiple simultaneous entries.

---

## Objective

Model a **capacity-limited shared resource** using a FreeRTOS Counting Semaphore distributed across three competing tasks, and examine resource contention, concurrent access control, and task interleaving through the SWV ITM Data Console.

---

## Components Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |

---

## The Counting Semaphore — Token Pool Model

```
 Counting Semaphore (Max=2, Initial=2)

 +----------------------------------+
 |  Token Pool                      |
 |  +------+  +------+             |
 |  | [T]  |  | [T]  |             |  <-- 2 tokens available at startup
 |  +------+  +------+             |
 +----------------------------------+

 Task A acquires --> consumes 1 token --> count: 2->1
 Task B acquires --> consumes 1 token --> count: 1->0
 Task C acquires --> no tokens left   --> BLOCKED (must wait!)

 Task A releases --> returns 1 token  --> count: 0->1 --> Task C UNBLOCKS
```

---

## Comparing Synchronization Primitives

```
+------------------+----------+---------------+--------------------+
| Primitive        | Max Count| Best Use Case | Mutual Exclusion?  |
+------------------+----------+---------------+--------------------+
| Binary Semaphore |    1     | Signaling     | Yes (1 at a time)  |
| Mutex            |    1     | Shared data   | Yes + ownership    |
| Counting Semaphore|   N     | Resource pools| No (N at a time)   |
+------------------+----------+---------------+--------------------+
```

---

## What Task Interleaving Looks Like

With semaphore count = **2**, tasks A and B hold it simultaneously and both write to ITM at the same time, producing **garbled output**:

```
Expected (using a mutex):     Actual (counting semaphore, count=2):
  1AAAAAAAAAA                   1A2BAB2BBBAAAB1AAAB2BB
  2BBBBBBBBBB                   (characters interleaved -- unpredictable)
  3CCCCCCCCCC
```

This is **intentional, not a defect** — it precisely demonstrates that counting semaphores manage *how many* tasks may concurrently access a resource, not the *order or atomicity* of those accesses.

---

## CubeMX Configuration

### Step 1 — System Settings

| Setting | Value |
|---------|-------|
| RCC | BYPASS |
| HCLK | 84 MHz |
| SYS → Debug | Trace Asynchronous Sw |
| SYS → Timebase | **TIM6** |

### Step 2 — FreeRTOS (CMSIS_V2)

**Tasks & Queues tab — Three Tasks:**

| Field | TaskA | TaskB | TaskC |
|-------|-------|-------|-------|
| Task Name | `TaskA` | `TaskB` | `TaskC` |
| Entry Function | `func_TaskA` | `func_TaskB` | `func_TaskC` |
| Priority | `osPriorityNormal` | `osPriorityNormal` | `osPriorityNormal` |

**Timers & Semaphores tab — Counting Semaphore:**

| Field | Value |
|-------|-------|
| Semaphore Name | `myCountingSem01` |
| **Max Count** | **`2`** -- at most 2 tasks hold the semaphore simultaneously |
| **Initial Count** | **`2`** -- fully available at program start |

### Step 3 — Advanced Settings
- Enable **USE_NEWLIB_REENTRANT**

### Step 4 — Printf and SWV Setup
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

### Task A
```c
void func_TaskA(void *argument) {
    /* USER CODE BEGIN 5 */
    char ch = 'A';
    for (;;) {
        osSemaphoreAcquire(myCountingSem01Handle, osWaitForever); // Claim a token
        printf("1");
        for (int i = 0; i < 10; i++) {
            printf("%c", ch);   // Prints 'A' ten times
            HAL_Delay(50);
        }
        osSemaphoreRelease(myCountingSem01Handle);  // Return the token
        osDelay(5);
    }
    /* USER CODE END 5 */
}
```

### Task B
```c
void func_TaskB(void *argument) {
    /* USER CODE BEGIN func_TaskB */
    char ch = 'B';
    for (;;) {
        osSemaphoreAcquire(myCountingSem01Handle, osWaitForever);
        printf("2");
        for (int i = 0; i < 10; i++) {
            printf("%c", ch);   // Prints 'B' ten times
            HAL_Delay(50);
        }
        osSemaphoreRelease(myCountingSem01Handle);
        osDelay(5);
    }
    /* USER CODE END func_TaskB */
}
```

### Task C
```c
void func_TaskC(void *argument) {
    /* USER CODE BEGIN func_TaskC */
    char ch = 'C';
    for (;;) {
        osSemaphoreAcquire(myCountingSem01Handle, osWaitForever);
        printf("3");
        for (int i = 0; i < 10; i++) {
            printf("%c", ch);   // Prints 'C' ten times
            HAL_Delay(50);
        }
        osSemaphoreRelease(myCountingSem01Handle);
        osDelay(5);
    }
    /* USER CODE END func_TaskC */
}
```

---

## Running and Observing

```
1. Build --> Debug --> Switch to debug perspective
2. Window --> Show View --> SWV ITM Data Console
3. Port 0 --> Start Trace --> Resume

With count=2, two tasks hold the semaphore concurrently:
   1A2BABABABABAB2BBBBA1AABAABBA...
   (interleaved -- two tasks printing at the same time)

With count=1 (modified for comparison):
   1AAAAAAAAAA
   2BBBBBBBBBB
   3CCCCCCCCCC
   (clean sequential blocks -- one task at a time)
```

---

## Observation Table

| # | Query | Response |
|---|-------|----------|
| 1 | How many tasks can access the ITM trace resource simultaneously? | |
| 2 | Describe the character interleaving pattern seen in the trace output | |
| 3 | What type of scheduling produces the observed interleaving? | |
| 4 | What code change produces a different access pattern? | |

---

## Experiment Variations

| Modification | Expected Outcome |
|-------------|-----------------|
| Set semaphore count = **1** | Only 1 task prints at a time --> clean, readable output |
| Set semaphore count = **3** | All 3 tasks access simultaneously --> maximum interleaving |
| Skip `osSemaphoreRelease()` in one task | That task holds the token permanently -- others may starve |
| Raise one task to `osPriorityHigh` | That task acquires the token more frequently |
| Replace with a **mutex** | True mutual exclusion -- clean sequential output, one task at a time |

---

## Core Takeaway

```
Counting Semaphore (count=2):
     Governs HOW MANY tasks may enter a shared region simultaneously
     Does NOT enforce atomic or exclusive access per individual task

Mutex:
     Guarantees exactly ONE task at a time (true mutual exclusion)
     Carries ownership -- only the acquiring task may release it
     Not intended for task-to-task event signaling

Use counting semaphores for: connection pools, hardware unit pools, rate limiters
Use mutexes for: shared data structures, single shared peripherals (SPI, UART)
```

---

## Reflection Questions

1. What does "thread-safe" mean -- and why does a counting semaphore with count > 1 not fully guarantee it for a single ITM or UART port?
2. Implement a fast producer and slow consumer using the counting semaphore -- what happens when the count drops to 0?
3. Implement a slow producer and fast consumer -- what happens when the count reaches its maximum?
4. Write a summary of your key learnings about RTOS-based system design accumulated across all ten experiments in this series.

---

## Result

Three tasks competed for a shared resource (the ITM trace port) governed by a Counting Semaphore initialized to count = 2. The SWV output clearly revealed task interleaving — two tasks wrote simultaneously, generating irregular mixed character sequences. This confirmed that counting semaphores regulate resource **availability** rather than **atomicity**, drawing a clear distinction from mutexes in practical RTOS system design.
