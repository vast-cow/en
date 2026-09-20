---
title: "Using Raspberry Pi 4, Raspberry Pi OS, and Docker to Connect Home Assistant with a Eufy IndoorCam 2K Pan & Tilt and Save Images When a Pet Is Detected"
description: "Deploy Home Assistant and an Eufy websocket bridge on a Raspberry Pi, configure HACS integration, and save timestamped pet-event images with authentication and automation troubleshooting."
pubDatetime: 2026-04-23T08:00:06.700Z
updatedDate: 2026-04-23T08:14:32.401Z
---

I wanted to connect an **Anker Eufy IndoorCam 2K Pan & Tilt** to **Home Assistant** and automatically **save an image whenever my pet is detected**.

This article explains how to build that setup on a **Raspberry Pi 4 running Raspberry Pi OS with Docker**.

I’ll start with the practical setup steps needed to get everything working, then summarize the common issues and questions that tend to come up along the way.

---

## What this setup does

The goal is straightforward. We want to create the following flow:

1. Run **Home Assistant** in Docker on a Raspberry Pi 4
2. Run a separate **websocket bridge** for Eufy
3. Install the **Eufy Security custom integration** in Home Assistant
4. Create an **automation** that saves the camera’s Event Image whenever a pet is detected

The overall structure looks like this:

```text
Raspberry Pi OS 64-bit
  └─ Docker / Docker Compose
      ├─ Home Assistant Container
      └─ eufy-security-ws

Home Assistant
  ├─ HACS
  ├─ Eufy Security custom integration
  └─ automation
       petDetected → save Event Image as snapshot
```

---

## Conclusion first

Yes, this setup works on **Raspberry Pi 4 + Raspberry Pi OS + Docker**.

The main points are:

* Use **Raspberry Pi OS 64-bit**
* Run Home Assistant as a **container**
* Use the **`eufy_security` custom integration**, not the standard integration
* Run **`eufy-security-ws`** as a separate container
* Save the pet-detection image by calling **`image.snapshot`** on the camera’s **Event Image**

---

# Step-by-step setup

## 1. Prepare Raspberry Pi OS

Install **Raspberry Pi OS 64-bit** on your Raspberry Pi 4.

I recommend using the 64-bit version rather than 32-bit. It is the safer choice for Docker and for future maintenance.

Then update the OS:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

After rebooting, verify that the system is running in 64-bit mode:

```bash
uname -m
```

If the output is `aarch64`, you are on 64-bit.

---

## 2. Install Docker

Install Docker and make sure Docker Compose is available:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
```

---

## 3. Create a working directory

Create directories for Home Assistant and the Eufy bridge data:

```bash
mkdir -p ~/ha-stack
cd ~/ha-stack

mkdir -p homeassistant/config
mkdir -p homeassistant/media/eufy_snapshots
mkdir -p eufy-security-ws
```

---

## 4. Create `compose.yaml`

In this setup, Home Assistant and `eufy-security-ws` run as separate containers.

Create `~/ha-stack/compose.yaml`:

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      TZ: Asia/Tokyo
    volumes:
      - ./homeassistant/config:/config
      - ./homeassistant/media:/media
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro

  eufy-security-ws:
    container_name: eufy-security-ws
    image: bropat/eufy-security-ws:latest
    restart: unless-stopped
    network_mode: host
    environment:
      TZ: Asia/Tokyo
      USERNAME: 'your-eufy-mail@example.com'
      PASSWORD: 'your-password'
      COUNTRY: 'JP'
      TRUSTED_DEVICE_NAME: 'rpi4-ha'
      EVENT_DURATION_SECONDS: '10'
    volumes:
      - ./eufy-security-ws:/data
```

Start the containers:

```bash
docker compose up -d
```

Home Assistant should then be available at:

```text
http://<your-pi-ip>:8123
```

---

## 5. Allow Home Assistant to write snapshots

Since the pet-detection images will be saved under `/media/eufy_snapshots`, Home Assistant needs permission to write there.

Add the following to `~/ha-stack/homeassistant/config/configuration.yaml`:

```yaml
homeassistant:
  allowlist_external_dirs:
    - /media/eufy_snapshots
```

This is important: **do not overwrite your existing `configuration.yaml`**.

