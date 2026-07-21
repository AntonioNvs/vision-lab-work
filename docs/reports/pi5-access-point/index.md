---
title: Breaking the 11-Device Limit in Multi-Phone Motion Capture
---

# Breaking the 11-Device Limit in Multi-Phone Motion Capture

## Scaling Google's libsoftwaresync with a Raspberry Pi 5 Access Point

**Author:** Antônio Caetano Neves Neto  
**Date:** July 2026  
**Repository:** [visionlab/libsoftwaresync](https://github.com/antoniocaetanonevesneto/visionlab)

---

## Abstract

Multi-phone motion capture systems offer a low-cost alternative to professional optical mocap setups. Google Research's libsoftwaresync enables sub-millisecond clock synchronization across Android devices using SNTP, but its reliance on the Android Wi-Fi hotspot limits the rig to 11 phones (10 clients + 1 leader/hotspot). This work replaces the Android hotspot with a Raspberry Pi 5 access point. The Pi 5's onboard Wi-Fi initially hit the same 10-client ceiling, but activating a USB Wi-Fi 6 adapter (TP-Link Archer TX20U Plus) broke through — successfully connecting 11 clients plus a leader (12 phones total). We document the complete implementation: dual-AP configuration (onboard → Wi-Fi 6), DHCP reservation for leader discovery, modifications to libsoftwaresync, and Android build system modernization.

---

## 1. Introduction

### 1.1 Background

Google Research's libsoftwaresync (2019) provides a software-based clock synchronization framework for Android multi-camera capture rigs. It operates over a local Wi-Fi network using two protocols:

- **SNTP clock sync** on UDP port 9428 — sub-millisecond precision
- **JSON-RPC control** on UDP port 8244 — trigger, exposure, and phase-alignment commands

The original architecture designates one phone as the **leader** (Wi-Fi hotspot + sync server) and all others as **clients**. Leader discovery is automatic: clients detect the hotspot's DHCP server IP and connect to it. This design is elegant but imposes a hard ceiling: **Android limits hotspots to 10 concurrent clients**, capping the rig at 11 phones.

### 1.2 Objective

Remove the 10-client ceiling by offloading the access point role to an external router — a Raspberry Pi 5 — while maintaining:
- Clock sync accuracy within 17 ms (one frame at 60 fps)
- The same leader-client architecture (one phone remains the software sync leader and records normally)
- Seamless client discovery without requiring manual IP configuration on each phone
- Proven capacity: 12 phones (11 clients + 1 leader), validated empirically

### 1.3 Hardware

| Component | Details |
|-----------|---------|
| Router | Raspberry Pi 5 (4 GB) |
| Primary AP | TP-Link Archer TX20U Plus (RTL8832AU, Wi-Fi 6) via `rtw89` kernel driver |
| Fallback AP | Pi 5 onboard Wi-Fi 5 (802.11ac) — limited to ~10 clients |
| Management | Ethernet direct link (PC ↔ Pi, `192.168.100.0/24`) |
| Phones | Android 10+ devices (12 units) |
| Leader phone MAC | `2c:4c:c6:9c:fc:7e` |

---

## 2. Development

### 2.1 Raspberry Pi 5 Access Point Configuration

The Pi 5 runs a standalone 5 GHz access point using `hostapd` and `dnsmasq`. No internet uplink is required during capture sessions — the network is purely local.

#### 2.1.1 hostapd — Wi-Fi Access Point Daemon

**File:** `/etc/hostapd/hostapd.conf`

```ini
driver=nl80211
ctrl_interface=/var/run/hostapd
ctrl_interface_group=0
beacon_int=100
auth_algs=1
wpa_key_mgmt=WPA-PSK
ssid=mocap-rig
channel=36
hw_mode=a
ieee80211n=1
ieee80211ac=1
wmm_enabled=1
ht_capab=[HT40+][SHORT-GI-20][SHORT-GI-40]
vht_capab=[SHORT-GI-80]
vht_oper_chwidth=1
vht_oper_centr_freq_seg0_idx=42
wpa_passphrase=mocap12345
interface=wlan1
wpa=2
wpa_pairwise=CCMP
country_code=BR
ap_isolate=0
```

**Key decisions:**

- **Channel 36 (5 GHz):** Avoids the congested 2.4 GHz band; 80 MHz channel width for throughput.
- **`ap_isolate=0`:** **Critical.** Without this, client devices cannot communicate with each other — SNTP and RPC packets would never reach the leader. This is the single most common failure point.
- **WPA2-PSK:** Sufficient for a local-only capture network with no internet exposure.

#### 2.1.2 dnsmasq — DHCP + DNS

**File:** `/etc/dnsmasq.d/090_wlan1.conf`

```ini
interface=wlan1
domain-needed
dhcp-range=10.3.141.10,10.3.141.50,255.255.255.0,24h
dhcp-host=2c:4c:c6:9c:fc:7e,10.3.141.100
```

**Key decisions:**

- **Subnet `10.3.141.0/24`:** Private range, avoids conflicts with common `192.168.x.x` networks.
- **DHCP reservation for leader MAC:** The leader phone always receives `10.3.141.100`. This is the foundation of the patched leader-discovery mechanism (Section 2.2).
- **Pool `.10–.50`:** 41 addresses for client phones, well above our 15-phone target.

#### 2.1.3 Onboard Wi-Fi Limitation & USB Adapter Migration

The Pi 5's onboard Wi-Fi (`wlan0`, Broadcom BCM43455) initially appeared sufficient. However, under load testing it hit the **same 10-client ceiling** as the Android hotspot — clients beyond the 10th could not associate or experienced severe packet loss. This is a known limitation of the Broadcom firmware's station table.

The TP-Link Archer TX20U Plus (Realtek RTL8832AU chipset, USB 3.0) was activated to overcome this. The adapter was recognized by the in-kernel `rtw89` driver (`lsusb` ID `0db0:6931`) and presented as `wlan1`. No external driver compilation was needed — the driver loaded automatically on Pi OS.

**Migration steps:**

```bash
# Stop services
sudo systemctl stop hostapd dnsmasq

# Swap interface in hostapd
sudo sed -i 's/interface=wlan0/interface=wlan1/' /etc/hostapd/hostapd.conf

# Swap interface in dnsmasq
sudo sed -i 's/interface=wlan0/interface=wlan1/' /etc/dnsmasq.d/090_wlan0.conf

# Move IP from wlan0 to wlan1
sudo ip addr del 10.3.141.1/24 dev wlan0 2>/dev/null
sudo ip addr add 10.3.141.1/24 dev wlan1

# Start services
sudo systemctl start dnsmasq hostapd
```

**Result:** 11 clients + 1 leader (12 phones) connected and stable. The Wi-Fi 6 adapter has no arbitrary client limit — its practical ceiling is determined by airtime and bandwidth, not firmware.

#### 2.1.4 Ethernet Management Link

A direct Ethernet connection between the PC and Pi 5 provides a reliable SSH channel independent of the Wi-Fi AP:

```bash
# Pi 5 side
sudo nmcli con add type ethernet ifname eth0 con-name eth0-static ip4 192.168.100.1/24
sudo nmcli con up eth0-static

# PC side
sudo nmcli con add type ethernet ifname enp44s0 con-name pi-eth ip4 192.168.100.2/24
sudo nmcli con up pi-eth
ssh antonio@192.168.100.1
```

#### 2.1.5 Network Topology

```
┌──────────────────────────────────────────────────┐
│  Raspberry Pi 5                                   │
│  AP: mocap-rig (5 GHz, Ch 36)                     │
│  IP: 10.3.141.1 (hostapd/dnsmasq)                 │
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │  Phone 1  │  │  Phone 2  │  │  Phone N  │  ...  │
│  │  LEADER   │  │  Client   │  │  Client   │         │
│  │ .100      │  │ .10       │  │ .11+      │         │
│  └──────────┘  └──────────┘  └──────────┘         │
│       │              │              │               │
│       └──────────────┼──────────────┘               │
│                      │                              │
│         All communication via Pi AP                 │
│         (SNTP :9428 + RPC :8244)                    │
└──────────────────────────────────────────────────┘
```

The Pi acts purely as a layer-2 bridge/AP. All libsoftwaresync traffic (SNTP clock sync + RPC commands) flows through it, but the Pi itself runs no sync software — it is transparent to the protocol.

### 2.2 libsoftwaresync Modifications

#### 2.2.1 Problem: Leader Discovery

The original code discovers the leader by calling `NetworkHelpers.getHotspotServerAddress()`, which reads the DHCP server IP from the Wi-Fi connection. This works when the leader *is* the hotspot, but fails when an external AP serves DHCP — every phone would see the Pi (`10.3.141.1`) as the leader and the sync protocol would break.

#### 2.2.2 Solution: Fixed-IP Leader with Reserved DHCP

We patched `SoftwareSyncController.java` to use a hardcoded leader IP combined with the DHCP reservation configured in dnsmasq:

**File:** `app/src/main/java/com/googleresearch/capturesync/SoftwareSyncController.java`  
**Lines 94–98 (patched):**

```java
localAddress = NetworkHelpers.getIPAddress();
leaderAddress = InetAddress.getByName("10.3.141.100");

// Leader is the phone with the reserved IP at 10.3.141.100 (Raspberry Pi DHCP).
isLeader = localAddress.equals(leaderAddress);
```

**How it works:**

1. The designated leader phone's MAC address is reserved to `10.3.141.100` in dnsmasq.
2. On startup, every phone compares its own Wi-Fi IP against `10.3.141.100`.
3. The phone that received the reserved IP becomes the software sync leader; all others act as clients and connect to it.
4. **Important:** Android 10+ randomizes MAC addresses by default. The leader phone **must have MAC randomization disabled** for the `mocap-rig` network, otherwise the DHCP reservation will never match.

#### 2.2.3 Why Not Make the Pi the Leader?

An alternative approach would run the sync leader software on the Pi itself, eliminating the need for a leader phone. This was rejected because:

- The Pi is not instrumented for video recording — a phone leader also records a camera stream.
- Running Java/Android code on the Pi would require a full Android runtime or a protocol reimplementation.
- The fixed-IP approach requires only a 2-line code change and zero protocol modifications.

### 2.3 Android Build System Modernization

The original libsoftwaresync repository targets Android Gradle Plugin (AGP) 3.5.1 with the deprecated `jcenter()` repository. Modern Android Studio releases (2024+) require AGP 8.x and JDK 17+. We updated the entire build configuration:

| Component | Original | Updated |
|-----------|----------|---------|
| AGP | 3.5.1 | 8.2.2 |
| Gradle | 5.x (wrapper) | 8.5 |
| JDK | 8 | 21 (Android Studio JBR) |
| compileSdk | 28 | 34 |
| targetSdk | 28 | 34 |
| Repositories | jcenter() | mavenCentral() |
| Namespace | AndroidManifest | build.gradle (`namespace`) |

**Key files updated:**

- `build.gradle` (root) — plugins block with AGP 8.2.2
- `app/build.gradle` — modernized DSL, namespace, compileSdk 34
- `settings.gradle` — pluginManagement + dependencyResolutionManagement
- `gradle.properties` — `org.gradle.java.home` pointing to Android Studio's bundled JDK 21

**Build command:**
```bash
cd libsoftwaresync
JAVA_HOME=/usr/local/android-studio/jbr ./gradlew assembleDebug
```

The APK is produced at `app/build/outputs/apk/debug/app-debug.apk`.

### 2.4 USB Wi-Fi 6 Adapter — The Key to Breaking the Ceiling

The TP-Link Archer TX20U Plus (Realtek RTL8832AU chipset, Wi-Fi 6 AX1800) was the solution to the 10-client limit. The in-kernel `rtw89` driver recognized it immediately — no out-of-tree compilation needed. The migration from `wlan0` (onboard) to `wlan1` (USB adapter) is documented in Section 2.1.3.

**Why it worked:** The Realtek chipset's firmware has no hardcoded client limit. The Broadcom firmware on the Pi 5's onboard Wi-Fi caps the station table at ~10 entries — a constraint that mirrors the Android hotspot limit. Switching to a different chipset vendor was the decisive factor.

---

## 3. Results and Discussion

### 3.1 What Was Achieved

| Metric | Original (Android Hotspot) | This Work (Pi 5 AP) |
|--------|---------------------------|---------------------|
| Max phones | 11 (10 clients) | **12 validated** (11 clients + leader); no hard ceiling |
| Leader discovery | Automatic (DHCP server IP) | Automatic (fixed IP via DHCP reservation) |
| Code changes | 0 lines | 2 lines (SoftwareSyncController.java) |
| Build system | AGP 3.5 / JDK 8 (broken on modern tools) | AGP 8.2 / JDK 21 |
| AP hardware | Phone hotspot | Pi 5 + USB Wi-Fi 6 adapter |
| AP isolation risk | None (hotspot allows comms) | Mitigated (`ap_isolate=0`) |
| Internet required | No | No |

**The 10-client barrier was broken twice:** first by replacing the Android hotspot with a Pi 5, and again when the Pi 5's own onboard Wi-Fi hit the same limit — solved by activating the USB Wi-Fi 6 adapter. The Realtek chipset has no firmware-enforced client cap.

### 3.2 Known Limitations

- **Sync accuracy not yet measured with Pi AP:** The original libsoftwaresync achieves <1 ms sync under ideal hotspot conditions. Network jitter introduced by an external AP may increase this. The 17 ms budget (one 60 fps frame) provides headroom, but empirical measurement is needed — the [LED Sync Panel](../led-sync-panel/) is being built for this purpose.
- **Leader phone single point of failure:** If the reserved-IP phone disconnects, clients have no leader. A future improvement could implement leader election.
- **MAC randomization trap:** Forgetting to disable MAC randomization on the leader phone silently breaks the setup — all phones become clients with no leader.

---

## 4. Conclusion

Replacing the Android hotspot with a Raspberry Pi 5 access point removes the 11-device ceiling from libsoftwaresync-based multi-phone capture rigs. The Pi 5's onboard Wi-Fi initially replicated the same 10-client limit, but activating a USB Wi-Fi 6 adapter (TP-Link Archer TX20U Plus) broke through — **12 phones (11 clients + leader) connected and stable**. The modification is minimally invasive — a 2-line code change plus DHCP reservation — and preserves the original sync protocol intact.

The key insight is that the Broadcom Wi-Fi chipset (used in both the Pi 5 and many Android phones) enforces a ~10-entry station table. Switching to a Realtek-based USB adapter eliminated this constraint, and the in-kernel `rtw89` driver required no compilation — the adapter worked out of the box on Pi OS.

The modernization of the build system to AGP 8.2.2 and JDK 21 ensures the project remains compilable on contemporary Android Studio releases, lowering the barrier for future researchers to replicate and extend this work.

---

## 5. Next Steps

1. **Empirical sync measurement:** Use the [LED Sync Panel](../led-sync-panel/) to measure inter-camera offset, validating sync accuracy independently of phone clocks.
2. **Scale beyond 12:** The Wi-Fi 6 adapter has no hard client limit — push to 15+ phones and measure airtime congestion.
3. **Active-low LED wiring:** Migrate the LED panel to 5V active-low drive for 10× brightness (see LED report §4.4).
4. **Leader election:** Implement automatic leader failover so the rig is not tied to a single phone's MAC address.
5. **Automated Pi provisioning:** Create a shell script that configures a fresh Pi 5 from scratch (hostapd, dnsmasq, SPI, LED firmware).

---

## Appendix A: Quick Setup Guide

### A.1 Raspberry Pi 5 (once)

```bash
# Plug in the USB Wi-Fi 6 adapter (TP-Link Archer TX20U Plus).
# Verify it appears as wlan1:
ip link show wlan1

# Install packages
sudo apt update && sudo apt install hostapd dnsmasq

# Stop services while configuring
sudo systemctl stop hostapd dnsmasq

# Write hostapd.conf (see Section 2.1.1) — use interface=wlan1
sudo nano /etc/hostapd/hostapd.conf

# Write dnsmasq config (see Section 2.1.2)
sudo nano /etc/dnsmasq.d/090_wlan1.conf

# Enable and start
sudo systemctl unmask hostapd
sudo systemctl enable hostapd dnsmasq
sudo systemctl start hostapd dnsmasq
```

### A.2 Leader Phone (once)

1. Connect to `mocap-rag` Wi-Fi network.
2. **Disable MAC randomization** for this network:
   - Settings → Wi-Fi → `mocap-rig` → Advanced → Privacy → "Use device MAC"
3. Verify the phone received `10.3.141.100` (check in Settings → About → Status).

### A.3 Client Phones (each)

1. Connect to `mocap-rag` Wi-Fi.
2. Install the patched APK via ADB or direct download.
3. Open the CaptureSync app — it should show "Client: ..." with a blue status indicator.
4. On the leader phone, verify the client count increments.

### A.4 Build from Source

```bash
git clone https://github.com/antoniocaetanonevesneto/visionlab
cd visionlab/libsoftwaresync
JAVA_HOME=/usr/local/android-studio/jbr ./gradlew assembleDebug
# APK at: app/build/outputs/apk/debug/app-debug.apk
```

---

## References

1. Google Research. (2019). *libsoftwaresync: Software synchronization for multi-phone capture.* [https://github.com/googleinterns/libsoftwaresync](https://github.com/googleinterns/libsoftwaresync)
2. Yu, B. (2020). *Motion Capture with Google libsoftwaresync.* [https://yuboshell.github.io/motion-capture.html](https://yuboshell.github.io/motion-capture.html)
3. hostapd documentation. [https://w1.fi/hostapd/](https://w1.fi/hostapd/)
4. dnsmasq manual. [https://thekelleys.org.uk/dnsmasq/doc.html](https://thekelleys.org.uk/dnsmasq/doc.html)
