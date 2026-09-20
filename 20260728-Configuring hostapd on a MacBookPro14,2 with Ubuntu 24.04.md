---
title: "Configuring hostapd on a MacBookPro14,2 with Ubuntu 24.04"
description: "This is a somewhat specialized use case, but this article documents how to use a MacBook Pro as a..."
pubDatetime: 2026-07-28T12:17:39.041Z
updatedDate: 2026-09-08T01:41:30.073Z
---

This is a somewhat specialized use case, but this article documents how to use a MacBook Pro as a Linux router and Wi-Fi access point.

In this setup, I installed **Ubuntu 24.04** on a **MacBookPro14,2 (2017, 13-inch)** and configured its built-in Broadcom wireless adapter as an access point using `hostapd`.

This article focuses specifically on **configuring hostapd**. It does not cover bridge configuration such as creating `br0` or assigning IP addresses.

## Environment

* MacBookPro14,2
* Ubuntu 24.04 LTS
* hostapd
* Wireless interface: `wlp2s0`
* Wired interface: `enx68da73adde3c`
* Bridge: `br0`

`hostapd` uses the `nl80211` driver. On modern Linux systems, `nl80211` is typically used with mac80211/cfg80211-based wireless drivers.

---

# hostapd.conf

The current configuration is as follows:

```ini
driver=nl80211

country_code=JP
ieee80211d=1

# 5 GHz
hw_mode=a
channel=36

# 802.11n/ac
ieee80211n=1
ieee80211ac=1
wmm_enabled=1

# 40 MHz
ht_capab=[HT40+]

# 80 MHz disabled
# no_pri_sec_switch=1
# vht_oper_chwidth=1
# vht_oper_centr_freq_seg0_idx=42

# WPA2
wpa=2
# wpa_key_mgmt=WPA-PSK SAE
# wpa_key_mgmt=SAE
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP

# PMF disabled
# ieee80211w=1
# ieee80211w=2
ieee80211w=0

ignore_broadcast_ssid=1
disable_pmksa_caching=1

# For SAE configurations
# sae_pwe=2

# interface=wlx3476c5d38aef
interface=wlp2s0
bridge=br0

ssid=YOUR_SSID
wpa_passphrase=YOUR_PASSWORD
# wpa_psk=...
```

---

# Configuration Details

## nl80211

```ini
driver=nl80211
```

This tells `hostapd` to use the Linux `nl80211` interface to communicate with the wireless driver.

For typical modern Linux wireless drivers based on mac80211/cfg80211, `nl80211` is the standard choice.

---

## Bridge

```ini
bridge=br0
```

This places wireless clients on the `br0` bridge, allowing them to join the same Layer 2 network segment as the wired LAN.

If a DHCP server already exists on the LAN, Wi-Fi clients can obtain their addresses from that server through the bridge. There is therefore no need to run a separate DHCP server specifically for `hostapd`.

---

## Fixed 5 GHz Operation

```ini
hw_mode=a
channel=36
```

This configures the access point to operate on the 5 GHz band using channel 36.

In this setup, channel 36 in the W52 band is used so that operation does not require DFS radar detection or a Channel Availability Check (CAC).

---

## Japan Regulatory Domain

```ini
country_code=JP
ieee80211d=1
```

`country_code=JP` configures the AP for the Japanese regulatory domain.

The channels and operating parameters actually available to `hostapd` are also affected by cfg80211, the Linux regulatory database, the wireless driver, firmware, and hardware capabilities.

`ieee80211d=1` enables IEEE 802.11d functionality, allowing the AP to advertise country information to clients.

---

# 802.11n / 802.11ac

```ini
ieee80211n=1
ieee80211ac=1
wmm_enabled=1
```

This enables 802.11n (HT) and 802.11ac (VHT) support.

`wmm_enabled=1` enables Wi-Fi Multimedia (WMM). WMM/QoS support is required for HT operation such as 802.11n, so it is enabled here.

---

# 40 MHz Channel Width (HT40)

An earlier version of this configuration disabled `ht_capab` and operated with a 20 MHz channel width.

The current configuration enables:

```ini
ht_capab=[HT40+]
```

`HT40+` specifies 40 MHz HT operation with the secondary channel above the primary channel.

With channel 36 as the primary channel, this allows a 40 MHz channel to be formed using the adjacent upper channel.

Previously, enabling `HT40+` caused `hostapd` to fail with:

```plaintext
Could not set channel for kernel driver
```

After revisiting the configuration and environment, `hostapd` now starts successfully with `HT40+` enabled.

Therefore, the conclusion in the earlier version of this article—that HT40 was unavailable due to a driver or firmware limitation—no longer applies to the current setup.

At least with the current environment, **5 GHz / channel 36 / HT40+** can be used successfully.

---

# 80 MHz (VHT80) Is Disabled

Although HT40 is now enabled, 80 MHz operation is still not enabled.

The relevant VHT80 options remain commented out:

```ini
# no_pri_sec_switch=1
# vht_oper_chwidth=1
# vht_oper_centr_freq_seg0_idx=42
```

For an 80 MHz VHT configuration with channel 36 as the primary channel, settings such as `vht_oper_chwidth=1` and `vht_oper_centr_freq_seg0_idx=42` would normally be involved.

In the current setup, these options are left disabled and the AP is configured to use **HT40 rather than VHT80**.

It is also important to note that:

```ini
ieee80211ac=1
```

does not by itself mean that the AP will operate with an 80 MHz channel width.

Enabling 802.11ac/VHT capability and selecting an 80 MHz operating channel width are separate parts of the configuration.

---

# WPA2-Personal Only

The current security configuration is:

