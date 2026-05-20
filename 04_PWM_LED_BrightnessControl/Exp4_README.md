# Experiment 4 — PWM-Based LED Brightness Control

> Apply mathematics to govern light. A rapidly switching square wave with an adjustable ON-time translates into perceived analog brightness.

---

## Objective

Generate a PWM signal on **TIM2 Channel 1 (PA5)** of the STM32F446RE and drive the onboard LED through a smooth, continuous fade cycle from 0% through 100% and back to 0% duty cycle.

---

## Components Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |
| (No external components needed — the onboard LED on PA5 is used directly) | — |

---

## Anatomy of a PWM Signal

```
 Vmax |  ######        ######        ######
      |  #    #        #    #        #    #
  0V  |  #    ######## #    ######## #    ########
      +--+----+--------+----+--------+------------>  time
         |TON |  TOFF  |

   Duty Cycle  D = TON / T x 100%
   Average Voltage = Vmax x D / 100

   25% duty:  ##........  -->  faint glow
   50% duty:  #####.....  -->  mid brightness
   75% duty:  ########..  -->  near full brightness
  100% duty:  ##########  -->  maximum output
```

---

## Design Equations

```
PWM Frequency  =  f_CLK / (PSC + 1) / (ARR + 1)

With:  f_CLK = 84 MHz,  PSC = 839,  ARR = 1000
       = 84,000,000 / 840 / 1001 ~ 100 Hz

Duty Cycle (%) =  CCR / (ARR + 1) x 100

Example:   CCR = 500  -->  500/1000 x 100 = 50%
```

---

## CubeMX Configuration

### Step 1 — Pin Assignment
Right-click **PA5** and set mode to **TIM2_CH1** (Alternate Function)

### Step 2 — Clock Setup
- **RCC** → BYPASS Clock Source
- Set HCLK to **84 MHz**

### Step 3 — TIM2 Parameters

| Parameter | Value | Effect |
|-----------|-------|--------|
| Prescaler (PSC) | `840 - 1` | Timer frequency = 84M/840 = **100 kHz** |
| Counter Mode | `Up` | Counts from 0 up to ARR |
| Auto-Reload (ARR) | `1000` | PWM frequency ~ **100 Hz** |
| CH1 Mode | `PWM Generation CH1` | PA5 outputs the PWM signal |
| CH1 Pulse (CCR) | `0` (initial) | Starts at 0% duty cycle |

### Step 4 — Code Generation
- Enter project name, select **STM32CubeIDE**, and click Generate Code
- Under Properties → MCU Post Build: enable **.bin** and **.hex** output

---

## Source Code — `main.c`

### Start PWM Output (after `MX_TIM2_Init()`)
```c
/* USER CODE BEGIN 2 */
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
/* USER CODE END 2 */
```

### Breathing LED Loop (inside `while(1)`)
```c
/* USER CODE BEGIN WHILE */
while (1) {
    int x;

    /* -- Fade IN: 0% to 100% -- */
    for (x = 0; x < 1000; x++) {
        __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, x);
        HAL_Delay(1);   // 1 ms per step --> 1-second total ramp up
    }

    /* -- Fade OUT: 100% to 0% -- */
    for (x = 1000; x > 0; x--) {
        __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, x);
        HAL_Delay(1);
    }
}
/* USER CODE END WHILE */
```

---

## CCR Value to Brightness Mapping

```
  CCR    0  |....................   0%    LED completely OFF
  CCR  250  |####................  25%    faint glow
  CCR  500  |##########..........  50%    half brightness
  CCR  750  |###############.....  75%    approaching full brightness
  CCR 1000  |####################  100%   maximum brightness
```

---

## Observation Table

| CCR Value | Duty Cycle (%) | Brightness Level | Observation |
|-----------|---------------|-----------------|-------------|
| 0 | 0% | | |
| 250 | 25% | | |
| 500 | 50% | | |
| 750 | 75% | | |
| 1000 | 100% | | |

---

## Running the Program

```
1. Connect the board via USB
2. Build --> OK --> Switch (to debug perspective)
3. Resume -- observe the LED fade smoothly in and out continuously
```

---

## Suggested Modifications

| Modification | How to Apply | Expected Result |
|-------------|-------------|-----------------|
| Speed up the fade | Change `HAL_Delay(1)` to `HAL_Delay(0)` | LED cycles in roughly 0.1 s |
| Slow down the fade | Change `HAL_Delay(1)` to `HAL_Delay(5)` | LED cycles over roughly 5 s |
| Lock at 33% brightness | Remove loops; set `CCR = 333` directly | LED remains at a fixed dim level |
| Increase PWM frequency | Reduce `ARR` to 99 | Frequency increases tenfold |
| Decrease PWM frequency | Increase `ARR` to 9999 | Visible flicker near 10 Hz |

---

## Reflection Questions

1. What is PWM and how does adjusting duty cycle influence the average output voltage?
2. Calculate the PWM frequency when PSC = 179 and ARR = 499 on a 90 MHz clock.
3. What visual effect results if the PWM frequency drops to 10 Hz?
4. How do you obtain exactly 33% duty cycle with ARR = 999?
5. How would this technique be adapted to govern the rotational speed of a DC motor?

---

## Result

PWM output was successfully generated via TIM2 on the STM32F446RE. The onboard LED exhibited a smooth, repeating fade-in and fade-out cycle, demonstrating the direct relationship between the CCR register value (duty cycle) and perceived LED brightness through pulse-width modulation.
