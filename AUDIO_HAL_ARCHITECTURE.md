# 🎵 Android Audio HAL Architecture - Complete Technical Documentation

## 📋 Document Overview
**Platform:** Raspberry Pi 5 AOSP (aosp_rpi5_car-bp2a-userdebug)  
**HAL Version:** Android Audio HAL API v2.0  
**Implementation:** TinyALSA-based Audio HAL  
**File:** `device/brcm/rpi5/audio/audio_hw.c` (811 lines)  
**Codec Support:** WM8960, HDMI, Generic ALSA devices

---

## 🏗️ Architecture Overview

The Android Audio HAL (Hardware Abstraction Layer) is a critical component that bridges the gap between Android's audio framework (AudioFlinger/AudioPolicyManager) and the underlying Linux ALSA (Advanced Linux Sound Architecture) drivers.

### **Three-Layer Architecture:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          ANDROID FRAMEWORK                              │
│                                                                         │
│  ┌──────────────────┐         ┌──────────────────┐                    │
│  │  AudioFlinger    │         │ AudioPolicy      │                    │
│  │  (Mixing, Routing│◄───────►│ Manager          │                    │
│  │   Threading)     │         │ (Device Selection)│                   │
│  └────────┬─────────┘         └─────────┬────────┘                    │
│           │                             │                              │
│           │ audio_hw_device API         │                              │
│           └─────────────┬───────────────┘                              │
│                         │                                              │
└─────────────────────────┼──────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         AUDIO HAL LAYER                                 │
│                    (audio.primary.rpi.so)                               │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │               HAL Module (audio_hw.c)                        │     │
│  │  ┌────────────┐  ┌──────────────┐  ┌──────────────────┐    │     │
│  │  │  Device    │  │   Output     │  │   Input          │    │     │
│  │  │  Operations│  │   Stream     │  │   Stream         │    │     │
│  │  │  (Init,    │  │   (Playback) │  │   (Recording)    │    │     │
│  │  │   Config)  │  │              │  │   (Stub)         │    │     │
│  │  └────────────┘  └──────────────┘  └──────────────────┘    │     │
│  │                                                              │     │
│  │  ┌────────────────────────────────────────────────────┐    │     │
│  │  │        Mixer Initialization (WM8960)               │    │     │
│  │  │   - PCM routing switches                           │    │     │
│  │  │   - Volume controls                                │    │     │
│  │  │   - Codec-specific configuration                   │    │     │
│  │  └────────────────────────────────────────────────────┘    │     │
│  └──────────────────────────────────────────────────────────────┘     │
│                         │                                              │
└─────────────────────────┼──────────────────────────────────────────────┘
                          │ TinyALSA API
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      KERNEL ALSA SUBSYSTEM                              │
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐            │
│  │ Card 0:      │    │ Card 1:      │    │ Card 2:      │            │
│  │ vc4-hdmi-0   │    │ vc4-hdmi-1   │    │ wm8960-      │            │
│  │ (HDMI Audio) │    │ (HDMI Audio) │    │ soundcard    │            │
│  └──────────────┘    └──────────────┘    └──────┬───────┘            │
│                                                   │                    │
│  /dev/snd/pcmC2D0p ──────────────────────────────┘                    │
│  /dev/snd/controlC2                                                   │
│                                                                         │
└─────────────────────────┼──────────────────────────────────────────────┘
                          │ I2C (Control) + I2S (Data)
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      HARDWARE LAYER                                     │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────┐          │
│  │              WM8960 Audio Codec                         │          │
│  │   - DAC (Digital to Analog)                             │          │
│  │   - ADC (Analog to Digital)                             │          │
│  │   - Class-D Amplifier                                   │          │
│  │   - Mixer Controls                                      │          │
│  └───────────────────────┬─────────────────────────────────┘          │
│                          │ Analog Audio                                │
│                          ▼                                              │
│                    🔊 SPEAKER / 🎧 HEADPHONES                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 📁 Source Code Structure

### **File Location:**
```
/mnt/build-storage/rpi5_aosp/device/brcm/rpi5/audio/
├── audio_hw.c                      (811 lines) ← Main HAL implementation
├── audio_hw_hdmi.c                 (HDMI variant)
├── audio_policy_configuration.xml  (Device routing policy)
└── Android.bp                      (Build configuration)
```

### **Build Configuration (`Android.bp`):**
```bp
cc_library_shared {
    name: "audio.primary.rpi",          // Library name
    relative_install_path: "hw",        // Install to /vendor/lib/hw/
    vendor: true,                       // Vendor partition
    srcs: ["audio_hw.c"],               // Source file
    shared_libs: [
        "libcutils",                    // Android utilities
        "liblog",                       // Logging
        "libtinyalsa",                  // ALSA wrapper
    ],
    cflags: ["-Wno-unused-parameter"],
}
```

**Output:** `/vendor/lib/hw/audio.primary.rpi.so`

---

## 🧩 HAL Module Components

## 1️⃣ **Data Structures**

### **1.1 Device Structure (`alsa_audio_device`)**

```c
struct alsa_audio_device {
    struct audio_hw_device hw_device;    // Android HAL interface (base)
    
    pthread_mutex_t lock;                // Device-level mutex
    int devices;                         // Active device mask
    struct alsa_stream_in *active_input; // Current input stream
    struct alsa_stream_out *active_output; // Current output stream
    bool mic_mute;                       // Microphone mute state
};
```

