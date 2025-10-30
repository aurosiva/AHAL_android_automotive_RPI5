# 📋 Complete Documentation: WM8960 Audio HAL & Bluetooth Configuration for Raspberry Pi 5 AOSP

## 📌 Project Overview
**Device:** Raspberry Pi 5  
**AOSP Version:** aosp_rpi5_car-bp2a-userdebug  
**Audio Codec:** WM8960 (I2C address: 0x1a, I2S interface)  
**Objective:** Enable WM8960 audio output for system audio, media playback, and Bluetooth A2DP Sink

---

## 🎯 Problems Identified & Solved

### **Initial Issues:**
1. ❌ WM8960 driver loaded but no audio output
2. ❌ Mixer controls muted by default (PCM routing disabled, volumes at 0)
3. ❌ Audio routing to HDMI instead of WM8960 speaker
4. ❌ Duplicate speaker device definitions causing confusion
5. ❌ Wrong PCM card property (string instead of numeric)
6. ❌ Bluetooth A2DP Sink paired but audio not playing
7. ❌ Android media player silent despite WM8960 detection

### **Root Causes:**
- Audio policy had duplicate `AUDIO_DEVICE_OUT_SPEAKER` definitions
- `persist.vendor.audio.pcm.card` set to card name instead of card number
- No mixer initialization in audio HAL
- WM8960 not set as default output device
- Sample rate mismatch between policy and codec capabilities

---

## 📂 File Modifications (7 Files Total)

---

## **1. Boot Configuration**
### 📁 `device/brcm/rpi5/boot/config.txt`

**Location:** `/mnt/build-storage/rpi5_aosp/device/brcm/rpi5/boot/config.txt`

**Purpose:** Load WM8960 device tree overlay and enable required interfaces

#### **Changes Made:**

**BEFORE:**
```ini
# Default configuration without WM8960
# I2C and I2S disabled
# No audio codec overlay
```

**AFTER:**
```ini
# I2C (required for WM8960 control interface)
dtparam=i2c_arm=on

# I2S (required for WM8960 audio data)
dtparam=i2s=on

# WM8960 Audio Codec Device Tree Overlay
dtoverlay=wm8960-soundcard
```

#### **Explanation:**
- **`dtparam=i2c_arm=on`**: Enables I2C bus for WM8960 register configuration (address 0x1a)
- **`dtparam=i2s=on`**: Enables I2S (Inter-IC Sound) for digital audio data transfer
- **`dtoverlay=wm8960-soundcard`**: Loads WM8960 kernel driver and creates ALSA sound card
- **Result:** Creates `/proc/asound/card2` (`wm8960-soundcard`)

⚠️ **IMPORTANT:** This file must be manually copied to the boot partition:
```bash
# After flashing, mount boot partition and copy:
cp device/brcm/rpi5/boot/config.txt /path/to/boot/partition/config.txt
```

---

## **2. Vendor Properties Configuration**
### 📁 `device/brcm/rpi5/vendor.prop`

**Location:** `/mnt/build-storage/rpi5_aosp/device/brcm/rpi5/vendor.prop`

**Purpose:** Configure system properties for audio routing and Bluetooth profiles

#### **Changes Made:**

**BEFORE:**
```properties
# Audio (INCORRECT CONFIGURATION)
persist.vendor.audio.device=wm8960
persist.vendor.audio.pcm.card=wm8960soundcard    # ❌ WRONG: String value
persist.vendor.audio.pcm.card.auto=false
ro.hardware.audio.primary=rpi_hdmi               # ❌ HDMI audio enabled

# Bluetooth (DEFAULT)
bluetooth.profile.a2dp.source.enabled?=true      # ❌ Source conflicts with Sink
```

**AFTER:**
```properties
# Audio
# WM8960 Audio Codec Configuration
persist.vendor.audio.device=wm8960               # ✅ Device identifier for HAL
persist.vendor.audio.pcm.card=2                  # ✅ FIXED: Numeric card ID
persist.vendor.audio.pcm.device=0                # ✅ PCM device 0
persist.vendor.audio.pcm.card.auto=false         # ✅ Disable auto-detection
ro.config.media_vol_default=20                   # ✅ Default volume level
ro.config.media_vol_steps=25                     # ✅ Volume steps
ro.hardware.audio.primary=rpi                    # ✅ Use rpi HAL (not HDMI)

# HDMI Audio (disabled in favor of WM8960)
#persist.vendor.audio.hdmi.device=vc4hdmi0       # ✅ Commented out
#ro.hardware.audio.primary=rpi_hdmi              # ✅ Commented out

# Bluetooth
bluetooth.profile.a2dp.source.enabled?=false     # ✅ Disable A2DP Source
bluetooth.profile.a2dp.sink.enabled?=true        # ✅ Enable A2DP Sink
```

