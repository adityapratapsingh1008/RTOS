# 📬 Inter-Task Communication via FreeRTOS Queue

> Semaphores say *"something happened."*  
> Queues say *"something happened — and here's the data."*

---

## 🎯 Aim

Implement a **Producer-Consumer** inter-task communication pattern using a FreeRTOS **Message Queue**, where `Sensor_Read` produces distance data every second and `Motion_Control` consumes it for processing.

---

## 🔧 Hardware Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |
| *(Ultrasonic sensor optional — experiment uses simulated data)* | — |

---

## 🔄 The Producer-Consumer Pipeline

```
┌──────────────────┐          ┌─────────────────────┐          ┌──────────────────────┐
│                  │          │                     │          │                      │
│  SENSOR_READ     │──PUT──►  │   FreeRTOS QUEUE    │──GET──►  │  MOTION_CONTROL      │
│  (Producer)      │          │   [16 slots]        │          │  (Consumer)          │
│                  │          │   ┌─┬─┬─┬─┬─┬─┐    │          │                      │
│  dist += 1       │          │   │1│2│3│4│ │ │    │          │  receives distance   │
│  every 1000ms    │          │   └─┴─┴─┴─┴─┴─┘    │          │  prints to ITM       │
│                  │          │   FIFO order        │          │  blocks if empty     │
└──────────────────┘          └─────────────────────┘          └──────────────────────┘
        │                              │                                  │
    osDelay(1000)              Copy-by-Value:                    osMessageQueueGet()
    blocks for 1s             data copied INTO                   blocks until data
                              the queue buffer                   arrives
```

---

## 📦 Why Queues — Not Global Variables?

```c
// ❌ Global variable (unsafe in multi-task)
unsigned int dist = 0;  // Task A writes, Task B reads — RACE CONDITION!
                        // Scheduler can context-switch mid-read

// ✅ FreeRTOS Queue (thread-safe)
osMessageQueuePut(queue, &dist, ...);  // Kernel-managed, atomic copy
osMessageQueueGet(queue, &dist, ...);  // Blocks gracefully if empty
```

### Queue Blocking Behavior

```
Queue EMPTY → Consumer blocks (zero CPU) until Producer sends data
                   Consumer: BLOCKED ──────────── READY ──► RUNNING
                                       data arrives ↑

Queue FULL  → Producer blocks until Consumer reads and frees a slot
              (natural back-pressure — no data loss)
```

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

**Tasks & Queues tab — Tasks:**

| Field | Task 1 | Task 2 |
|-------|--------|--------|
| Task Name | `Sensing_Task` | `Navigation_Task` |
| Entry Function | `Sensor_Read` | `Motion_Control` |
| Priority | `osPriorityNormal` | `osPriorityNormal` |

**Tasks & Queues tab — Queue:**

| Field | Value |
|-------|-------|
| Queue Name | `myQueue01` |
| Queue Size | `16` (slots) |
| Item Size | `sizeof(unsigned int)` = 4 bytes |

### Step 3 — Advanced Settings
- Enable **USE_NEWLIB_REENTRANT** (required for `printf` in multiple tasks)

### Step 4 — Printf / SWV
- Enable `printf`/`scanf` in C/C++ Build Settings
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

### Verify Queue Creation
```c
/* Confirm element type is unsigned int — update if needed */
myQueue01Handle = osMessageQueueNew(16, sizeof(unsigned int), &myQueue01_attributes);
```

### Producer Task — `Sensor_Read`
```c
void Sensor_Read(void *argument) {
    /* USER CODE BEGIN 5 */
    unsigned int dist = 0;
    for (;;) {
        printf("Inside Data Producer Task\n");

        dist = dist + 1;  // Simulated sensor reading (replace with real sensor code)

        /* Put value into queue — blocks if queue is full */
        osMessageQueuePut(myQueue01Handle, &dist, 0, osWaitForever);

        osDelay(1000);  // Produce a new reading every 1 second
    }
    /* USER CODE END 5 */
}
```

### Consumer Task — `Motion_Control`
```c
void Motion_Control(void *argument) {
    /* USER CODE BEGIN Motion_Control */
    unsigned int distance;
    for (;;) {
        printf("Inside Data Consumer Task\n");

        /* Block here until a value is available in the queue */
        osMessageQueueGet(myQueue01Handle, &distance, NULL, osWaitForever);

        printf("Distance is %u\n", distance);
        /* Add processing logic here: motor control, display, etc. */
    }
    /* USER CODE END Motion_Control */
}
```

---

## 🖥️ Running & Observing

```
1. Build → Debug → Switch perspective
2. Window → Show View → SWV ITM Data Console
3. Port 0 → Start Trace ● → Resume ▶

Expected SWV output:
   Inside Data Producer Task
   Inside Data Consumer Task
   Distance is 1
   Inside Data Consumer Task       ← consumer blocks, waits
   Inside Data Producer Task       ← 1 second later
   Inside Data Consumer Task
   Distance is 2
   ...
```

---

## 🔧 Modifications to Try

### Replace Simulated Data with Real HC-SR04
Add ultrasonic measurement code inside `Sensor_Read` before the `osMessageQueuePut()` call, replacing `dist = dist + 1` with the distance calculation from Experiment 3.

### Test Queue Full Behavior
Change queue size to `3`, keep `osDelay(1000)` in producer but remove `osDelay` from consumer — observe the producer block when the queue fills.

### Test Queue Empty Behavior
Add `osDelay(2000)` in consumer and `osDelay(1000)` in producer — observe consumer blocks for 2 seconds at a time waiting for new data.

---

## ✅ Result

Inter-task communication was successfully demonstrated using a FreeRTOS Message Queue. The `Sensor_Read` producer task placed data into the kernel-managed queue every second, while `Motion_Control` remained efficiently blocked until data arrived — implementing a clean, race-condition-free Producer-Consumer pipeline.
