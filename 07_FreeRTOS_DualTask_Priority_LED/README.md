# ⚖️ FreeRTOS Dual Tasks & Priority Scheduling

> Two tasks. Same delay. Different priorities. Completely different outcomes.

---

## 🎯 Aim

Create two FreeRTOS tasks with configurable priority levels and analyze how the **preemptive scheduler** allocates CPU time — demonstrated through LED blink rates and SWV ITM trace output.

---

## 🔧 Hardware Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| External LED | 1 |
| 220Ω Resistor | 1 |
| Breadboard + connecting wires | — |
| USB Type-A to Mini-B cable | 1 |

### External LED Wiring
```
PA6 ──── [220Ω] ──── LED(+) ──── LED(-) ──── GND
```

---

## 🧠 How FreeRTOS Priority Scheduling Works

### Equal Priority — Round Robin
```
Time:    0ms      500ms    1000ms   1500ms
         │         │         │         │
Task_1:  ▓▓░░░░░░░░▓▓░░░░░░░░▓▓░░░░░░░░   LED1 blinks @ 1Hz
Task_2:  ░░▓▓░░░░░░░░▓▓░░░░░░░░▓▓░░░░░░   LED2 blinks @ 1Hz

Both tasks block via osDelay(500) — CPU shared fairly.
```

### Unequal Priority — Preemption
```
Time:    0ms      500ms    1000ms
         │         │         │
Task_1:  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓   (HIGH priority — gets CPU first)
         (while blocking, Task_2 gets a chance)
Task_2:  ░░░░░▓░░░░░▓░░░░░▓░░░░░▓   LED2 blinks slower / starves
```

### Extreme Priority Difference — Starvation
```
Task_HIGH (Realtime): ████████████████████████████████
Task_LOW  (Low):      ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ (never runs!)
```

---

## ⚙️ CubeMX Configuration

### Step 1 — GPIO

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
- **Tasks & Queues** → create **two tasks**:

| Field | Task 1 | Task 2 |
|-------|--------|--------|
| Task Name | `LED_1` | `LED_2` |
| Entry Function | `Task1_function` | `StartLED_2` |
| Priority | *(vary as per table)* | *(vary as per table)* |
| Stack | 128 words | 128 words |

### Step 4 — Printf & SWV (same as Exp 6)
- Enable `printf`/`scanf` float in C/C++ Build Settings
- Debugger → SWV enabled, Core Clock = 84 MHz

---

## 💻 Code — `main.c`

### Include + ITM Retarget
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

### Task Functions
```c
/* Task 1 — controls LED on PA5 */
void Task1_function(void *argument) {
    /* USER CODE BEGIN 5 */
    for (;;) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        printf("Task_1 Executing for LED Toggle \n");
        osDelay(500);
    }
    /* USER CODE END 5 */
}

/* Task 2 — controls LED on PA6 */
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

## 📊 Observation Table — Run for Each Priority Combination

Change priorities in CubeMX Tasks & Queues tab, regenerate code, then observe:

| S.No | Task_1 Priority | Task_2 Priority | LED_1 Rate | LED_2 Rate | SWV Console Pattern | Remarks |
|------|----------------|----------------|-----------|-----------|---------------------|---------|
| 1 | `osPriorityLow` | `osPriorityLow` | | | | Both equal |
| 2 | `osPriorityNormal` | `osPriorityLow` | | | | Slight difference |
| 3 | `osPriorityHigh` | `osPriorityNormal` | | | | Clear difference |
| 4 | `osPriorityRealtime` | `osPriorityNormal` | | | | Starvation risk |
| 5 | `osPriorityError` | `osPriorityLow` | | | | Extreme case |

### What to Look For in SWV Console

```
Equal priorities:
  Task_1 Executing for LED Toggle
  Task_2 Executing for LED Toggle
  Task_1 Executing for LED Toggle
  Task_2 Executing for LED Toggle   ← alternates fairly

High vs Low:
  Task_1 Executing for LED Toggle
  Task_1 Executing for LED Toggle
  Task_1 Executing for LED Toggle
  Task_2 Executing for LED Toggle   ← Task_2 appears rarely
```

---

## 🔁 Running the Program

```
1. Set priorities in CubeMX → Regenerate Code
2. Update task functions (re-add code after regeneration)
3. Connect board → Build → Debug
4. Window → Show View → SWV ITM Data Console
5. Enable Port 0 → Start Trace ● → Resume ▶
6. Observe LED blink rates + console message frequency
7. Repeat for each row in the observation table
```

---

## 🔧 Modifications to Try

| Experiment | How |
|-----------|-----|
| Make Task_2 run more than Task_1 | Give Task_2 a higher priority |
| Demonstrate starvation | Set Task_1 to `osPriorityRealtime`, Task_2 to `osPriorityLow` — Task_2 LED should stop |
| Add a third task | Create `LED_3` on a third GPIO pin, set intermediate priority |
| Remove `osDelay` from Task_1 | Task_1 will spin indefinitely and starve Task_2 completely |

---

## ✅ Result

Two FreeRTOS tasks were created with configurable priorities using CMSIS-RTOS v2. Priority-based preemptive scheduling behavior was observed through LED blink rates and SWV ITM console output, demonstrating how higher-priority tasks dominate CPU access and can cause lower-priority task starvation.
