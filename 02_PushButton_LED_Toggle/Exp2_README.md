# Experiment 2 — GPIO Digital Input: Push Button Controlled LED Toggle

## Objective
Interface a push button as a digital input device and demonstrate LED state control by toggling it upon every confirmed button press.

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

This experiment extends the GPIO output knowledge from Experiment 1 and introduces **GPIO input**, where the onboard user button is used to control the onboard LED.

### GPIO Configured as Input
When a GPIO pin is placed in **digital input** mode, the microcontroller passively monitors the voltage on that pin and interprets it as logic `1` (HIGH) or `0` (LOW). Unlike output mode, the MCU does not drive the pin — it only reads the incoming signal.

### Board Pin Assignments

| Function | Arduino Label | MCU Pin | Default Logic Level |
|----------|---------------|---------|---------------------|
| User LED (LD2) | D13 | **PA5** | Output, Push-Pull |
| User Button (B1) | — | **PC13** | Input, Active LOW |

### Understanding Active-Low Button Logic
The onboard user button **B1** on the Nucleo-F446RE is connected to **PC13**, which has a hardware pull-up resistor already present on the board. This results in the following behavior:
- Button **not pressed** → PC13 reads **HIGH (1)**
- Button **pressed** → PC13 reads **LOW (0)**

This convention is referred to as **active-low** — the signal transitions to low when the intended event (button press) takes place.

### What is Contact Bounce?
Mechanical push buttons do not produce a single clean transition when actuated. When pressed, the metal contacts physically bounce multiple times before settling, creating a burst of rapid HIGH/LOW transitions. This is known as **contact bounce**, and it can cause the MCU to misinterpret a single press as several events.

A straightforward **software debounce** is implemented here — after a press is detected, a short `HAL_Delay(200)` pause allows the signal to stabilize before the next read cycle.

### HAL Functions Used

| Function | Purpose |
|----------|---------|
| `HAL_GPIO_ReadPin(GPIOx, GPIO_Pin)` | Returns the current logic level of an input pin (`0` or `1`) |
| `HAL_GPIO_TogglePin(GPIOx, GPIO_Pin)` | Inverts the current state of a GPIO output pin |
| `HAL_Delay(ms)` | Blocking millisecond pause — used here to debounce the button signal |

---

## STM32CubeMX Setup

### Step 1 — MCU Selection
- Open STM32CubeMX and go to **ACCESS TO MCU SELECTOR**
- Search for and select: **STM32F446RETx**
- Click **Start Project**

### Step 2 — GPIO Configuration

**PA5 — LED Output:**

| Parameter | Value |
|-----------|-------|
| GPIO Mode | Output Push Pull |
| GPIO Pull-up/Pull-down | No pull-up and no pull-down |
| Maximum Output Speed | Low |
| User Label | LED (optional) |

**PC13 — Push Button Input:**

| Parameter | Value |
|-----------|-------|
| GPIO Mode | Input mode |
| GPIO Pull-up/Pull-down | No pull-up and no pull-down |
| User Label | BTN (optional) |

> The Nucleo board already has a hardware pull-up resistor on PC13, so no internal pull-up needs to be set in CubeMX.

### Step 3 — Clock Source (RCC)
- Navigate to **System Core → RCC**
- High Speed Clock (HSE): **BYPASS Clock Source**

### Step 4 — Clock Configuration

| Clock Domain | Prescaler | Frequency |
|--------------|-----------|-----------|
| PLL Source | HSE | — |
| SYSCLK | PLL output | 180 MHz |
| AHB (HCLK) | /1 | 180 MHz |
| APB1 | /4 | 45 MHz |
| APB2 | /2 | 90 MHz |

### Step 5 — Project Manager
- Enter a **Project Name** (e.g., `Experiment_2`)
- Toolchain/IDE: **STM32CubeIDE**
- Click **Generate Code** then **Open Project**

### Step 6 — Post-Build Output Settings
- Right-click the project and navigate to **Properties → C/C++ Build → Settings → MCU Post Build Outputs**
- Enable **Convert to Binary File (.bin)**
- Enable **Convert to Intel Hex File (.hex)**
- Click **Apply and Close**

---

## Source Code

Open `Core/Src/main.c` and insert the following code inside the `while(1)` loop between the user code markers:

```c
/* USER CODE BEGIN WHILE */
while (1)
{
    // Read PC13 — Active LOW: a reading of 0 indicates a press
    if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == 0)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);   // Toggle LED state on PA5
        HAL_Delay(200);                           // Debounce settling pause
    }
    /* USER CODE END WHILE */
}
```

### Program Flow

```
Loop starts
    |
    v
Read PC13
    |
    |-- HIGH (button not pressed) --> skip --> loop again
    |
    +-- LOW (button held down)
            |
            v
        Toggle PA5 (LED changes state)
            |
            v
        Wait 200 ms  <- allows contact bounce to settle
            |
            v
        Loop again
```

> Keep all user code strictly between `USER CODE BEGIN` and `USER CODE END` markers to prevent overwriting during CubeMX regeneration.

---

## Build and Run

1. Connect the Nucleo board via USB
2. Click **Build** (hammer icon) and confirm **0 errors**
3. Click **Debug**, then click **Switch** on the perspective dialog
4. Click **Resume (▶)** to begin execution
5. Press the blue **B1 user button** on the board — the green LED **LD2** should toggle with each press

---

## Recorded Observations

### Button Press vs. LED State

| Trial No. | Button Action | PC13 State | LED State Before | LED State After | Remark |
|-----------|---------------|------------|------------------|-----------------|--------|
| 1 | No press (initial) | HIGH | OFF | OFF | No change |
| 2 | 1st Press | LOW | OFF | ON | LED turns ON |
| 3 | Release | HIGH | ON | ON | LED stays ON |
| 4 | 2nd Press | LOW | ON | OFF | LED turns OFF |
| 5 | 3rd Press | LOW | OFF | ON | LED turns ON again |
| 6 | Rapid Press | Bouncing | Varies | Toggles | LED flickers at high speed |

**Key observations:**
- The LED state is **latched** — it retains the new value even after the button is released.
- Every confirmed press triggers exactly one state change.
- Very rapid repeated presses may cause flickering since the 200 ms window can be insufficient.

---

## Result

For every valid button press, the LED on PA5 toggled exactly once and held its state after the button was released. This confirms correct configuration of **PC13 as a digital input** and **PA5 as a digital output**, with functional software-level debounce implemented via `HAL_Delay()`.

---

## Key Takeaways

- A microcontroller has no built-in notion of a "button press" as an event — it only samples binary logic levels at a given instant. The developer is responsible for interpreting signal transitions meaningfully.
- **Active-low logic** is a widely adopted convention in embedded hardware. Always refer to the board schematic to determine the idle-state polarity of any input signal.
- **Contact bounce** is an inherent physical characteristic of mechanical switches. A single press may generate dozens of spurious transitions within microseconds. A delay-based debounce is the simplest countermeasure, though timer or state-machine approaches offer greater robustness.
- The LED in this experiment is **stateful** — unlike the periodic blink in Experiment 1, it retains and holds its last known condition. This concept underpins latches and toggles in real-world embedded control applications.
- Combining GPIO input and output in one application creates a stimulus-response relationship — the foundational building block of embedded control systems.

---

## Project Layout

```
02_PushButton_LED_Toggle/
├── Core/
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       └── main.c          <- User code goes here (while loop)
├── Drivers/
│   └── STM32F4xx_HAL_Driver/
├── Experiment_2.ioc        <- CubeMX pin and clock settings file
└── README.md               <- This document
```
