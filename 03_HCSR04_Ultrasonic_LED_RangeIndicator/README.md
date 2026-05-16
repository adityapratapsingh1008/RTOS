# ◈ HC-SR04 Ultrasonic Distance Sensor

```
┌─────────────────────────────────────────────────────────────────┐
│          STM32F446RE  ←──────────→  HC-SR04 SENSOR             │
│                                                                 │
│   PA8 (TRIG) ──────────────────────► TRIG                      │
│   PA9 (ECHO) ◄────────────────────── ECHO                      │
│   3.3V  ───────────────────────────► VCC                       │
│   GND   ───────────────────────────► GND                       │
│                                                                 │
│   PA5  ──► [LED LD2]    PA2/PA3 ──► USART2 Terminal            │
└─────────────────────────────────────────────────────────────────┘
```

**Aim:** Interface an HC-SR04 ultrasonic sensor with STM32F446RE and classify distance using LED visual indication transmitted over UART.

---

## 🔩 Hardware Required

| Component | Qty |
|-----------|-----|
| STM32 Nucleo-F446RE | 1 |
| HC-SR04 Ultrasonic Sensor | 1 |
| Jumper wires | Several |
| USB Type-A to Mini-B cable | 1 |

---

## 📐 How It Works

The HC-SR04 measures distance using **time-of-flight** of ultrasonic sound waves:

```
 STM32              HC-SR04
   │                  │
   │── 10µs HIGH ───► TRIG
   │                  │ (emits 8 ultrasonic bursts at 40kHz)
   │                  │
   │◄─── ECHO HIGH ──── (echo pulse width ∝ distance)
   │                  │
   │── ECHO LOW ──────┘

  d = (v × Δt) / 2
    where v ≈ 343 m/s = 0.0343 cm/µs
```

TIM4 is configured with **Prescaler = 83** (84–1) so at 84 MHz system clock:

```
Timer tick = 84 MHz / 84 = 1 MHz → 1 tick = 1 µs
```

---

## ⚙️ CubeMX Configuration

### Step 1 — Pin Setup

| Pin | Label | Configuration |
|-----|-------|---------------|
| PA5 | LED | GPIO Output |
| PC13 | USER Button | GPIO Input |
| PA8 | TRIG | GPIO Output |
| PA9 | ECHO | GPIO Input |
| PA2 | USART2_TX | Alternate Function |
| PA3 | USART2_RX | Alternate Function |

### Step 2 — Clock
- **RCC** → BYPASS Clock Source
- **Clock Configuration** → set HCLK to **84 MHz**

### Step 3 — Timer (TIM4)
- Enable **TIM4** → Internal Clock
- **Prescaler:** `84 - 1` = 83
- **Counter Period (ARR):** leave default (we set it dynamically)
- This gives **1 µs resolution**

### Step 4 — USART2
- Mode: **Asynchronous**
- Baud Rate: 115200 (default)

### Step 5 — Code Generation
- Project Name → select **STM32CubeIDE** → **Generate Code**
- Properties → C/C++ Build → Settings → MCU Post Build: enable **Convert to Binary** and **Convert to Hex**

---

## 💻 Code — `main.c`

### Includes & Defines
```c
/* USER CODE BEGIN Includes */
#include <string.h>
#include <stdio.h>
/* USER CODE END Includes */

/* USER CODE BEGIN PTD */
#define usTIM TIM4
/* USER CODE END PTD */
```

### Function Prototype + Globals
```c
/* USER CODE BEGIN PFP */
void usDelay(uint32_t uSec);
/* USER CODE END PFP */

/* USER CODE BEGIN 0 */
const float speedOfSound = 0.0343 / 2;  // cm/µs (half for round-trip)
float distance;
char uartBuf[100];
/* USER CODE END 0 */
```

### Local Variable (before while loop)
```c
/* USER CODE BEGIN 1 */
uint32_t numTicks = 0;
/* USER CODE END 1 */
```

### Main Loop Logic
```c
/* USER CODE BEGIN 3 */
// 1. Pull TRIG low briefly
HAL_GPIO_WritePin(TRIG_GPIO_Port, TRIG_Pin, GPIO_PIN_RESET);
usDelay(3);

// 2. Send 10µs trigger pulse
HAL_GPIO_WritePin(TRIG_GPIO_Port, TRIG_Pin, GPIO_PIN_SET);
usDelay(10);
HAL_GPIO_WritePin(TRIG_GPIO_Port, TRIG_Pin, GPIO_PIN_RESET);

// 3. Wait for ECHO to go HIGH
while (HAL_GPIO_ReadPin(ECHO_GPIO_Port, ECHO_Pin) == GPIO_PIN_RESET);

// 4. Count µs while ECHO stays HIGH
numTicks = 0;
while (HAL_GPIO_ReadPin(ECHO_GPIO_Port, ECHO_Pin) == GPIO_PIN_SET) {
    numTicks++;
    usDelay(2);   // each iteration ≈ 2.8 µs
}

// 5. Compute distance
distance = (numTicks + 0.0f) * 2.8 * speedOfSound;

// 6. Send over UART
sprintf(uartBuf, "Distance (cm) = %.1f\r\n", distance);
HAL_UART_Transmit(&huart2, (uint8_t *)uartBuf, strlen(uartBuf), 100);
HAL_Delay(1000);

// 7. LED threshold indicator
if (distance <= 20.0f)
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);   // ON
else
    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET); // OFF
/* USER CODE END 3 */
```

### Microsecond Delay Function
```c
/* USER CODE BEGIN 4 */
void usDelay(uint32_t uSec) {
    if (uSec < 2) uSec = 2;
    usTIM->ARR = uSec - 1;  // Set reload value
    usTIM->EGR = 1;          // Reinitialize timer
    usTIM->SR  &= ~1;        // Clear overflow flag
    usTIM->CR1 |= 1;         // Start counter
    while ((usTIM->SR & 0x0001) != 1); // Wait for overflow
    usTIM->SR &= ~(0x0001);
}
/* USER CODE END 4 */
```

> **Why direct register access?** `HAL_Delay()` has 1 ms granularity — far too coarse for the µs timing needed here.

---


## 🔁 Running the Program

```
1. Connect board via USB
2. Build → click OK → click Switch (to debug perspective)
3. Click Resume ▶ to execute
4. Open serial terminal at 115200 baud to see distance values
```

---

## ✅ Result

The HC-SR04 was successfully interfaced with STM32F446RE. Distance values were computed via `d = (v × Δt) / 2` and displayed on UART. The onboard LED activated when measured distance fell below the 20 cm threshold.
