---
pubDatetime: 2026-02-14T17:33:19+09:00
title: "Configuring Network Disconnection During Modern Standby in Windows 11"
description: "This article provides a comprehensive guide to disabling network connectivity during Modern Standby in Windows 11. It covers the background, steps, verification methods, and precautions, focusing on disabling 'Network Connections During Standby' through Group Policy (for Pro/Enterprise editions), checking if the system uses Modern Standby (S0), and potential side effects."
---

In Windows 11, **Modern Standby (S0 Low Power Idle)** maintains network connectivity even during sleep to enable notifications and synchronization.

If you want to **disable network connectivity during sleep** or **prevent background communication at night**, configuring the system to **not allow network connections during Modern Standby** is effective.

This article explains how to do this, covering **Windows 11 Pro / Enterprise**, and also providing instructions for **Windows 11 Home Edition**.

## What This Article Covers

- Disabling **network connectivity** during Modern Standby (S0).
- Controlling both when connected to AC power and when running on battery.
- Configuring the setting via the Registry in Windows 11 Home Edition.
- Verifying that the PC supports Modern Standby after configuration.

## Disabling 'Network Connections During Standby' Using Group Policy (for Pro / Enterprise)

In Windows 11 Pro / Enterprise, you can configure this through the **Local Group Policy Editor**.

### Steps

1. Press `Win + R` to open **Run**.
2. Enter `gpedit.msc` and launch the **Local Group Policy Editor**.
3. Navigate to the following location:

**Computer Configuration**  
→ **Administrative Templates**  
→ **System**  
→ **Power Management**  
→ **Sleep Settings**

4. Find and set the following policies to **"Disabled"**:

- **Allow network connectivity during connected standby (plugged in)**
- **Allow network connectivity during connected standby (on battery)**

5. Apply the settings:

Open Command Prompt as an administrator and run the following command:

```bat
gpupdate /force
```

Alternatively, restart Windows.

---

## How to Configure in Windows 11 Home Edition

Since Windows 11 Home Edition typically doesn't include `gpedit.msc`, you'll need to configure the equivalent settings **directly through the Registry**.

### Method 1: Configuring with Commands

Open **Windows Terminal (Administrator)** or **Command Prompt (Administrator)** and execute the following two commands:

```bat
reg add "HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9" /v ACSettingIndex /t REG_DWORD /d 0 /f

reg add "HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9" /v DCSettingIndex /t REG_DWORD /d 0 /f
```

After applying the settings, **restart Windows**.

Here's what each command does:

- `ACSettingIndex = 0`: Disables network connectivity during Modern Standby when connected to AC power.
- `DCSettingIndex = 0`: Disables network connectivity during Modern Standby when running on battery.

This is equivalent to disabling both "plugged in" and "on battery" settings in the Group Policy for Pro / Enterprise.

### Method 2: Configuring with Registry Editor

If you prefer a GUI, use `regedit`.

1. Press `Win + R`.
2. Enter `regedit` and launch the **Registry Editor**.
3. Navigate to the following key:

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9
```

If the `Power`, `PowerSettings`, or GUID keys don't exist, create them.

4. Create the following **DWORD (32-bit) Value** within the GUID key:

```text
ACSettingIndex
DCSettingIndex
```

5. Set both values to `0`.
6. Restart Windows.

### Reverting the Settings in Home Edition

To undo the policy enforcement, delete the created values.

In the administrator Command Prompt, run the following:

```bat
reg delete "HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9" /v ACSettingIndex /f

reg delete "HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9" /v DCSettingIndex /f
```

Then, restart Windows.

---

## Verifying if Modern Standby (S0) is Enabled

Before configuring, verify that your PC uses Modern Standby.

In Command Prompt or Windows Terminal, run the following:

```bat
powercfg /a
```

In the output, check if the following is listed under "The following sleep states are available on this system":

```text
Standby (S0 Low Power Idle)
```

If it is, your PC supports Modern Standby.

Depending on your environment, you might see:

```text
Standby (S0 Low Power Idle) Network Connected
```

or a similar network-related variation.

## Verifying the Configuration

Keep in mind that Modern Standby behavior is also influenced by the PC manufacturer, NIC, drivers, and firmware. Simply setting the Registry or Policy doesn't guarantee that the NIC will completely shut down.

After putting the PC to sleep, you can use the following command to verify its behavior during Modern Standby:

```bat
powercfg /sleepstudy
```

This generates a **SleepStudy Report**.

Use this to check if there is any unintended network communication or background activity occurring during sleep.

## Precautions (Side Effects)

- **Notifications, synchronization, email reception, and background communication** during sleep may be stopped or limited.
- Behavior may vary depending on the PC manufacturer, NIC, drivers, and firmware.
- This setting does not disable Modern Standby altogether and switch to the traditional S3 sleep state.
- Be careful when modifying the Registry to avoid errors.

## Summary

If you want to suppress network communication during Modern Standby in Windows 11, the configuration method depends on your Windows edition.

**Windows 11 Pro / Enterprise**

Set the following two items to **"Disabled"** in Group Policy:

- **Allow network connectivity during connected standby (plugged in)**
- **Allow network connectivity during connected standby (on battery)**

**Windows 11 Home Edition**

Since the Group Policy Editor is not included by default, use the following Registry policy:

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Power\PowerSettings\f15576e8-98b7-4186-b944-eafa664402d9
```

Set the following:

```text
ACSettingIndex = 0
DCSettingIndex = 0
```

This will disable network connectivity during Modern Standby when connected to AC power and when running on battery.