#### **Explanation:**

| Property | Value | Purpose |
|----------|-------|---------|
| `persist.vendor.audio.device` | `wm8960` | Identifies WM8960 for mixer initialization |
| `persist.vendor.audio.pcm.card` | `2` | **CRITICAL FIX:** Points to card number (not name) |
| `persist.vendor.audio.pcm.device` | `0` | PCM device index on card 2 |
| `ro.hardware.audio.primary` | `rpi` | Selects `audio.primary.rpi.so` HAL library |
| `bluetooth.profile.a2dp.sink.enabled?` | `true` | Enables receiving audio from phone via Bluetooth |
| `bluetooth.profile.a2dp.source.enabled?` | `false` | Disables sending audio to other devices |

**Key Fix:** Changed `persist.vendor.audio.pcm.card=wm8960soundcard` → `persist.vendor.audio.pcm.card=2`

The audio HAL code reads this property:
```c
property_get("persist.vendor.audio.pcm.card", card, "0");
return atoi(card);  // Expects numeric value!
```

---

## **3. Audio Policy Configuration**
### 📁 `device/brcm/rpi5/audio/audio_policy_configuration.xml`

**Location:** `/mnt/build-storage/rpi5_aosp/device/brcm/rpi5/audio/audio_policy_configuration.xml`

**Purpose:** Define audio devices, routing, and default output for Android AudioFlinger

#### **Changes Made:**

**BEFORE:**
```xml
<module name="primary" halVersion="2.0">
    <attachedDevices>
        <item>Speaker</item>                      <!-- ❌ Generic speaker -->
        <item>Built-In Mic</item>
    </attachedDevices>
    <defaultOutputDevice>Speaker</defaultOutputDevice>  <!-- ❌ Points to generic -->
    
    <mixPorts>
        <mixPort name="primary output" role="source" flags="AUDIO_OUTPUT_FLAG_PRIMARY">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="48000"        <!-- ❌ Limited sample rates -->
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </mixPort>
    </mixPorts>
    
    <devicePorts>
        <!-- ❌ DUPLICATE SPEAKER DEFINITIONS -->
        <devicePort tagName="Speaker" type="AUDIO_DEVICE_OUT_SPEAKER" role="sink">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="48000"        <!-- ❌ Limited -->
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        
        <devicePort tagName="WM8960" type="AUDIO_DEVICE_OUT_SPEAKER" role="sink">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="8000 16000 32000 44100 48000"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        <!-- Both using same type causes confusion! -->
        
        <devicePort tagName="Wired Headset" type="AUDIO_DEVICE_OUT_WIRED_HEADSET" role="sink">
            <profile samplingRates="48000" ... />   <!-- ❌ Limited -->
        </devicePort>
    </devicePorts>
    
    <routes>
        <route type="mix" sink="Speaker" sources="primary output"/>  <!-- ❌ Generic speaker -->
        <route type="mix" sink="WM8960" sources="primary output"/>   <!-- ❌ Duplicate route -->
    </routes>
</module>
```

