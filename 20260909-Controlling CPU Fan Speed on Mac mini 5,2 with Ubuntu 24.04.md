---
title: "Controlling CPU Fan Speed on Mac mini 5,2 with Ubuntu 24.04"
description: "If you have a Mac mini 5,2 (Mid 2011) running Ubuntu 24.04, a convenient setup is to use applesmc +..."
pubDatetime: 2026-09-09T01:54:03.910Z
---

If you have a Mac mini 5,2 (Mid 2011) running Ubuntu 24.04, a convenient setup is to use **`applesmc` + `mbpfan`**. `mbpfan 2.4.0` is also included as a standard package in Ubuntu 24.04 (Noble). [Ubuntu Packages][1]

The Linux `applesmc` driver provides Intel Mac temperature sensors and fan control via sysfs. [GitHub][2]

## 1. First, check `applesmc`

```bash
sudo modprobe applesmc
sudo modprobe coretemp

lsmod | grep -E 'applesmc|coretemp'
```

Next, check the fan interface.

```bash
SMC=$(find /sys/devices/platform -maxdepth 1 -type d -name 'applesmc.*' -print -quit)
echo "$SMC"

ls -l "$SMC"/fan*
```

For example, if you see something like this, you can control it.

```text
fan1_input
fan1_manual
fan1_max
fan1_min
fan1_output
```

Be sure to check `fan1_min` and `fan1_max`.

```bash
cat "$SMC/fan1_min"
cat "$SMC/fan1_max"
cat "$SMC/fan1_input"
```

The values will vary depending on the Mac, so it is safer **not to assume values like 1800-5500 RPM**.

---

## 2. Manually change the rotation speed

You can first test it temporarily at, for example, 3000 RPM.

```bash
echo 1 | sudo tee "$SMC/fan1_manual"
echo 3000 | sudo tee "$SMC/fan1_output"
```

Current actual rotation speed:

```bash
cat "$SMC/fan1_input"
```

After a few seconds, it should be around 3000 RPM.

On Intel Macs in Linux, the method of setting `fan1_manual=1` and then writing the target RPM to `fan1_output` is used. [Ask Ubuntu][3]

### Return to automatic control

This is important.

```bash
echo 0 | sudo tee "$SMC/fan1_manual"
```

If you leave it in manual mode and fix it at a low rotation speed, the rotation speed will not increase even when the CPU/GPU load increases.

---

## 3. Recommended: Use `mbpfan` to link to temperature

On Ubuntu 24.04,

```bash
sudo apt update
sudo apt install mbpfan lm-sensors
```

Enable:

```bash
sudo systemctl enable --now mbpfan
```

Check status:

```bash
systemctl status mbpfan
```

`mbpfan` reads the CPU temperature from `coretemp` and changes the fan speed via `applesmc`. [GitHub][4]

Configuration:

```bash
sudo nano /etc/mbpfan.conf
```

For example, for Mac mini 5,2, I would first try the following as a gentle setting.

```ini
[general]

low_temp = 55
high_temp = 65
max_temp = 80

polling_interval = 1
```

For the fan's min/max, `mbpfan` can use the values reported by `applesmc`. If you want to specify it explicitly,

```bash
cat "$SMC/fan1_min"
cat "$SMC/fan1_max"
```

and then

```ini
min_fan1_speed = 2000
max_fan1_speed = 5500
```

Set it using the **actual values of your machine**. `mbpfan`'s default setting also specifies that it refers to `/sys/devices/platform/applesmc.768/fan*_min` and `fan*_max`. [GitHub][5]

After changing:

```bash
sudo systemctl restart mbpfan
```

---

## 4. Monitor temperature and RPM in real time

```bash
sensors
```

Or,

```bash
watch -n 1 sensors
```

If you want to see the fan RPM directly:

```bash
watch -n 1 "cat $SMC/fan1_input; sensors"
```