```ini
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

An earlier version of the configuration used:

```ini
wpa_key_mgmt=WPA-PSK SAE
```

to provide WPA2/WPA3 transition mode.

The current configuration no longer enables SAE. The AP therefore uses **WPA2-Personal (WPA-PSK) only**.

`wpa=2` enables RSN, commonly referred to as WPA2 in this configuration, while `wpa_key_mgmt=WPA-PSK` selects pre-shared-key authentication.

---

# AES (CCMP) Only

```ini
rsn_pairwise=CCMP
```

CCMP is used as the pairwise cipher.

TKIP is not enabled, resulting in a conventional WPA2-Personal configuration using CCMP.

---

# WPA3 / SAE Is Currently Disabled

WPA3-Personal using SAE is no longer enabled.

The configuration file still contains several commented-out options for testing or future use:

```ini
# wpa_key_mgmt=WPA-PSK SAE
# wpa_key_mgmt=SAE
# sae_pwe=2
```

Because these options are commented out, the current AP does not advertise or accept WPA3-Personal/SAE authentication.

Likewise, `sae_pwe` has no effect while SAE is disabled.

---

# PMF Is Disabled

The current configuration uses:

```ini
ieee80211w=0
```

`ieee80211w` controls Protected Management Frames (PMF), also known as IEEE 802.11w.

The values are generally:

```plaintext
0 = disabled
1 = optional
2 = required
```

The previous WPA2/WPA3 transition-mode configuration enabled PMF. Since WPA3/SAE is no longer being used, the current configuration explicitly disables PMF.

The configuration file retains the alternatives as comments:

```ini
# ieee80211w=1
# ieee80211w=2
ieee80211w=0
```

If WPA3-Personal/SAE is enabled again in the future, the PMF configuration will also need to be reconsidered to satisfy the applicable WPA3 requirements.

---

# PMKSA Caching Is Disabled

The current configuration also includes:

```ini
disable_pmksa_caching=1
```

PMKSA (Pairwise Master Key Security Association) caching allows previously established authentication state to be reused, reducing the work required during subsequent authentication.

Setting:

```ini
disable_pmksa_caching=1
```

disables this cache.

Disabling PMKSA caching is not generally required for a normal access point. In this setup, it is disabled to simplify authentication behavior and eliminate PMKSA caching as a variable when troubleshooting client compatibility and reconnection behavior.

---

# Hidden SSID

```ini
ignore_broadcast_ssid=1
```

This configures the AP as a so-called hidden SSID network.

Hiding an SSID should not be considered a meaningful security measure. It can also have several disadvantages:

* Clients may require manual network configuration.
* Some devices may have connectivity issues.
* Troubleshooting can become more complicated.

In this setup, the hidden SSID is a configuration choice for the intended use case rather than a security mechanism.

---

# Wireless Interface

```ini
interface=wlp2s0
```

This specifies the wireless interface that `hostapd` uses for AP mode.

The configuration file also retains an interface used during testing:

```ini
# interface=wlx3476c5d38aef
```

but the currently active interface is:

```ini
interface=wlp2s0
```

---

# Starting hostapd

To start `hostapd` with verbose debugging enabled:

```bash
sudo hostapd -dd /etc/hostapd/hostapd.conf
```

When startup succeeds, you should see output similar to:

```plaintext
wlp2s0: AP-ENABLED
```

When investigating channel-width or driver-related issues, running `hostapd` directly with `-dd` can be particularly useful because it exposes detailed information about the interaction between `hostapd`, `nl80211`, and the wireless driver.

---

# HT40 / VHT80 Troubleshooting

During the initial setup, attempts to use 40 MHz and 80 MHz configurations resulted in errors such as:

```plaintext
Could not set channel for kernel driver
```

and:

```plaintext
80/80+80 MHz: no second channel offset
```

Because of these errors, an earlier version of the setup removed the HT40 and VHT80 options entirely and operated with a 20 MHz channel width.

After revisiting the configuration and environment, the current setup now works with:

```ini
hw_mode=a
channel=36

ieee80211n=1
ieee80211ac=1

ht_capab=[HT40+]
```

The situation can therefore be summarized as follows:

* 20 MHz only — **previous configuration**
* HT40 — **currently enabled**
* VHT80 — **currently disabled**

One important lesson from this setup is that the capabilities reported by `iw list` do not necessarily guarantee that every corresponding channel configuration can be used in AP mode.

For example, `iw list` may report:

```plaintext
HT20/HT40
VHT Capabilities
```

but actual AP operation still depends on the wireless driver, firmware, regulatory domain, channel configuration, and other hardware or software constraints.

The most reliable way to determine whether a particular configuration works is therefore to test it with `hostapd` and inspect the detailed debug output.

---

# Final Configuration

The current setup consists of:

* 5 GHz operation
* Channel 36
* **40 MHz channel width (HT40+)**
* 802.11n enabled
* 802.11ac enabled
* **WPA2-Personal (WPA-PSK) only**
* AES (CCMP) only
* **WPA3/SAE disabled**
* **PMF disabled**
* PMKSA caching disabled
* Bridged networking through `br0`
* Hidden SSID

The earlier configuration used a 20 MHz channel width because HT40/VHT80 prevented the AP from starting, while WPA2/WPA3 transition mode and PMF were enabled.

The current configuration takes a different approach: **HT40 is enabled for 40 MHz operation, while authentication has been simplified to WPA2-PSK only**.

On a MacBookPro14,2 running Ubuntu 24.04 with `hostapd`, the capabilities shown by `iw list` should not be treated as proof that a particular AP channel configuration will work. In practice, testing the configuration with `hostapd -dd` and verifying what the `nl80211` driver actually accepts is the most reliable approach.