**AFTER:**
```xml
<module name="primary" halVersion="2.0">
    <attachedDevices>
        <item>WM8960</item>                       <!-- ✅ WM8960 as primary -->
        <item>Built-In Mic</item>
    </attachedDevices>
    <defaultOutputDevice>WM8960</defaultOutputDevice>  <!-- ✅ Explicit default -->
    
    <mixPorts>
        <mixPort name="primary output" role="source" flags="AUDIO_OUTPUT_FLAG_PRIMARY">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="8000 16000 32000 44100 48000"  <!-- ✅ All WM8960 rates -->
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </mixPort>
        <mixPort name="primary input" role="sink">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="8000 11025 12000 16000 22050 24000 32000 44100 48000"
                     channelMasks="AUDIO_CHANNEL_IN_MONO AUDIO_CHANNEL_IN_STEREO"/>
        </mixPort>
    </mixPorts>
    
    <devicePorts>
        <!-- ✅ WM8960 as ONLY speaker device -->
        <devicePort tagName="WM8960" type="AUDIO_DEVICE_OUT_SPEAKER" role="sink">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="8000 16000 32000 44100 48000"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        
        <!-- ✅ Updated sample rates for headphones -->
        <devicePort tagName="Wired Headset" type="AUDIO_DEVICE_OUT_WIRED_HEADSET" role="sink">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="8000 16000 32000 44100 48000"  <!-- ✅ Expanded -->
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        
        <devicePort tagName="Wired Headphones" type="AUDIO_DEVICE_OUT_WIRED_HEADPHONE" role="sink">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="8000 16000 32000 44100 48000"  <!-- ✅ Expanded -->
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        
        <!-- ✅ WM8960 Mic for input -->
        <devicePort tagName="WM8960 Mic" type="AUDIO_DEVICE_IN_BUILTIN_MIC" role="source">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="8000 16000 32000 44100 48000"
                     channelMasks="AUDIO_CHANNEL_IN_STEREO"/>
        </devicePort>
        
        <!-- Bluetooth devices -->
        <devicePort tagName="BT SCO" type="AUDIO_DEVICE_OUT_BLUETOOTH_SCO" role="sink">
            <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                     samplingRates="8000 16000"
                     channelMasks="AUDIO_CHANNEL_OUT_MONO"/>
        </devicePort>
        <!-- ... other BT profiles ... -->
    </devicePorts>
    
    <routes>
        <!-- ✅ WM8960 as first/primary route -->
        <route type="mix" sink="WM8960" sources="primary output"/>
        <route type="mix" sink="Wired Headset" sources="primary output"/>
        <route type="mix" sink="Wired Headphones" sources="primary output"/>
        <route type="mix" sink="BT SCO" sources="primary output"/>
        <route type="mix" sink="BT SCO Headset" sources="primary output"/>
        <route type="mix" sink="BT SCO Car Kit" sources="primary output"/>
        
        <!-- ✅ Input routing including WM8960 Mic -->
        <route type="mix" sink="primary input"
               sources="Built-In Mic,Wired Headset Mic,BT SCO Headset Mic,WM8960 Mic"/>
    </routes>
</module>

<!-- Bluetooth audio modules -->
<xi:include href="a2dp_in_audio_policy_configuration_7_0.xml"/>
<xi:include href="usb_audio_policy_configuration.xml"/>
<xi:include href="r_submix_audio_policy_configuration.xml"/>
<xi:include href="bluetooth_audio_policy_configuration_7_0.xml"/>
```

#### **Explanation:**

**Key Changes:**

1. **Removed Duplicate Speaker Device:**
   - Old: Had both "Speaker" and "WM8960" as `AUDIO_DEVICE_OUT_SPEAKER`
   - New: Only "WM8960" exists, eliminating confusion
   - Impact: AudioFlinger now correctly identifies speaker device

2. **Set WM8960 as Default:**
   - `<defaultOutputDevice>WM8960</defaultOutputDevice>`
   - Ensures all audio routes to WM8960 by default
   - Critical for media playback and system sounds

3. **Expanded Sample Rates:**
   - Old: `samplingRates="48000"` (only 48kHz)
   - New: `samplingRates="8000 16000 32000 44100 48000"`
   - Supports Bluetooth SCO (8/16kHz), music (44.1kHz), and system (48kHz)

4. **Routing Priority:**
   - WM8960 route placed first in routing list
   - AudioFlinger evaluates routes in order
   - Ensures WM8960 is preferred over other outputs

**Audio Routing Flow:**
```
Android App (Media Player)
    ↓
AudioFlinger
    ↓
Check defaultOutputDevice = "WM8960"
    ↓
Find devicePort tagName="WM8960" type=AUDIO_DEVICE_OUT_SPEAKER
    ↓
Route "primary output" → "WM8960" sink
    ↓
audio.primary.rpi.so HAL
    ↓
PCM card 2, device 0 (WM8960)
    ↓
🔊 Sound output!
```

---

## **4. Audio HAL Implementation**
### 📁 `device/brcm/rpi5/audio/audio_hw.c`

**Location:** `/mnt/build-storage/rpi5_aosp/device/brcm/rpi5/audio/audio_hw.c`

**Purpose:** Hardware Abstraction Layer for audio - initializes WM8960 mixer controls

#### **Changes Made:**

**BEFORE:**
```c
// No mixer initialization
// Mixer controls defaulted to muted state
// No WM8960-specific configuration
```

**AFTER:**

