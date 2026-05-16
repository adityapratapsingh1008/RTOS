# GPIO Digital Output: LED Blink with Software Delay

## 🎯 Aim
To configure a GPIO pin of **STM32F446RE** as a digital output and verify LED blinking operation using software delay routines.

---

## 🧰 Apparatus

| Component | Details |
|-----------|---------|
| Microcontroller Board | STM32 Nucleo-F446RE |
| Cable | USB Type-A to Mini-B |
| IDE | STM32CubeIDE |
| Configuration Tool | STM32CubeMX (integrated in CubeIDE) |

---

## 📖 Theory

The **STM32F446RE** microcontroller provides multiple general-purpose I/O (GPIO) ports — **GPIOA through GPIOH** — each of which can be independently configured into one of four modes:

| Mode | Description |
|------|-------------|
| **Input** | Read external digital signal |
| **Output** | Drive a digital high/low signal |
| **Alternate Function** | Used by peripherals (UART, SPI, I2C, etc.) |
| **Analog** | Used by ADC/DAC |

### Why PA5?
On the **Nucleo-F446RE** board, the onboard green user LED (**LD2**) is hardwired to **Arduino pin D13**, which maps to microcontroller pin **PA5**. This means no external LED or wiring is required — PA5 can directly drive LD2.

### Push-Pull Output Mode
PA5 is configured in **push-pull output** mode, which allows the pin to:
- Drive the output **HIGH** (~3.3 V) → LED **ON**
- Drive the output **LOW** (0 V) → LED **OFF**

This is in contrast to open-drain mode, which can only pull the pin low and requires an external pull-up resistor.

### HAL Functions Used

| Function | Description |
|----------|-------------|
| `HAL_GPIO_TogglePin(GPIOx, GPIO_Pin)` | Flips the current state of the specified GPIO pin |
| `HAL_Delay(ms)` | Blocks execution for the given number of milliseconds using SysTick |

---

## ⚙️ STM32CubeMX Configuration

### Step 1 — MCU Selection
- Open STM32CubeMX → **ACCESS TO MCU SELECTOR**
- Search and select: **STM32F446RETx**
- Click **Start Project**

### Step 2 — GPIO Configuration
- Navigate to **System Core → GPIO**
- Right-click on pin **PA5** → Set as **GPIO_Output**
- GPIO Settings:

| Parameter | Value |
|-----------|-------|
| GPIO Mode | Output Push Pull |
| GPIO Pull-up/Pull-down | No pull-up and no pull-down |
| Maximum Output Speed | Low |
| User Label | LD2 *(optional)* |

### Step 3 — Clock Source (RCC)
- Navigate to **System Core → RCC**
- High Speed Clock (HSE): **BYPASS Clock Source**
- This enables use of the external clock signal provided by the ST-Link on the Nucleo board

### Step 4 — Clock Configuration
- Open the **Clock Configuration** tab
- Configure to achieve **maximum internal clock frequency (180 MHz)**

| Clock Domain | Prescaler | Frequency |
|--------------|-----------|-----------|
| PLL Source | HSE | — |
| SYSCLK | PLL × multiplier | 180 MHz |
| AHB (HCLK) | /1 | 180 MHz |
| APB1 | /4 | 45 MHz |
| APB2 | /2 | 90 MHz |

### Step 5 — Project Manager
- Set **Project Name** (e.g., `Experiment_1`)
- Toolchain/IDE: **STM32CubeIDE**
- Click **Generate Code** → **Open Project**

### Step 6 — Build Output Settings
- Right-click project → **Properties → C/C++ Build → Settings → MCU Post Build Outputs**
- Enable ✅ **Convert to Binary File (.bin)**
- Enable ✅ **Convert to Intel Hex File (.hex)**
- Click **Apply and Close**

---

## 💻 Code

After code generation, open `Core/Src/main.c` and add the following two lines inside the `while(1)` loop, between the user code markers:

```c
/* USER CODE BEGIN WHILE */
while (1)
{
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);   // Toggle LED LD2 on PA5
    HAL_Delay(500);                           // 500 ms software delay

    /* USER CODE END WHILE */
}
```

> ⚠️ Always write your code between the `USER CODE BEGIN` and `USER CODE END` comment markers. Code placed outside these markers will be **overwritten** if you regenerate from CubeMX.

---

## 🔨 Build & Flash

1. Connect the Nucleo board via USB
2. Click **Build** (hammer icon) → verify **0 errors, 0 warnings**
3. Click **Debug** → click **Switch** on the perspective-switch popup
4. Click **Resume (▶)** to start program execution
5. The onboard LED LD2 should begin blinking at the configured rate

---

## 📊 Observations

The delay value was varied and the effect on LED blink speed was recorded:

| S. No. | GPIO Pin | Mode / Pull | Delay (ms) | Approx. Blink Period (s) | Comment |
|--------|----------|-------------|------------|--------------------------|---------|
| 1 | PA5 | Output, PP, No PU/PD | 100 | 0.1 s | Fast — barely visible toggle |
| 2 | PA5 | Output, PP, No PU/PD | 200 | 0.3 s | Visible — moderate speed |
| 3 | PA5 | Output, PP, No PU/PD | 500 | 1 s | Clearly visible — comfortable rate |
| 4 | PA5 | Output, PP, No PU/PD | 1000 | 2 s | Slow — clearly distinct ON/OFF phases |

> **Note:** Blink period ≈ 2 × Delay value, since the LED spends one delay duration ON and one delay duration OFF per cycle.

---

## ✅ Result

The GPIO pin **PA5** of the STM32F446RE on the Nucleo-F446RE board was successfully configured as a **digital push-pull output** using STM32CubeIDE. The onboard user LED **LD2** blinked correctly at the configured rate using HAL-based software delay routines. Varying the delay value produced a proportional and clearly observable change in blink speed, confirming correct GPIO output and clock configuration.

---

## 💡 Key Takeaways

- GPIO pins are multi-purpose — the same physical pin can act as input, output, alternate function, or analog depending on the software configuration.
- **Push-pull mode** actively drives the pin both HIGH and LOW, making it ideal for directly driving LEDs without external components.
- `HAL_Delay()` is a **blocking delay** based on SysTick. During the delay, the CPU is stalled and cannot perform any other task — this is a key limitation of bare-metal super loop programs.
- The **accuracy of `HAL_Delay()`** depends entirely on correct SYSCLK configuration. A misconfigured clock will produce incorrect timing.
- The **super loop (`while(1)`)** is the simplest embedded program structure — sequential, single-task, and runs forever.
- Blink period = 2 × delay, because the LED completes one full ON-OFF cycle per two delay calls.

---

## 📁 Project Structure

```
01_GPIO_LED_Blink_SoftwareDelay/
├── Core/
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       └── main.c          ← User code added here (while loop)
├── Drivers/
│   └── STM32F4xx_HAL_Driver/
├── Experiment_1.ioc        ← CubeMX pin & clock configuration file
└── README.md               ← This file
```
