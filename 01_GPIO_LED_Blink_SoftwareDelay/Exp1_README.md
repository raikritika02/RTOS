# Experiment 1 — GPIO Digital Output: LED Blink Using Software Delay

## Objective
Configure a GPIO pin on the **STM32F446RE** microcontroller to function as a digital output and verify LED blinking behavior using software-based delay routines.

---

## Components Required

| Component | Details |
|-----------|---------|
| Microcontroller Board | STM32 Nucleo-F446RE |
| Cable | USB Type-A to Mini-B |
| IDE | STM32CubeIDE |
| Configuration Tool | STM32CubeMX (integrated within CubeIDE) |

---

## Background Theory

The **STM32F446RE** microcontroller provides multiple general-purpose I/O (GPIO) ports — **GPIOA through GPIOH** — each of which can be independently assigned to one of four operating modes:

| Mode | Description |
|------|-------------|
| **Input** | Captures an incoming digital signal |
| **Output** | Drives a digital HIGH or LOW level |
| **Alternate Function** | Delegated to on-chip peripherals (UART, SPI, I2C, etc.) |
| **Analog** | Linked to internal ADC/DAC circuitry |

### Why Pin PA5?
On the **Nucleo-F446RE** development board, the green user LED (**LD2**) is physically wired to **Arduino header pin D13**, which maps to microcontroller pin **PA5**. No external LED or additional wiring is necessary — simply configuring PA5 as an output directly drives LD2.

### Push-Pull Output Mode
PA5 is configured in **push-pull output** mode. In this mode, the internal driver circuit can:
- Actively pull the pin **HIGH** (~3.3 V) — LED switches **ON**
- Actively pull the pin **LOW** (0 V) — LED switches **OFF**

This differs from open-drain mode, which can only pull the signal low and relies on an external pull-up resistor to restore the high state.

### HAL Functions Used

| Function | Purpose |
|----------|---------|
| `HAL_GPIO_TogglePin(GPIOx, GPIO_Pin)` | Reverses the current logic state of a specified GPIO pin |
| `HAL_Delay(ms)` | Halts execution for a defined number of milliseconds using the SysTick timer |

---

## STM32CubeMX Setup

### Step 1 — MCU Selection
- Open STM32CubeMX and select **ACCESS TO MCU SELECTOR**
- Search for and choose: **STM32F446RETx**
- Click **Start Project**

### Step 2 — GPIO Configuration
- Navigate to **System Core → GPIO**
- Right-click pin **PA5** and select **GPIO_Output**
- Apply the following GPIO settings:

| Parameter | Value |
|-----------|-------|
| GPIO Mode | Output Push Pull |
| GPIO Pull-up/Pull-down | No pull-up and no pull-down |
| Maximum Output Speed | Low |
| User Label | LD2 (optional) |

### Step 3 — Clock Source (RCC)
- Go to **System Core → RCC**
- Set High Speed Clock (HSE) to: **BYPASS Clock Source**
- This enables the board to use the external clock signal provided by the ST-Link interface

### Step 4 — Clock Configuration

| Clock Domain | Prescaler | Frequency |
|--------------|-----------|-----------|
| PLL Source | HSE | — |
| SYSCLK | PLL multiplier | 180 MHz |
| AHB (HCLK) | /1 | 180 MHz |
| APB1 | /4 | 45 MHz |
| APB2 | /2 | 90 MHz |

### Step 5 — Project Manager
- Enter a **Project Name** (e.g., `Experiment_1`)
- Set Toolchain/IDE to: **STM32CubeIDE**
- Click **Generate Code** then **Open Project**

### Step 6 — Post-Build Output Settings
- Right-click the project and go to **Properties → C/C++ Build → Settings → MCU Post Build Outputs**
- Enable **Convert to Binary File (.bin)**
- Enable **Convert to Intel Hex File (.hex)**
- Click **Apply and Close**

---

## Source Code

After code generation, open `Core/Src/main.c` and insert the following lines inside the `while(1)` loop between the designated user code markers:

```c
/* USER CODE BEGIN WHILE */
while (1)
{
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);   // Toggle LED LD2 on PA5
    HAL_Delay(500);                           // Hold for 500 ms

    /* USER CODE END WHILE */
}
```

> Always write code between `USER CODE BEGIN` and `USER CODE END` markers. Code placed outside these boundaries will be **removed** upon CubeMX code regeneration.

---

## Build and Run

1. Connect the Nucleo board via USB
2. Click **Build** (hammer icon) and verify **0 errors, 0 warnings**
3. Click **Debug**, then click **Switch** when prompted to change perspective
4. Click **Resume (▶)** to begin program execution
5. The onboard LED LD2 should begin blinking at the configured rate

---

## Recorded Observations

The delay parameter was modified across multiple trials to examine its effect on blink rate:

| S. No. | GPIO Pin | Mode / Pull | Delay (ms) | Approx. Blink Period (s) | Comment |
|--------|----------|-------------|------------|--------------------------|---------|
| 1 | PA5 | Output, PP, No PU/PD | 100 | 0.1 s | Very rapid — barely visible to the eye |
| 2 | PA5 | Output, PP, No PU/PD | 200 | 0.3 s | Moderate speed, clearly perceptible |
| 3 | PA5 | Output, PP, No PU/PD | 500 | 1 s | Comfortable pace, easily observable |
| 4 | PA5 | Output, PP, No PU/PD | 1000 | 2 s | Slow — ON and OFF phases distinctly visible |

> **Note:** The total blink period is roughly 2 times the delay value, since each ON and OFF phase independently consumes one complete delay interval.

---

## Result

GPIO pin **PA5** on the STM32F446RE Nucleo board was successfully configured as a **push-pull digital output** using STM32CubeIDE. The onboard user LED **LD2** blinked correctly at every configured rate. Modifying the delay value produced a proportional and clearly visible change in blink frequency, confirming correct GPIO output behavior and system clock configuration.

---

## Key Takeaways

- GPIO pins are multi-purpose — the same physical pin can act as an input, output, alternate function, or analog connection, determined entirely by software configuration.
- **Push-pull mode** actively controls both the HIGH and LOW states of the pin, making it suitable for direct LED driving without additional components.
- `HAL_Delay()` is a **blocking function** built on the SysTick timer. During the wait period, the CPU is completely occupied and cannot execute any other code — a fundamental constraint of bare-metal super loop programs.
- The **accuracy of `HAL_Delay()`** depends on a correctly configured SYSCLK. Any clock misconfiguration will result in inaccurate delay durations.
- The **super loop (`while(1)`)** is the most elementary program structure in embedded software — linear, single-task, and running indefinitely.
- Blink period = 2 × delay, because one full ON-OFF cycle requires two delay durations.

---

## Project Layout

```
01_GPIO_LED_Blink_SoftwareDelay/
├── Core/
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       └── main.c          <- User code goes here (while loop)
├── Drivers/
│   └── STM32F4xx_HAL_Driver/
├── Experiment_1.ioc        <- CubeMX pin and clock settings file
└── README.md               <- This document
```
