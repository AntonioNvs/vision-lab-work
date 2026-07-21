---
title: LED Sync Panel — Camera-Decodable Timecode
description: Independent sync measurement for multi-camera rigs via Raspberry Pi 5
---

# LED Sync Panel

## Measuring Inter-Camera Sync Without Trusting Phone Clocks

**Author:** Antônio Caetano Neves Neto  
**Date:** July 2026  
**Repository:** [yuboshell/led-sync-panel](https://github.com/yuboshell/led-sync-panel) (antonio-development branch)

---

## 1. Motivation

Multi-phone capture rigs use software clock synchronization (SNTP), but the true test of sync is **what the cameras actually record**. The LED sync panel is a flat array of 7 LEDs displaying a rapidly advancing Gray-coded timecode. All cameras film it simultaneously; decoding each camera's frame tells us the exact capture-time offset — **independently of any phone clock or network timestamp**.

This is the hardware counterpart to the [Pi 5 Access Point](../pi5-access-point/) setup. Together they form a complete synchronization stack: the AP ensures the phones agree on time; the LED panel **measures** how well they actually do.

---

## 2. Why Raspberry Pi 5 Instead of Pico H

The original design used a Raspberry Pi Pico H microcontroller. The migration to a Raspberry Pi 5 was driven by future integration needs:

| | Pico H | Raspberry Pi 5 |
|---|---|---|
| **Role** | Dedicated LED driver | Hub for the entire rig |
| **Connectivity** | USB only | Wi-Fi AP + Ethernet + SPI |
| **Software** | Bare-metal C (RP2040) | Linux + Python + C |
| **Future** | Single-purpose | Control panel + decode pipeline + remote access |

**Key reasons for the switch:**

1. **Unified hardware.** The Pi 5 already runs the Wi-Fi access point for the phones. Running the LED panel from the same device eliminates a separate microcontroller, power supply, and USB cable.

2. **Remote control.** A Python web server on the Pi 5 can start/stop the timecode and adjust parameters from any browser on the capture network — no need to physically touch the panel during a shoot.

3. **On-device decode pipeline.** The Pi 5 can run the decode scripts directly after footage is transferred, turning the rig into a self-contained measurement system.

4. **SPI is native.** The Pi 5's `/dev/spidev0.0` interface drives the 74HC595 identically to the Pico's hardware SPI, with the added benefit that CE0 auto-toggles the latch.

---

## 3. Hardware

### 3.1 Circuit

```
Raspberry Pi 5                   74HC595 Shift Register
─────────────────────────────────────────────────────
Pin 19 (GPIO10/MOSI)  ─────────  Pin 14 (SER)    — serial data
Pin 23 (GPIO11/SCLK)  ─────────  Pin 11 (SRCLK)  — shift clock
Pin 24 (GPIO8/CE0)    ─────────  Pin 12 (RCLK)   — latch (auto from SPI CS)
Pin 1/17 (3.3V)       ─────────  Pin 16 (VCC)    — logic power
Pin 1/17 (3.3V)       ─────────  Pin 10 (MR)     — disable reset
Pin 6 (GND)           ─────────  Pin 8 (GND)     — ground
Pin 6 (GND)           ─────────  Pin 13 (OE)     — outputs enabled
                              QA (Pin 15) — unused

LEDs: 595 QB..QH → 150Ω → LED (+) → LED (-) → GND
```

### 3.2 Components

| Part | Qty | Notes |
|------|-----|-------|
| Raspberry Pi 5 | 1 | Any RAM variant |
| 74HC595 shift register | 1 | SN74HC595N, DIP-16 |
| Red LED (5 mm) | 7 | Standard through-hole |
| 150Ω resistor (¼ W) | 7 | One per LED (see §4) |
| 0.1 µF ceramic capacitor | 1 | Decoupling, VCC↔GND |

---

## 4. Resistor Selection: 150Ω vs 240Ω

The current-limiting resistor determines LED brightness, which directly affects the signal-to-noise ratio in the decode pipeline. Two values were tested experimentally.

### 4.1 Experiment

Two videos of the panel running `./timecode` (default: 500 µs/step Gray code) were recorded with a Pixel 7 at 4K30, identical framing and lighting. The only variable was the resistor.

### 4.2 Results

| Metric | 150Ω | 240Ω | Improvement |
|--------|------|------|-------------|
| **SNR (mean)** | 12.4 dB | 4.1 dB | **+200%** |
| **Contrast (on−off)** | 70.6 | 52.5 | **+34%** |
| **Brightness (ON)** | 97.3 | 86.5 | +12% |
| **Brightness (OFF)** | 26.6 | 34.0 | — |
| **Variance (signal)** | 2114 | 1640 | +29% |

### 4.3 Analysis

The 150Ω resistor delivers **3× the SNR** because:

- **Higher current**: 8.7 mA vs 5.4 mA (+60%)
- **Better contrast**: Larger separation between ON and OFF brightness levels
- **Lower noise floor**: The OFF state is darker (26.6 vs 34.0), reducing ambiguity

The 240Ω is sufficient for a dark room close-up, but the 150Ω provides margin for suboptimal conditions (ambient light, camera distance, lower-quality phone sensors).

### 4.4 Active-Low Future Enhancement

The 74HC595 is weak at sourcing current (VOH drops to ~2.4V at 6 mA) but strong at sinking (VOL ≈ 0.3V). An active-low configuration — powering LED anodes from the Pi's 5V rail and connecting cathodes to 595 outputs — would deliver **~18 mA per LED** (10× the current active-high setup):

```
5V → 150Ω → LED (+) → LED (-) → 595 QB..QH  (active-low, 595 sinks)
I_LED = (5 − 2.0 − 0.3) / 150 = 18 mA
```

This requires inverting the firmware output bits. The code is prepared; wiring migration is pending.

---

## 5. Timecode & Decode

### 5.1 Gray Code

The panel displays a 7-bit Gray code counter (0–127), advancing every τ = 500 µs. Gray coding ensures **only one LED changes per step** — even the 127→0 wrap. A camera frame whose exposure straddles a tick decodes to within ±1 count, never to a garbage multi-bit value.

### 5.2 Firmware

The Pi 5 firmware (`rpi5_timecode/main.c`) is a native C port of the Pico version. It uses Linux `spidev` for SPI communication and `clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME)` for drift-free timing.

### 5.3 Decode Pipeline

```
ffmpeg frame extraction → LED localization (variance + max-image)
→ Gaussian-weighted sampling → Otsu threshold → un-Gray → count
→ timestamp = count × τ
```

**Improvements over the original decoder:**

- **Variance-based localization**: Finds blinking LEDs by temporal variance, robust to static reflections
- **Sub-pixel refinement**: Parabolic interpolation around each peak for precise center coordinates
- **Gaussian-weighted sampling**: Pixels near the LED center contribute more, reducing cross-talk
- **Otsu threshold**: Adaptive per-LED thresholding handles varying brightness
- **SNR reporting**: Per-LED signal-to-noise ratio in dB
- **Confidence scoring**: Per-frame quality metric (margin to threshold, 0–1)

### 5.4 Sub-Tau Resolution (In Development)

The Gray code changes one LED per step. When a camera frame captures the LED mid-transition, its **analog brightness** encodes the exact instant within the τ interval:

```
timestamp = count × τ + fraction × τ
where fraction = (brightness − off_level) / (on_level − off_level)
```

This promises **~50 µs effective resolution** from the existing 500 µs hardware — a 10× improvement requiring no hardware changes.

---

## 6. Next Steps

1. **Active-low wiring**: Migrate LEDs to 5V rail with 595 sinking for 10× brightness
2. **Sub-tau decode**: Implement analog brightness fraction in `two_camera_offset_v2.py`
3. **Web control panel**: Flask server on Pi 5 for remote start/stop and tau adjustment
4. **End-to-end validation**: 11-phone rig with simultaneous AP sync + LED panel measurement
5. **Two-row vernier**: Add a second LED row (coarse) for disambiguating the fine-row wrap, as modeled in `sim/`