# GPIO Digital Input: Push Button LED Toggle

##  Aim
To interface a push button as digital input and demonstrate LED control by toggling its state on each valid button press.

---

##  Apparatus

| Component | Details |
|-----------|---------|
| Microcontroller Board | STM32 Nucleo-F446RE |
| Cable | USB Type-A to Mini-B |
| IDE | STM32CubeIDE |
| Configuration Tool | STM32CubeMX (integrated in CubeIDE) |

---

##  Theory

This experiment extends GPIO knowledge from output to **input**, reading the state of the onboard user push button and using it to control the onboard LED.

### GPIO as Input
When a GPIO pin is configured as a **digital input**, the microcontroller reads the voltage level present on that pin and interprets it as logic `1` (HIGH) or logic `0` (LOW). Unlike output mode where the MCU drives the pin, in input mode the MCU only listens.

### Board Pin Mapping

| Function | Arduino Label | MCU Pin | Default State |
|----------|---------------|---------|---------------|
| User LED (LD2) | D13 | **PA5** | Output, Push-Pull |
| User Button (B1) | — | **PC13** | Input, Active LOW |

### Active LOW Button Logic
The onboard user button **B1** on the Nucleo-F446RE is connected to **PC13** with a hardware pull-up resistor. This means:
- Button **not pressed** → PC13 reads **HIGH (1)**
- Button **pressed** → PC13 reads **LOW (0)**

This is called **active-low** logic — the signal goes low when the event occurs.

### Button Debouncing
Mechanical buttons do not produce a clean single transition when pressed. The contacts physically bounce, generating multiple rapid HIGH/LOW transitions before settling. This is called **contact bounce** and can cause the MCU to register one press as multiple events.

A simple **software debounce** is applied here — after detecting a press, a short `HAL_Delay(200)` pause waits for the signal to stabilize before the loop checks again.

### HAL Functions Used

| Function | Description |
|----------|-------------|
| `HAL_GPIO_ReadPin(GPIOx, GPIO_Pin)` | Returns the current logic state of an input pin (`0` or `1`) |
| `HAL_GPIO_TogglePin(GPIOx, GPIO_Pin)` | Flips the current state of a GPIO output pin |
| `HAL_Delay(ms)` | Blocking delay in milliseconds — used here as a software debounce |

---

##  STM32CubeMX Configuration

### Step 1 — MCU Selection
- Open STM32CubeMX → **ACCESS TO MCU SELECTOR**
- Search and select: **STM32F446RETx**
- Click **Start Project**

### Step 2 — GPIO Configuration

**PA5 — LED Output:**

| Parameter | Value |
|-----------|-------|
| GPIO Mode | Output Push Pull |
| GPIO Pull-up/Pull-down | No pull-up and no pull-down |
| Maximum Output Speed | Low |
| User Label | LED *(optional)* |

**PC13 — Push Button Input:**

| Parameter | Value |
|-----------|-------|
| GPIO Mode | Input mode |
| GPIO Pull-up/Pull-down | No pull-up and no pull-down |
| User Label | BTN *(optional)* |

> The Nucleo board already has a hardware pull-up on PC13, so no internal pull-up is needed in CubeMX.

### Step 3 — Clock Source (RCC)
- Navigate to **System Core → RCC**
- High Speed Clock (HSE): **BYPASS Clock Source**

### Step 4 — Clock Configuration

| Clock Domain | Prescaler | Frequency |
|--------------|-----------|-----------|
| PLL Source | HSE | — |
| SYSCLK | PLL output | 180 MHz |
| AHB (HCLK) | /1 | 180 MHz |
| APB1 | /4 | 45 MHz |
| APB2 | /2 | 90 MHz |

### Step 5 — Project Manager
- Set **Project Name** (e.g., `Experiment_2`)
- Toolchain/IDE: **STM32CubeIDE**
- Click **Generate Code** → **Open Project**

