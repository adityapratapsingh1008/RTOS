# 🔔 EXPERIMENT 8 — External Interrupt + Binary Semaphore

> Don't do work in an ISR. Signal a task to do it instead.

---

## 🎯 Aim

Configure an **EXTI interrupt** on the user button (PC13) and use a **Binary Semaphore** to synchronize an LED control task — implementing the **Deferred Interrupt Processing** pattern.

---

## 🔧 Hardware Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |

---

## 🧠 Core Concept — Deferred Interrupt Processing

### ❌ Naive Approach (Do Work in ISR)
```
Button pressed
    │
    ▼
ISR fires → blink LED 5 times (2.5 seconds of HAL_Delay inside ISR!)
              ↑
              THIS BLOCKS ALL OTHER INTERRUPTS — dangerous!
```

### ✅ Correct RTOS Approach (This Experiment)
```
Button pressed
    │
    ▼
ISR fires → osSemaphoreRelease()   ← ISR does ONE fast operation
    │                                 (microseconds)
    │
    ▼
Scheduler sees LED_Control task is now READY
    │
    ▼
LED_Control task UNBLOCKS → acquires semaphore → blinks LED 5×
    (normal task context — HAL_Delay is fine here)
```

---

## 📡 Full Sequence Diagram

```
USER        BUTTON HW      EXTI/NVIC      SEMAPHORE      LED_Control TASK
 │               │               │               │               │
 │── press ──►   │               │               │               │
 │           falling             │               │               │
 │           edge  ──────────►   │               │               │
 │                           IRQ fires           │               │
 │                               │──  Release ──►│  count: 0→1   │
 │                               │               │               │
 │                               │               │  task unblocks│
 │                               │               │◄── Acquire ───│ count: 1→0
 │                               │               │               │
 │                               │               │          blink×5
 │                               │               │         (10 toggles
 │                               │               │          × 250ms)
 │                               │               │               │
 │                               │               │          blocks again
 │                               │               │          (waits for next
 │                               │               │           semaphore)
```

---

## ⚙️ CubeMX Configuration

### Step 1 — GPIO

| Pin | Label | Mode |
|-----|-------|------|
| PA5 | LED LD2 | GPIO Output |
| PC13 | USER Button | **GPIO_EXTI13** |

### Step 2 — EXTI / Interrupt Setup
1. **GPIO → PC13** → Detection: **Falling Edge Trigger**
2. **GPIO → NVIC tab** → enable: `EXTI line[15:10] interrupt` ☑
3. **System Core → NVIC** → set EXTI[15:10] preemption priority to **7**

> ⚠️ FreeRTOS requires NVIC priorities for any ISR using FreeRTOS API calls to be **≥ configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY** (typically 5 or higher numerically). Priority 7 is safe.

### Step 3 — System Settings

| Setting | Value |
|---------|-------|
| RCC | BYPASS |
| HCLK | 84 MHz |
| SYS → Debug | Trace Asynchronous Sw |
| SYS → Timebase | **TIM6** |

### Step 4 — FreeRTOS (CMSIS_V2)
- **Tasks & Queues** → rename default task to `LED_Control`
- **Timers & Semaphores** → **Add** Binary Semaphore:

| Field | Value |
|-------|-------|
| Semaphore Name | `myBinarySem01` |
| Initial Count | `1` *(will be changed to 0 in code — see below)* |

### Step 5 — Printf / SWV
- Enable `printf`/`scanf` in C/C++ Build Settings
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

### LED Control Task
```c
void StartDefaultTask(void *argument) {
    uint8_t i;
    for (;;) {
        /* Block here until semaphore is released by button ISR */
        if (osSemaphoreAcquire(myBinarySem01Handle, 100) == osOK) {
            printf("Inside LEDControl Task\n");
            i = 0;
            while (i < 10) {                         // 10 toggles = 5 blinks
                HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
                HAL_Delay(250);                      // 250ms ON + 250ms OFF
                i++;
            }
        }
        /* Loop back and block again — task is dormant until next press */
    }
}
```

### Button ISR (EXTI Callback)
```c
/* Called automatically by HAL when EXTI fires on PC13 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    osSemaphoreRelease(myBinarySem01Handle);  // Signal the LED task
}
```

### ⚠️ CRITICAL — Fix Semaphore Initial Count

Find this auto-generated line in `main.c` and change the **second argument from `1` to `0`**:

```c
// ❌ Auto-generated (wrong — task would immediately run without button press):
myBinarySem01Handle = osSemaphoreNew(1, 1, &myBinarySem01_attributes);

// ✅ Required change (task blocks on startup, waits for button):
myBinarySem01Handle = osSemaphoreNew(1, 0, &myBinarySem01_attributes);
//                                      ↑
//                               Initial count = 0 → task starts BLOCKED
```

> **⚠️ Repeat this change every time you regenerate code from CubeMX.**

---

## 🖥️ Running & Observing

```
1. Build → Debug → Switch perspective
2. Window → Show View → SWV ITM Data Console
3. Enable Port 0 → Start Trace ● → Resume ▶
4. Application starts — LED_Control is BLOCKED (no output yet)
5. Press USER button on board
6. LED blinks 5 times rapidly
7. "Inside LEDControl Task" appears in console
8. Task goes BLOCKED again — waiting for next press
```

---

## 📊 Observation Table

| S.No | Query | Response |
|------|-------|----------|
| 1 | State of LED_Control task at application start | BLOCKED |
| 2 | State when button is pressed | READY → RUNNING |
| 3 | Does LED_Control run continuously after button press? | No — blocks again |
| 4 | Priority level of EXTI interrupt line | |

---

## 🔬 Bonus Experiment — Behavior Change

Replace the task code with this variant and observe the difference:

```c
/* Variant: No osOK check — always tries to acquire */
for (;;) {
    osSemaphoreAcquire(myBinarySem01Handle, 100);  // note: no if()
    i = 0;
    while (i < 10) {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        osDelay(250);
        i++;
    }
    printf("Inside LEDControl Task\n");
}
```

**Question:** What changes when the timeout (100 ms) expires without a button press?

---

## ❓ Reflection Questions

1. Why doesn't `LED_Control` run continuously after the button is pressed?
2. What is a breakpoint and how is it useful in debug mode for this experiment?
3. Trace through the full `osSemaphoreAcquire` loop — what happens at each iteration?
4. What behavior changes when you remove the `if (... == osOK)` check?

---

## ✅ Result

The Binary Semaphore successfully synchronized the LED blinking task to the hardware button press event. The ISR performed only a single non-blocking `osSemaphoreRelease()` call, while all time-consuming LED toggling was deferred safely to the `LED_Control` task context — demonstrating the Deferred Interrupt Processing pattern.