**Purpose:** Root device object representing the entire audio hardware interface.

**Lifecycle:**
1. Created in `adev_open()`
2. Lives for entire Android runtime
3. Destroyed in `adev_close()`

**Key Responsibilities:**
- Maintain global audio device state
- Manage active streams (one input, one output)
- Provide thread safety via mutex
- Track device routing

---

### **1.2 Output Stream Structure (`alsa_stream_out`)**

```c
struct alsa_stream_out {
    struct audio_stream_out stream;    // Android stream interface (base)
    
    pthread_mutex_t lock;              // Stream-level mutex
    struct pcm_config config;          // PCM configuration (rate, format, etc.)
    struct pcm *pcm;                   // TinyALSA PCM handle
    bool unavailable;                  // Hardware availability flag
    int standby;                       // Standby state (1=standby, 0=active)
    struct alsa_audio_device *dev;     // Parent device pointer
    int write_threshold;               // Buffer threshold for writes
    unsigned int written;              // Total frames written
};
```

**Purpose:** Represents an active audio output stream (playback).

**Lifecycle:**
1. Created in `adev_open_output_stream()`
2. Opened/closed based on playback activity
3. Enters standby when idle
4. Destroyed in `adev_close_output_stream()`

**Key Responsibilities:**
- PCM data writing
- Buffer management
- Standby power management
- Latency calculation

---

### **1.3 PCM Configuration (`pcm_config`)**

```c
struct pcm_config {
    unsigned int channels;      // 2 (stereo)
    unsigned int rate;          // 48000 Hz
    unsigned int period_size;   // 1024 frames
    unsigned int period_count;  // 4 periods
    enum pcm_format format;     // PCM_FORMAT_S16_LE (16-bit signed)
    unsigned int start_threshold;
    unsigned int avail_min;
};
```

**Configured Values:**
```c
#define CODEC_BASE_FRAME_COUNT 32
#define PERIOD_MULTIPLIER 32
#define PERIOD_SIZE (32 * 32) = 1024 frames
#define PLAYBACK_PERIOD_COUNT 4
#define CODEC_SAMPLING_RATE 48000
#define CHANNEL_STEREO 2
```

**Buffer Calculations:**
```
Frame Size      = 2 channels × 2 bytes/sample = 4 bytes
Period Size     = 1024 frames × 4 bytes = 4096 bytes
Total Buffer    = 4 periods × 4096 bytes = 16384 bytes (16 KB)
Latency         = (1024 × 4 / 48000) = ~85 ms
```

---

## 2️⃣ **HAL Module Entry Point**

### **2.1 Module Definition**

```c
struct audio_module HAL_MODULE_INFO_SYM = {
    .common = {
        .tag = HARDWARE_MODULE_TAG,
        .module_api_version = AUDIO_MODULE_API_VERSION_0_1,
        .hal_api_version = HARDWARE_HAL_API_VERSION,
        .id = AUDIO_HARDWARE_MODULE_ID,              // "audio"
        .name = "Raspberry Pi audio HW HAL",
        .author = "The Android Open Source Project",
        .methods = &hal_module_methods,
    },
};
```

**Loading Process:**
```
1. Android System Boot
2. AudioFlinger starts
3. hw_get_module("audio", &module)  ← Loads audio.primary.rpi.so
4. Looks for HAL_MODULE_INFO_SYM symbol
5. Calls module->methods->open()
```

---

### **2.2 Module Methods**

```c
static struct hw_module_methods_t hal_module_methods = {
    .open = adev_open,    // Called to open audio device
};
```

---

## 3️⃣ **Device Initialization Flow**

### **3.1 Device Open (`adev_open`)**

```c
static int adev_open(const hw_module_t* module, const char* name,
                     hw_device_t** device)
```

**Detailed Flow Diagram:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                    adev_open() Initialization                          │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────┐
    │ 1. Read System Properties                       │
    │    - persist.vendor.audio.pcm.card              │
    │    - persist.vendor.audio.pcm.device            │
    │    - persist.vendor.audio.pcm.card.auto         │
    │    - persist.vendor.audio.device                │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 2. Get PCM Card/Device                          │
    │    pcm_card = get_pcm_card()                    │
    │    pcm_device = get_pcm_device()                │
    │                                                 │
    │    If auto=true:                                │
    │      → probe_pcm_out_card()                     │
    │         - Scan /proc/asound/card*/id            │
    │         - Match "wm8960", "Headphones", etc.    │
    │    Else:                                        │
    │      → Read property value (e.g., "2")          │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 3. Allocate Device Structure                    │
    │    adev = calloc(1, sizeof(alsa_audio_device))  │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 4. Initialize Function Pointers                 │
    │    adev->hw_device.init_check = adev_init_check │
    │    adev->hw_device.open_output_stream = ...     │
    │    adev->hw_device.close_output_stream = ...    │
    │    adev->hw_device.open_input_stream = ...      │
    │    adev->hw_device.set_master_volume = ...      │
    │    adev->hw_device.set_mode = ...               │
    │    ... (25+ function pointers)                  │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 5. Set Default State                            │
    │    adev->devices = AUDIO_DEVICE_NONE            │
    │    adev->active_input = NULL                    │
    │    adev->active_output = NULL                   │
    │    pthread_mutex_init(&adev->lock)              │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 6. Codec-Specific Initialization                │
    │    if (persist.vendor.audio.device == "wm8960") │
    │      init_wm8960_mixer(pcm_card)                │
    │        ↓                                        │
    │        • Open ALSA mixer                        │
    │        • Enable PCM routing switches            │
    │        • Set volume controls                    │
    │        • Configure amplifier gains              │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 7. Return Device Handle                         │
    │    *device = &adev->hw_device.common            │
    │    return 0 (success)                           │
    └─────────────────────────────────────────────────┘
