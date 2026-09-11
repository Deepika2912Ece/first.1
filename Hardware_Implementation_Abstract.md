# Hardware Implementation Abstract: ESP32-Based Heart Rate Monitoring System

## Abstract

The hardware implementation of this heart rate monitoring system is built around the **ESP32 microcontroller**, which serves as the central processing unit for signal acquisition, filtering, and peripheral control. An analog **Pulse Sensor** is connected to an ADC1 input pin (GPIO 34) of the ESP32 to capture the photoplethysmogram (PPG) signal, a voltage waveform that varies with blood volume changes at the fingertip. ADC1 is deliberately chosen over ADC2 because the latter shares internal hardware with the ESP32's Wi-Fi radio, which can introduce measurement instability during wireless activity. The captured analog signal is converted to a 12-bit digital value (0–4095) by the ESP32's onboard ADC, configured with 11 dB attenuation to accommodate the sensor's 0–3.3V output range.

For real-time feedback, a **128x64 pixel I²C OLED display (SSD1306)** is wired to the ESP32's default I²C bus (SDA on GPIO 21, SCL on GPIO 22) and used to render the live BPM reading and system status text. A **buzzer** connected to a digital GPIO output (GPIO 25) provides an audible alert, toggled on and off by the firmware whenever the computed heart rate falls outside a configured safe range. All peripherals share a common ground with the ESP32 and are powered from its onboard 3.3V regulator, allowing the entire circuit to be assembled on a breadboard and powered through a single USB connection — resulting in a compact, low-cost, self-contained embedded prototype with no external computation or wireless dependency required for core operation.

## Hardware Components and Pin Mapping

| Component | Interface | ESP32 Pin |
|---|---|---|
| Pulse Sensor (Signal) | Analog (ADC1_CH6) | GPIO 34 |
| Pulse Sensor (VCC / GND) | Power | 3V3 / GND |
| OLED Display SSD1306 (SDA) | I²C | GPIO 21 |
| OLED Display SSD1306 (SCL) | I²C | GPIO 22 |
| Buzzer (+) | Digital Output | GPIO 25 |

## ESP32 Hardware Initialization Code

The following code excerpt shows the hardware-level setup performed in firmware — configuring the ADC for the Pulse Sensor, initializing the I²C bus for the OLED, and preparing the buzzer output:

```cpp
// ---------------------------------------------------------------------------
// Pin Definitions
// ---------------------------------------------------------------------------
#define PULSE_SENSOR_PIN   34   // ADC1_CH6 - analog input, input-only pin
#define BUZZER_PIN         25
#define OLED_SDA           21
#define OLED_SCL           22

// ---------------------------------------------------------------------------
// OLED Configuration
// ---------------------------------------------------------------------------
#define SCREEN_WIDTH   128
#define SCREEN_HEIGHT  64
#define OLED_RESET     -1
#define OLED_ADDRESS   0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Serial.begin(115200);

  // --- Buzzer output ---
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  // --- ADC configuration for Pulse Sensor ---
  analogReadResolution(12);          // ESP32 ADC: 0-4095
  analogSetAttenuation(ADC_11db);    // Full 0-3.3V input range

  // --- I2C bus initialization for OLED ---
  Wire.begin(OLED_SDA, OLED_SCL);

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDRESS)) {
    Serial.println(F("SSD1306 allocation failed"));
    while (true) { delay(10); }
  }

  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println(F("Heart Rate Monitor"));
  display.println(F("Initializing..."));
  display.display();
}
```

## ESP32 Sensor Reading Code

The raw analog PPG value is acquired directly from the Pulse Sensor pin using the ESP32's ADC read function, forming the entry point for all downstream signal processing:

```cpp
void processSample(unsigned long now) {
  int raw = analogRead(PULSE_SENSOR_PIN);   // Read raw PPG value from hardware ADC
  // -> raw is passed on to the filtering and peak-detection pipeline
}
```

## Buzzer Alert Hardware Control

The buzzer is driven directly by toggling a digital GPIO pin, producing an intermittent audible tone when the heart rate crosses a safe threshold:

```cpp
void updateBuzzer() {
  bool alertCondition = signalPresent &&
      (currentBPM > BPM_HIGH_THRESHOLD || currentBPM < BPM_LOW_THRESHOLD);

  if (alertCondition) {
    static unsigned long lastToggle = 0;
    static bool buzzerState = false;
    unsigned long now = millis();
    if (now - lastToggle >= 300) {
      lastToggle = now;
      buzzerState = !buzzerState;
      digitalWrite(BUZZER_PIN, buzzerState ? HIGH : LOW);
    }
  } else {
    digitalWrite(BUZZER_PIN, LOW);
  }
}
```

## Summary

The hardware implementation demonstrates efficient use of the ESP32's built-in ADC and I²C peripherals to interface with an analog biomedical sensor and standard display/alert hardware, without requiring any external signal-conditioning circuitry, analog-to-digital converter modules, or microcontroller add-ons. This makes the design compact, inexpensive, and easily reproducible on a breadboard for prototyping or educational purposes.
