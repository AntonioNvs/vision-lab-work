---
title: VisionLab — Multi-Phone Motion Capture
description: Scaling beyond 11 devices with Raspberry Pi 5 — University of Alberta
---

# VisionLab

**University of Alberta — Vision & Learning Lab**

Multi-phone motion capture offers a low-cost alternative to professional optical mocap, but Google's libsoftwaresync framework imposes a hard ceiling: 11 phones (10 clients + 1 leader/hotspot). This project attacks that bottleneck from two complementary angles.

---

## [Pi 5 Access Point](reports/pi5-access-point/)

Replaces the Android Wi-Fi hotspot with a Raspberry Pi 5 + USB Wi-Fi 6 adapter, removing the 10-client ceiling. Covers `hostapd`/`dnsmasq` configuration, DHCP reservation for leader discovery, onboard Wi-Fi failure analysis, and Wi-Fi 6 adapter migration.

**Key result:** 12 phones validated (11 clients + leader), no hard ceiling.

---

## [LED Sync Panel](reports/led-sync-panel/)

A camera-decodable LED timecode panel that measures inter-camera synchronization error directly from video — independently of phone clocks. Driven by a Raspberry Pi 5 via SPI through a 74HC595 shift register.

**Key result:** SNR analysis comparing resistors (150Ω vs 240Ω), improved decode pipeline, sub-tau resolution method.

---

**Author:** Antônio Caetano Neves Neto  
**Lab:** VisionLab, University of Alberta  
**Date:** July 2026  
**Repository:** [github.com/antoniocaetanonevesneto/visionlab](https://github.com/antoniocaetanonevesneto/visionlab)