# 🎫 EXPERIMENT 10 — Counting Semaphore & Shared Resource Control

> A binary semaphore is a single ticket.  
> A counting semaphore is a **pool of tickets**.

---

## 🎯 Aim

Model a **limited shared resource** using a FreeRTOS Counting Semaphore with three competing tasks, and study access control, resource contention, and task interleaving behavior on the SWV ITM console.

---

## 🔧 Hardware Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |

---

## 🧠 Counting Semaphore — The Token Pool Model

```
 Counting Semaphore (Max=2, Initial=2)

 ┌──────────────────────────────────┐
 │  Token Pool                      │
 │  ┌──────┐  ┌──────┐             │
 │  │  🎫  │  │  🎫  │             │  ← 2 tokens available
 │  └──────┘  └──────┘             │
 └──────────────────────────────────┘

 Task A acquires → takes 1 token → count: 2→1
 Task B acquires → takes 1 token → count: 1→0
 Task C acquires → count is 0 → BLOCKED (no tokens left!)

 Task A releases → returns token → count: 0→1 → Task C UNBLOCKS
```

---

## ⚖️ Counting Semaphore vs. Binary Semaphore vs. Mutex

```
┌──────────────────┬──────────────┬───────────────┬────────────────────┐
│ Primitive        │ Max Count    │ Use Case      │ Mutual Exclusion?  │
├──────────────────┼──────────────┼───────────────┼────────────────────┤
│ Binary Semaphore │ 1            │ Signaling     │ Yes (1 at a time)  │
│ Mutex            │ 1            │ Shared data   │ Yes + ownership    │
│ Counting Semaphore│ N (configurable)│ Resource pools│ No (N at a time) │
└──────────────────┴──────────────┴───────────────┴────────────────────┘
```

---

## 🔀 What "Interleaving" Looks Like

With semaphore count = **2**, tasks A and B can hold it simultaneously → both print to ITM at the same time → **garbled output**:

```
Expected (if mutex):     Actual (with counting semaphore, count=2):
  1AAAAAAAAAA              1A2BAB2BBBAAAB1AAAB2BB
  2BBBBBBBBBB              (characters interleaved — unpredictable order)
  3CCCCCCCCCC
```

This is **not a bug** — it perfectly illustrates that counting semaphores manage *how many* can access a resource, not *atomicity* of the access.

---

## ⚙️ CubeMX Configuration

### Step 1 — System Settings

| Setting | Value |
|---------|-------|
| RCC | BYPASS |
| HCLK | 84 MHz |
| SYS → Debug | Trace Asynchronous Sw |
| SYS → Timebase | **TIM6** |

### Step 2 — FreeRTOS (CMSIS_V2)

**Tasks & Queues tab — Three Tasks:**

| Field | TaskA | TaskB | TaskC |
|-------|-------|-------|-------|
| Task Name | `TaskA` | `TaskB` | `TaskC` |
| Entry Function | `func_TaskA` | `func_TaskB` | `func_TaskC` |
| Priority | `osPriorityNormal` | `osPriorityNormal` | `osPriorityNormal` |

**Timers & Semaphores tab — Counting Semaphore:**

| Field | Value |
|-------|-------|
| Semaphore Name | `myCountingSem01` |
| **Max Count** | **`2`** — maximum 2 tasks hold it simultaneously |
| **Initial Count** | **`2`** — starts fully available |

### Step 3 — Advanced Settings
- Enable **USE_NEWLIB_REENTRANT**

### Step 4 — Printf / SWV
- Enable `printf`/`scanf` float in C/C++ Build Settings
- Debugger → SWV, Core Clock = 84 MHz

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

### Task A
```c
void func_TaskA(void *argument) {
    /* USER CODE BEGIN 5 */
    char ch = 'A';
    for (;;) {
        osSemaphoreAcquire(myCountingSem01Handle, osWaitForever); // Take token
        printf("1");
        for (int i = 0; i < 10; i++) {
            printf("%c", ch);   // Prints 'A' ten times
            HAL_Delay(50);
        }
        osSemaphoreRelease(myCountingSem01Handle);  // Return token
        osDelay(5);
    }
    /* USER CODE END 5 */
}
```

