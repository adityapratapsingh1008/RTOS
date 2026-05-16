# 🟢 EXPERIMENT 6 — FreeRTOS: First Task & SWV ITM Tracing

> Your first step into the RTOS world. One task. One LED. One console message. But an entirely new paradigm.

---

## 🎯 Aim

Create a single **FreeRTOS task** on STM32F446RE using CMSIS-RTOS v2 that toggles the onboard LED every 500 ms, and observe real-time execution via the **SWV ITM Data Console** in STM32CubeIDE.

---

## 🔧 Hardware Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |

---

## 🧩 How FreeRTOS Changes Everything

### Old Way (Super Loop)
```
main() → while(1) { toggle; HAL_Delay(500); toggle; ... }
         CPU is STUCK during HAL_Delay — can't do anything else
```

### New Way (FreeRTOS)
```
Scheduler starts
    │
    └─► Task_1 (RUNNING)
            │
            ├── HAL_GPIO_TogglePin()
            ├── printf("Task_1 Executing...")
            └── osDelay(500)  ◄── Task enters BLOCKED state
                                   CPU is FREE for other tasks
                    │
                   500ms later...
                    │
            Task_1 → READY → RUNNING again
```

### Task State Machine
```
              osThreadNew()
                   │
                   ▼
              ┌─────────┐
              │  READY  │◄──────────────────────┐
              └────┬────┘                       │
                   │  Scheduler selects it      │
                   ▼                            │
              ┌─────────┐                       │
              │ RUNNING │──── osDelay(500) ───► │
              └─────────┘                  ┌────┴────┐
                                           │ BLOCKED │
                                           └─────────┘
```

---

## ⚙️ CubeMX Configuration

### Step 1 — GPIO
- **PA5** → GPIO Output (LED LD2)

### Step 2 — System Settings

| Setting | Value |
|---------|-------|
| RCC | BYPASS Clock Source |
| HCLK | 84 MHz |
| **SYS → Debug** | **Trace Asynchronous Sw** |
| **SYS → Timebase Source** | **TIM6** *(not SysTick — FreeRTOS uses it)* |

### Step 3 — FreeRTOS
- **Middleware & Software Packs** → **FreeRTOS** → Interface: **CMSIS_V2**
- **Tasks & Queues** tab → configure the default task:

| Field | Value |
|-------|-------|
| Task Name | `Task_1` |
| Priority | `osPriorityNormal` |
| Entry Function | `Task1_function` |
| Stack Size | `128` words (default) |

### Step 4 — Printf Float Support
- **Project Properties** → **C/C++ Build** → **Settings** → check `printf` and `scanf` float support

### Step 5 — Generate Code
- Project name → **STM32CubeIDE** → **Generate Code** → Open Project

---

## 🛠️ Debugger / SWV Setup

```
Debug icon → Edit Debug Configurations
    │
    ├── Debugger tab:
    │     └── ST-LINK → click SCAN → board S/N auto-detected
    │
    └── Serial Wire Viewer (SWV):
          ├── ☑ Enable
          ├── Core Clock: 84.0 MHz   ← must match your clock config!
          └── Apply → Close
```

---

## 💻 Code — `main.c`

### Include
```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
/* USER CODE END Includes */
```

### Retarget `printf` → ITM Port 0
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

> `ITM_SendChar()` sends one character at a time through the **Instrumentation Trace Macrocell** — a built-in Cortex-M4 debug channel that requires **zero UART pins**.

### Task Function
```c
/* USER CODE END Header_Task1_function */
void Task1_function(void *argument) {
    /* USER CODE BEGIN 5 */
    for (;;) {
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
        printf("Task_1 Executing for LED Toggle \n");
        osDelay(500);   // Blocks task for 500ms; CPU is free
    }
    /* USER CODE END 5 */
}
```

---

## 🖥️ Launching the SWV ITM Console

```
Step 1:  Window → Show View → Other → SWV → SWV ITM Data Console
Step 2:  Click Debug button → Switch to Debug perspective
Step 3:  In SWV ITM Data Console:
           ☑ Enable PC Sampling
           ☑ Port 0  ← this is where ITM_SendChar(0) outputs
Step 4:  Click ● Start Trace (red dot)
Step 5:  Click ▶ Resume

Expected output on Port 0:
   Task_1 Executing for LED Toggle
   Task_1 Executing for LED Toggle
   Task_1 Executing for LED Toggle
   ...  (every 500 ms)
```

---

## 📊 Observation Template

Fill in during the lab session:

| Parameter | Observed Value |
|-----------|---------------|
| LED blink period (ms) | |
| Console messages per second | |
| Compilation errors | |
| Compilation warnings | |
| Observation duration (min) | |
| Any timing drift noticed? | |

---

## 🆚 FreeRTOS `osDelay` vs. Super Loop `HAL_Delay`

| Feature | `HAL_Delay(500)` | `osDelay(500)` |
|---------|-----------------|----------------|
| CPU during delay | Spinning (busy) | Released to scheduler |
| Other tasks can run | ❌ No | ✅ Yes |
| Timing accuracy | Depends on SysTick | Managed by RTOS kernel |
| Use in ISR | Allowed | ❌ Not allowed |

---

## ❓ Reflection Questions

1. How does `osDelay(500)` differ conceptually from `HAL_Delay(500)` in a super loop?
2. What advantages appear *now* with one task, and what more appears when you add a second?
3. What did you observe in the SWV ITM console — was the output synchronized with LED toggling?
4. If the LED didn't blink or no messages appeared, which configuration steps would you recheck first?

---

## ✅ Result

A single FreeRTOS task (`Task_1`) was created using the CMSIS-RTOS v2 interface. The LED toggled at a precise 500 ms interval driven by `osDelay()`, and real-time execution was verified non-intrusively via the SWV ITM Data Console on Port 0.
