# Experiment 5 — Non-Blocking Super Loop Architecture

> No RTOS. No blocking delays. Just effective use of a millisecond timestamp.

---

## Objective

Implement a **non-blocking super loop** that runs three independent periodic tasks concurrently — LED toggling, button polling, and IR sensor sampling — on a single CPU core without any `HAL_Delay()` blocking calls.

---

## Components Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| IR Sensor module | 1 |
| Jumper wires | Several |
| USB Type-A to Mini-B cable | 1 |

---

## Core Concept — Foreground / Background Architecture

```
+-------------------------- FOREGROUND ---------------------------+
|  SysTick ISR fires every 1 ms                                  |
|  --> increments the internal HAL tick counter                  |
+----------------------------------------------------------------+
                            |
                     HAL_GetTick()
                            |
+-------------------------- BACKGROUND ---------------------------+
|                   while(1) SUPER LOOP                          |
|                                                                |
|  dt = now - last_time                                          |
|                                                                |
|  led_counter    += dt  --> fires every 500 ms --> LED toggle   |
|  button_counter += dt  --> fires every  50 ms --> btn poll     |
|  ir_counter     += dt  --> fires every 200 ms --> ADC read     |
+----------------------------------------------------------------+
```

### Task Timeline Visualization

```
Time (ms): 0   50  100 150 200 250 300 350 400 450 500
           |    |    |    |    |    |    |    |    |    |    |
LED        |-----------------------------------------------*    |  (500ms toggle)
Button     |----*----*----*----*----*----*----*----*----*------|  (50ms poll)
IR Sensor  |-----------*-----------*-----------*--------------|  (200ms ADC)
```

**Core idea:** Each task only *checks* whether its scheduled interval has elapsed — none of them *waits* for it.

---

## CubeMX Configuration

### Step 1 — Pin Assignments

| Pin | Label | Configuration |
|-----|-------|---------------|
| PA5 | LED LD2 | GPIO Output |
| PC13 | USER Button | GPIO Input, Pull-up |
| PA0 | IR Sensor | Analog (ADC1_IN0) |

### Step 2 — ADC1 Setup
- Enable **ADC1**, Channel **IN0** (PA0)
- Resolution: 12-bit (default) — raw values span 0 to 4095

### Step 3 — Clock Setup
- **RCC** → BYPASS, HCLK = **84 MHz**

### Step 4 — Code Generation
- Enter project name, select STM32CubeIDE, and click Generate Code

---

## Source Code — `main.c`

### Global Variables and Time Helper Function
```c
/* USER CODE BEGIN 0 */
volatile uint32_t led_counter_ms    = 0;
volatile uint32_t button_counter_ms = 0;
volatile uint32_t ir_counter_ms     = 0;
volatile uint32_t last_time_ms      = 0;

volatile uint8_t  button_pressed = 0;
volatile uint16_t ir_value       = 0;   // 12-bit ADC result

static inline uint32_t GetTimeMs(void) {
    return HAL_GetTick();   // SysTick-based, 1 ms resolution
}
/* USER CODE END 0 */
```

### Initialization Block (before while loop)
```c
/* USER CODE BEGIN 2 */
last_time_ms = GetTimeMs();
/* USER CODE END 2 */
```

### Super Loop Body
```c
/* USER CODE BEGIN WHILE */
while (1) {

    /* -- Compute elapsed time since previous iteration -- */
    uint32_t now_ms = GetTimeMs();
    uint32_t dt     = now_ms - last_time_ms;
    last_time_ms    = now_ms;

    led_counter_ms    += dt;
    button_counter_ms += dt;
    ir_counter_ms     += dt;

    /* -- Task 1: Toggle LED every 500 ms -- */
    if (led_counter_ms >= 500) {
        led_counter_ms = 0;
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    }

    /* -- Task 2: Sample Button every 50 ms -- */
    if (button_counter_ms >= 50) {
        button_counter_ms = 0;
        GPIO_PinState raw = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);
        if (raw == GPIO_PIN_RESET) {
            button_pressed = 1;
            HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET); // force LED ON
        } else {
            button_pressed = 0;
        }
    }

    /* -- Task 3: Read IR Sensor every 200 ms -- */
    if (ir_counter_ms >= 200) {
        ir_counter_ms = 0;
        HAL_ADC_Start(&hadc1);
        HAL_ADC_PollForConversion(&hadc1, 10);  // allow up to 10 ms
        if (HAL_ADC_GetState(&hadc1) & HAL_ADC_STATE_REG_EOC) {
            ir_value = (uint16_t)HAL_ADC_GetValue(&hadc1);
        }
        HAL_ADC_Stop(&hadc1);
    }

    // No HAL_Delay() here -- that would block all three tasks!
}
/* USER CODE END WHILE */
```

---

## Super Loop vs. Blocking Delays

| Approach | LED @ 500ms | Button @ 50ms | IR @ 200ms | CPU During Wait |
|----------|-------------|--------------|------------|-----------------|
| `HAL_Delay()` blocking | Yes | Stalled | Stalled | 100% busy-wait |
| **Super Loop (this exp)** | Yes | Yes | Yes | 0% (loop runs freely) |

---

## Observation Table

| S.No | Item | Status / Value | Remarks |
|------|------|---------------|---------|
| 1 | LED | | Toggles every 500 ms? |
| 2 | Button Pressed | | LED forced ON when held? |
| 3 | IR Sensor (`ir_value`) | | Value changes with proximity? |

---

## Running the Program

```
1. Connect the board via USB
2. Build --> OK --> Switch (debug perspective)
3. Add 'ir_value' and 'button_pressed' to the Watch window
4. Resume and observe variables updating live
```

---

## Suggested Modifications

| Modification | Code Change |
|-------------|-------------|
| Blink LED every 250 ms | `if (led_counter_ms >= 250)` |
| Poll button every 100 ms | `if (button_counter_ms >= 100)` |
| Stream ir_value over UART | Insert `sprintf` + `HAL_UART_Transmit` inside the IR task block |
| Add a 4th task (e.g., buzzer) | Declare `buzzer_counter_ms` and increment it the same way |

---

## Reflection Questions

1. In what specific ways does the super loop outperform `HAL_Delay()`-based timing in a multi-task scenario?
2. What role do the three dedicated software counters play in maintaining independent task periods?
3. What did you observe in `ir_value` as an object was brought closer to the IR sensor?
4. How do the PC13 pull-up and PA5 output configurations work together for button-driven LED control?
5. Under what conditions would a super loop become insufficient, necessitating a full RTOS instead?

---

## Result

The non-blocking super loop successfully executed three independent periodic tasks in parallel without any CPU stall, demonstrating that `HAL_GetTick()`-based software counters are a lightweight and efficient substitute for RTOS task scheduling in resource-limited embedded applications.
