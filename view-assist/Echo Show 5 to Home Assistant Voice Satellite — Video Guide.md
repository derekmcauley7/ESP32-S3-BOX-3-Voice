# Echo Show 5 to Home Assistant Voice Satellite — Guide

## Overview

This guide turns an Echo Show 5 (2nd gen) into a local Home Assistant voice satellite with a touchscreen: jailbreak, LineageOS, then View Assist 
**Device:** Echo Show 5 2nd gen (2021), model **C76N8S**, codename **cronos**. 

**What you need**

- Echo Show 5 2nd gen (C76N8S) and its power brick
- Micro-USB **data** cable
- A Windows PC for the jailbreak and flashing (the exploit needs Windows or Linux; the later adb tweaks also work from a Mac)
- A free XDA account (needed to download attachments)
- Home Assistant with HACS and a working Assist pipeline

**Structure:** prep → jailbreak → boot image → LineageOS → apps → Home Assistant → test.

## Part 1: Prep

Keep the Echo **off Wi-Fi** from the moment you unbox it — an Amazon update could patch the exploit. You don't need to finish the Echo setup; the jailbreak script checks the firmware itself.

1. Install the [Kindle Fire USB driver](https://developer.amazon.com/docs/fire-tablets/connecting-adb-to-device.html#install_usb_driver) and reboot Windows. The installer may say it failed, it usually works anyway.
2. Make a folder `C:\echo` and download into it:
   - **amonet-cronos.zip** and **boot-root.zip** — first post of the [cronos unlock thread](https://xdaforums.com/t/unlock-root-twrp-unbrick-amazon-echo-show-5-2nd-gen-2021-cronos.4772596/) (sign in to see attachments)
   - **lineage-18.1-…-UNOFFICIAL-cronos.zip** — [cronos LineageOS thread](https://xdaforums.com/t/rom-unofficial-11-cronos-lineageos-18-1-for-the-amazon-echo-show-5-2021.4772598/) (don't unzip)
   - **Platform Tools** — [direct link](https://dl.google.com/android/repository/platform-tools-latest-windows.zip)
3. Unzip amonet-cronos, boot-root and platform-tools. Copy `boot.img` (from boot-root.zip) and the Lineage ZIP into the `platform-tools` folder.

**Check the driver:** put the Echo in fastboot (Part 2, step 1), plug in USB, open Device Manager. You want **Android Device → Android Bootloader Interface**. A yellow warning icon means install the [Google USB driver](https://developer.android.com/studio/run/win-usb) instead.

## Part 2: Jailbreak

The amonet exploit unlocks the bootloader and installs TWRP. On my unit it worked first time with the Echo never connected to Wi-Fi.

1. Unplug the Echo. Hold **all three top buttons** and plug the power back in until **FASTBOOT** shows bottom-left.
2. Connect the micro-USB cable to the PC.
3. Open **Command Prompt as Administrator**:

```
cd C:\echo\amonet-cronos\amonet
.\fastbrick.bat
```

4. Type `YES` when it asks. The amonet screen flashes on the Echo, then it reboots into **TWRP**.
5. On the TWRP "Keep System Read only?" screen, tick **Never show this screen during boot again** and **Swipe to Allow Modifications**.
 
## Part 3: Flash the boot image

This flashes the rooted boot image from the cronos thread. Note: boot-root.zip contains a file called **boot.img**, not boot-root.img.

1. Unplug the Echo, hold **Volume Down** and plug in power. (The Gen 1 guides say Mute — on Gen 2 it's Volume Down.) The screen shows the Amazon logo with **hacked fastboot** at the bottom. It looks frozen; it's waiting for commands.
2. With USB connected, run:

```
cd C:\echo\platform-tools-latest-windows\platform-tools
fastboot devices
fastboot oem flags 61
fastboot flash boot boot.img
fastboot reboot
```

3. `fastboot devices` should list a serial number before you go on.
4. It reboots to the Fire OS setup screen. **Don't set it up or join Wi-Fi** — Fire OS is wiped in the next part.


## Part 4: Install LineageOS

TWRP wipes Fire OS and flashes LineageOS 18.1 (Android 11). This is the point of no return for Fire OS.

1. Unplug the Echo, hold **Volume Up** and plug in power → TWRP main menu.
2. Copy the Lineage ZIP to the Echo (`/sdcard/` is the Echo's internal storage, not an SD card):

```
cd C:\echo\platform-tools-latest-windows\platform-tools
adb devices
adb push lineage-18.1-XXXXXXXX-UNOFFICIAL-cronos.zip /sdcard/
```

3. On the Echo: **Wipe → Advanced Wipe**, tick **only System, Data and Cache**, swipe. Wiping anything else can brick it.
4. Tap the **⌂ Home** icon in TWRP's bottom bar (there's no Back button), then **Install** → the Lineage ZIP → tick reboot → swipe to flash.
5. First boot takes a few minutes; you'll get the white Lineage welcome screen.

**If `adb devices` shows nothing in TWRP**

- Run `adb kill-server` and replug.
- Device Manager may show an unknown device in TWRP mode — install the Google USB driver and pick **Android ADB Interface**.
- Or skip adb: TWRP → **Mount → Enable MTP**, then drag the ZIP onto the Echo in File Explorer.

## Part 5: LineageOS setup and the Companion App

From here no cable is needed — everything is done on the Echo, or from the same PC over Wi-Fi in Part 7.

1. Go through the Lineage welcome screens and join Wi-Fi (safe now — no Amazon updates).
2. Install the **View Assist Companion App** (package `com.msp1974.vacompanion`): download the latest APK from the [releases page](https://github.com/msp1974/ViewAssist_Companion_App/releases) in the browser and install it. (Obtainium also works, but F-Droid's first catalogue load on this CPU took several minutes.)
3. Open the app and grant the microphone and other permissions.

## Part 6: Home Assistant side

The Echo appears in HA on its own; View Assist plus two custom cards turn it from "your normal dashboard" into a proper voice display.

1. **Add the device:** Settings → Devices & Services → Discovered → **VACA …** (found via zeroconf). Assign it an area and your Assist pipeline.
2. **Install View Assist** from HACS and add the **Master Configuration** entry. It creates the View Assist dashboard (`/view-assist`).
3. **Install the required cards** from HACS — the View Assist screens are blank without them:
   - **button-card** (custom-cards)
   - **layout-card** (thomasloven)  
4. **Add the Echo to View Assist:** View Assist → Add entry → type **View Assist Companion App device** → pick the VACA satellite → name it.
5. Press **Refresh** on the VACA device page (or reopen the app). The Echo shows the View Assist clock.
6. Test: "Ok Nabu, what time is it?" Check the run under **Settings → Voice assistants → your pipeline → ⋮ → Debug**.

**Links**

- [XDA: cronos unlock / root / TWRP](https://xdaforums.com/t/unlock-root-twrp-unbrick-amazon-echo-show-5-2nd-gen-2021-cronos.4772596/)
- [XDA: LineageOS 18.1 for cronos](https://xdaforums.com/t/rom-unofficial-11-cronos-lineageos-18-1-for-the-amazon-echo-show-5-2021.4772598/)
- [View Assist docs](https://dinki.github.io/View-Assist/)
- [View Assist Companion App](https://github.com/msp1974/ViewAssist_Companion_App)
- [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools)