**New Function Added (lines 689-739):**
```c
/* Initialize WM8960 mixer controls for proper audio routing */
static void init_wm8960_mixer(int card)
{
    struct mixer *mixer;
    struct mixer_ctl *ctl;
    
    mixer = mixer_open(card);
    if (!mixer) {
        ALOGE("Failed to open mixer for card %d", card);
        return;
    }
    
    ALOGI("Initializing WM8960 mixer controls for card %d", card);
    
    /* Enable PCM playback routing */
    ctl = mixer_get_ctl_by_name(mixer, "Left Output Mixer PCM Playback Switch");
    if (ctl) mixer_ctl_set_value(ctl, 0, 1);
    
    ctl = mixer_get_ctl_by_name(mixer, "Right Output Mixer PCM Playback Switch");
    if (ctl) mixer_ctl_set_value(ctl, 0, 1);
    
    /* Set speaker volumes */
    ctl = mixer_get_ctl_by_name(mixer, "Speaker Playback Volume");
    if (ctl) {
        mixer_ctl_set_value(ctl, 0, 110);  /* Left channel: 110/127 (86%) */
        mixer_ctl_set_value(ctl, 1, 110);  /* Right channel: 110/127 (86%) */
    }
    
    ctl = mixer_get_ctl_by_name(mixer, "Speaker DC Volume");
    if (ctl) mixer_ctl_set_value(ctl, 0, 5);
    
    ctl = mixer_get_ctl_by_name(mixer, "Speaker AC Volume");
    if (ctl) mixer_ctl_set_value(ctl, 0, 5);
    
    /* Set headphone volumes */
    ctl = mixer_get_ctl_by_name(mixer, "Headphone Playback Volume");
    if (ctl) {
        mixer_ctl_set_value(ctl, 0, 110);  /* Left channel: 110/127 (86%) */
        mixer_ctl_set_value(ctl, 1, 110);  /* Right channel: 110/127 (86%) */
    }
    
    /* Set main playback volume to maximum */
    ctl = mixer_get_ctl_by_name(mixer, "Playback Volume");
    if (ctl) {
        mixer_ctl_set_value(ctl, 0, 255);  /* Left channel: max */
        mixer_ctl_set_value(ctl, 1, 255);  /* Right channel: max */
    }
    
    ALOGI("WM8960 mixer initialization complete");
    mixer_close(mixer);
}
```

**Modified `adev_open()` Function (lines 785-791):**
```c
static int adev_open(const hw_module_t* module, const char* name,
        hw_device_t** device)
{
    struct alsa_audio_device *adev;

    ALOGV("adev_open: %s", name);

    pcm_card = get_pcm_card();
    pcm_device = get_pcm_device();
    ALOGI("adev_open: pcm_card %d, pcm_device %d", pcm_card, pcm_device);

    if (strcmp(name, AUDIO_HARDWARE_INTERFACE) != 0)
        return -EINVAL;

    adev = calloc(1, sizeof(struct alsa_audio_device));
    if (!adev)
        return -ENOMEM;

    // ... HAL function assignments ...

    adev->devices = AUDIO_DEVICE_NONE;
    *device = &adev->hw_device.common;

    /* ✅ NEW: Initialize WM8960 mixer controls if using WM8960 codec */
    char card_prop[PROPERTY_VALUE_MAX];
    property_get("persist.vendor.audio.device", card_prop, "");
    if (strcmp(card_prop, "wm8960") == 0) {
        ALOGI("Detected WM8960 audio device, initializing mixer");
        init_wm8960_mixer(pcm_card);
    }

    return 0;
}
```

#### **Explanation:**

**Mixer Controls Configured:**

| Control Name | Value | Range | Purpose |
|--------------|-------|-------|---------|
| `Left Output Mixer PCM Playback Switch` | **On (1)** | 0-1 | ✅ Routes PCM data to left output amplifier |
| `Right Output Mixer PCM Playback Switch` | **On (1)** | 0-1 | ✅ Routes PCM data to right output amplifier |
| `Speaker Playback Volume` | **110** | 0-127 | ✅ Sets speaker output level to 86% |
| `Speaker DC Volume` | **5** | 0-5 | ✅ DC bias for Class-D amplifier |
| `Speaker AC Volume` | **5** | 0-5 | ✅ AC gain for Class-D amplifier |
| `Headphone Playback Volume` | **110** | 0-127 | ✅ Sets headphone output level to 86% |
| `Playback Volume` | **255** | 0-255 | ✅ Master DAC volume at 100% |

**Critical Mixer Switches:**

The **PCM Playback Switches** are the most critical:
```
[DAC] → [PCM Data] → [Left Output Mixer] ← PCM Switch ON ✅
                    ↓
                [Left Speaker Amplifier]
```

**Without these switches enabled:**
- PCM audio data doesn't reach the output mixers
- Speaker/headphone amplifiers receive no signal
- Result: Silent output even with correct routing

**Initialization Flow:**
```
Boot Sequence
    ↓
AudioService starts
    ↓
Loads audio.primary.rpi.so
    ↓
Calls adev_open()
    ↓
Reads persist.vendor.audio.device = "wm8960"
    ↓
Calls init_wm8960_mixer(2)  ← Card 2
    ↓
Opens /dev/snd/controlC2
    ↓
Enables PCM switches (controls #51, #54)
    ↓
Sets volumes (controls #10, #12, #14, #15)
    ↓
Closes mixer
    ↓
✅ WM8960 ready for audio playback!
```

