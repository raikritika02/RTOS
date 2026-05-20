# Experiment 3 — HC-SR04 Ultrasonic Distance Measurement

```
+------------------------------------------------------------------+
|          STM32F446RE  <---------->  HC-SR04 SENSOR              |
|                                                                  |
|   PA8 (TRIG) ----------------------------------------> TRIG    |
|   PA9 (ECHO) <---------------------------------------- ECHO    |
|   3.3V  ---------------------------------------> VCC            |
|   GND   ---------------------------------------> GND            |
|                                                                  |
|   PA5  --> [LED LD2]    PA2/PA3 --> USART2 Terminal             |
+------------------------------------------------------------------+
```

**Objective:** Interface an HC-SR04 ultrasonic sensor with the STM32F446RE, compute distance from time-of-flight measurements, and signal proximity thresholds using the onboard LED and UART output.

---

## Components Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| HC-SR04 Ultrasonic Sensor | 1 |
| Jumper wires | Several |
| USB Type-A to Mini-B cable | 1 |

---

## Operating Principle

The HC-SR04 measures distance by tracking the **travel time** of ultrasonic pulses:

```
 STM32              HC-SR04
   |                  |
   |-- 10us HIGH ---> TRIG
   |                  | (emits 8 ultrasonic pulses at 40 kHz)
   |                  |
   |<--- ECHO HIGH --- (echo pulse width proportional to distance)
   |                  |
   |-- ECHO LOW ------+

  d = (v x dt) / 2
    where v ~ 343 m/s = 0.0343 cm/us
```

TIM4 is set up with **Prescaler = 83** (84 - 1) so that at an 84 MHz system clock:

```
Timer tick = 84 MHz / 84 = 1 MHz  -->  1 tick = 1 us
```

---

## CubeMX Configuration

### Step 1 — Pin Assignments

| Pin | Label | Configuration |
|-----|-------|---------------|
| PA5 | LED | GPIO Output |
| PC13 | USER Button | GPIO Input |
| PA8 | TRIG | GPIO Output |
| PA9 | ECHO | GPIO Input |
| PA2 | USART2_TX | Alternate Function |
| PA3 | USART2_RX | Alternate Function |

### Step 2 — Clock Setup
- **RCC** → BYPASS Clock Source
- **Clock Configuration** → set HCLK to **84 MHz**

### Step 3 — Timer (TIM4)
- Enable **TIM4** with Internal Clock
- **Prescaler:** `84 - 1` = 83
- **Counter Period (ARR):** leave at default (dynamically modified at runtime)
- Resulting resolution: **1 us per tick**

### Step 4 — USART2
- Mode: **Asynchronous**
- Baud Rate: 115200 (default)

### Step 5 — Code Generation
- Enter the project name, select **STM32CubeIDE**, and click **Generate Code**
- Under Properties → C/C++ Build → Settings → MCU Post Build: enable **Convert to Binary** and **Convert to Hex**

---

## Source Code — `main.c`

### Includes and Macro Definitions
```c
/* USER CODE BEGIN Includes */
#include <string.h>
#include <stdio.h>
/* USER CODE END Includes */

/* USER CODE BEGIN PTD */
#define usTIM TIM4
/* USER CODE END PTD */
```

### Function Prototype and Global Variables
```c
/* USER CODE BEGIN PFP */
void usDelay(uint32_t uSec);
/* USER CODE END PFP */

/* USER CODE BEGIN 0 */
const float speedOfSound = 0.0343 / 2;  // cm/us, halved for round-trip
float distance;
char uartBuf[100];
/* USER CODE END 0 */
```

### Local Variable Declaration (before while loop)
```c
/* USER CODE BEGIN 1 */
uint32_t numTicks = 0;
/* USER CODE END 1 */
```

### Main Loop Logic
```c
/* USER CODE BEGIN 3 */
// 1. Pull TRIG low to ensure a clean start
HAL_GPIO_WritePin(TRIG_GPIO_Port, TRIG_Pin, GPIO_PIN_RESET);
usDelay(3);

// 2. Send a 10 us trigger pulse
HAL_GPIO_WritePin(TRIG_GPIO_Port, TRIG_Pin, GPIO_PIN_SET);
usDelay(10);
HAL_GPIO_WritePin(TRIG_GPIO_Port, TRIG_Pin, GPIO_PIN_RESET);

// 3. Wait until ECHO transitions HIGH
while (HAL_GPIO_ReadPin(ECHO_GPIO_Port, ECHO_Pin) == GPIO_PIN_RESET);

// 4. Count elapsed us while ECHO remains HIGH
numTicks = 0;
while (HAL_GPIO_ReadPin(ECHO_GPIO_Port, ECHO_Pin) == GPIO_PIN_SET) {
    numTicks++;
    usDelay(2);   // each iteration takes approximately 2.8 us
}

// 5. Derive distance from tick count
distance = (numTicks + 0.0f) * 2.8 * speedOfSound;

// 6. Send measurement over UART
sprintf(uartBuf, "Distance (cm) = %.1f\r\n", distance);
HAL_UART_Transmit(&huart2, (uint8_t *)uartBuf, strlen(uartBuf), 100);
HAL_Delay(1000);

// 7. Proximity indicator via LED
if (distance <= 20.0f)
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);   // ON
else
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET); // OFF
/* USER CODE END 3 */
```

### Microsecond Delay Function
```c
/* USER CODE BEGIN 4 */
void usDelay(uint32_t uSec) {
    if (uSec < 2) uSec = 2;
    usTIM->ARR = uSec - 1;   // Load the target count value
    usTIM->EGR = 1;           // Reinitialize counter and prescaler
    usTIM->SR  &= ~1;         // Clear the update event flag
    usTIM->CR1 |= 1;          // Start the counter
    while ((usTIM->SR & 0x0001) != 1); // Spin until overflow occurs
    usTIM->SR &= ~(0x0001);
}
/* USER CODE END 4 */
```

> **Why direct register access?** `HAL_Delay()` only provides 1 ms resolution — far too imprecise for the microsecond-level timing this sensor requires.

---

## Observation Table

| S.No | Actual Distance (cm) | UART Reading (cm) | LED Status | Echo Pulse Width (us) | Remarks |
|------|---------------------|-------------------|------------|----------------------|---------|
| 1 | 5 | | | | |
| 2 | 10 | | | | |
| 3 | 15 | | | | |
| 4 | 20 | | | | |
| 5 | 25 | | | | |

---

## Running the Program

```
1. Connect the board via USB
2. Build --> click OK --> click Switch (enters debug perspective)
3. Click Resume to begin execution
4. Open a serial terminal at 115200 baud to view live distance readings
```

---

## Reflection Questions

1. Why is TIM4's prescaler set to `84 - 1`?
2. How do the `ARR`, `EGR`, `SR`, and `CR1` registers interact inside `usDelay()`?
3. What is the purpose of calling `usDelay(2)` within the echo pulse measurement loop?
4. What does the guard `if(uSec < 2) uSec = 2` protect against?
5. How could this setup be extended to build a simple obstacle-avoidance robot?

---

## Result

The HC-SR04 ultrasonic sensor was successfully interfaced with the STM32F446RE. Distance values were computed using `d = (v x dt) / 2` and transmitted to a terminal over UART. The onboard LED activated whenever the measured distance dropped below the 20 cm threshold, confirming correct sensor integration and proximity detection behavior.
