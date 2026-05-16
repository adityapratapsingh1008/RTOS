# 🔄 Super Loop Architecture

> *"No RTOS. No blocking. Just clever use of a millisecond clock."*

---

## 🎯 Aim

Implement a **non-blocking super loop** that runs three independent periodic tasks — LED blinking, button reading, and IR sensor sampling — all sharing a single CPU core without any `HAL_Delay()` blocking calls.

---

## 🔧 Hardware Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| IR Sensor module | 1 |
| Jumper wires | Several |
| USB Type-A to Mini-B cable | 1 |

---

## 🧠 The Core Idea — Foreground / Background

```
┌──────────────────────── FOREGROUND ─────────────────────────┐
│  SysTick ISR fires every 1 ms                               │
│  → increments HAL tick counter                              │
└─────────────────────────────────────────────────────────────┘
                            │
                     HAL_GetTick()
                            │
┌──────────────────────── BACKGROUND ─────────────────────────┐
│                   while(1) SUPER LOOP                       │
│                                                             │
│  dt = now - last_time                                       │
│                                                             │
│  led_counter    += dt  ──► fires every 500 ms  → LED blink │
│  button_counter += dt  ──► fires every  50 ms  → btn read  │
│  ir_counter     += dt  ──► fires every 200 ms  → ADC read  │
└─────────────────────────────────────────────────────────────┘
```

### Timeline Visualization

```
Time (ms): 0   50  100 150 200 250 300 350 400 450 500
           │    │    │    │    │    │    │    │    │    │    │
LED        │───────────────────────────────────────────●    │  (500ms toggle)
Button     │────●────●────●────●────●────●────●────●────●───│  (50ms poll)
IR Sensor  │─────────────●───────────────●───────────────●──│  (200ms ADC)
```

**Key insight:** Every task *checks* if its time has come — it never *waits*.

---

## ⚙️ CubeMX Configuration

### Step 1 — Pin Setup

| Pin | Label | Configuration |
|-----|-------|---------------|
| PA5 | LED LD2 | GPIO Output |
| PC13 | USER Button | GPIO Input, **Pull-up** |
| PA0 | IR Sensor | Analog (ADC1_IN0) |

### Step 2 — ADC1
- Enable **ADC1**, Channel **IN0** (PA0)
- Resolution: 12-bit (default) → values 0–4095

### Step 3 — Clock
- **RCC** → BYPASS, HCLK = **84 MHz**

### Step 4 — Generate Code
- Write project name → STM32CubeIDE → Generate Code

---

## 💻 Code — `main.c`

### Global Variables & Time Helper
```c
/* USER CODE BEGIN 0 */
volatile uint32_t led_counter_ms    = 0;
volatile uint32_t button_counter_ms = 0;
volatile uint32_t ir_counter_ms     = 0;
volatile uint32_t last_time_ms      = 0;

volatile uint8_t  button_pressed = 0;
volatile uint16_t ir_value       = 0;   // 0–4095 raw ADC

static inline uint32_t GetTimeMs(void) {
    return HAL_GetTick();   // SysTick 1 ms resolution
}
/* USER CODE END 0 */
```

### Initialization (before while loop)
```c
/* USER CODE BEGIN 2 */
last_time_ms = GetTimeMs();
/* USER CODE END 2 */
```

### Super Loop
```c
/* USER CODE BEGIN WHILE */
while (1) {

    /* ── Update elapsed time ── */
    uint32_t now_ms = GetTimeMs();
    uint32_t dt     = now_ms - last_time_ms;
    last_time_ms    = now_ms;

    led_counter_ms    += dt;
    button_counter_ms += dt;
    ir_counter_ms     += dt;

    /* ── Task 1: Blink LED @ 500 ms ── */
    if (led_counter_ms >= 500) {
        led_counter_ms = 0;
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    }

    /* ── Task 2: Read Button @ 50 ms ── */
    if (button_counter_ms >= 50) {
        button_counter_ms = 0;
        GPIO_PinState raw = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);
        if (raw == GPIO_PIN_RESET) {
            button_pressed = 1;
            HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET); // force LED ON
        } else {
            button_pressed = 0;
        }
    }

    /* ── Task 3: Read IR Sensor @ 200 ms ── */
    if (ir_counter_ms >= 200) {
        ir_counter_ms = 0;
        HAL_ADC_Start(&hadc1);
        HAL_ADC_PollForConversion(&hadc1, 10);  // wait up to 10 ms
        if (HAL_ADC_GetState(&hadc1) & HAL_ADC_STATE_REG_EOC) {
            ir_value = (uint16_t)HAL_ADC_GetValue(&hadc1);
        }
        HAL_ADC_Stop(&hadc1);
    }

    // ⚠ NO HAL_Delay() here — that would block everything!
}
/* USER CODE END WHILE */
```

---

## 🆚 Super Loop vs. Blocking Delays

| Approach | LED @ 500ms | Button @ 50ms | IR @ 200ms | CPU usage during wait |
|----------|-------------|--------------|------------|----------------------|
| `HAL_Delay()` blocking | ✅ | ❌ blocked | ❌ blocked | 100% (busy-wait) |
| **Super Loop (this exp)** | ✅ | ✅ | ✅ | 0% (loop runs freely) |

---

## 🔁 Running the Program

```
1. Connect board via USB
2. Build → OK → Switch (debugger perspective)
3. Add 'ir_value' and 'button_pressed' to Watch window
4. Resume ▶ and observe variables updating live
```

---

## 🔧 Modifications to Try

| Modification | Code Change |
|-------------|-------------|
| Change LED blink rate to 250 ms | `if (led_counter_ms >= 250)` |
| Change button poll to 100 ms | `if (button_counter_ms >= 100)` |
| Print ir_value over UART | Add `sprintf` + `HAL_UART_Transmit` in the IR task block |
| Add a 4th task (e.g., buzzer) | Add `buzzer_counter_ms`, increment it like the others |

---

## ✅ Result

The super loop successfully ran three concurrent periodic tasks without blocking, demonstrating non-blocking scheduling via `HAL_GetTick()` software counters as a lightweight alternative to RTOS task management for resource-constrained systems.
