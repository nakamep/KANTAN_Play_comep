# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

KANTAN Play Core is an open-source music gadget firmware for M5Stack Core2/CoreS3 devices that works with dedicated KANTAN Play base hardware. The project enables chord playing with simple finger gestures and requires the specialized hardware to function.

## Architecture

The codebase follows a task-based architecture with separate modules for different hardware interfaces:

### Core Components
- **main/main.cpp**: Application entry point that initializes all task modules
- **system_registry.cpp/.hpp**: Central system state and configuration management
- **common_define.cpp/.hpp**: Shared definitions and constants

### Task Modules (FreeRTOS tasks on ESP32, SDL threads on native)
- **task_kantanplay**: Core music playing logic using KANTAN Music API
- **task_midi**: MIDI input/output handling (BLE and UART transports)
- **task_i2s**: Audio I/O management
- **task_i2c**: Internal device communication (BMI270, ES8388, SI5351, etc.)
- **task_spi**: SPI communication
- **task_wifi**: WiFi connectivity and HTTP client
- **task_port_a/b**: Physical port management
- **task_operator**: User input handling
- **task_commander**: Command processing system

### Hardware Interfaces
- **in_i2c/**: Internal I2C devices (accelerometer, audio codec, clock generator)
- **ex_i2c/**: External I2C devices (M5 button units, ExtIO2)
- **midi/**: MIDI transport layers (BLE, UART)

### Third-Party Libraries
- **kantan-music/**: Proprietary music engine (precompiled libraries for ESP32, x86, M1 Mac)
- Uses M5Unified for M5Stack hardware abstraction
- ArduinoJson for JSON processing

## Build System

Uses PlatformIO with multiple build environments:

### Development Commands
```bash
# Build for ESP32 Core2 (debug)
pio run -e esp32_arduino

# Build for ESP32-S3 CoreS3 (debug)  
pio run -e esp32s3_arduino

# Build release version for Core2
pio run -e release

# Build release version for CoreS3 (default)
pio run -e release_s3

# Upload to device
pio run -e release_s3 -t upload

# Monitor serial output
pio device monitor

# Native development (x86)
pio run -e native_x86

# Native development (M1 Mac)
pio run -e native_m1mac
```

### Project Structure
- Source code in `main/` directory (configured via `src_dir = main`)
- Components/libraries in `components/` directory
- Binary output goes to `ota_bin/` for OTA updates
- Post-build script `generate_user_custom.py` creates custom firmware files

## Key Development Notes

- The firmware will not run without KANTAN Play base hardware connected
- Native builds use SDL2 for cross-platform development and testing
- KANTAN Music API provides the core music functionality via precompiled libraries
- System uses FreeRTOS tasks on ESP32 with specific CPU core assignments
- Memory management is critical - debug builds include detailed heap monitoring
- WiFi connectivity supports OTA updates and web-based configuration

## Hardware Dependencies

- M5Stack Core2 or CoreS3 required
- KANTAN Play base hardware (switches, MIDI, amplifier, battery)
- Optional: M5 expansion units (buttons, ExtIO2) via external I2C