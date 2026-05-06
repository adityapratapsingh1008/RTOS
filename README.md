#  STM32F446RE — Firmware Development using FreeRTOS

This repository documents my hands-on projects with the **STM32F446RE Nucleo Board**, progressing from bare-metal GPIO programming to real-time operating system concepts using **FreeRTOS**, **STM32CubeIDE** and **STM32CubeMX**.

Each folder represents a standalone project focused on a specific peripheral, concept, or RTOS feature of the STM32 microcontroller.

---

##  Topics Covered

- GPIO Digital Output & LED Control
- Digital Input with Push Buttons
- Ultrasonic Sensor Interfacing (HC-SR04)
- PWM Signal Generation & Duty Cycle Control
- Super Loop Architecture & Timing Counters
- FreeRTOS Task Creation & Scheduling
- FreeRTOS Task Priorities & Preemption
- External Interrupts (EXTI) with RTOS Synchronization
- Inter-Task Communication using Queues
- Shared Resource Management with Counting Semaphores

---

##  Project Structure

| Folder | Experiment | Description |
|--------|-----------|-------------|
| `01_GPIO_LED_Blink_SoftwareDelay` | GPIO Digital Output — LED Blink | Configure a GPIO pin as digital output and verify LED blinking using software delay routines |
| `02_PushButton_LED_Toggle` | Digital Input — Push Button LED Toggle | Interface a push button as digital input and toggle LED state on each valid button press |
| `03_HCSR04_Ultrasonic_LED_RangeIndicator` | HC-SR04 Ultrasonic Sensor — Distance Classification | Interface the HC-SR04 ultrasonic sensor with STM32F446RE and classify distance ranges using LED visual indication |
| `04_PWM_LED_BrightnessControl` | PWM Generation — LED Brightness Control | Generate a PWM signal using a timer and control onboard LED brightness by varying the duty cycle |
| `05_SuperLoop_LED_Button_Sensor` | Super Loop Architecture — Multi-Peripheral Polling | Implement a super loop-based embedded program that sequentially handles LED blinking, button reading, and sensor acquisition using timing counters |
| `06_FreeRTOS_SingleTask_LED_Blink` | FreeRTOS Basics — Single Task LED Blink | Develop a basic FreeRTOS project in STM32CubeIDE and validate LED blinking using a single RTOS task |
| `07_FreeRTOS_DualTask_Priority_LED` | FreeRTOS Task Priorities — Dual Task Analysis | Create and execute two FreeRTOS tasks with different priorities and analyze their effect on LED blinking behavior |
| `08_FreeRTOS_EXTI_Semaphore_LED` | FreeRTOS Synchronization — EXTI + Binary Semaphore | Configure an external interrupt (EXTI) for a user button and use a binary semaphore to synchronize an LED control task |
| `09_FreeRTOS_Queue_Sensor_UART` | FreeRTOS Inter-Task Communication — Queue + UART | Implement inter-task communication using a FreeRTOS queue where one task acquires sensor data and another transmits it over UART |
| `10_FreeRTOS_CountingSemaphore_SharedResource` | FreeRTOS Resource Management — Counting Semaphore | Model a limited shared resource using a FreeRTOS counting semaphore and study access control when multiple tasks request the resource simultaneously |

---

##  Development Environment

| Tool | Details |
|------|---------|
| **Board** | STM32F446RE Nucleo |
| **IDE** | STM32CubeIDE |
| **RTOS** | FreeRTOS (via STM32Cube middleware) |
| **Language** | C |
| **Flashing** | STM32CubeIDE built-in / STM32CubeProgrammer |

---

##  Experiment Progression

```
GPIO Output → Button Input → Ultrasonic Sensor → PWM
       ↓
   Super Loop (Bare-Metal Multi-Peripheral)
       ↓
FreeRTOS Single Task → Dual Task Priorities → EXTI + Semaphore
       ↓
   Queue-based Sensor→UART Pipeline → Counting Semaphore
```

---

##  Notes

- Experiments 1–5 follow a **bare-metal / HAL-based** approach with polling and software delays.
- Experiments 6–10 progressively introduce **FreeRTOS** concepts, building on each other.
- Each folder contains the full STM32CubeIDE project with source code and configuration files.
- This repository serves as a personal learning reference for embedded systems and RTOS programming.

Contributions, suggestions, and improvements are welcome!
