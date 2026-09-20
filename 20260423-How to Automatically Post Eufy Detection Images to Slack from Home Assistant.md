---
title: "How to Automatically Post Eufy Detection Images to Slack from Home Assistant"
description: "If you already have Eufy integrated with Home Assistant and can access the camera’s event image, the..."
pubDatetime: 2026-04-23T07:25:35.834Z
updatedDate: 2026-04-23T08:11:29.086Z
---

If you already have Eufy integrated with Home Assistant and can access the camera’s event image, the next step is straightforward: save that image and send it to a Slack channel automatically.

This article assumes the following is already in place:

* Eufy is connected to Home Assistant
* The camera exposes an `image.xxx_event_image` entity
* Detection-related entities such as `binary_sensor.xxx_pet_detected` are available if needed
* Eufy image retrieval itself is already working 

The goal is simple:

1. Create a Slack app
2. Add the Slack integration to Home Assistant
3. Extend your existing automation so it posts the saved image to Slack
4. Avoid the common configuration mistakes that tend to break file uploads

---

# Step-by-Step Setup

## 1. Understand the overall flow

At a high level, the automation looks like this:

```text
Eufy event image updates
  → Home Assistant automation runs
  → Image is saved to /media/eufy_snapshots
  → The saved image is uploaded to Slack
```

Since image retrieval is already working, the only new part is the Slack side.
Your existing setup already covers the image snapshot flow and local storage under `/media/eufy_snapshots`. 

---

## 2. Create a Slack app

To let Home Assistant post messages and upload files, you need a Slack app with a bot user.

Create a new app in the Slack API dashboard and attach it to the workspace where you want notifications to appear.

Then open **OAuth & Permissions** and configure the bot token scopes.

### Recommended bot scopes

If you want both text messages and file attachments to work reliably, add these scopes:

* `chat:write`
* `chat:write.public`
* `chat:write.customize`
* `files:write`
* `channels:read`
* `groups:read`
* `mpim:read`
* `im:read`

Text-only notifications may work with fewer permissions, but file uploads often fail unless the read-related scopes are also present. In this setup, Home Assistant may need to resolve the channel ID before uploading the file.

---

## 3. Reinstall the app to the workspace

This step is easy to miss.

After changing scopes, reinstall the Slack app to the workspace so the updated permissions actually take effect. Without reinstalling, the new scopes may not be reflected in the token.

---

## 4. Copy the Bot User OAuth Token

Once the app is installed, Slack will issue a **Bot User OAuth Token**, typically starting with `xoxb-`.

Keep that token handy. You will use it as the API key in Home Assistant.

---

## 5. Invite the bot to the destination channel

If you want to post into a channel such as `#pet-camera`, make sure the bot has been added to that channel.

For example:

```text
/invite @your-bot-name
```

If the bot is not in the channel, posting may fail even if the rest of the setup is correct.

---

## 6. Add the Slack integration in Home Assistant

Now switch to Home Assistant.

Go to:

* **Settings**
* **Devices & Services**
* **Add Integration**
* **Slack**

When prompted, enter:

* **API Key**
  → the `xoxb-...` Bot User OAuth Token

* **Default Channel**
  → for example `#pet-camera`

* **Username / Icon**
  → optional

Once configured, Home Assistant will create a notify service such as `notify.xxx`, which you will use in your automation.

---

## 7. Find the actual `notify.xxx` service name

Do not guess the notify service name.

Open **Developer Tools → Actions** and type `notify.` to see the exact service name Home Assistant generated for your Slack workspace.

This matters more than it sounds. If your Slack workspace name includes Japanese or Chinese characters, the resulting `notify.xxx` service name may not be the romanization you expect. In practice, it can end up looking like a transliterated form you would not guess correctly by hand.

Always verify the real service name in Home Assistant before using it in YAML.

---

## 8. Confirm that the snapshot directory is allowlisted

Since you are attaching a local file, Home Assistant must be allowed to access the directory where the image is stored.

Your existing image-save setup already uses this path: `/media/eufy_snapshots`. 

Make sure `configuration.yaml` includes:

```yaml
homeassistant:
  allowlist_external_dirs:
    - /media/eufy_snapshots
```

If you add or change this setting, restart Home Assistant afterward.

---

## 9. Test a plain text Slack notification first

Before touching your automation, verify that basic Slack notifications work.

In **Developer Tools → Actions**, call your Slack notify service with something like:

```yaml
message: "Test notification"
title: "Home Assistant Test"
target: "#pet-camera"
```

If this works, authentication and basic Slack delivery are fine.

---

## 10. Test a fixed image upload

Next, test file upload with a known file path before introducing templates or variables.

Use a simple payload like this:

