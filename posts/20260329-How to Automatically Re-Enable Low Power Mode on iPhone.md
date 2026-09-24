---
title: "How to Automatically Re-Enable Low Power Mode on iPhone"
description: "Combine iPhone Shortcuts and a personal automation to restore Low Power Mode when it turns off, using a local flag file and a separate toggle for manual control."
pubDatetime: 2026-03-29T07:54:15.908Z
---

## And Add a Shortcut to Toggle the Automation On and Off

Low Power Mode on iPhone is useful, but in some cases it can turn itself off automatically.
When that happens, turning it back on manually every time gets old fast.

A practical solution is to combine:

* an automation that turns **Low Power Mode back on whenever it gets turned off**
* a separate Shortcut that lets you **enable or disable that behavior whenever you want**

In this setup, a hidden file stored in **On My iPhone** is used as a flag to determine whether automatic re-enabling should run. The toggle Shortcut switches between `.lowpowermode-enabled` and `.lowpowermode-disabled`, and also turns Low Power Mode itself on or off at the same time. 
The re-enable Shortcut checks for `.lowpowermode-enabled`, and only turns Low Power Mode back on when that flag is active. It can also display a notification afterward. 

---

## What this article will build

By the end of this guide, you will have three parts working together.

### 1. A toggle Shortcut

This Shortcut lets you switch the behavior between:

* **enabled**
* **disabled**

Internally, it does that by renaming a flag file stored in **On My iPhone**.
When enabled, the file is `.lowpowermode-enabled`.
When disabled, the file is `.lowpowermode-disabled`. 

### 2. A re-enable Shortcut

This Shortcut is triggered by automation when Low Power Mode turns off.
It checks whether the enabled flag is present, and if so, it turns Low Power Mode back on. 

### 3. A personal automation

This automation runs when Low Power Mode is turned off and launches the re-enable Shortcut.

---

## How the system works

The logic is straightforward.

1. A toggle Shortcut changes whether the feature is enabled or disabled.
2. That state is stored using a flag file name.
3. Low Power Mode turns off.
4. A personal automation runs.
5. The re-enable Shortcut checks the flag.
6. If enabled, it turns Low Power Mode back on.

The key idea is that the automation itself remains in place all the time.
Whether it actually re-enables Low Power Mode is controlled by the flag file.

That is much easier to manage than editing the automation every time you want to pause or resume the behavior.

---

## Preparation

Before building the Shortcuts, create the flag file structure.

### Storage location

This setup uses **On My iPhone** as the storage location.
The uploaded Shortcut definitions also point to local storage under that location.

### Files used as flags

Create one of these files to start with:

* `.lowpowermode-enabled`
* `.lowpowermode-disabled`

If you want the auto-re-enable behavior to start in the active state, create `.lowpowermode-enabled` first.

### Important note

The file names need to match exactly.
If the names are off by even one character, the logic will fail.

---

## Part 1: Build the toggle Shortcut

Start by creating the Shortcut that enables or disables the entire behavior.

### What this Shortcut does

Based on the uploaded Shortcut definition, the flow works like this:

* Try to get `.lowpowermode-enabled`
* If that file is returned, rename it to `.lowpowermode-disabled`
* Turn Low Power Mode off
* Otherwise, get `.lowpowermode-disabled`
* Rename it to `.lowpowermode-enabled`
* Turn Low Power Mode on 

So this Shortcut handles two things at once:

* switching the flag state
* switching Low Power Mode itself

### How to create it

#### 1. Create a new Shortcut

Open the Shortcuts app and create a new Shortcut.

A few good names:

* `Low Power Mode Auto Toggle`
* `Toggle Low Power Mode Auto Restore`

#### 2. Add “Get File from Folder”

Add **Get File from Folder** as the first action.

Configure it like this:

* Location: `On My iPhone`
* File path: `.lowpowermode-enabled`
* Error if not found: Off

#### 3. Add an “If” action

Set the condition to:

* **If File has any value**

This checks whether `.lowpowermode-enabled` was successfully returned.

#### 4. In the “If” branch

Add these actions:

1. **Rename File**
   `.lowpowermode-enabled` → `.lowpowermode-disabled`
2. **Set Low Power Mode**
   Off

This means the feature is currently enabled, so the Shortcut disables it. 

#### 5. In the “Otherwise” branch

Add these actions:

1. **Get File from Folder**
   `.lowpowermode-disabled`
