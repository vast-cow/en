---
title: "How to Disable Network Connectivity During Modern Standby in Windows 11"
description: "Configure plugged-in and battery Group Policy settings to reduce network activity during S0 sleep, verify Modern Standby support, and account for hardware-dependent behavior."
pubDatetime: 2026-02-14T08:35:22.345Z
---

In Windows 11, **Modern Standby (S0 Low Power Idle)** may keep the network connected during sleep to allow notifications and background synchronization.

If you prefer the network to be fully disconnected while the device is sleeping — for example, to stop overnight background traffic — you can configure the system to shift from **Connected Standby** behavior toward a **Disconnected** state.

This article explains how to disable network connectivity during Modern Standby in Windows 11.



## What This Guide Covers

* Disabling **network connectivity during Modern Standby (S0)**
* Configuring settings for both **plugged-in** and **battery** modes
* Verifying that your device supports Modern Standby



## Disable Network Connectivity During Standby via Group Policy (Pro / Enterprise)

The most reliable method is through **Local Group Policy Editor**, available in Windows 11 Pro and Enterprise editions.

### Steps

1. Press `Win + R` to open **Run**

2. Type `gpedit.msc` and launch **Local Group Policy Editor**

3. Navigate to:

   ```
   Computer Configuration
     → Administrative Templates
       → System
         → Power Management
           → Sleep Settings
   ```

4. Locate the following policies and set both to **Disabled**:

   * **Allow network connectivity during connected-standby (plugged in)**
   * **Allow network connectivity during connected-standby (on battery)**

5. Apply the changes:

   Open Command Prompt as Administrator and run:

   ```bat
   gpupdate /force
   ```

   Alternatively, restart the system.



## Expected Behavior

After disabling these policies:

* Modern Standby shifts from **Connected (network maintained)**
  to **Disconnected (network suspended)**
* Background synchronization and sleep-time network activity are reduced or stopped



## Verify That Your Device Uses Modern Standby (S0)

Before applying these settings, confirm that your device supports Modern Standby.

Open Command Prompt as Administrator and run:

```bat
powercfg /a
```

If the output lists:

```
Standby (S0 Low Power Idle)
```

your device is using Modern Standby.



## Considerations

* Notifications, email synchronization, and background updates may stop while the device is asleep.
* Some OEM firmware, NIC drivers, or hardware implementations may partially maintain connectivity despite the policy change.
* Behavior can vary depending on system firmware and network adapter capabilities.



## Summary

To disable network connectivity during Modern Standby in Windows 11, the most direct approach is to set the following two Group Policy settings to **Disabled**:

* **Allow network connectivity during connected-standby (plugged in)**
* **Allow network connectivity during connected-standby (on battery)**

This configuration is appropriate for users who want to minimize network activity during sleep.