```

**Key Functions:**

#### **A. Card Selection (`get_pcm_card`)**
```c
static int get_pcm_card()
{
    char card[PROPERTY_VALUE_MAX];
    
    // Check if auto-detection enabled
    property_get("persist.vendor.audio.pcm.card.auto", card, "false");
    if (!strcmp(card, "true"))
        return probe_pcm_out_card();    // Auto-detect
    
    // Use explicit card number
    property_get("persist.vendor.audio.pcm.card", card, "0");
    return atoi(card);                  // Convert to integer
}
```

**Property Examples:**
```properties
# Manual configuration (WM8960)
persist.vendor.audio.pcm.card=2
persist.vendor.audio.pcm.device=0

# Auto-detection
persist.vendor.audio.pcm.card.auto=true
persist.vendor.audio.device=wm8960
```

#### **B. Card Probing (`probe_pcm_out_card`)**
```c
static int probe_pcm_out_card() {
    for (int i = 0; i < 5; i++) {
        snprintf(card_node, sizeof(card_node), "/proc/asound/card%d/id", i);
        if ((fp = fopen(card_node, "r")) != NULL) {
            fgets(card_id, sizeof(card_id), fp);
            
            // Match 3.5mm jack
            if (!strcmp(card_prop, "jack") && !strncmp(card_id, "Headphones", 10)) {
                return i;
            }
            // Match DAC (excluding HDMI)
            else if (!strcmp(card_prop, "dac") && 
                     strncmp(card_id, "Headphones", 10) &&
                     strncmp(card_id, "vc4hdmi", 7)) {
                return i;
            }
        }
    }
    return 0;  // Default to card 0
}
```

---

### **3.2 WM8960 Mixer Initialization**

```c
static void init_wm8960_mixer(int card)
{
    struct mixer *mixer;
    struct mixer_ctl *ctl;
    
    // Open ALSA mixer for card
    mixer = mixer_open(card);
    if (!mixer) {
        ALOGE("Failed to open mixer for card %d", card);
        return;
    }
    
    // Configure routing switches
    ctl = mixer_get_ctl_by_name(mixer, "Left Output Mixer PCM Playback Switch");
    if (ctl) mixer_ctl_set_value(ctl, 0, 1);  // Enable
    
    ctl = mixer_get_ctl_by_name(mixer, "Right Output Mixer PCM Playback Switch");
    if (ctl) mixer_ctl_set_value(ctl, 0, 1);  // Enable
    
    // Set volumes
    ctl = mixer_get_ctl_by_name(mixer, "Speaker Playback Volume");
    if (ctl) {
        mixer_ctl_set_value(ctl, 0, 110);  // Left: 86%
        mixer_ctl_set_value(ctl, 1, 110);  // Right: 86%
    }
    
    // ... more controls ...
    
    mixer_close(mixer);
}
```

**Mixer Control Architecture:**

```
┌────────────────────────────────────────────────────────────────────┐
│                    WM8960 Mixer Control Path                       │
└────────────────────────────────────────────────────────────────────┘

PCM Audio Data
      │
      ▼
┌─────────────────┐
│   DAC (L/R)     │  ← "Playback Volume" (0-255)
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Left/Right Output Mixer            │
│  ┌──────────────────────────────┐   │
│  │ PCM Playback Switch          │◄──┼── Control #51, #54 (ON/OFF)
│  │ (Routes DAC to output)       │   │
│  └──────────────────────────────┘   │
└───────────────┬─────────────────────┘
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
┌───────────────┐   ┌───────────────┐
│   Speaker     │   │  Headphone    │
│   Amplifier   │   │  Amplifier    │
│               │   │               │
│  - DC Volume  │   │  - Volume     │
│  - AC Volume  │   │    (0-127)    │
│  - Volume     │   └───────┬───────┘
│    (0-127)    │           │
└───────┬───────┘           ▼
        │              🎧 Headphones
        ▼
    🔊 Speaker
