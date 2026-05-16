# ⚡ EXPERIMENT 4 — PWM LED Brightness Control

> **Control light with math.** A square wave with a variable duty cycle becomes analog brightness.

---

## 🎯 Aim

Generate a PWM signal on **TIM2 Channel 1 (PA5)** of the STM32F446RE and smoothly fade the onboard LED in and out by sweeping the duty cycle from 0% → 100% → 0%.

---

## 🔧 Hardware Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| USB Type-A to Mini-B cable | 1 |
| *(no external components needed — uses onboard LED on PA5)* | — |

---

## 📡 PWM Signal Anatomy

```
 Vmax ┤  ██████        ██████        ██████
      │  █    █        █    █        █    █
  0V  ┤  █    ████████ █    ████████ █    ████████
      └──┬────┬────────┬────┬────────►  time
         │TON │  TOFF  │

   Duty Cycle  D = TON / T × 100%
   Avg Voltage = Vmax × D / 100

   25% duty:  ██░░░░░░░░░░  →  dim
   50% duty:  █████░░░░░░░  →  medium
   75% duty:  ████████░░░░  →  bright
  100% duty:  ████████████  →  full on
```

---

## 🔢 Key Formulas

```
PWM Frequency  =  f_CLK / (PSC + 1) / (ARR + 1)

With:  f_CLK = 84 MHz,  PSC = 839,  ARR = 1000
       = 84,000,000 / 840 / 1001 ≈ 100 Hz

Duty Cycle (%) =  CCR / (ARR + 1) × 100

Example:   CCR = 500 → 500/1000 × 100 = 50%
```

---

## ⚙️ CubeMX Configuration

### Step 1 — Pin Setup
Right-click **PA5** → Set as **TIM2_CH1** (Alternate Function)

### Step 2 — Clock
- **RCC** → BYPASS Clock Source
- Set HCLK to **84 MHz**

### Step 3 — TIM2 Parameters

| Parameter | Value | Effect |
|-----------|-------|--------|
| Prescaler (PSC) | `840 - 1` | Timer clock = 84M/840 = **100 kHz** |
| Counter Mode | `Up` | Counts 0 → ARR |
| Auto-Reload (ARR) | `1000` | Period = 100kHz/1001 ≈ **100 Hz** |
| CH1 Mode | `PWM Generation CH1` | PA5 outputs PWM |
| CH1 Pulse (CCR) | `0` (initial) | Start at 0% duty cycle |

### Step 4 — Code Generation
- Write Project name → select **STM32CubeIDE** → Generate Code
- Properties → MCU Post Build: enable **.bin** and **.hex** output

---

## 💻 Code — `main.c`

### Start PWM (after `MX_TIM2_Init()`)
```c
/* USER CODE BEGIN 2 */
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
/* USER CODE END 2 */
```

### Fade Loop (inside `while(1)`)
```c
/* USER CODE BEGIN WHILE */
while (1) {
    int x;

    /* ── Fade IN: 0% → 100% ── */
    for (x = 0; x < 1000; x++) {
        __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, x);
        HAL_Delay(1);   // 1ms per step → 1 second total fade
    }

    /* ── Fade OUT: 100% → 0% ── */
    for (x = 1000; x > 0; x--) {
        __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, x);
        HAL_Delay(1);
    }
}
/* USER CODE END WHILE */
```

---

## 🔬 CCR → Brightness Mapping

```
  CCR   0  ▏░░░░░░░░░░░░░░░░░░░░  0%    LED OFF
  CCR 250  ▏████░░░░░░░░░░░░░░░░ 25%    dim glow
  CCR 500  ▏██████████░░░░░░░░░░ 50%    half brightness
  CCR 750  ▏███████████████░░░░░ 75%    bright
  CCR 1000 ▏████████████████████100%    full brightness
```

---

## 📊 Observation Table

| CCR Value | Duty Cycle (%) | Brightness Level | Observation |
|-----------|---------------|-----------------|-------------|
| 0 | 0% | | |
| 250 | 25% | | |
| 500 | 50% | | |
| 750 | 75% | | |
| 1000 | 100% | | |

---

## 🔁 Running the Program

```
1. Connect board via USB
2. Build project → OK → Switch (to debug perspective)
3. Resume ▶ — watch the LED breathe in and out
```

---

## 🔧 Modifications to Try

| Change | How | Expected Outcome |
|--------|-----|-----------------|
| Faster fade | Change `HAL_Delay(1)` → `HAL_Delay(0)` | LED fades in ~0.1s |
| Slower fade | Change `HAL_Delay(1)` → `HAL_Delay(5)` | LED fades in ~5s |
| Fixed 33% brightness | Remove loops; set `CCR = 333` | LED stays dim |
| Higher PWM frequency | Reduce `ARR` to 99 | 10x frequency (check flicker) |
| Lower PWM frequency | Increase `ARR` to 9999 | Visible flicker at ~10Hz |

---

## ❓ Reflection Questions

1. What is PWM and how does duty cycle affect average output voltage?
2. Calculate PWM frequency if PSC = 179 and ARR = 499 (90 MHz clock).
3. What happens if PWM frequency drops to 10 Hz — will the LED flicker?
4. How do you achieve exactly 33% duty cycle with ARR = 999?
5. How would you control an external motor speed using this same technique?

---

## ✅ Result

PWM was successfully generated using TIM2 on the STM32F446RE. The onboard LED demonstrated smooth fade-in and fade-out, confirming the direct relationship between CCR duty cycle value and perceived LED brightness through pulse-width modulation.