2. **Rename File**
   `.lowpowermode-disabled` → `.lowpowermode-enabled`
3. **Set Low Power Mode**
   On

This means the feature is currently disabled, so the Shortcut enables it. 

### Result

Once finished, this Shortcut becomes a single tap toggle for the whole system:

* enabled → disabled
* disabled → enabled

It also keeps the actual Low Power Mode state synchronized with the flag state. 

---

## Part 2: Build the re-enable Shortcut

Now create the Shortcut that turns Low Power Mode back on when needed.

### What this Shortcut does

According to the uploaded definition, its logic is:

* get `.lowpowermode-enabled`
* check it with an If condition
* if the file has a value, turn Low Power Mode on
* show a notification saying: **“Turned Low Power Mode back on”** 

### How to create it

#### 1. Create a new Shortcut

Create another Shortcut in the Shortcuts app.

Suggested names:

* `Re-enable Low Power Mode`
* `Restore Low Power Mode`

#### 2. Add “Get File from Folder”

Add **Get File from Folder**.

Configure it like this:

* Location: `On My iPhone`
* File path: `.lowpowermode-enabled`

#### 3. Add an “If” action

Use this condition:

* **If File has any value**

This means the rest of the Shortcut only runs if the enabled flag is present.

#### 4. In the “If” branch

Add these actions:

1. **Set Low Power Mode**
   On
2. **Show Notification**
   Title: `Turned Low Power Mode back on`

That matches the uploaded Shortcut behavior. 

---

## Part 3: Create the personal automation

With both Shortcuts ready, the last step is connecting the re-enable Shortcut to a personal automation.

### Steps

#### 1. Open the Automation tab

In the Shortcuts app, go to **Automation**.

#### 2. Create a new personal automation

Tap **Create Personal Automation**.

#### 3. Choose “Low Power Mode” as the trigger

From the available triggers, select **Low Power Mode**.

#### 4. Set the trigger to “Is Turned Off”

This is the critical setting.
You want the automation to run **when Low Power Mode is turned off**.

#### 5. Add “Run Shortcut”

For the action, choose **Run Shortcut** and select your re-enable Shortcut, such as:

* `Re-enable Low Power Mode`

#### 6. Turn off confirmation prompts

Adjust these settings as needed:

* turn off **Ask Before Running**
* optionally adjust **Notify When Run**

Now, whenever Low Power Mode turns off, the automation runs the re-enable Shortcut automatically.

---

## How to use it

Daily use is simple.

### When you want auto re-enable active

Run the toggle Shortcut once.

That changes:

* `.lowpowermode-disabled` → `.lowpowermode-enabled`
* Low Power Mode → On 

### When you want to pause the behavior

Run the same toggle Shortcut again.

That changes:

* `.lowpowermode-enabled` → `.lowpowermode-disabled`
* Low Power Mode → Off 

So in practice, one Shortcut controls the entire system.

---

## Why this setup is useful

### You do not need to edit the automation every time

The automation can stay in place permanently.
You only change the flag state through the toggle Shortcut.

### The state is easy to understand

The file name tells you exactly what mode the system is in:

* `.lowpowermode-enabled` = active
* `.lowpowermode-disabled` = inactive

### It is easy to extend

Because the system is flag-based, it is also easy to expand later.
For example, you could:

* change the notification text
* add logging
* allow auto re-enable only during certain hours
* disable it while a Focus mode is active

---

## Things to watch out for

### Keep the file location consistent

Both the toggle Shortcut and the re-enable Shortcut need to read from the same place.
If one points to a different folder, the system will break.

### File names must be exact

Use these names exactly:

* `.lowpowermode-enabled`
* `.lowpowermode-disabled`

### Notifications may appear often

The re-enable Shortcut includes a notification action.
If Low Power Mode is turned off frequently, you may see that alert more often than you want. 

---

## Conclusion

This approach gives you a clean way to make iPhone automatically turn Low Power Mode back on, while still keeping the behavior easy to control.

The core idea is simple:

* use **Get File from Folder** to read a flag file
* use **If File has any value** to determine whether the feature is enabled
* use one Shortcut to toggle the flag state
* use another Shortcut to restore Low Power Mode when the automation fires

The toggle Shortcut handles both renaming the flag file and switching Low Power Mode itself, while the re-enable Shortcut checks the enabled flag and only restores the setting when appropriate.

That makes this more practical than a basic always-on automation.
It gives you automatic recovery **and** manual control.