```

**Critical Controls:**

| Control ID | Control Name | Function | Values | Impact |
|------------|--------------|----------|--------|--------|
| #51 | Left Output Mixer PCM Playback Switch | Routes left DAC to output | 0=Off, 1=On | **CRITICAL** - No audio if off |
| #54 | Right Output Mixer PCM Playback Switch | Routes right DAC to output | 0=Off, 1=On | **CRITICAL** - No audio if off |
| #12 | Speaker Playback Volume | Speaker output level | 0-127 | Sets speaker loudness |
| #10 | Headphone Playback Volume | Headphone output level | 0-127 | Sets headphone loudness |
| #15 | Playback Volume | Master DAC gain | 0-255 | Master digital volume |
| #13 | Speaker DC Volume | DC bias for Class-D amp | 0-5 | Amplifier efficiency |
| #14 | Speaker AC Volume | AC gain for Class-D amp | 0-5 | Amplifier gain |

---

## 4️⃣ **Output Stream Lifecycle**

### **4.1 Stream Creation (`adev_open_output_stream`)**

```
┌────────────────────────────────────────────────────────────────────────┐
│              adev_open_output_stream() Flow                            │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────┐
    │ 1. Validate PCM Capabilities                    │
    │    params = pcm_params_get(card, device, OUT)   │
    │    Check hardware supports requested config     │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 2. Allocate Stream Structure                    │
    │    out = calloc(1, sizeof(alsa_stream_out))     │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 3. Assign Function Pointers                     │
    │    out->stream.write = out_write                │
    │    out->stream.get_latency = out_get_latency    │
    │    out->stream.set_volume = out_set_volume      │
    │    out->stream.standby = out_standby            │
    │    ... (15+ functions)                          │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 4. Configure PCM Parameters                     │
    │    out->config.channels = 2                     │
    │    out->config.rate = 48000                     │
    │    out->config.format = PCM_FORMAT_S16_LE       │
    │    out->config.period_size = 1024               │
    │    out->config.period_count = 4                 │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 5. Negotiate with AudioFlinger                  │
    │    if (requested_config != hal_config)          │
    │      config->sample_rate = 48000                │
    │      config->format = AUDIO_FORMAT_PCM_16_BIT   │
    │      config->channel_mask = STEREO              │
    │      return -EINVAL (ask AudioFlinger to retry) │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 6. Initialize Stream State                      │
    │    out->dev = adev                              │
    │    out->standby = 1 (standby mode)              │
    │    out->unavailable = false                     │
    │    out->written = 0                             │
    │    pthread_mutex_init(&out->lock)               │
    └─────────────────┬───────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────────┐
    │ 7. Return Stream Handle                         │
    │    *stream_out = &out->stream                   │
    │    return 0                                     │
    └─────────────────────────────────────────────────┘
```

**Key Points:**
- **Stream NOT opened yet** - PCM device opened on first write
- **Standby mode** - No power consumption until audio plays
- **Config negotiation** - HAL tells AudioFlinger what formats it supports

---

### **4.2 Stream Start (`start_output_stream`)**

Called on first `out_write()` after standby.

```c
static int start_output_stream(struct alsa_stream_out *out)
{
    struct alsa_audio_device *adev = out->dev;
    
    if (out->unavailable)
        return -ENODEV;
    
    // Configure buffer thresholds
    out->write_threshold = PLAYBACK_PERIOD_COUNT * PERIOD_SIZE;
    out->config.start_threshold = PLAYBACK_PERIOD_START_THRESHOLD * PERIOD_SIZE;
    out->config.avail_min = PERIOD_SIZE;
    
    // Open PCM device with MMAP mode
    out->pcm = pcm_open(pcm_card, pcm_device, 
                        PCM_OUT | PCM_MMAP | PCM_NOIRQ | PCM_MONOTONIC,
                        &out->config);
    
    if (!pcm_is_ready(out->pcm)) {
        ALOGE("cannot open pcm_out driver: %s", pcm_get_error(out->pcm));
        pcm_close(out->pcm);
        adev->active_output = NULL;
        out->unavailable = true;
        return -ENODEV;
    }
    
    adev->active_output = out;
    return 0;
}
```

**PCM Open Flags:**
- `PCM_OUT` - Output direction (playback)
- `PCM_MMAP` - Memory-mapped I/O (low latency)
- `PCM_NOIRQ` - No interrupt-driven mode
- `PCM_MONOTONIC` - Use monotonic clock for timestamps

**Buffer Thresholds:**
```c
write_threshold = 4 * 1024 = 4096 frames (85 ms)
start_threshold = 2 * 1024 = 2048 frames (42 ms)  ← Start playback when buffer reaches this
avail_min = 1024 frames (21 ms)                   ← Wake up when this much space available
```

---

### **4.3 Audio Write (`out_write`)**

```c
static ssize_t out_write(struct audio_stream_out *stream, const void* buffer,
                         size_t bytes)
{
    int ret;
    struct alsa_stream_out *out = (struct alsa_stream_out *)stream;
    struct alsa_audio_device *adev = out->dev;
    size_t frame_size = audio_stream_out_frame_size(stream);
    size_t out_frames = bytes / frame_size;
    
    // Lock device + stream
    pthread_mutex_lock(&adev->lock);
    pthread_mutex_lock(&out->lock);
    
    // Wake from standby if needed
    if (out->standby) {
        ret = start_output_stream(out);
        if (ret != 0) {
            pthread_mutex_unlock(&adev->lock);
            goto exit;
        }
        out->standby = 0;
    }
    
    pthread_mutex_unlock(&adev->lock);
    
    // Write to PCM using MMAP
    ret = pcm_mmap_write(out->pcm, buffer, out_frames * frame_size);
    if (ret == 0) {
        out->written += out_frames;  // Track position
    }
    
exit:
    pthread_mutex_unlock(&out->lock);
    
    // Simulate write time on error
    if (ret != 0) {
        usleep((int64_t)bytes * 1000000 / frame_size /
               out_get_sample_rate(&stream->common));
    }
    
    return bytes;  // Always return requested bytes
}
```

**Write Flow Diagram:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                       out_write() Execution                            │
└────────────────────────────────────────────────────────────────────────┘

AudioFlinger Thread
      │
      ▼
┌─────────────────────────┐
│ Call out_write()        │
│ buffer = 4096 bytes     │  ← One period of audio
│ (1024 frames × 4 bytes) │
└────────┬────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Lock device & stream mutexes   │  ← Thread safety
└────────┬───────────────────────┘
         │
         ▼
    ┌────────────────┐
    │  In standby?   │
    └────┬──────┬────┘
         │ Yes  │ No
         ▼      └───────────────┐
┌────────────────────────┐      │
│ start_output_stream()  │      │
│ - Open PCM device      │      │
│ - Configure buffers    │      │
│ - Exit standby         │      │
└────────┬───────────────┘      │
         │                      │
         ▼                      │
┌────────────────────────────────┴──┐
│ pcm_mmap_write()                  │
│ - Copy buffer to kernel ring      │
│ - Update write pointer            │
│ - Hardware DMA starts if needed   │
└────────┬──────────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Update position tracking       │
│ out->written += 1024           │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Unlock mutexes                 │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ Return bytes (4096)            │
└────────────────────────────────┘
```