### Task B
```c
void func_TaskB(void *argument) {
    /* USER CODE BEGIN func_TaskB */
    char ch = 'B';
    for (;;) {
        osSemaphoreAcquire(myCountingSem01Handle, osWaitForever);
        printf("2");
        for (int i = 0; i < 10; i++) {
            printf("%c", ch);   // Prints 'B' ten times
            HAL_Delay(50);
        }
        osSemaphoreRelease(myCountingSem01Handle);
        osDelay(5);
    }
    /* USER CODE END func_TaskB */
}
```

### Task C
```c
void func_TaskC(void *argument) {
    /* USER CODE BEGIN func_TaskC */
    char ch = 'C';
    for (;;) {
        osSemaphoreAcquire(myCountingSem01Handle, osWaitForever);
        printf("3");
        for (int i = 0; i < 10; i++) {
            printf("%c", ch);   // Prints 'C' ten times
            HAL_Delay(50);
        }
        osSemaphoreRelease(myCountingSem01Handle);
        osDelay(5);
    }
    /* USER CODE END func_TaskC */
}
```

---

## 🖥️ Running & Observing

```
1. Build → Debug → Switch perspective
2. Window → Show View → SWV ITM Data Console
3. Port 0 → Start Trace ● → Resume ▶

With count=2, TWO tasks hold the semaphore at a time:
   1A2BABABABABAB2BBBBA1AABAABBA...
   (interleaved — 2 tasks printing simultaneously)

With count=1 (change for comparison):
   1AAAAAAAAAA\n
   2BBBBBBBBBB\n
   3CCCCCCCCCC\n
   (clean blocks — only 1 task at a time)
```

---

## 📊 Observation Table

| # | Query | Response |
|---|-------|----------|
| 1 | How many tasks can use the ITM trace resource concurrently? | |
| 2 | Describe the interleaving pattern in the trace output | |
| 3 | Type of scheduling that leads to interleaving | |
| 4 | What different outcome can you produce by changing the code? | |

---

## 🔬 Experiment Variations

| Change | Expected Outcome |
|--------|-----------------|
| Set semaphore count = **1** | Only 1 task prints at a time → clean output |
| Set semaphore count = **3** | All 3 tasks access simultaneously → maximum interleaving |
| Remove `osSemaphoreRelease()` from a task | That task holds token forever — other tasks may starve |
| Change one task priority to `osPriorityHigh` | High-priority task gets token more often |
| Replace counting semaphore with **mutex** | Mutual exclusion — clean output, 1 task at a time |

---

## 🔑 Key Takeaway

```
Counting Semaphore (count=2):
    ✅ Controls HOW MANY tasks access a resource pool
    ❌ Does NOT guarantee atomic/exclusive access per task

Mutex:
    ✅ Guarantees ONE task at a time (true mutual exclusion)
    ✅ Ownership concept (only acquirer can release)
    ❌ Not suitable for signaling between tasks

Use counting semaphores for: connection pools, hardware unit pools, rate limiting
Use mutexes for: shared data structures, single peripherals (SPI, UART)
```

---

## ❓ Reflection Questions

1. What does "thread-safe" mean — and why does a counting semaphore with count > 1 not fully guarantee it for a single UART/ITM port?
2. Implement a fast producer + slow consumer using the counting semaphore. What happens when count reaches 0?
3. Implement a slow producer + fast consumer. What happens when count reaches max?
4. Write a reflection summarizing your key learnings about RTOS-based system design across all experiments.

---

## ✅ Result

Three tasks competed for a shared resource (ITM trace port) managed by a Counting Semaphore with count = 2. The SWV output clearly demonstrated task interleaving — two tasks printed simultaneously, producing garbled character sequences. This confirmed that counting semaphores manage resource **availability** but not **atomicity**, distinguishing them from mutexes in practical RTOS design.
