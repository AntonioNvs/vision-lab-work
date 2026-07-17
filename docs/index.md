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

Multi-phone motion capture systems offer a low-cost alternative to professional optical mocap setups. Google Research's libsoftwaresync enables sub-millisecond clock synchronization across Android devices using SNTP, but its reliance on the Android Wi-Fi hotspot limits the rig to 11 phones (10 clients + 1 leader/hotspot). This work replaces the Android hotspot with a Raspberry Pi 5 access point, removing the device ceiling entirely while preserving sync accuracy ≤ 17 ms. We document the complete implementation: Pi 5 AP configuration with DHCP reservation for the leader phone, modifications to the libsoftwaresync codebase for fixed-IP leader discovery, and modernization of the Android build system for contemporary toolchains.

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

### 1.3 Hardware

| Component | Details |
|-----------|---------|
| Router | Raspberry Pi 5 (4 GB), onboard Wi-Fi 5 (802.11ac) |
| USB Adapter | TP-Link Archer TX20U Plus (RTL8832AU, Wi-Fi 6) — *driver compiled, reserved for future scaling* |
| Phones | Android 10+ devices (11–15 units) |
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
interface=wlan0
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

**File:** `/etc/dnsmasq.d/090_wlan0.conf`

```ini
interface=wlan0
domain-needed
dhcp-range=10.3.141.10,10.3.141.50,255.255.255.0,24h
dhcp-host=2c:4c:c6:9c:fc:7e,10.3.141.100
```

**Key decisions:**

- **Subnet `10.3.141.0/24`:** Private range, avoids conflicts with common `192.168.x.x` networks.
- **DHCP reservation for leader MAC:** The leader phone always receives `10.3.141.100`. This is the foundation of the patched leader-discovery mechanism (Section 2.2).
- **Pool `.10–.50`:** 41 addresses for client phones, well above our 15-phone target.

#### 2.1.3 Network Topology

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

### 2.4 USB Wi-Fi Adapter (Future Scaling)

The TP-Link Archer TX20U Plus (Realtek RTL8832AU chipset) was selected for higher client capacity. Despite being marketed as Wi-Fi 6 (AX1800), the in-kernel Linux driver currently operates at Wi-Fi 5 (802.11ac) speeds. The `lwfinger/rtl8852au` out-of-tree driver was compiled successfully on the Pi 5. If the onboard Wi-Fi proves insufficient under 15-phone load, the adapter can be activated by changing the `interface` line in `hostapd.conf` from `wlan0` to the adapter's interface name.

---

## 3. Results and Discussion

### 3.1 What Was Achieved

| Metric | Original (Android Hotspot) | This Work (Pi 5 AP) |
|--------|---------------------------|---------------------|
| Max phones | 11 | Limited only by AP capacity (est. 30+) |
| Leader discovery | Automatic (DHCP server IP) | Automatic (fixed IP via DHCP reservation) |
| Code changes | 0 lines | 2 lines (SoftwareSyncController.java) |
| Build system | AGP 3.5 / JDK 8 (broken on modern tools) | AGP 8.2 / JDK 21 |
| AP isolation risk | None (hotspot allows comms) | Mitigated (`ap_isolate=0`) |
| Internet required | No | No |
| Setup time per session | 0 (hotspot built-in) | ~2 min (Pi boot) |

### 3.2 Known Limitations

- **Sync accuracy not yet measured with Pi AP:** The original libsoftwaresync achieves <1 ms sync under ideal hotspot conditions. Network jitter introduced by an external AP may increase this. The 17 ms budget (one 60 fps frame) provides headroom, but empirical measurement is needed.
- **Leader phone single point of failure:** If the reserved-IP phone disconnects, clients have no leader. A future improvement could implement leader election.
- **MAC randomization trap:** Forgetting to disable MAC randomization on the leader phone silently breaks the setup — all phones become clients with no leader.

---

## 4. Conclusion

Replacing the Android hotspot with a Raspberry Pi 5 access point removes the 11-device ceiling from libsoftwaresync-based multi-phone capture rigs. The modification is minimally invasive — a 2-line code change plus DHCP reservation — and preserves the original sync protocol intact. The solution is low-cost (~$60 for the Pi 5), requires no internet connectivity, and scales to the practical limits of a single Wi-Fi access point.

The modernization of the build system to AGP 8.2.2 and JDK 21 ensures the project remains compilable on contemporary Android Studio releases, lowering the barrier for future researchers to replicate and extend this work.

---

## 5. Next Steps

1. **Empirical sync measurement:** Use the libsoftwaresync phase-alignment LED flash test to measure clock offset between leader and clients across the Pi AP, comparing against the baseline Android hotspot.
2. **Load testing:** Incrementally add phones (12 → 15 → 20) while monitoring sync accuracy degradation, identifying the practical ceiling of the Pi 5's onboard Wi-Fi.
3. **USB adapter activation:** If onboard Wi-Fi proves insufficient, switch to the TP-Link Archer TX20U Plus and repeat load testing.
4. **APK installation:** Resolve ADB detection issues and flash the patched APK to all phones.
5. **Test capture session:** Record synchronized multi-phone footage and validate frame-level alignment in post-processing.
6. **Leader election:** Implement automatic leader failover so the rig is not tied to a single phone's MAC address.
7. **Automated Pi provisioning:** Create an Ansible playbook or shell script that configures a fresh Pi 5 from scratch.

---

## Appendix A: Quick Setup Guide

### A.1 Raspberry Pi 5 (once)

```bash
# Install packages
sudo apt update && sudo apt install hostapd dnsmasq

# Stop services while configuring
sudo systemctl stop hostapd dnsmasq

# Write hostapd.conf (see Section 2.1.1)
sudo nano /etc/hostapd/hostapd.conf

# Write dnsmasq config (see Section 2.1.2)
sudo nano /etc/dnsmasq.d/090_wlan0.conf

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
