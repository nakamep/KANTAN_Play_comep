# Technical Design Document: ESP32 Software MIDI Synthesizer Extension for KANTAN Play Core

## Table of Contents
1. [M5Stack Core S3 Hardware Overview](#1-m5stack-core-s3-hardware-overview)
2. [Open-Source ESP32 MIDI Synthesizer Survey](#2-open-source-esp32-midi-synthesizer-survey)
3. [Composite Integration Strategy](#3-composite-integration-strategy)
4. [Feature Extension Roadmap](#4-feature-extension-roadmap)
5. [Appendices](#5-appendices)

---

## 1. M5Stack Core S3 Hardware Overview

### 1.1 CPU and Memory Specifications

| Component | Specification | Impact on Audio Processing |
|-----------|---------------|---------------------------|
| CPU | ESP32-S3 Dual-core Xtensa LX7 @ 240MHz | Sufficient for real-time audio synthesis with proper task allocation |
| RAM | 512KB SRAM + 8MB PSRAM | PSRAM enables large sample buffers and multiple synth engines |
| Flash | 16MB | Adequate for firmware, samples, and soundfonts |
| FPU | Single-precision floating-point | Hardware-accelerated audio DSP operations |

### 1.2 Audio Interface Analysis

**Current I2S Implementation (task_i2s.cpp:64-67):**
```cpp
i2s_config.gpio_cfg.bclk = def::hw::pin::i2s_bck;   // Bit Clock
i2s_config.gpio_cfg.ws   = def::hw::pin::i2s_ws;    // Word Select
i2s_config.gpio_cfg.dout = def::hw::pin::i2s_out;   // Data Out
i2s_config.gpio_cfg.mclk = def::hw::pin::i2s_mclk;  // Master Clock
```

**Audio Configuration:**
- Sample Rate: 48kHz (configurable)
- Bit Depth: 32-bit I2S
- Channels: Stereo
- DMA Buffers: 4 descriptors × 96 frames
- Audio Codec: ES8388 (hardware MIDI synthesis)

### 1.3 Real-Time Processing Considerations

**Current Audio Pipeline:**
1. ES8388 generates audio → I2S input
2. Volume control and mixing in task_i2s
3. Audio output via I2S to ES8388 DAC

**CPU Allocation (from common_define.hpp):**
- Audio tasks typically run on Core 1
- MIDI tasks on Core 0
- Available CPU cycles for software synthesis: ~40-60%

**Memory Bottlenecks:**
- DMA-capable memory required for I2S buffers
- PSRAM access latency: ~80ns (acceptable for non-real-time operations)
- Internal SRAM: Critical for real-time audio buffers

### 1.4 GPIO and Interface Availability

**Available Interfaces:**
- I2C: Internal (ES8388, sensors) and External (expansion modules)
- SPI: Display, SD card, external modules
- UART: MIDI input/output
- USB: MIDI over USB-C (ESP32-S3 native USB)
- WiFi/Bluetooth: Wireless MIDI

**Power Constraints:**
- Battery operation requires power-efficient audio processing
- Dynamic frequency scaling may be needed for battery life

---

## 2. Open-Source ESP32 MIDI Synthesizer Survey

### 2.1 Primary Candidates

| Project | License | Polyphony | Features | Dependencies | Portability |
|---------|---------|-----------|----------|--------------|-------------|
| **ESP32Synth** | MIT | 20+ voices | Wavetable, ADSR, Oscilloscope | Arduino Core | ★★★★☆ |
| **esp32_fm_synth** | Custom | 6 voices | FM synthesis, YM2612 emulation | Arduino Core, ML_SynthTools | ★★★☆☆ |
| **TinySoundFont** | MIT | Variable | SoundFont2 playback | Standard C libraries | ★★★★★ |
| **esp32_midi_sampler** | Custom | Variable | Sample playback, LittleFS | Arduino Core, AC101 | ★★☆☆☆ |
| **esp32soundsynth** | MIT | 8-bit | Lo-fi wavetable | Arduino Core | ★★★★☆ |

### 2.2 Detailed Analysis

#### 2.2.1 ESP32Synth by bartpleiter
- **URL:** https://github.com/bartpleiter/ESP32Synth
- **License:** MIT
- **Core Features:**
  - 20+ voice polyphony
  - Square, Saw, Sine, Triangle waveforms
  - Exponential ADSR envelopes
  - 40kHz sample rate, 8-bit DAC
  - Nokia 5110 display integration
- **Dependencies:** Arduino Core for ESP32
- **Memory Usage:** ~200KB RAM (estimated)
- **CPU Usage:** ~30-40% at full polyphony
- **Integration Compatibility:** ★★★★☆ (Good MIDI interface, display conflicts)

#### 2.2.2 TinySoundFont Integration
- **URL:** https://github.com/schellingb/TinySoundFont
- **License:** MIT
- **Core Features:**
  - SoundFont2 (.sf2) file support
  - High-quality sample-based synthesis
  - Minimal dependencies (fopen, math, malloc)
  - Single header implementation
- **Memory Requirements:** 2-20MB for soundfonts (PSRAM compatible)
- **CPU Usage:** Variable based on polyphony and sample complexity
- **Integration Advantages:**
  - Professional-quality sound
  - Extensive instrument libraries available
  - Proven embedded compatibility (ESP8266Audio library)

#### 2.2.3 marcel-licence FM Synthesizer
- **URL:** https://github.com/marcel-licence/esp32_fm_synth
- **License:** Non-commercial
- **Core Features:**
  - 6-voice polyphony with 4 operators each
  - 8 FM algorithms (YM2612 compatible)
  - Reverb and delay effects
  - Float-precision FM core
- **Memory Usage:** ~150KB RAM
- **CPU Usage:** ~25-35% (optimized assembler)
- **License Limitation:** Restricts commercial use

### 2.3 Performance Comparison Matrix

| Synthesizer | RAM Usage | CPU Load | Audio Quality | MIDI Compatibility | Real-time Suitability |
|-------------|-----------|----------|---------------|-------------------|----------------------|
| ESP32Synth | 200KB | Medium | Good | Excellent | ★★★★☆ |
| TinySoundFont | 2-20MB | High | Excellent | Good | ★★★☆☆ |
| FM Synth | 150KB | Medium | Very Good | Excellent | ★★★★☆ |
| MIDI Sampler | 500KB-2MB | Variable | Excellent | Good | ★★★☆☆ |
| SoundSynth | 100KB | Low | Fair | Good | ★★★★★ |

---

## 3. Composite Integration Strategy

### 3.1 Architecture Decision: Hybrid Multi-Engine Approach

**Recommended Strategy:** Implement a layered architecture supporting multiple synthesis engines with dynamic switching based on performance requirements and user preferences.

### 3.2 Primary Engine Selection

**Tier 1 - High Performance:**
- **ESP32Synth** (wavetable synthesis)
- **esp32soundsynth** (lo-fi synthesis)

**Tier 2 - Medium Performance:**
- **FM Synthesizer** (when licensing permits)
- **TinySoundFont** (small soundfonts only)

**Tier 3 - Specialized:**
- **MIDI Sampler** (for specific instruments)

### 3.3 System Registry Integration Points

Based on the current `system_registry.hpp` analysis, the following integration points are identified:

#### 3.3.1 New Registry Sections

```cpp
// Software synthesizer control
struct reg_software_synth_t : public registry_t {
    enum index_t : uint16_t {
        SYNTH_ENGINE_SELECT,      // 0=Hardware, 1=ESP32Synth, 2=TinySoundFont, etc.
        SYNTH_POLYPHONY_LIMIT,    // Maximum voices
        SYNTH_SAMPLE_RATE,        // Audio sample rate
        SYNTH_BUFFER_SIZE,        // Audio buffer size
        SYNTH_MASTER_VOLUME,      // Software synth volume
        SYNTH_ENGINE_STATUS,      // Current engine status
        SOUNDFONT_BANK_SELECT,    // Active soundfont bank
        WAVETABLE_SELECT,         // Active wavetable
    };
} software_synth;

// Audio mixing control
struct reg_audio_mixer_t : public registry_t {
    enum index_t : uint16_t {
        HARDWARE_SYNTH_LEVEL,     // ES8388 level
        SOFTWARE_SYNTH_LEVEL,     // Software synth level
        MIX_MODE,                 // Mixing mode (replace/mix/layer)
        CROSSFADE_TIME,           // Switching transition time
    };
} audio_mixer;
```

#### 3.3.2 Task Architecture Integration

**New Tasks:**
- `task_software_synth`: Software synthesis engine management
- `task_audio_mixer`: Hardware/software audio mixing

**Modified Tasks:**
- `task_i2s`: Enhanced for bidirectional audio mixing
- `task_midi`: MIDI routing to multiple synthesis engines
- `task_kantanplay`: Engine selection and switching logic

### 3.4 Audio Pipeline Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   MIDI Input    │───▶│  task_midi       │───▶│ Engine Router   │
│ (BLE/USB/UART)  │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                         │
                       ┌─────────────────────────────────┼─────────────────────────────────┐
                       ▼                                 ▼                                 ▼
        ┌─────────────────────┐           ┌─────────────────────┐           ┌─────────────────────┐
        │   ES8388 Hardware   │           │   ESP32Synth        │           │   TinySoundFont     │
        │   Synthesis         │           │   Wavetable         │           │   SoundFont2        │
        └─────────────────────┘           └─────────────────────┘           └─────────────────────┘
                       │                                 │                                 │
                       └─────────────────────────────────┼─────────────────────────────────┘
                                                         ▼
                                              ┌─────────────────┐
                                              │  task_i2s       │
                                              │  Audio Mixer    │
                                              └─────────────────┘
                                                         │
                                                         ▼
                                              ┌─────────────────┐
                                              │   I2S Output    │
                                              │   (ES8388 DAC)  │
                                              └─────────────────┘
```

### 3.5 Memory Management Strategy

**Static Allocation:**
- Core synthesis buffers in internal SRAM
- Fixed-size voice pools

**Dynamic Allocation (PSRAM):**
- SoundFont sample data
- Large wavetables
- Audio effect buffers

**Memory Partitioning:**
```cpp
// Memory allocation strategy
#define SYNTH_VOICE_POOL_SIZE    24        // Maximum concurrent voices
#define AUDIO_BUFFER_SIZE        512       // Samples per buffer
#define SOUNDFONT_CACHE_SIZE     (4*1024*1024)  // 4MB PSRAM cache
#define WAVETABLE_CACHE_SIZE     (1*1024*1024)  // 1MB wavetable cache
```

---

## 4. Feature Extension Roadmap

### 4.1 Milestone 1: Environment Setup & Build Verification
**Duration:** 3-5 days

#### Tasks:
1. **Development Environment Setup** (1 day)
   - Install ESP32-S3 development tools
   - Configure PlatformIO for Core S3 target
   - Set up audio analysis tools (oscilloscope software)
   - **Acceptance Criteria:** Clean build of existing KANTAN Play Core for ESP32-S3

2. **Hardware Audio Interface Verification** (1 day)
   - Test I2S bidirectional communication
   - Verify ES8388 codec functionality
   - Measure audio latency and quality metrics
   - **Acceptance Criteria:** Baseline audio performance documented

3. **System Registry Extension** (1 day)
   - Add software synthesizer registry sections
   - Implement audio mixer registry
   - Add configuration persistence
   - **Acceptance Criteria:** Registry compiles and persists settings

4. **Initial Synthesis Engine Integration** (2 days)
   - Port ESP32Synth library to project structure
   - Create task_software_synth skeleton
   - Implement basic MIDI note on/off
   - **Acceptance Criteria:** Single-voice software synthesis functional

### 4.2 Milestone 2: Single Engine Implementation
**Duration:** 5-7 days

#### Tasks:
1. **ESP32Synth Full Integration** (2 days)
   - Implement full polyphony (20+ voices)
   - Add ADSR envelope control
   - Integrate wavetable selection
   - **Acceptance Criteria:** Full ESP32Synth feature parity

2. **Audio Pipeline Integration** (2 days)
   - Implement task_audio_mixer
   - Add hardware/software audio mixing
   - Optimize I2S buffer management
   - **Acceptance Criteria:** Clean audio mixing without artifacts

3. **MIDI Routing Enhancement** (1 day)
   - Extend task_midi for engine routing
   - Add MIDI channel assignment per engine
   - Implement MIDI controller mapping
   - **Acceptance Criteria:** MIDI controls both hardware and software engines

4. **Basic UI Integration** (2 days)
   - Add software synth settings to menu system
   - Implement engine selection interface
   - Add audio level visualization
   - **Acceptance Criteria:** User can switch between synthesis engines

### 4.3 Milestone 3: Multi-Engine Support
**Duration:** 7-10 days

#### Tasks:
1. **TinySoundFont Integration** (3 days)
   - Port TinySoundFont library
   - Implement SoundFont loading from SD card
   - Add PSRAM-based sample caching
   - **Acceptance Criteria:** Basic SoundFont playback functional

2. **Engine Arbitration System** (2 days)
   - Implement voice stealing algorithms
   - Add CPU load monitoring
   - Create dynamic polyphony adjustment
   - **Acceptance Criteria:** System maintains real-time performance under load

3. **Advanced Audio Mixing** (2 days)
   - Implement crossfading between engines
   - Add per-engine volume control
   - Optimize mixing algorithms for ESP32-S3
   - **Acceptance Criteria:** Smooth transitions between synthesis engines

4. **Memory Management Optimization** (2 days)
   - Implement efficient PSRAM usage
   - Add garbage collection for unused samples
   - Optimize buffer allocation strategies
   - **Acceptance Criteria:** Memory usage remains stable during operation

5. **Configuration System** (1 day)
   - Add engine-specific parameter storage
   - Implement preset management
   - Add configuration backup/restore
   - **Acceptance Criteria:** Settings persist across power cycles

### 4.4 Milestone 4: Performance Optimization
**Duration:** 5-7 days

#### Tasks:
1. **CPU Load Optimization** (2 days)
   - Profile synthesis engine performance
   - Implement assembly optimizations where needed
   - Add dynamic frequency scaling
   - **Acceptance Criteria:** CPU usage <60% at maximum polyphony

2. **Audio Latency Reduction** (2 days)
   - Optimize I2S buffer sizes
   - Reduce synthesis engine processing delays
   - Implement priority-based task scheduling
   - **Acceptance Criteria:** Audio latency <10ms end-to-end

3. **Memory Access Optimization** (1 day)
   - Optimize PSRAM access patterns
   - Implement intelligent caching
   - Reduce memory fragmentation
   - **Acceptance Criteria:** Consistent memory performance

4. **Power Consumption Optimization** (2 days)
   - Implement idle synthesis engine power-down
   - Add adaptive sample rate adjustment
   - Optimize FreeRTOS task scheduling
   - **Acceptance Criteria:** Battery life impact <20%

### 4.5 Milestone 5: Advanced Features
**Duration:** 8-10 days

#### Tasks:
1. **Dynamic Engine Switching** (2 days)
   - Implement seamless engine transitions
   - Add morph/blend capabilities between engines
   - Create smart engine selection based on MIDI content
   - **Acceptance Criteria:** Smooth transitions without audio dropouts

2. **Effect Processing Integration** (3 days)
   - Add reverb/delay to software engines
   - Implement filter modules
   - Add modulation capabilities (LFO, envelope)
   - **Acceptance Criteria:** High-quality audio effects functional

3. **External Control Integration** (2 days)
   - Add USB MIDI controller support
   - Implement CC parameter mapping
   - Add external device synchronization
   - **Acceptance Criteria:** External controllers modify synthesis parameters

4. **Performance Monitoring** (1 day)
   - Add real-time CPU usage display
   - Implement audio quality metrics
   - Create diagnostic tools
   - **Acceptance Criteria:** System performance visible to user

5. **Documentation and Examples** (2 days)
   - Create user manual sections
   - Add example configurations
   - Document API changes
   - **Acceptance Criteria:** Complete documentation for new features

### 4.6 Milestone 6: Testing & Validation
**Duration:** 5-7 days

#### Tasks:
1. **Unit Testing** (2 days)
   - Test individual synthesis engines
   - Validate MIDI processing accuracy
   - Test memory management under stress
   - **Acceptance Criteria:** All unit tests pass

2. **Integration Testing** (2 days)
   - Test engine switching scenarios
   - Validate audio quality across configurations
   - Test power consumption in various modes
   - **Acceptance Criteria:** System stable under all test conditions

3. **Performance Validation** (1 day)
   - Benchmark against baseline performance
   - Validate latency requirements
   - Test maximum polyphony limits
   - **Acceptance Criteria:** Performance meets or exceeds specifications

4. **User Acceptance Testing** (2 days)
   - Test with real musical scenarios
   - Validate UI/UX improvements
   - Test configuration persistence
   - **Acceptance Criteria:** System ready for end-user deployment

### 4.7 Total Project Timeline
**Estimated Duration:** 33-46 days
**Resource Requirements:** 1-2 embedded software engineers
**Risk Factors:** Audio quality, real-time performance, memory constraints

---

## 5. Appendices

### 5.1 Hardware Datasheets and References

#### 5.1.1 ESP32-S3 Technical Reference
- **URL:** https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf
- **Key Sections:**
  - Chapter 29: I2S Controller
  - Chapter 4: Memory Management Unit
  - Chapter 7: System and Memory

#### 5.1.2 ES8388 Audio Codec
- **URL:** https://www.everest-semi.com/pdf/ES8388%20DS.pdf
- **Key Features:**
  - 24-bit ADC/DAC
  - I2S interface
  - Built-in microphone preamp
  - Hardware volume control

#### 5.1.3 M5Stack Core S3 Specifications
```
CPU: ESP32-S3 Dual-core @ 240MHz
RAM: 512KB SRAM + 8MB PSRAM
Flash: 16MB
Audio: ES8388 codec with built-in amplifier
Display: 2.0" IPS LCD (320×240)
Connectivity: WiFi 802.11 b/g/n, Bluetooth 5.0, USB-C
```

### 5.2 ESP-IDF and Arduino Core Documentation

#### 5.2.1 I2S Configuration Example
```cpp
i2s_config_t i2s_config = {
    .mode = I2S_MODE_MASTER | I2S_MODE_TX | I2S_MODE_RX,
    .sample_rate = 48000,
    .bits_per_sample = I2S_BITS_PER_SAMPLE_32BIT,
    .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT,
    .communication_format = I2S_COMM_FORMAT_STAND_I2S,
    .intr_alloc_flags = ESP_INTR_FLAG_LEVEL1,
    .dma_buf_count = 4,
    .dma_buf_len = 512,
    .use_apll = true,
    .tx_desc_auto_clear = true,
    .fixed_mclk = 0
};
```

#### 5.2.2 FreeRTOS Task Creation
```cpp
xTaskCreatePinnedToCore(
    synthesis_task,           // Task function
    "synth_engine",          // Task name
    4096,                    // Stack size
    &engine_params,          // Task parameters
    configTIMER_TASK_PRIORITY + 1,  // Priority
    &synth_task_handle,      // Task handle
    1                        // CPU core (Core 1 for audio)
);
```

#### 5.2.3 MIDI Parsing Implementation
```cpp
void process_midi_message(uint8_t* midi_data, size_t length) {
    if (length >= 3) {
        uint8_t status = midi_data[0];
        uint8_t channel = status & 0x0F;
        uint8_t command = status & 0xF0;
        
        switch (command) {
            case 0x90: // Note On
                synth_note_on(channel, midi_data[1], midi_data[2]);
                break;
            case 0x80: // Note Off
                synth_note_off(channel, midi_data[1]);
                break;
            case 0xB0: // Control Change
                synth_control_change(channel, midi_data[1], midi_data[2]);
                break;
        }
    }
}
```

### 5.3 Memory Allocation Strategies

#### 5.3.1 PSRAM-Aware Allocation
```cpp
// Allocate audio buffers in PSRAM for non-real-time data
void* psram_audio_malloc(size_t size) {
    return heap_caps_malloc(size, MALLOC_CAP_SPIRAM);
}

// Allocate real-time buffers in internal RAM
void* dma_audio_malloc(size_t size) {
    return heap_caps_malloc(size, MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL);
}
```

#### 5.3.2 Voice Pool Management
```cpp
typedef struct {
    float frequency;
    float amplitude;
    float phase;
    uint32_t note_on_time;
    uint8_t midi_note;
    uint8_t midi_channel;
    bool active;
} synth_voice_t;

#define MAX_VOICES 24
synth_voice_t voice_pool[MAX_VOICES];
```

### 5.4 Performance Benchmarking Tools

#### 5.4.1 CPU Usage Monitoring
```cpp
void monitor_cpu_usage() {
    static uint32_t last_idle_time = 0;
    uint32_t current_idle = xTaskGetIdleRunTimeCounter();
    uint32_t cpu_usage = 100 - ((current_idle - last_idle_time) * 100 / 
                                 (configTICK_RATE_HZ * portNUM_PROCESSORS));
    last_idle_time = current_idle;
    
    system_registry.runtime_info.setCPUUsage(cpu_usage);
}
```

#### 5.4.2 Audio Latency Measurement
```cpp
// Measure round-trip audio latency
void measure_audio_latency() {
    uint32_t start_time = esp_timer_get_time();
    // Trigger test tone
    generate_test_pulse();
    // Measure response time via ADC
    uint32_t end_time = wait_for_audio_response();
    uint32_t latency_us = end_time - start_time;
    
    system_registry.runtime_info.setAudioLatency(latency_us);
}
```

### 5.5 Integration Testing Framework

#### 5.5.1 Automated Audio Quality Tests
```cpp
typedef struct {
    const char* test_name;
    float expected_thd;        // Total Harmonic Distortion
    float expected_snr;        // Signal-to-Noise Ratio
    uint32_t test_frequency;
} audio_test_case_t;

audio_test_case_t audio_tests[] = {
    {"440Hz Sine Wave", 0.1f, 80.0f, 440},
    {"1kHz Square Wave", 5.0f, 70.0f, 1000},
    {"Polyphonic Chord", 2.0f, 75.0f, 0},
};
```

### 5.6 Licensing Considerations

#### 5.6.1 Compatible Open Source Licenses
- **MIT License:** ESP32Synth, TinySoundFont - Full commercial use permitted
- **Apache 2.0:** Some audio libraries - Commercial use with attribution
- **BSD 3-Clause:** Various synthesis libraries - Commercial use permitted

#### 5.6.2 Restricted Licenses
- **marcel-licence projects:** Non-commercial use only
- **GPL libraries:** Require open-source distribution of derivative works

**Recommendation:** Prioritize MIT and Apache 2.0 licensed components for maximum deployment flexibility.

---

## Conclusion

This technical design document provides a comprehensive roadmap for extending KANTAN Play Core with ESP32-based software MIDI synthesizers. The hybrid multi-engine approach balances performance, flexibility, and maintainability while leveraging the existing system_registry architecture for seamless integration.

The proposed 6-milestone development plan provides clear deliverables and acceptance criteria, with an estimated timeline of 33-46 days for full implementation. The emphasis on open-source components with permissive licenses ensures maximum commercial viability while maintaining the project's open-source heritage.

Key success factors include:
1. Proper memory management leveraging ESP32-S3's PSRAM
2. Real-time audio processing optimization
3. Seamless integration with existing system architecture
4. Comprehensive testing and validation throughout development

The resulting system will provide users with professional-quality software synthesis options while maintaining the real-time performance and user experience expected from the KANTAN Play platform.