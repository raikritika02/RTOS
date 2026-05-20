# STM32F446RE — Embedded Firmware Development with FreeRTOS

This repository documents a structured series of hands-on experiments with the **STM32F446RE Nucleo Board**, progressing from foundational bare-metal GPIO programming through to real-time operating system implementation using **FreeRTOS**, **STM32CubeIDE**, and **STM32CubeMX**.

Each folder in this repository represents a self-contained project targeting a specific peripheral, architectural concept, or RTOS feature of the STM32 microcontroller platform.

---

## Topics Covered

- GPIO Digital Output and LED Blink Control
- Digital Input Interfacing with Push Buttons
- Ultrasonic Distance Sensing using HC-SR04
- PWM Signal Generation and Duty Cycle Control
- Non-Blocking Super Loop Design with Software Timing Counters
- FreeRTOS Task Creation and Scheduler Fundamentals
- FreeRTOS Task Priorities and Preemptive Scheduling
- External Interrupt Handling (EXTI) with RTOS Synchronization
- Inter-Task Data Transfer using Message Queues
- Shared Resource Access Control with Counting Semaphores

---

## Project Structure

| Folder | Experiment | Description |
|--------|------------|-------------|
| `01_GPIO_LED_Blink_SoftwareDelay` | GPIO Digital Output — LED Blink | Configure a GPIO pin as a push-pull digital output and validate LED blinking behavior using software delay routines |
| `02_PushButton_LED_Toggle` | Digital Input — Push Button LED Toggle | Interface the onboard push button as a digital input and toggle the LED state on every confirmed button press |
| `03_HCSR04_Ultrasonic_LED_RangeIndicator` | HC-SR04 Ultrasonic Sensor — Distance Classification | Interface an HC-SR04 ultrasonic sensor with the STM32F446RE and indicate distance ranges using LED visual feedback and UART output |
| `04_PWM_LED_BrightnessControl` | PWM Generation — LED Brightness Control | Generate a timer-based PWM signal and control onboard LED brightness by continuously sweeping the duty cycle |
| `05_SuperLoop_LED_Button_Sensor` | Super Loop Architecture — Multi-Peripheral Polling | Build a non-blocking super loop that concurrently manages LED toggling, button polling, and IR sensor sampling using independent software timing counters |
| `06_FreeRTOS_SingleTask_LED_Blink` | FreeRTOS Basics — Single Task LED Blink | Create a minimal FreeRTOS project in STM32CubeIDE and verify periodic LED toggling using a single RTOS task with SWV ITM tracing |
| `07_FreeRTOS_DualTask_Priority_LED` | FreeRTOS Task Priorities — Dual Task Analysis | Configure two FreeRTOS tasks with varying priority levels and study the effect of preemptive scheduling on individual LED blink rates |
| `08_FreeRTOS_EXTI_Semaphore_LED` | FreeRTOS Synchronization — EXTI and Binary Semaphore | Set up an external interrupt on the user button and employ a binary semaphore to defer LED control work to a dedicated RTOS task |
| `09_FreeRTOS_Queue_Sensor_UART` | FreeRTOS Inter-Task Communication — Queue and UART | Implement a producer-consumer pipeline using a FreeRTOS message queue, where one task generates sensor data and another receives and processes it |
| `10_FreeRTOS_CountingSemaphore_SharedResource` | FreeRTOS Resource Management — Counting Semaphore | Model a capacity-limited shared resource using a FreeRTOS counting semaphore and observe concurrent access behavior across three competing tasks |

---

## Development Environment

| Tool | Details |
|------|---------|
| **Board** | STM32F446RE Nucleo |
| **IDE** | STM32CubeIDE |
| **RTOS** | FreeRTOS (via STM32Cube middleware) |
| **Language** | C |
| **Flashing** | STM32CubeIDE built-in debugger / STM32CubeProgrammer |

---

## Experiment Progression

```
GPIO Output --> Button Input --> Ultrasonic Sensor --> PWM
       |
   Super Loop (Bare-Metal Multi-Peripheral Polling)
       |
FreeRTOS Single Task --> Dual Task Priorities --> EXTI + Semaphore
       |
   Queue-based Producer-Consumer Pipeline --> Counting Semaphore
```

---

## Notes

- Experiments 1 through 5 follow a **bare-metal, HAL-based** approach built around polling loops and software delays.
- Experiments 6 through 10 progressively introduce **FreeRTOS** concepts, with each experiment building directly on the previous one.
- Every project folder includes the complete STM32CubeIDE project along with source code and the CubeMX configuration file.
- This repository serves as a structured personal reference for embedded systems development and RTOS-based firmware design.

Contributions, suggestions, and improvements are welcome.