**API Used:**
- `mixer_open(card)`: Opens ALSA mixer for specified card
- `mixer_get_ctl_by_name(mixer, name)`: Gets control by name
- `mixer_ctl_set_value(ctl, index, value)`: Sets control value
- `mixer_close(mixer)`: Closes mixer handle

**Why This is Necessary:**

ALSA/TinyALSA drivers default all mixer controls to **muted/off** for safety:
- Prevents speaker damage from unexpected signals
- Avoids loud pops during boot
- Requires explicit configuration by audio HAL

Without this initialization:
```bash
# Default state after boot:
tinymix -D 2 51  # Left PCM Switch: Off  ❌
tinymix -D 2 54  # Right PCM Switch: Off  ❌
tinymix -D 2 12  # Speaker Volume: 0  ❌
```

With initialization:
```bash
# After HAL initialization:
tinymix -D 2 51  # Left PCM Switch: On  ✅
tinymix -D 2 54  # Right PCM Switch: On  ✅
tinymix -D 2 12  # Speaker Volume: 110 110  ✅
```

---

## **5. Device Makefile**
### 📁 `device/brcm/rpi5/device.mk`

**Location:** `/mnt/build-storage/rpi5_aosp/device/brcm/rpi5/device.mk`

**Purpose:** Build configuration for device-specific packages

#### **Changes Made:**

**BEFORE:**
```makefile
# Audio
PRODUCT_PACKAGES += \
    android.hardware.audio.service \
    android.hardware.audio@7.1-impl \
    android.hardware.audio.effect@7.0-impl \
    audio.primary.rpi \
    audio.primary.rpi_hdmi \          # ❌ HDMI audio HAL included
    audio.r_submix.default \
    audio.usb.default
    # ❌ No TinyALSA tools
```

**AFTER:**
```makefile
# Audio
PRODUCT_PACKAGES += \
    android.hardware.audio.service \
    android.hardware.audio@7.1-impl \
    android.hardware.audio.effect@7.0-impl \
    audio.primary.rpi \                # ✅ RPi audio HAL (WM8960)
    audio.r_submix.default \
    audio.usb.default \
    tinyplay \                         # ✅ Added for testing
    tinycap \                          # ✅ Added for recording
    tinymix \                          # ✅ Added for mixer control
    tinypcminfo                        # ✅ Added for PCM info

# HDMI Audio (optional, disabled for WM8960)
#    audio.primary.rpi_hdmi \          # ✅ Commented out
```

#### **Explanation:**

**TinyALSA Tools Added:**

| Tool | Purpose | Usage Example |
|------|---------|---------------|
| `tinyplay` | Play audio files | `tinyplay test.wav -D 2 -d 0` |
| `tinycap` | Record audio | `tinycap recorded.wav -D 2 -d 0` |
| `tinymix` | Control mixer | `tinymix -D 2 51 On` |
| `tinypcminfo` | Show PCM info | `tinypcminfo -D 2 -d 0` |

**Why HDMI Audio is Disabled:**

- `audio.primary.rpi_hdmi` conflicts with `audio.primary.rpi`
- Both try to register as primary audio HAL
- `ro.hardware.audio.primary=rpi` in vendor.prop selects rpi HAL
- HDMI audio still available via HDMI device if needed

---

## **6. Bluetooth Service Fixes (Previous Work)**
### 📁 `packages/modules/Bluetooth/android/app/src/com/android/bluetooth/a2dpsink/A2dpSinkService.java`

**Location:** `/mnt/build-storage/rpi5_aosp/packages/modules/Bluetooth/android/app/src/com/android/bluetooth/a2dpsink/A2dpSinkService.java`

**Purpose:** Handle A2DP Sink profile (receive audio from phone)

#### **Previous Changes (From Earlier Bluetooth Fixes):**
- Added null checks for device objects
- Removed overly aggressive dual-role prevention
- Fixed connection policy handling
- Improved error logging

**Result:** A2DP Sink now connects successfully (seen in logs: "Stream opened", "Stream started")

---

## **7. Bluetooth HAL (Previous Work)**
### 📁 `device/brcm/rpi5/bluetooth/net_bluetooth_mgmt.cpp`
### 📁 `device/brcm/rpi5/bluetooth/BluetoothHci.cpp`

**Previous Changes:**
- Removed `rfKill()` calls causing disconnect loops
- Fixed variable declarations for goto compatibility
- Improved state management and error handling
- Fixed death recipient logic

