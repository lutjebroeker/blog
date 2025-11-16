---

title: "Fixing Aqara Lights That Turn On Randomly via ZHA"

date: 2025-11-16

tags: ["home-automation", "aqara", "zha", "zigbee", "home-assistant", "troubleshooting"]

topics: ["Smart Home", "Zigbee Devices", "Home Assistant"]

---

## The Problem
Your Aqara lamp (likely a **T1M model**) that keeps turning on by itself through ZHA is a well-known issue. This happens because of the `power_on_behavior` setting, which determines what the lamp does after a power outage. By default, it's set to "On" (lamp turns on), but you want it set to "PreserveState" (maintains the previous state).

The reason this problem occurs is that ZHA doesn't provide default access to all Aqara-specific settings. You need a custom quirk that exposes the **Manufacturer Specific Cluster (0xfcc0)** from Aqara with all the hidden attributes.
## Solution: Step by Step
## Step 1: Create a Directory for Custom Quirks
First, create a folder in your Home Assistant configuration directory where you can store custom quirks.
1. Open the **File Editor** in Home Assistant (or use Samba share if you have it installed)
2. Navigate to your `/config` folder
3. Create a new directory called `zha_quirks` or `custom_zha_quirks`
## Step 2: Configure configuration.yaml
Add the following to your `configuration.yaml` file:
```yaml

zha:

  enable_quirks: true

  custom_quirks_path: /config/zha_quirks/

```
**Note:** If you used the `custom_zha_quirks` folder name, adjust the path accordingly:

```yaml

zha:

  enable_quirks: true

  custom_quirks_path: /config/custom_zha_quirks/

```
## Step 3: Download the Quirk File
Download the specific quirk for the Aqara T1M light:
1. Go to: `https://raw.githubusercontent.com/zigpy/zha-device-handlers/dev/zhaquirks/xiaomi/aqara/light_acn.py`
2. Right-click and select **"Save as"** or copy the entire content
3. Save the file as `light_acn.py` in the `zha_quirks` folder you created in Step 1
**Important:** Make sure the filename ends with `.py` and that it's a Python file, not a text file.
## Step 4: Restart Home Assistant
1. Go to **Settings** → **System** → **Restart**
2. Wait until Home Assistant fully restarts
## Step 5: Remove and Re-pair the Light
This is a **crucial step** — the quirk is only applied to new devices or when you re-pair an existing device.
1. Go to **Settings** → **Devices & Services** → **ZHA**
2. Find your Aqara light in the device list
3. Click on the device and remove it completely
4. Restart Home Assistant again (this clears all cache)
5. Add the light again via **Add Device** in ZHA
6. Reset the light according to Aqara instructions (usually by quickly toggling the light on and off multiple times until it flashes)
## Step 6: Verify the Quirk is Loaded
After re-pairing:
1. Go to your Aqara light's device page
2. Scroll down to **Zigbee Device Signature**
3. You should now see a **quirk** section mentioning the custom quirk
4. A **"Manage Zigbee Device"** option should now be available
## Step 7: Set the power_on_behavior
This is the step that actually fixes the problem!
1. Go to **Settings** → **Devices & Services** → **ZHA**
2. Open your Aqara light device
3. Click on **"Manage Zigbee Device"**
4. Scroll through the clusters until you find **"AqaraLight T1M (Endpoint id: 1, Id: 0xfcc0, Type: in)"**
   - This is Aqara's Manufacturer Specific Cluster
1. Click on it to expand
2. Find the attribute **`power_on_state (id: 0x0517)`**
3. Click on **"Read attribute"** to check the current value
4. Click on **"Write attribute"**
5. In the **"Value"** field, enter: `1`
6. Click **"Write attribute"** to save

**Value options for power_on_behavior:** 

| Value | Behavior                                 | Recommendation             |
| ----- | ---------------------------------------- | -------------------------- |
| **0** | On — lamp turns on after power loss      | ❌ This causes your problem |
| **1** | PreserveState — maintains the last state | ✅ This is what you want!   |
| **2** | Off — lamp stays off after power loss    | ⚠️ May be impractical      |

## Step 8: Verify It Works
Check that the setting persists:
1. Click **"Read attribute"** again for `power_on_state`
2. Verify that the value now shows **1** and doesn't revert to 0
3. Leave the light off for a few days and check if the problem is resolved
Multiple users report that after setting `power_on_state` to "1", the light no longer turns on by itself.
---
## Summary
The key to solving your Aqara light problem is:

1. ✅ Install a custom quirk that exposes the Manufacturer Specific Cluster (0xfcc0)
2. ✅ Completely remove the light from ZHA and re-pair it
3. ✅ Set the `power_on_state` attribute to value **1** for PreserveState
4. ✅ Verify that the value persists

With these steps, your Aqara T1M light should no longer turn on by itself after a power outage!