**MMAP Write Details:**

```
User Space (HAL)                  Kernel Space (ALSA)
┌──────────────┐                 ┌─────────────────┐
│  AudioFlinger│                 │ Kernel Ring     │
│    Buffer    │                 │ Buffer          │
│              │                 │  ┌───┬───┬───┐  │
│  [PCM Data]  │──pcm_mmap_write→│  │ 0 │ 1 │ 2 │  │
│   4096 bytes │                 │  └───┴───┴───┘  │
└──────────────┘                 │     ↓ ↓ ↓       │
                                 │   Hardware      │
                                 │   DMA Transfer  │
                                 └─────────────────┘
                                          │
                                          ▼
                                 ┌─────────────────┐
                                 │  WM8960 Codec   │
                                 │  (I2S Interface)│
                                 └─────────────────┘
```

---

### **4.4 Standby Mode (`out_standby`)**

```c
static int do_output_standby(struct alsa_stream_out *out)
{
    struct alsa_audio_device *adev = out->dev;
    
    if (!out->standby) {
        pcm_close(out->pcm);         // Close PCM device
        out->pcm = NULL;
        adev->active_output = NULL;
        out->standby = 1;            // Mark as standby
    }
    return 0;
}
```

**Standby Trigger:**
- AudioFlinger calls `standby()` after ~3 seconds of inactivity
- Closes PCM device to save power
- Next write will reopen device

**State Transitions:**

```
┌──────────┐   out_write()    ┌──────────┐   3s idle    ┌──────────┐
│  Initial │─────────────────►│  Active  │────────────►│ Standby  │
│ (Standby)│                  │ (Playing)│              │  (Idle)  │
└──────────┘                  └─────┬────┘              └────┬─────┘
      ▲                             │                        │
      │                             │ out_standby()          │
      │                             └────────────────────────┘
      │                                                      │
      └──────────────────────────────────────────────────────┘
                          out_write()
```

---

## 5️⃣ **Audio Routing & Device Selection**

### **5.1 Device Routing**

```c
static int out_set_parameters(struct audio_stream *stream, const char *kvpairs)
{
    struct alsa_stream_out *out = (struct alsa_stream_out *)stream;
    struct alsa_audio_device *adev = out->dev;
    struct str_parms *parms;
    char value[32];
    int val = 0;
    int ret = -EINVAL;
    
    parms = str_parms_create_str(kvpairs);
    
    // Check for routing parameter
    if (str_parms_get_str(parms, AUDIO_PARAMETER_STREAM_ROUTING, 
                          value, sizeof(value)) >= 0) {
        val = atoi(value);
        pthread_mutex_lock(&adev->lock);
        pthread_mutex_lock(&out->lock);
        
        // Update device mask if changed
        if (((adev->devices & AUDIO_DEVICE_OUT_ALL) != val) && (val != 0)) {
            adev->devices &= ~AUDIO_DEVICE_OUT_ALL;
            adev->devices |= val;
        }
        
        pthread_mutex_unlock(&out->lock);
        pthread_mutex_unlock(&adev->lock);
        ret = 0;
    }
    
    str_parms_destroy(parms);
    return ret;
}
```

**Routing Example:**

```
AudioFlinger: "routing=2"
              ↓
    AUDIO_DEVICE_OUT_SPEAKER (0x2)
              ↓
    adev->devices = 0x2
              ↓
    Audio routes to WM8960 speaker
```

**Device Mask Values:**
```c
AUDIO_DEVICE_OUT_SPEAKER          = 0x00000002
AUDIO_DEVICE_OUT_WIRED_HEADSET    = 0x00000004
AUDIO_DEVICE_OUT_WIRED_HEADPHONE  = 0x00000008
AUDIO_DEVICE_OUT_BLUETOOTH_SCO    = 0x00000010
AUDIO_DEVICE_OUT_BLUETOOTH_A2DP   = 0x00000080
```

---

## 6️⃣ **Latency & Timing**

### **6.1 Latency Calculation**

