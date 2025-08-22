# Seven-Segment Display with STM32F103C8  
STM32F103C8 project driving a seven-segment display via GPIO and SPI (74HC595)

## Description  
This project demonstrates two different approaches to drive a **common-cathode seven-segment display** using the STM32F103C8 microcontroller:  

1. **Direct GPIO control** – each segment is driven directly by GPIO pins.  
2. **SPI with 74HC595 shift register** – reduces the number of pins required by using a shift register.  

Two push buttons are used to increment and decrement numbers from **0 to 9** in both implementations.  

---

## Hardware Used  
- STM32F103C8 (Blue Pill)  
- Common-cathode seven-segment display  
- 2× Push buttons  
- Breadboard, jumper wires  
- 74HC595 shift register (for Version 2)  

---

## Software & Tools  
- STM32CubeIDE and STM32CubeMX  
- ST-LINK programmer  

---

## Implementation Versions  

### 🔹 Version 1 – Direct GPIO  
- Segments connected directly to GPIO pins.  
- A lookup table defines which segments to turn on for digits `0–9`.  
- Simple to implement but consumes many I/O pins.  

### 🔹 Version 2 – SPI with 74HC595  
- Segments controlled through the **74HC595 shift register**.  
- Data is sent via the SPI interface, then latched to update the display.  
- Saves GPIO pins and is scalable for multi-digit displays.  

---

## How It Works  
- **GPIO version:** Each segment (A–G, DP) is controlled directly using output pins.  
- **SPI version:** Segment data is shifted serially into the 74HC595 and displayed once latched.  
- Two push buttons:  
  - **Increment** → increases digit (wraps 0 → 9)  
  - **Decrement** → decreases digit (wraps 9 → 0)  

---

## Demo  