**Result:** Bluetooth pairing stable, no connection/disconnection loops

---

## 🔄 Complete Audio Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    ANDROID APPLICATION                          │
│              (Media Player, YouTube, Spotify)                   │
└─────────────────────────┬───────────────────────────────────────┘
                          │ AudioTrack API
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                   ANDROID AUDIOFLINGER                          │
│  - Reads audio_policy_configuration.xml                         │
│  - Finds defaultOutputDevice = "WM8960"                         │
│  - Routes to AUDIO_DEVICE_OUT_SPEAKER (0x2)                     │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HAL Interface
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                AUDIO HAL (audio.primary.rpi.so)                 │
│  - adev_open() called on boot                                   │
│  - Reads persist.vendor.audio.pcm.card = 2                      │
│  - Reads persist.vendor.audio.device = "wm8960"                 │
│  - Calls init_wm8960_mixer(2)                                   │
│  - Enables PCM switches, sets volumes                           │
│  - Opens PCM: /dev/snd/pcmC2D0p                                 │
└─────────────────────────┬───────────────────────────────────────┘
                          │ ALSA/TinyALSA
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                    ALSA SUBSYSTEM                               │
│  - Card 2: wm8960-soundcard                                     │
│  - PCM Device 0: playback                                       │
│  - Mixer controls initialized                                   │
└─────────────────────────┬───────────────────────────────────────┘
                          │ I2S + I2C
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                    WM8960 CODEC                                 │
│  - I2C (0x1a): Register configuration                           │
│  - I2S: Digital audio data (PCM)                                │
│  - DAC: Digital to Analog conversion                            │
│  - Class-D Amplifier: Speaker output                            │
└─────────────────────────┬───────────────────────────────────────┘
                          │ Analog Output
                          ↓
                    🔊 SPEAKER 🔊
```

---

## 🔊 Bluetooth A2DP Sink Audio Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   BLUETOOTH PHONE (Source)                      │
│              (Playing music, YouTube, etc.)                     │
└─────────────────────────┬───────────────────────────────────────┘
                          │ Bluetooth A2DP
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                BLUETOOTH HAL (RPi5)                             │
│  - device/brcm/rpi5/bluetooth/                                  │
│  - Receives Bluetooth audio packets                             │
│  - Profile: A2DP_SINK                                           │
└─────────────────────────┬───────────────────────────────────────┘
                          │ Bluetooth Stack
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│              A2DP SINK SERVICE (Framework)                      │
│  - A2dpSinkService.java                                         │
│  - Connection state: CONNECTED                                  │
│  - Audio state: PLAYING                                         │
│  - Codec: SBC/AAC                                               │
└─────────────────────────┬───────────────────────────────────────┘
                          │ AudioTrack
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                   ANDROID AUDIOFLINGER                          │
│  - Creates AudioTrack for A2DP Sink                             │
│  - Routes to primary output (WM8960)                            │
└─────────────────────────┬───────────────────────────────────────┘
                          │ (Same path as above)
                          ↓
                    audio.primary.rpi.so
                          ↓
                       ALSA Card 2
                          ↓
                       WM8960 Codec
                          ↓
                    🔊 SPEAKER 🔊
```

---

## 🧪 Verification Commands

### **1. Check Boot Configuration Applied:**
```bash
adb shell "dmesg | grep -i wm8960"
# Expected output:
# [  3.827945] wm8960 1-001a: supply DCVDD not found, using dummy regulator
# [  3.834698] wm8960 1-001a: supply DBVDD not found, using dummy regulator
# [  4.428742] #2: wm8960-soundcard
```

### **2. Check ALSA Card Detection:**
```bash
adb shell "cat /proc/asound/cards"
# Expected output:
# 0 [vc4hdmi0      ]: vc4-hdmi - vc4-hdmi-0
# 1 [vc4hdmi1      ]: vc4-hdmi - vc4-hdmi-1
# 2 [wm8960soundcard]: simple-card - wm8960-soundcard  ✅
```

### **3. Check Audio HAL Initialization:**
```bash
adb logcat -d | grep -E "adev_open|wm8960.*mixer"
# Expected output:
# AudioHAL: adev_open: pcm_card 2, pcm_device 0
# AudioHAL: Detected WM8960 audio device, initializing mixer
# AudioHAL: Initializing WM8960 mixer controls for card 2
# AudioHAL: WM8960 mixer initialization complete
```

### **4. Check Mixer Controls:**
```bash
adb shell "tinymix -D 2 51"  # Left Output Mixer PCM Playback Switch
# Expected: On

adb shell "tinymix -D 2 54"  # Right Output Mixer PCM Playback Switch
# Expected: On

adb shell "tinymix -D 2 12"  # Speaker Playback Volume
# Expected: 110 110

adb shell "tinymix -D 2 10"  # Headphone Playback Volume
# Expected: 110 110
```