```c
static uint32_t out_get_latency(const struct audio_stream_out *stream)
{
    struct alsa_stream_out *out = (struct alsa_stream_out *)stream;
    return (PERIOD_SIZE * PLAYBACK_PERIOD_COUNT * 1000) / out->config.rate;
}
```

**Calculation:**
```
Latency = (1024 × 4 × 1000) / 48000 = 85.33 ms
```

**Latency Breakdown:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Audio Latency Components                           │
└────────────────────────────────────────────────────────────────────────┘

AudioFlinger Buffer (16 KB)                    Hardware Buffer (16 KB)
┌────────────────────────┐                    ┌────────────────────────┐
│ Period 0: 1024 frames  │                    │ Period 0: 1024 frames  │
│ Period 1: 1024 frames  │ ─── HAL write() ──►│ Period 1: 1024 frames  │
│ Period 2: 1024 frames  │                    │ Period 2: 1024 frames  │
│ Period 3: 1024 frames  │                    │ Period 3: 1024 frames  │
└────────────────────────┘                    └────────┬───────────────┘
                                                       │ DMA
  ◄──────── 85 ms ──────►                             ▼
                                              ┌────────────────────────┐
                                              │  WM8960 DAC            │
                                              │  (Analog conversion)   │
                                              └────────┬───────────────┘
                                                       │
                                                       ▼ ~2 ms
                                                  🔊 Speaker

Total Latency: ~85-100 ms (acceptable for media, not ideal for gaming/VoIP)
```

---

### **6.2 Presentation Timestamp**

```c
static int out_get_presentation_position(const struct audio_stream_out *stream,
                                         uint64_t *frames, struct timespec *timestamp)
{
    struct alsa_stream_out *out = (struct alsa_stream_out *)stream;
    int ret = -1;
    
    if (out->pcm) {
        unsigned int avail;
        if (pcm_get_htimestamp(out->pcm, &avail, timestamp) == 0) {
            // Calculate frames actually played
            size_t kernel_buffer_size = out->config.period_size * out->config.period_count;
            int64_t signed_frames = out->written - kernel_buffer_size + avail;
            
            if (signed_frames >= 0) {
                *frames = signed_frames;
                ret = 0;
            }
        }
    }
    
    return ret;
}
```

**Position Calculation:**

```
Frames Played = Total Written - Buffer Size + Available Space

Example:
  Written:      10000 frames
  Buffer Size:   4096 frames
  Available:     1024 frames (1 period free)
  
  Played = 10000 - 4096 + 1024 = 6928 frames
  Time   = 6928 / 48000 = 0.144 seconds
```

---

## 7️⃣ **Input Stream (Stub Implementation)**

The current implementation provides a **stub input stream** (recording not implemented):

```c
static ssize_t in_read(struct audio_stream_in *stream, void* buffer, size_t bytes)
{
    // Fake timing for audio input
    usleep((int64_t)bytes * 1000000 / audio_stream_in_frame_size(stream) /
           in_get_sample_rate(&stream->common));
    
    memset(buffer, 0, bytes);  // Return silence
    return bytes;
}
```

**To implement real recording:**
1. Add `struct alsa_stream_in` with PCM handle
2. Implement `start_input_stream()` with `pcm_open(PCM_IN)`
3. Replace `memset` with `pcm_read()` in `in_read()`
4. Add mixer configuration for microphone input

---

## 8️⃣ **Thread Safety & Locking**

### **Mutex Hierarchy**

```
┌────────────────────────────────────────────────────────────────────────┐
│                         Lock Ordering Rules                            │
└────────────────────────────────────────────────────────────────────────┘

ALWAYS lock in this order to prevent deadlock:

1. Device Lock (adev->lock)
   ↓
2. Stream Lock (out->lock)

Example (from out_write):
    pthread_mutex_lock(&adev->lock);     // 1. Device first
    pthread_mutex_lock(&out->lock);      // 2. Stream second
    
    // ... critical section ...
    
    pthread_mutex_unlock(&adev->lock);   // Unlock in any order
    pthread_mutex_unlock(&out->lock);
```

**Protected Resources:**

| Mutex | Protects | Used By |
|-------|----------|---------|
| `adev->lock` | `adev->devices`, `adev->active_output`, `adev->active_input` | Device operations |
| `out->lock` | `out->pcm`, `out->standby`, `out->written` | Stream operations |

---

## 9️⃣ **Power Management**

### **Standby Strategy**

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Power States                                      │
└────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│ DEEP STANDBY     │  Power: 0 mW
│ - PCM closed     │  State: standby=1, pcm=NULL
│ - No clocks      │  Entry: 3s idle
│ - Instant-on     │  Exit:  First write
└──────────────────┘
        ▲ │
      3s│ │ out_write()
  idle  │ │
        │ ▼
┌──────────────────┐
│ ACTIVE PLAYBACK  │  Power: ~50 mW (codec active)
│ - PCM open       │  State: standby=0, pcm!=NULL
│ - DMA running    │  Entry: out_write() from standby
│ - Audio playing  │  Exit:  out_standby()
└──────────────────┘
```

**Power Consumption Estimates:**
- Deep Standby: ~0 mW (PCM device closed)
- Active Playback: ~50 mW (WM8960 DAC + amplifier)
- Wake-up Time: <10 ms (PCM open + first write)