```yaml
message: "Image upload test"
title: "test image"
data:
  file:
    path: /media/eufy_snapshots/test.jpg
```

At first, it can be helpful to omit `target` and let the message go to the integration’s default channel. That removes one more variable from the test.

If this succeeds, then the file upload mechanism itself is working.

---

## 11. Add Slack upload to your automation

Once the fixed upload test works, extend your existing automation.

Here is a complete example:

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
  - delay:
      hours: 0
      minutes: 0
      seconds: 2
      milliseconds: 0
  - action: notify.YOUR_SLACK_SERVICE
    data:
      message: ""
      data:
        file:
          path: "{{ file }}"
```

This automation does four things:

1. Waits for the Eufy event image to update
2. Checks whether the pet detection sensor is on
3. Saves the event image to a timestamped JPEG file
4. Uploads that saved file to Slack

The image-save portion follows the same event image approach already established in the existing Eufy/Home Assistant setup. 

---

## 12. Reload or restart Home Assistant

If you edited `automations.yaml` directly, the automation will not become active until you reload automations or restart Home Assistant.

You can either:

* Run `automation.reload` from **Developer Tools → Actions**
* Restart Home Assistant

---

## 13. Verify end-to-end behavior

Finally, test the full flow:

1. Trigger a real detection event
2. Confirm that `binary_sensor.xxx_pet_detected` changes to `on`
3. Confirm that `image.xxx_event_image` updates
4. Confirm that a JPEG appears in `/media/eufy_snapshots`
5. Confirm that Slack receives the message with the attached image

To inspect saved files on disk:

```bash
ls -lh ~/ha-stack/homeassistant/media/eufy_snapshots
```

---

# Common Pitfalls and Troubleshooting

## File uploads fail, but text messages work

This is one of the most common failure modes.

If plain text Slack messages succeed but file uploads fail, the problem is usually not authentication. It is usually one of these:

* missing Slack scopes
* missing `files:write`
* missing read scopes for channel resolution
* the bot is not in the target channel
* the local file path is wrong
* the directory is not allowlisted

A particularly common error is `missing_scope` when Home Assistant tries to resolve the channel ID through Slack. In that case, adding the following scopes usually fixes it:

* `channels:read`
* `groups:read`
* `mpim:read`
* `im:read`

In this setup, adding `files:write` and reinstalling the app to the workspace resolved the issue.

---

## Do not guess the `notify.xxx` service name

If your Slack workspace name contains Japanese or other non-Latin characters, the generated notify service name may not match what you intuitively expect.

Check the exact name under **Developer Tools → Actions** rather than typing it from memory.

This avoids a surprisingly frustrating class of errors.

---

## `Message malformed: extra keys not allowed @ data['0']`

This error usually has nothing to do with Slack itself.

It almost always means the YAML structure is wrong for the place where you pasted it.

Typical causes include:

* pasting a full automation where only an action payload is expected
* pasting a list-based block with a leading `-` into a single-object YAML editor
* mixing up Developer Tools action YAML and full automation YAML

For example, in a single automation editor, this often causes trouble:

```yaml
- alias: ...
  trigger:
    ...
```

In that context, the editor usually expects:

```yaml
alias: ...
trigger:
  ...
condition:
  ...
action:
  ...
```

Likewise, if you are testing only the Slack call in Developer Tools, use a single action payload rather than a full automation.

---

## Test with a fixed file before using templates

Do not start with the full templated automation if file uploads are failing.

First test with a known-good file path:

```yaml
message: "Image upload test"
title: "test image"
data:
  file:
    path: /media/eufy_snapshots/test.jpg
```

Once that works, switch to `{{ file }}` in the automation.

This makes troubleshooting much easier.

---

## Omit `target` at first if necessary

Channel resolution introduces another possible point of failure.

If uploads are still failing, temporarily remove `target` and let the Slack integration post to its configured default channel. Once that works, add the explicit `target` back.

---

## Add a short delay after saving the image

Uploading immediately after saving sometimes creates a race condition, especially on smaller systems.

A short delay like this improves reliability:

```yaml
- delay:
    seconds: 2
```

It is a small detail, but it helps.

---

# Final Thoughts

If Eufy event images are already available in Home Assistant, posting them to Slack is a relatively small next step.

The core work is:

1. create a Slack app
2. grant the right scopes
3. connect Slack to Home Assistant
4. upload the saved event image from your automation

The main things that tend to trip people up are not the automation logic itself, but the surrounding details:

* missing Slack scopes for file uploads
* unexpected `notify.xxx` service names
* YAML pasted into the wrong place
* file path and allowlist issues

Once those are handled, the setup is quite clean: Eufy detection image in, Slack notification out.