If the file already contains a `homeassistant:` block, just add `allowlist_external_dirs` inside it.

After that, restart Home Assistant:

```bash
docker compose restart homeassistant
```

---

## 6. Install HACS

Because `eufy_security` is not a built-in integration, you need **HACS**.

Install it from inside the Home Assistant container:

```bash
docker exec -it homeassistant bash
wget -O - https://get.hacs.xyz | bash -
exit
```

Then restart Home Assistant:

```bash
docker compose restart homeassistant
```

After the restart, go through the HACS setup in the Home Assistant UI:

* **Settings → Devices & Services**
* **Add Integration**
* **HACS**
* Complete the GitHub device login flow

---

## 7. Configure the `eufy-security-ws` login settings

This part is critical.

If `docker logs -f eufy-security-ws` shows:

```text
Missing one of USERNAME or PASSWORD
```

then your Eufy credentials are not being passed into the container.

Make sure the following variables are present in the `eufy-security-ws` section of `compose.yaml`:

* `USERNAME`
* `PASSWORD`
* `COUNTRY`
* `TRUSTED_DEVICE_NAME`
* `EVENT_DURATION_SECONDS`

After making changes, recreate or restart the container:

```bash
docker compose up -d
docker logs -f eufy-security-ws
```

---

## 8. Be careful if your password contains `$`

Docker Compose may interpret `$` as variable expansion.

If your password contains dollar signs, do not write it naively. A safer approach is to wrap it in single quotes and escape as needed with `$$`.

For example:

```yaml
PASSWORD: 'pa$$word$$with$$dollar'
```

This is an easy place to get stuck.

---

## 9. Install the Eufy Security integration from HACS

Once HACS is available:

* Go to **HACS → Integrations**
* Search for **Eufy Security**
* Install it
* Restart Home Assistant

Then add the integration:

* **Settings → Devices & Services**
* **Add Integration**
* **Eufy Security**

For the connection settings, use:

* **Host:** `127.0.0.1`
* **Port:** `3000`

---

## 10. Check the Eufy app settings

Even if Home Assistant is configured correctly, you may not receive pet-detection events unless the Eufy app itself is configured properly.

In the Eufy app, verify the following:

* **Pet Detection** is enabled
* **Push notifications** are enabled
* Shared account settings are configured if needed

---

## 11. Confirm the created entities

If the integration is working, Home Assistant should create entities like these for your camera:

```text
binary_sensor.living_room_pet_detected
image.living_room_event_image
```

The exact entity names will vary depending on your environment, so check them in the Home Assistant UI:

* **Settings → Devices & Services**
* Open the Eufy camera device
* Review the entity list

---

## 12. Create `automations.yaml`

You can save an image when a pet is detected with an automation like this:

```yaml
alias: Eufy pet detection posts image to Slack
description: ""
triggers:
  - trigger: state
    entity_id:
      - image.living_room_event_image
conditions:
  - condition: state
    entity_id: binary_sensor.living_room_pet_detected
    state:
      - "on"
actions:
  - variables:
      ts: "{{ now().strftime('%Y%m%d-%H%M%S') }}"
      file: /media/eufy_snapshots/pet_{{ ts }}.jpg
  - action: image.snapshot
    metadata: {}
    target:
      entity_id: image.living_room_event_image
    data:
      filename: "{{ file }}"
```

Replace the `entity_id` values with the ones from your own environment.

---

## 13. Apply the automation changes

If you edit `automations.yaml` directly, saving the file is **not enough**. Home Assistant will not automatically reload it.

You have two options.

### Option 1: Reload automations from the UI

* Go to **Developer Tools**
* Go to **Actions**
* Run `automation.reload`

### Option 2: Restart Home Assistant

```bash
docker compose restart homeassistant
```

---

## 14. Verify that the automation is active

You can confirm that the automation has been loaded in several ways:

1. Check whether the alias appears under **Settings → Automations & Scenes → Automations**
2. Open the automation and see whether you can click **Run**
3. Review the **Trace**
4. Confirm that image files are actually being created in the target directory

To check the saved files from the shell:

```bash
ls -lh ~/ha-stack/homeassistant/media/eufy_snapshots
```

---

## 15. Test the full flow

Finally, test the actual detection flow:

1. Let your pet move in front of the camera
2. Confirm that `binary_sensor.xxx_pet_detected` changes to `on`
3. Confirm that `image.xxx_event_image` updates
4. Confirm that a JPEG file is saved under `/media/eufy_snapshots`

At that point, the setup is complete.

---

# Common questions and troubleshooting

## Can this really be done with Raspberry Pi 4, Raspberry Pi OS, and Docker?

Yes.

The main assumption is that you are using:

* **Home Assistant Container**, not Home Assistant OS
* A separate **`eufy-security-ws`** container
* The **`eufy_security`** integration installed through HACS

So the structure is a little different from a Home Assistant OS setup that relies on add-ons.

---

## Should I append to `configuration.yaml` or replace it?

Append to it. Do not replace it.

Most Home Assistant installations already have settings in `configuration.yaml`. In this case, you only need to add `allowlist_external_dirs`.

If there is no existing `homeassistant:` block:

```yaml
homeassistant:
  allowlist_external_dirs:
    - /media/eufy_snapshots
```

If there is already a `homeassistant:` block, add it inside that section:

```yaml
homeassistant:
  name: Home
  time_zone: Asia/Tokyo
  allowlist_external_dirs:
    - /media/eufy_snapshots
```

---

## HACS does not appear under Add Integration

The most common causes are:

1. Home Assistant was not restarted fully
2. Browser cache is getting in the way
3. The HACS files were not installed correctly

First, restart Home Assistant:

```bash
docker compose restart homeassistant
```

Then hard-refresh the browser:

* **Windows / Linux:** `Ctrl + F5`
* **Mac:** `Cmd + Shift + R`

You can also verify that the HACS files exist:

```bash
ls ~/ha-stack/homeassistant/config/custom_components/hacs
```

---

## `eufy-security-ws` says `Missing one of USERNAME or PASSWORD`

That means the Eufy credentials are not reaching the container.

Make sure the `environment:` section includes:

```yaml
environment:
  TZ: Asia/Tokyo
  USERNAME: 'your-eufy-mail@example.com'
  PASSWORD: 'your-password'
  COUNTRY: 'JP'
  TRUSTED_DEVICE_NAME: 'rpi4-ha'
  EVENT_DURATION_SECONDS: '10'
```

---

## What if the password contains `$`?

Be careful.

Docker Compose treats `$` specially, so passwords containing dollar signs can break unless written correctly.

For example:

```yaml
PASSWORD: 'pa$$word$$with$$dollar'
```

This is one of the most common small but frustrating issues in the setup.

---

## How can I tell whether `automations.yaml` has been applied?

Use the following checklist:

1. Run `automation.reload`
2. Open **Settings → Automations & Scenes → Automations**
3. Confirm that your automation alias appears
4. Try running it manually
5. Check whether an image file is saved in the destination directory

The key point is that editing `automations.yaml` alone does not automatically apply the changes.

---

## Do I need live video streaming too?

Not for this use case.

If your goal is simply to **save an image when a pet is detected**, then using only the **Event Image** is lighter and easier to manage, especially on a Raspberry Pi 4.

Once you add live streaming or recording, the load and the number of moving parts both increase. For a first version, saving still images is the more practical approach.

---

## What should I keep in mind for real-world use?

A few recommendations:

* Use **Raspberry Pi OS 64-bit**
* Prefer an **SSD over microSD** for storage
* Start with **image saving only**
* Check both the **Home Assistant logs** and the **eufy-security-ws logs** when troubleshooting
* Do not forget the **notification settings** in the Eufy app

---

# Summary

It is entirely possible to use a Raspberry Pi 4 with Raspberry Pi OS and Docker to build a setup where **Home Assistant receives pet-detection events from a Eufy IndoorCam 2K Pan & Tilt and saves an image automatically**.

The three most important pieces are:

* Run Home Assistant as a **container**
* Use **`eufy-security-ws` + `eufy_security`** for the Eufy integration
* Save the image by calling **`image.snapshot`** on the **Event Image**

The initial setup can be slightly tricky around HACS, authentication settings, and reloading `automations.yaml`, but once those are in place, the overall system is fairly straightforward.

For a Raspberry Pi 4, this is a practical and realistic setup if your goal is simply to **save a still image whenever your pet is detected**.