---

## 🔟 **Complete Call Flow Example**

### **Scenario: Play 1 second of audio**

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Complete Audio Playback Flow                        │
└────────────────────────────────────────────────────────────────────────┘

APPLICATION LAYER:
┌──────────────────────────────────────────────────────────────────────┐
│ MediaPlayer.start()                                                  │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
FRAMEWORK LAYER:                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ AudioFlinger::createTrack()                                          │
│   - Allocate shared memory buffer                                    │
│   - Determine output device (speaker=0x2)                            │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ AudioFlinger::openOutput()                                           │
│   - hw_get_module("audio")  ← Load audio.primary.rpi.so              │
│   - module->open(AUDIO_HARDWARE_INTERFACE)                           │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
HAL LAYER:                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│ adev_open()                                                          │
│   - pcm_card = 2 (WM8960)                                            │
│   - pcm_device = 0                                                   │
│   - init_wm8960_mixer(2)                                             │
│     ✓ Enable PCM routing switches                                   │
│     ✓ Set volumes                                                    │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ adev_open_output_stream()                                            │
│   - Allocate alsa_stream_out                                         │
│   - config: 48kHz, stereo, S16_LE, 1024×4                            │
│   - standby = 1 (not yet opened)                                     │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ AudioFlinger::PlaybackThread::threadLoop()                           │
│   - Mix audio from all tracks                                        │
│   - Every 21ms (1024 frames @ 48kHz)                                 │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
                                         ▼ (First write)
┌──────────────────────────────────────────────────────────────────────┐
│ out_write(buffer=4096 bytes)                                         │
│   - standby==1? YES                                                  │
│   - Call start_output_stream()                                       │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ start_output_stream()                                                │
│   - pcm_open(2, 0, PCM_OUT|PCM_MMAP)                                 │
│   - Opens /dev/snd/pcmC2D0p                                          │
│   - standby = 0                                                      │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ pcm_mmap_write(4096 bytes)                                           │
│   - Copy to kernel ring buffer                                       │
│   - Kernel DMA starts transfer to I2S                                │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
KERNEL LAYER:                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│ ALSA PCM Core                                                        │
│   - Ring buffer: 16KB (4 periods)                                    │
│   - DMA engine: BCM2712 I2S DMA                                      │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
                                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│ WM8960 Driver (sound/soc/codecs/wm8960.c)                            │
│   - I2S data → DAC                                                   │
│   - Mixer routing (from init_wm8960_mixer)                           │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
HARDWARE LAYER:                          ▼
┌──────────────────────────────────────────────────────────────────────┐
│ WM8960 Codec Chip                                                    │
│   - I2S input (digital audio)                                        │
│   - DAC converts to analog                                           │
│   - Class-D amplifier drives speaker                                 │
└────────────────────────────────────────┬─────────────────────────────┘
                                         │
                                         ▼
                                   🔊 AUDIO OUTPUT!

Timeline (1 second of audio):
T=0ms:     MediaPlayer.start()
T=1ms:     adev_open(), adev_open_output_stream()
T=2ms:     First out_write(), start_output_stream()
T=3ms:     pcm_mmap_write() - first buffer
T=24ms:    out_write() - buffer 2
T=45ms:    out_write() - buffer 3
...
T=1000ms:  Last out_write()
T=1021ms:  Last audio reaches speaker
T=4000ms:  out_standby() - close PCM after 3s idle
```

---

## 📊 Performance Characteristics

### **Buffer Configuration Analysis**

| Parameter | Value | Impact |
|-----------|-------|--------|
| **Period Size** | 1024 frames | 21.3 ms per buffer |
| **Period Count** | 4 periods | 85.3 ms total latency |
| **Buffer Size** | 16384 bytes | 4 KB per period |
| **Sample Rate** | 48000 Hz | Standard audio rate |
| **Channels** | 2 (Stereo) | Full stereo output |
| **Format** | S16_LE | 16-bit signed little-endian |
| **Frame Size** | 4 bytes | 2 channels × 2 bytes |

**Latency Budget:**
```
AudioFlinger mixing:     ~10 ms
HAL buffering:           ~85 ms  (4 periods)
Kernel DMA:              ~5 ms
Codec DAC:               ~2 ms
Total:                   ~102 ms
```

**CPU Usage (estimated):**
- Mixing (AudioFlinger): ~5% (1 core)
- HAL overhead: <1%
- DMA: 0% (hardware)
- **Total: ~5-6% CPU for playback**

---

## 🛠️ Debugging & Diagnostics

### **Key Log Tags**

```bash
# HAL logs
adb logcat -s audio_hw_rpi:V

# AudioFlinger logs
adb logcat -s AudioFlinger:V

# Audio policy logs
adb logcat -s AudioPolicyManager:V

# TinyALSA logs
adb logcat -s tinyalsa:V
```

### **Dumpsys Commands**

```bash
# Audio service state
adb shell dumpsys audio

# Audio policy state
adb shell dumpsys media.audio_policy

# AudioFlinger state
adb shell dumpsys media.audio_flinger
```

### **PCM State Inspection**

```bash
# Show all ALSA cards
adb shell cat /proc/asound/cards

# Show PCM devices
adb shell cat /proc/asound/pcm