### Version 1 – Direct GPIO  
![Seven-Segment GPIO](https://github.com/Negar-Mahmoudy/stm32-seven-segment-gpio-spi/blob/acf085bbcd67c09e0e98c98953ab953fda11a270/images/version1-schematic.png)  
*Seven-segment driven directly by STM32F103C8 GPIO pins.*

### Version 2 – SPI with 74HC595  
![Seven-Segment SPI](images/version2-schematic.jpg)  
*Seven-segment driven through SPI and 74HC595 shift register.*

### Working GIF  
![Seven-Segment Demo GIF](images/version1.gif)  
*Incrementing and decrementing numbers with push buttons - Version1*

![Seven-Segment Demo GIF](images/version2.gif)  
*Incrementing and decrementing numbers with push buttons - Version2*

---

## Step-by-Step Walkthrough — Version 1 (Direct GPIO)

1. **Map the segments to GPIO pins**
   - Connect the 7-segment (common-cathode) segments to `GPIOB`:
     ```
     PB11 → a
     PB10 → b
     PB9  → c
     PB15 → d
     PB14 → e
     PB13 → f
     PB12 → g
     ```
   - Buttons: `BTN_UP`, `BTN_DOWN` on GPIOB with pull-ups, EXTI on falling edge.

2. **Define digit bitmasks**
   - Create a lookup table `digits[10]` where each bit indicates one segment (1 = ON, 0 = OFF).
   - Shift the masks so they align with PB9–PB15.  
     ```c
     const uint16_t digits[10] = {
       0x77<<9, 0x03<<9, 0x6e<<9, 0x4f<<9, 0x1b<<9,
       0x5d<<9, 0x7d<<9, 0x07<<9, 0x7f<<9, 0x5f<<9
     };
     uint16_t mask = 0x7f << 9;  // PB9..PB15
     uint8_t num = 0;            // current digit (0..9)
     ```

3. **Initialize clocks and GPIOs**
   - Enable clocks for `GPIOB` and configure:
     - Segments (`PB9..PB15`) as **Output Push-Pull** (low speed).
     - Buttons as **Input with Pull-Up**, **EXTI on falling edge**.
   - Initialize all segment pins to **RESET**.

4. **Configure EXTI and NVIC**
   - Enable EXTI lines for the two buttons.
   - Set NVIC priorities (e.g., `EXTI0_IRQn`, `EXTI1_IRQn`) and enable them.

5. **Handle button presses in the EXTI callback**
   - Use `HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)`:
     - If `BTN_UP` → `num++` (clamp at 9).
     - If `BTN_DOWN` → `num--` (clamp at 0).
   - Update the display:
     ```c
     HAL_GPIO_WritePin(GPIOB, digits[num], GPIO_PIN_SET);                  // turn ON required segments
     HAL_GPIO_WritePin(GPIOB, mask & ~(digits[num]), GPIO_PIN_RESET);      // turn OFF others
     ```

6. **Main loop stays idle**
   - The `while(1)` loop is empty—display updates are fully interrupt-driven for responsive button handling.

7. **(Optional) Debounce**
   - If needed, add a simple debounce (delay or state sampling) inside the EXTI callback or ignore repeated edges by checking time since last press.

8. **Test**
   - Verify each digit 0→9 appears correctly.
   - Check wrap behavior at boundaries (0 and 9) and confirm no ghost segments.

> ✅ Result: Minimal logic in the main loop, instant updates on button press, but uses many GPIO pins (7 segments + 2 buttons).

## Step-by-Step Walkthrough — Version 2 (SPI + 74HC595)

1. **Hardware setup**
   - Replace direct GPIO control with a **74HC595 shift register**.
   - Connect STM32F103 → 74HC595 using **SPI1**:
     - `SPI1_MOSI` → `DS` (serial input)
     - `SPI1_SCK`  → `SHCP` (shift clock)
     - `LATCH` pin → `STCP` (storage clock, manual GPIO toggle)
   - Seven-segment (common-cathode) connected to outputs `QA..QG`.

   Segment mapping:
     ```
     QA → c
     QB → b
     QC → a
     QD → g
     QE → f
     QF → e
     QG → d
     ```
    - Buttons: `BTN_UP`, `BTN_DOWN` on GPIOB with pull-ups, EXTI on falling edge.
3. **Digit encoding**
- Define a lookup table `digits[10]` with 7-bit masks for 0–9:
  ```c
  const uint8_t digits[10] = {
    0x77, 0x03, 0x6e, 0x4f, 0x1b,
    0x5d, 0x7d, 0x07, 0x7f, 0x5f
  };
  ```
- Each bit corresponds to one segment (1 = ON).

3. **Initialize peripherals**
- Enable GPIO clocks and configure:
  - **Buttons**: input with pull-up + EXTI interrupts (falling edge).
  - **LATCH pin**: output push-pull.
- Configure **SPI1** in master mode, 8-bit data, MSB first, low polarity/phase.

4. **Interrupt handling (buttons)**
- Software debounce prevents multiple increments.
- Callback identifies which button triggered the interrupt using GPIO_Pin.
- Main loop updates display automatically; CPU responds only to interrupts.
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin){
static uint32_t last_tick=0;
if(HAL_GetTick()-last_tick<200) return;
last_tick=HAL_GetTick();
if(GPIO_Pin==BTN_UP_Pin && num<9) num++;
else if(GPIO_Pin==BTN_DOWN_Pin && num>0) num--;
}
```
5. **Main loop logic**
- Continuously send the current digit to the shift register:
  ```c
  HAL_SPI_Transmit(&hspi1, &digits[num], 1, 1);
  LATCH();         // toggle latch pin to update outputs
  HAL_Delay(100);  // small refresh delay
  ```
- Display automatically updates whenever `num` changes.

6. **LATCH function**
- Ensures data from shift register is transferred to output pins:
  ```c
  void LATCH() {
    HAL_GPIO_WritePin(LATCH_GPIO_Port, LATCH_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(LATCH_GPIO_Port, LATCH_Pin, GPIO_PIN_RESET);
  }
  ```

7. **Advantages over Version 1**
- Uses **only 3 STM32 pins** (MOSI, SCK, LATCH) instead of 7 segment pins.
- Cleaner wiring and scalable if more digits or LEDs are added (chainable 74HC595).
- CPU load still minimal; EXTI interrupt handles button logic.

> ✅ Result: Same functionality as Version 1 (0–9 counting with buttons), but with much fewer GPIO pins used thanks to SPI and the shift register.