### Step 6 — Build Output Settings
- Right-click project → **Properties → C/C++ Build → Settings → MCU Post Build Outputs**
- Enable  **Convert to Binary File (.bin)**
- Enable  **Convert to Intel Hex File (.hex)**
- Click **Apply and Close**

---

##  Code

Open `Core/Src/main.c` and add the following inside the `while(1)` loop between the user code markers:

```c
/* USER CODE BEGIN WHILE */
while (1)
{
    // Read onboard button (PC13) — Active LOW
    if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == 0)  // Pressed = LOW
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);       // Toggle LED on PA5
        HAL_Delay(200);                              // Software debounce delay
    }
    /* USER CODE END WHILE */
}
```

### Program Flow

```
Loop starts
    │
    ▼
Read PC13
    │
    ├── HIGH (not pressed) ──→ do nothing, loop again
    │
    └── LOW (pressed)
            │
            ▼
        Toggle PA5 (LED flips state)
            │
            ▼
        Delay 200ms  ← debounce: ignore contact bounce
            │
            ▼
        Loop again
```

>  Always write your code between the `USER CODE BEGIN` and `USER CODE END` comment markers to prevent it from being overwritten on CubeMX code regeneration.

---

##  Build & Flash

1. Connect the Nucleo board via USB
2. Click **Build** (hammer icon) → verify **0 errors**
3. Click **Debug** → click **Switch** on the perspective popup
4. Click **Resume (▶)** to start execution
5. Press the blue **B1 user button** on the Nucleo board — the green LED **LD2** should toggle on each press

---

##  Observations

### Button Press vs LED State

| Trial No. | Button Action | Button Pin State (PC13) | LED State (PA5) Before | LED State (PA5) After | Observation |
|-----------|---------------|------------------------|------------------------|------------------------|-------------|
| 1 | Initial (No press) | HIGH | Low | Low | LED is OFF |
| 2 | 1st Press | LOW | Low | High | LED turns ON |
| 3 | Release | HIGH | High | High | LED stays ON |
| 4 | 2nd Press | LOW | High | Low | LED turns OFF |
| 5 | 3rd Press | LOW | Low | High | LED turns ON |
| 6 | Rapid Press | Toggling | Dependent on state | Toggles | LED flickers |

**Key observations:**
- The LED state is **latched** — it holds its new value even after the button is released.
- Each valid press causes exactly one toggle.
- Rapid pressing causes flickering as the 200 ms debounce window is too short for very fast repeated presses.

---

##  Result

For all valid button presses, the LED on PA5 toggled exactly once per press, and the new state was maintained after button release. This confirms correct configuration of **PC13 as a digital input** and **PA5 as a digital output**, with functional software-based debouncing using `HAL_Delay()`.

---

##  Key Takeaways

- A microcontroller does not inherently understand a "button press" as an event — it only reads binary logic levels (HIGH/LOW). The programmer is responsible for interpreting the signal transition.
- **Active-low logic** is common in embedded hardware design. Always check the board schematic to determine the idle (unpressed) state of an input pin.
- **Contact bounce** is a real hardware phenomenon. A single physical press can produce dozens of logic transitions in microseconds. A software delay is the simplest debounce technique, though timer-based or state-machine debouncing is more robust.
- The LED in this experiment is **stateful** — unlike the continuous blink in Experiment 1, it remembers and holds its last state. This is the foundation of toggle and latch logic used in real-world control systems.
- Combining GPIO input and output in the same program creates a meaningful stimulus-response relationship — the foundation of all embedded control systems.

---

##  Project Structure

```
02_PushButton_LED_Toggle/
├── Core/
│   ├── Inc/
│   │   └── main.h
│   └── Src/
│       └── main.c          ← User code added here (while loop)
├── Drivers/
│   └── STM32F4xx_HAL_Driver/
├── Experiment_2.ioc        ← CubeMX pin & clock configuration file
└── README.md               ← This file
```