# Show card 2 info
adb shell cat /proc/asound/card2/id
adb shell cat /proc/asound/card2/pcm0p/info
adb shell cat /proc/asound/card2/pcm0p/sub0/hw_params
adb shell cat /proc/asound/card2/pcm0p/sub0/status
```

**Example Output:**
```
# cat /proc/asound/card2/pcm0p/sub0/hw_params
access: MMAP_INTERLEAVED
format: S16_LE
subformat: STD
channels: 2
rate: 48000 (48000/1)
period_size: 1024
buffer_size: 4096

# cat /proc/asound/card2/pcm0p/sub0/status
state: RUNNING
owner_pid   : 3090
trigger_time: 1234567.123456789
tstamp      : 1234567.987654321
delay       : 2048
avail       : 1024
avail_max   : 4096
```

---

## 🔬 Advanced Topics

### **MMAP Mode Details**

**Traditional vs MMAP Mode:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Traditional Write Mode                          │
└─────────────────────────────────────────────────────────────────────┘

User Space              Kernel Space
┌──────────┐            ┌──────────┐
│  Buffer  │────copy───►│  Kernel  │
│  4096 B  │            │  Buffer  │
└──────────┘            └────┬─────┘
                             │ DMA
                             ▼
                         Hardware

Latency: +1 copy overhead (~0.5ms)


┌─────────────────────────────────────────────────────────────────────┐
│                        MMAP Mode                                    │
└─────────────────────────────────────────────────────────────────────┘

User Space              Kernel Space
┌──────────┐            ┌──────────┐
│  Buffer  │◄──mmap────►│  Kernel  │
│ (Shared) │            │  Buffer  │
└──────────┘            └────┬─────┘
                             │ DMA
                             ▼
                         Hardware

Latency: No copy, direct write (~0.1ms)
```

**Benefits:**
- Lower latency (no user→kernel copy)
- Lower CPU usage (no memcpy)
- Zero-copy architecture

---

### **Mixer Control Internals**

**ALSA Mixer Control Structure:**

```c
struct mixer_ctl {
    struct mixer *mixer;
    struct snd_ctl_elem_info *info;
    char **ename;
};

// Example: Get control by name
mixer = mixer_open(2);                                    // Card 2
ctl = mixer_get_ctl_by_name(mixer, "Speaker Volume");    // Find control
mixer_ctl_set_value(ctl, 0, 110);                        // Set left channel
mixer_ctl_set_value(ctl, 1, 110);                        // Set right channel
mixer_close(mixer);
```

**Kernel Path:**
```
mixer_ctl_set_value()
    ↓
ioctl(fd, SNDRV_CTL_IOCTL_ELEM_WRITE)
    ↓
snd_ctl_elem_write()
    ↓
wm8960_put_volsw()  ← Codec driver callback
    ↓
i2c_smbus_write_byte_data()  ← Write to WM8960 register
```

---

## 📚 References & Resources

### **Android Documentation**
- [Audio HAL Interface](https://source.android.com/docs/core/audio/implement-hal)
- [Audio Policy Configuration](https://source.android.com/docs/core/audio/audio-policy)
- [TinyALSA Library](https://github.com/tinyalsa/tinyalsa)

### **ALSA Documentation**
- [ALSA Project](https://www.alsa-project.org/)
- [Writing an ALSA Driver](https://www.alsa-project.org/~tiwai/writing-an-alsa-driver/)

### **WM8960 Codec**
- [WM8960 Datasheet](https://www.cirrus.com/products/wm8960/)
- [Device Tree Bindings](https://www.kernel.org/doc/Documentation/devicetree/bindings/sound/wm8960.txt)

### **Source Code Locations**
- HAL Implementation: `/device/brcm/rpi5/audio/audio_hw.c`
- Audio Header: `/hardware/libhardware/include/hardware/audio.h`
- TinyALSA: `/external/tinyalsa/`
- AudioFlinger: `/frameworks/av/services/audioflinger/`
- Audio Policy: `/frameworks/av/services/audiopolicy/`

---

## ✅ Summary

### **Key Takeaways:**

1. **Three-Layer Design:**
   - Framework (AudioFlinger) → HAL (audio.primary.rpi.so) → Kernel (ALSA)

2. **Core Components:**
   - Device: `alsa_audio_device` (singleton)
   - Output Stream: `alsa_stream_out` (per-stream)
   - PCM Config: 48kHz, stereo, 16-bit, 1024×4 buffer

3. **Critical Functions:**
   - `adev_open()` - Initialize HAL
   - `adev_open_output_stream()` - Create stream
   - `start_output_stream()` - Open PCM device
   - `out_write()` - Write audio data
   - `init_wm8960_mixer()` - Configure codec

4. **WM8960 Integration:**
   - Mixer initialization at boot
   - PCM routing switches (controls #51, #54)
   - Volume controls (speaker, headphone, master)

5. **Performance:**
   - Latency: ~85-100 ms
   - CPU: ~5-6% for playback
   - Power: 0 mW (standby), ~50 mW (active)

6. **Thread Safety:**
   - Device lock → Stream lock (always this order)
   - MMAP mode for zero-copy writes

---

**Document Version:** 1.0  
**Last Updated:** October 30, 2025  
**Author:** AI Assistant  
**Platform:** Raspberry Pi 5 AOSP

