# esp32_s3_composite_video

A lightweight, hardware-accelerated Arduino/ESP-IDF library for generating a **stable monochrome PAL 50Hz composite video signal** on the **ESP32-S3** microcontroller using a custom 8-bit R-2R resistor ladder DAC.

## Features
- **Pure ESP32-S3 Support:** Engineered specifically for the S3 architecture (Xtensa LX7), bypassing the lack of an internal DAC.
- **Hardware-Driven:** Uses the precise `esp_timer` peripheral for microsecond-accurate TV-line generation. 
- **Fixed High Resolution:** Optimized at a rock-solid **320x220** pixels (progressive scan, no flicker).
- **256 Grayscale Steps:** Full contrast control via an external 8-bit R-2R resistor ladder.
- **Built-in Sync Protection:** Automatically clamps active pixel brightness above the sync threshold (>= 60) to prevent display desynchronization.
- **Ultra-Lean Interface:** Stripped-down API containing only the highly optimized Bresenham `drawLine` function with variable thickness and brightness.

## Hardware Setup (8-bit R-2R DAC)
Connect 8 consecutive digital GPIOs (e.g., GP1, GP2, GP4, GP5, GP6, GP7, GP8, GP9) through a standard R-2R resistor network (R=1kΩ, 2R=2kΩ) [2.1]:
- **D0 (LSB):** Connects to the first pin.
- **D7 (MSB):** Connects to the last pin (strongest signal weight).
- **RCA Center Pin:** Connects to the end of the R-2R ladder (with a 2kΩ shunting resistor to GND).
- **RCA Outer Shield:** Connects directly to **ESP32 GND**.

## Minimal Example
```cpp
#include <Arduino.h>
#include <esp32_s3_composite_video.h>

esp32_s3_composite_video tv;

void setup() {
    // Initialize by passing 8 GPIO pins (D0 to D7)
    tv.begin(1, 2, 4, 5, 6, 7, 8, 9);

    // Can also be initialized with an array of pins
    // const int my_pins[] = {1, 2, 4, 5, 6, 7, 8, 9};
    // tv.begin(my_pins);
}

void loop() {
    tv.clear(60); // Clear screen to pure black
    
    // Draw a bright white diagonal line
    // tv.drawLine(x0, y0, x1, y1, thickness, brightness)
    tv.drawLine(1, 1, 10, 10, 5, 255);
    
    delay(20); // Maintain ~50 FPS
}
```

## API Reference

### 1. Initialization (`begin`)

Activates the hardware timers, registers the GPIO pins, and starts streaming the PAL signal via background interrupts.

#### Option A: List initialization
```cpp
bool begin(int p0, int p1, int p2, int p3, int p4, int p5, int p6, int p7);
```
Passes 8 GPIO pins directly as separate arguments. Order must be from LSB (`p0`) to MSB (`p7`).
- **Example:**
  ```cpp
  tv.begin(1, 2, 4, 5, 6, 7, 8, 9);
  ```

#### Option B: Array initialization
```cpp
bool begin(const int user_pins[]);
```
Passes an array containing exactly 8 pins. Order must be from LSB (`[0]`) to MSB (`[7]`).
- **Example:**
  ```cpp
  const int my_pins[] = {1, 2, 4, 5, 6, 7, 8, 9};
  tv.begin(my_pins);
  ```

---

### 2. Screen Management (`clear`)

```cpp
void clear(uint8_t color = 60);
```
Fills the entire framebuffer with a single color/brightness level. 
- `color`: Grayscale value from `60` (pure black) to `255` (pure white). Values below `60` are clamped to prevent breaking TV sync. Default is `60`.
- **Example:**
  ```cpp
  tv.clear();    // Blanks the screen to solid black
  tv.clear(128); // Fills the background with a medium gray tone
  ```

---

### 3. Graphics Rendering (`drawLine`)

```cpp
void drawLine(int x0, int y0, int x1, int y1, int thickness, uint8_t brightness);
```
The core rendering function. Draws any line using the hardware-optimized Bresenham algorithm with configurable line width and shade of gray.
- `x0, y0`: Starting coordinates (X: `0...319`, Y: `0...219`).
- `x1, y1`: Ending coordinates (X: `0...319`, Y: `0...219`).
- `thickness`: Line width in pixels (minimum `1`).
- `brightness`: Grayscale shade from `60` (black) to `255` (white).

#### Single Pixel / Dot Recipe:
Set the start and end points to the exact same coordinates.
- **Example:**
  ```cpp
  // Draws a bold white dot in the center of the screen, 8 pixels wide
  tv.drawLine(160, 110, 160, 110, 8, 255);
  ```

#### Border / Wireframe Rectangle Recipe:
Chain four lines to form a box.
- **Example:**
  ```cpp
  int x = 20, y = 30, w = 80, h = 40, thick = 2, gray = 180;
  tv.drawLine(x, y, x + w, y, thick, gray);         // Top edge
  tv.drawLine(x + w, y, x + w, y + h, thick, gray); // Right edge
  tv.drawLine(x + w, y + h, x, y + h, thick, gray); // Bottom edge
  tv.drawLine(x, y + h, x, y, thick, gray);         // Left edge
  ```

#### Solid / Filled Rectangle Recipe:
Draw a single horizontal line where the `thickness` matches the desired height of the rectangle.
- **Example:**
  ```cpp
  // Renders a solid gray block (width: 100px, height: 40px)
  tv.drawLine(50, 110, 150, 110, 40, 130);
  ```

---

### 4. Geometry Getters (`width` / `height`)

Used to dynamically fetch screen limits, making your application code fully responsive to resolution changes.

```cpp
int width() const;  // Always returns 320
int height() const; // Always returns 220
```

- **Example:**
  ```cpp
  // Automatically draws a border edge-to-edge with a 5px offset
  int w = tv.width();
  int h = tv.height();
  tv.drawLine(5, 5, w - 6, 5, 2, 255);
  ```