### **5. Check Audio Routing:**
```bash
adb shell "dumpsys audio | grep -A 5 'STREAM_MUSIC'"
# Expected:
# - STREAM_MUSIC:
#    Current: 2 (speaker): 20, ...  ✅ Speaker (2) present!
```

### **6. Test Audio Playback:**
```bash
# Push test file
adb push test_audio.wav /data/local/tmp/

# Test with tinyplay
adb shell "tinyplay /data/local/tmp/test_audio.wav -D 2 -d 0"
# Should hear sound from WM8960 speaker!

# Test with media player
adb shell am start -a android.intent.action.VIEW \
  -d "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
```

### **7. Check Bluetooth A2DP Status:**
```bash
adb shell "dumpsys bluetooth_manager | grep -A 20 'A2DP.*State'"
# Expected:
# A2DP Source State: Disabled
# A2DP Sink State: Enabled  ✅
# SDP A2DP sink handle: 65540
```

### **8. Monitor Bluetooth Audio Streaming:**
```bash
adb logcat | grep -E "A2DP|BTA_AV"
# When playing music from phone:
# shim::btm  A2DP Stream opened    : xx:xx:xx:xx:xx:xx BTA_AV_OPEN_EVT
# shim::btm  A2DP Stream started   : xx:xx:xx:xx:xx:xx BTA_AV_START_EVT
```

---

## 🏗️ Build Instructions

```bash
cd /mnt/build-storage/rpi5_aosp

# Setup environment
source build/envsetup.sh
lunch aosp_rpi5_car-bp2a-userdebug

# Option 1: Build audio module only (faster)
m audio.primary.rpi -j$(nproc)

# Option 2: Full system build (recommended for policy changes)
make -j$(nproc)

# Flash to device
# (Use your standard flashing procedure)

# IMPORTANT: Manually copy config.txt to boot partition
# Mount boot partition and run:
cp device/brcm/rpi5/boot/config.txt /path/to/boot/partition/config.txt
sync
```

---

## 📊 Configuration Summary Table

| Component | File | Key Setting | Value |
|-----------|------|-------------|-------|
| **Device Tree** | `config.txt` | `dtoverlay` | `wm8960-soundcard` |
| **I2C Enable** | `config.txt` | `dtparam` | `i2c_arm=on` |
| **I2S Enable** | `config.txt` | `dtparam` | `i2s=on` |
| **PCM Card** | `vendor.prop` | `persist.vendor.audio.pcm.card` | `2` |
| **PCM Device** | `vendor.prop` | `persist.vendor.audio.pcm.device` | `0` |
| **Audio Device** | `vendor.prop` | `persist.vendor.audio.device` | `wm8960` |
| **Primary HAL** | `vendor.prop` | `ro.hardware.audio.primary` | `rpi` |
| **Default Output** | `audio_policy_configuration.xml` | `defaultOutputDevice` | `WM8960` |
| **Attached Device** | `audio_policy_configuration.xml` | `attachedDevices` | `WM8960` |
| **Device Type** | `audio_policy_configuration.xml` | `type` | `AUDIO_DEVICE_OUT_SPEAKER` |
| **Sample Rates** | `audio_policy_configuration.xml` | `samplingRates` | `8000 16000 32000 44100 48000` |
| **Mixer Init** | `audio_hw.c` | `init_wm8960_mixer()` | Called in `adev_open()` |
| **PCM Switch L** | `audio_hw.c` | Control #51 | `On (1)` |
| **PCM Switch R** | `audio_hw.c` | Control #54 | `On (1)` |
| **Speaker Volume** | `audio_hw.c` | Control #12 | `110/127 (86%)` |
| **A2DP Sink** | `vendor.prop` | `bluetooth.profile.a2dp.sink.enabled?` | `true` |
| **A2DP Source** | `vendor.prop` | `bluetooth.profile.a2dp.source.enabled?` | `false` |

---

## 🎯 Expected Results

### ✅ **After Applying All Changes:**

1. **System Boot:**
   - WM8960 driver loads as card 2
   - Audio HAL initializes mixer controls
   - No errors in logcat

2. **Media Playback:**
   - Music apps play through WM8960 speaker
   - YouTube/streaming audio works
   - System sounds audible

3. **Bluetooth:**
   - Phone pairs successfully
   - A2DP Sink connects
   - Music from phone plays through WM8960 speaker
   - No connection/disconnection loops

