# Audio HAL Documentation for Android Automotive on Raspberry Pi 5

## 📚 Overview

This repository contains comprehensive documentation for the Audio Hardware Abstraction Layer (HAL) implementation for Android Automotive on Raspberry Pi 5 with WM8960 audio codec.

## 📂 Repository Structure

```
AHAL_docs/
├── README.md                                    (This file)
├── AUDIO_HAL_ARCHITECTURE.md                   (64 KB)
│   └── Complete technical architecture documentation
│       - HAL layer design and implementation
│       - Data structures and flow diagrams
│       - Performance analysis and debugging guide
│
├── WM8960_AUDIO_BLUETOOTH_CONFIGURATION.md     (36 KB)
│   └── Complete configuration documentation
│       - WM8960 codec setup and modifications
│       - Bluetooth A2DP Sink configuration
│       - All file changes with before/after comparisons
│       - Troubleshooting guide
│
└── docs/
    ├── 20251030_101753.mp4                     (Video demo)
    ├── 20251030_110336.mp4                     (Video demo)
    ├── 20251030_194008.jpg                     (Screenshot)
    └── 20251030_194019.jpg                     (Screenshot)
```

## 📖 Documentation Files

### 1. AUDIO_HAL_ARCHITECTURE.md
**Complete Audio HAL Technical Documentation**

- 🏗️ Architecture Overview (3-layer system)
- 🧩 HAL Module Components and Data Structures
- 🔄 Device Initialization Flow with Diagrams
- 🎵 WM8960 Mixer Architecture
- 📊 Output Stream Lifecycle
- ⚡ Performance Characteristics
- 🛠️ Debugging & Diagnostics Guide
- 🔬 Advanced Topics (MMAP mode, thread safety)

**Key Topics Covered:**
- Complete code analysis of `audio_hw.c` (811 lines)
- Buffer configuration and latency analysis
- Thread safety and locking mechanisms
- Power management and standby strategy
- End-to-end audio flow from MediaPlayer to speaker

### 2. WM8960_AUDIO_BLUETOOTH_CONFIGURATION.md
**Complete Configuration Guide**

- 🎯 Problems Identified & Solutions
- 📁 7 Files Modified with Complete Before/After
- 🔊 Audio Flow Diagrams
- 📱 Bluetooth A2DP Sink Flow
- 🧪 Verification Commands
- 🏗️ Build Instructions
- 🔧 Troubleshooting Guide

**Files Documented:**
- `config.txt` (Boot configuration)
- `vendor.prop` (System properties)
- `audio_policy_configuration.xml` (Audio routing)
- `audio_hw.c` (HAL mixer initialization)
- `device.mk` (Build configuration)
- Bluetooth services (Framework fixes)
- Bluetooth HAL (Driver improvements)

## 🎯 Features Implemented

### Audio HAL
✅ WM8960 codec integration  
✅ TinyALSA-based implementation  
✅ MMAP mode for low-latency playback  
✅ Automatic mixer initialization  
✅ PCM routing switch configuration  
✅ Volume control management  
✅ Power management with standby mode  

### Bluetooth
✅ A2DP Sink profile (receive audio from phone)  
✅ Stable connection management  
✅ Audio routing to WM8960 speaker  
✅ Crash resilience and error handling  

## 🔧 Hardware Configuration

**Platform:** Raspberry Pi 5  
**AOSP Build:** aosp_rpi5_car-bp2a-userdebug  
**Audio Codec:** WM8960  
- I2C Address: 0x1a  
- I2S Interface: Digital audio  
- ALSA Card: 2 (`wm8960-soundcard`)  
- PCM Device: 0  

**Audio Specifications:**
- Sample Rate: 48000 Hz (supports 8k-48k)
- Channels: 2 (Stereo)
- Format: PCM_FORMAT_S16_LE (16-bit signed)
- Latency: ~85-100 ms
- Buffer: 4 periods × 1024 frames (16 KB)

## 📊 Performance Metrics

| Metric | Value | Notes |
|--------|-------|-------|
| **Latency** | ~85-100 ms | Acceptable for media playback |
| **CPU Usage** | ~5-6% | During active playback |
| **Power (Standby)** | 0 mW | PCM device closed |
| **Power (Active)** | ~50 mW | Codec + amplifier |
| **Wake-up Time** | <10 ms | From standby to active |

## 🚀 Quick Start

### Prerequisites
```bash
# AOSP build environment
source build/envsetup.sh
lunch aosp_rpi5_car-bp2a-userdebug
```

### Build Audio HAL
```bash
# Build audio module only (faster)
m audio.primary.rpi -j$(nproc)

# Or full system build
make -j$(nproc)
```

### Verify Configuration
```bash
# Check WM8960 driver loaded
adb shell "dmesg | grep -i wm8960"

# Check ALSA cards
adb shell "cat /proc/asound/cards"

# Check mixer initialization
adb logcat -d | grep -E "adev_open|wm8960.*mixer"

# Test audio playback
adb shell "tinyplay /data/local/tmp/test.wav -D 2 -d 0"
```

## 🛠️ Troubleshooting

### No Sound from Speaker
```bash
# Check mixer controls
adb shell "tinymix -D 2 51"  # Left PCM Switch (should be "On")
adb shell "tinymix -D 2 54"  # Right PCM Switch (should be "On")
adb shell "tinymix -D 2 12"  # Speaker Volume (should be "110 110")

# Check audio routing
adb shell "dumpsys audio | grep -A 5 STREAM_MUSIC"
```

### Bluetooth Audio Not Working
```bash
# Check A2DP Sink enabled
adb shell "dumpsys bluetooth_manager | grep 'A2DP.*State'"

# Monitor Bluetooth audio streaming
adb logcat | grep -E "A2DP|BTA_AV"
```

## 📝 Key Configuration Files

### vendor.prop
```properties
persist.vendor.audio.device=wm8960
persist.vendor.audio.pcm.card=2
persist.vendor.audio.pcm.device=0
ro.hardware.audio.primary=rpi
bluetooth.profile.a2dp.sink.enabled?=true
bluetooth.profile.a2dp.source.enabled?=false
```

### config.txt (Boot Partition)
```ini
dtparam=i2c_arm=on
dtparam=i2s=on
dtoverlay=wm8960-soundcard
```

## 🔗 References

- [Android Audio HAL Documentation](https://source.android.com/docs/core/audio/implement-hal)
- [WM8960 Datasheet](https://www.cirrus.com/products/wm8960/)
- [TinyALSA Library](https://github.com/tinyalsa/tinyalsa)
- [ALSA Project](https://www.alsa-project.org/)

## 📄 License

Apache License 2.0 (following AOSP licensing)

## 👥 Authors

- Audio HAL Implementation: KonstaKANG, AOSP Project
- WM8960 Integration & Documentation: [Your Team/Organization]

## 📧 Contact

For questions or issues, please open an issue in this repository.

---

**Last Updated:** October 30, 2025  
**Platform:** Raspberry Pi 5 AOSP (Android Automotive)  
**Audio Codec:** WM8960
