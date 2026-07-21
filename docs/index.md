---
title: VisionLab — Multi-Phone Motion Capture
description: Scaling beyond 11 devices with Raspberry Pi 5
---

# VisionLab

Research infrastructure for scaling multi-phone motion capture beyond the 11-device limit. Two complementary projects address the synchronization bottleneck from different angles.

---

## Reports

### [Pi 5 Access Point](reports/pi5-access-point/)

Replaces the Android Wi-Fi hotspot with a Raspberry Pi 5 access point, removing the 10-client ceiling of Google's libsoftwaresync. Covers `hostapd`/`dnsmasq` configuration, DHCP reservation for leader discovery, and USB Wi-Fi 6 adapter activation.

**Key result:** 15+ phone rig with sync accuracy ≤ 17 ms.

---

### [LED Sync Panel](reports/led-sync-panel/)

An independent, camera-decodable LED timecode panel for measuring inter-camera synchronization error directly from video footage. Driven by a Raspberry Pi 5 via SPI through a 74HC595 shift register.

**Key result:** Sub-millisecond sync measurement resolution, independent of phone clocks.

---

## Repository

[github.com/antoniocaetanonevesneto/visionlab](https://github.com/antoniocaetanonevesneto/visionlab)

**Author:** Antônio Caetano Neves Neto  
**Date:** July 2026