4. **Audio Routing:**
   - `dumpsys audio` shows speaker (2) in STREAM_MUSIC
   - AudioFlinger output device: 0x2 (SPEAKER)
   - PCM card 2, device 0 active

5. **Volume Control:**
   - Android volume slider controls WM8960 output
   - Mute/unmute works correctly
   - Volume persists across reboots

---

## 🔧 Troubleshooting

### **Problem: No sound from speaker**

**Check 1: Kernel driver loaded?**
```bash
adb shell "dmesg | grep wm8960"
```
- If not found: Check `config.txt` in boot partition

**Check 2: Correct card selected?**
```bash
adb logcat -d | grep "adev_open: pcm_card"
```
- Should show: `pcm_card 2`
- If not: Check `vendor.prop` has `persist.vendor.audio.pcm.card=2`

**Check 3: Mixer initialized?**
```bash
adb shell "tinymix -D 2 51"
```
- Should show: `On`
- If `Off`: HAL mixer init didn't run

**Check 4: Audio routing correct?**
```bash
adb shell "dumpsys audio | grep -A 5 STREAM_MUSIC"
```
- Should include: `2 (speaker): XX`
- If not: Check audio_policy_configuration.xml

### **Problem: Bluetooth pairs but no audio**

**Check 1: A2DP Sink enabled?**
```bash
adb shell "dumpsys bluetooth_manager | grep 'A2DP.*State'"
```
- Should show: `A2DP Sink State: Enabled`

**Check 2: Audio stream status?**
```bash
adb logcat | grep "BTA_AV"
```
- Should show: `BTA_AV_OPEN_EVT`, `BTA_AV_START_EVT`

**Check 3: AudioTrack created?**
```bash
adb logcat | grep "AudioTrack"
```
- Should show AudioTrack creation for A2DP Sink

---

## 📝 Maintenance Notes

### **When Updating AOSP:**
- Verify `audio_policy_configuration.xml` not overwritten
- Check `vendor.prop` properties preserved
- Rebuild audio HAL (`m audio.primary.rpi`)

### **When Changing Audio Hardware:**
- Update `dtoverlay` in `config.txt`
- Modify `init_wm8960_mixer()` for new codec
- Update mixer control names/values

### **When Adding New Audio Devices:**
- Add `<devicePort>` in audio_policy_configuration.xml
- Add `<route>` for new device
- Update `attachedDevices` if always connected

---

## ✅ Checklist for New Build

- [ ] `config.txt` has WM8960 overlay
- [ ] `config.txt` copied to boot partition
- [ ] `vendor.prop` has card=2, device=0
- [ ] `audio_policy_configuration.xml` has WM8960 as default
- [ ] `audio_hw.c` has mixer initialization
- [ ] `device.mk` includes tinyalsa tools
- [ ] Build completed without errors
- [ ] Device boots successfully
- [ ] `dmesg` shows WM8960 driver loaded
- [ ] `logcat` shows mixer initialization
- [ ] `tinymix` controls show correct values
- [ ] Media playback produces sound
- [ ] Bluetooth pairs and connects
- [ ] Bluetooth audio plays through speaker

---

## 📚 References

- **WM8960 Datasheet:** https://www.cirrus.com/products/wm8960/
- **ALSA Mixer API:** https://www.alsa-project.org/alsa-doc/alsa-lib/group___mixer.html
- **TinyALSA Documentation:** https://github.com/tinyalsa/tinyalsa
- **Android Audio Policy:** https://source.android.com/docs/core/audio/audio-policy
- **Bluetooth A2DP:** https://source.android.com/docs/core/connect/bluetooth

---

## 👥 Author

**Sivaraman Ramamoorthy**
- 📧 Email: aurosivaraman@gmail.com
- 📱 Phone: +91 91592 29591
- 🔧 Specialization: WM8960 Codec Integration, Bluetooth A2DP Sink, Audio Policy Configuration
- 🎯 Contribution: Complete WM8960 setup, mixer initialization, Bluetooth framework fixes, comprehensive documentation

**Acknowledgments:**
- Audio HAL Base: KonstaKANG, AOSP Project
- Android Bluetooth Stack: The Android Open Source Project
- WM8960 Driver: Linux Kernel Community

---

**Document Version:** 1.1  
**Last Updated:** October 30, 2025  
**Author:** Sivaraman Ramamoorthy  
**Contact:** aurosivaraman@gmail.com | +91 91592 29591  
**Platform:** AOSP for Raspberry Pi 5 (Android Automotive 16)  
**Audio Codec:** WM8960 (I2C: 0x1a, I2S interface)  
**Repository:** https://github.com/aurosiva/AHAL_android_automotive_RPI5

