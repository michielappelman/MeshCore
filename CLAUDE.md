# MeshCore - AI Assistant Guide

This document provides context for AI assistants working with the MeshCore codebase.

## Project Overview

**MeshCore** is a lightweight, portable C++ library that enables multi-hop packet routing for embedded projects using LoRa and other packet radios. It creates resilient, decentralized communication networks that work without internet connectivity.

### Key Characteristics
- **Language**: C++ (embedded focus)
- **Platform**: PlatformIO-based embedded systems
- **Supported Hardware**: ESP32, NRF52, RP2040, STM32
- **Primary Use Cases**: Off-grid communication, emergency response, IoT sensor networks, outdoor activities
- **License**: MIT

## Architecture & Core Components

### Directory Structure

```
MeshCore/
├── src/                      # Core library source code
│   ├── Mesh.cpp/.h          # Main mesh networking logic
│   ├── Dispatcher.cpp/.h    # Message routing and handling
│   ├── Packet.cpp/.h        # Packet structure and handling
│   ├── Identity.cpp/.h      # Node identity and cryptography
│   ├── Utils.cpp/.h         # Utility functions
│   ├── MeshCore.h           # Main library header
│   └── helpers/             # Platform-specific helpers
├── examples/                 # Example applications
│   ├── companion_radio/     # BLE/USB/WiFi companion device
│   ├── simple_repeater/     # Network range extender
│   ├── simple_room_server/  # BBS-style shared posts
│   ├── simple_secure_chat/  # Secure terminal chat
│   └── simple_sensor/       # Sensor network node
├── lib/                     # Third-party libraries
├── variants/                # Device-specific configurations
├── boards/                  # Board definitions
├── arch/                    # Architecture-specific code
├── docs/                    # Documentation
└── platformio.ini           # PlatformIO configuration
```

### Core Components

1. **Mesh.cpp/h**: The heart of the mesh network
   - Multi-hop packet routing
   - Network topology management
   - Neighbor discovery
   - LoRa radio control

2. **Dispatcher.cpp/h**: Message handling and routing
   - Packet forwarding logic
   - Message type routing
   - Event callbacks

3. **Packet.cpp/h**: Packet structure and serialization
   - Packet format definitions
   - Encryption/decryption
   - Packet validation

4. **Identity.cpp/h**: Node identification and security
   - Ed25519 cryptography
   - Public/private key management
   - Node addressing

## Development Guidelines

### Code Style & Conventions

**From the Contributing section:**

1. **Think Embedded**: Keep code concise without unnecessary layers
   - Avoid high-level language patterns
   - Minimize abstraction overhead
   - Focus on efficiency and memory usage

2. **Memory Management**:
   - **NO dynamic memory allocation** except during setup/begin functions
   - Use static allocation wherever possible
   - Carefully manage stack usage

3. **Code Formatting**:
   - Use the existing brace and indenting style (`.clang-format` available)
   - **DO NOT** retroactively reformat existing code
   - Maintain consistency with the existing codebase

4. **Simplicity First**:
   - Keep it simple
   - Don't over-engineer solutions
   - Write clear, readable code

### Build Configuration

**Build System**: PlatformIO

**Base Dependencies** (from `platformio.ini`):
- RadioLib @ ^7.3.0 (LoRa radio control)
- Crypto @ ^0.4.0 (Cryptography)
- RTClib @ ^2.1.3 (Real-time clock)
- Melopero RV3028 @ ^1.1.0 (RTC module)
- CayenneLPP @ 1.6.1 (LPP encoding)

**Platform-Specific Bases**:
- `esp32_base`: ESP32 platforms (most common)
- `nrf52_base`: Nordic NRF52 platforms
- `rp2040_base`: Raspberry Pi Pico
- `stm32_base`: STM32 platforms

**Key Build Flags**:
```
-D LORA_FREQ=869.525          # Default LoRa frequency
-D LORA_BW=250                # Bandwidth
-D LORA_SF=11                 # Spreading factor
-D ENABLE_PRIVATE_KEY_IMPORT=1
-D ENABLE_PRIVATE_KEY_EXPORT=1
```

### Building Projects

```bash
# Using PlatformIO CLI
pio run                       # Build default environment
pio run -e <environment>      # Build specific environment
pio run -t upload             # Build and upload
pio device monitor            # Open serial monitor

# Using the build script
./build.sh
```

## Working with MeshCore

### When Making Changes

1. **Read Before Modifying**: Always read existing code before suggesting changes
2. **Maintain Node Roles**: Understand the difference between Companion nodes (non-repeating) and repeater nodes
3. **Consider Multi-hop**: Changes may affect packet routing across multiple hops
4. **Test on Hardware**: Embedded code should be tested on actual hardware when possible
5. **Check LoRa Limits**: Be aware of LoRa packet size constraints and duty cycle limitations

### Common Tasks

#### Adding a New Feature
1. Start with the appropriate example application
2. Check if similar functionality exists in helpers/
3. Maintain the no-dynamic-allocation rule
4. Test on target hardware platform

#### Fixing Bugs
1. Check recent commits for context
2. Test fix across multiple platforms if possible
3. Verify no regression in routing behavior

#### Modifying Protocol
1. **Coordinate with maintainers** - protocol changes affect all nodes
2. Review V2 protocol roadmap before proposing changes
3. Ensure backward compatibility when possible

### Important Constraints

1. **Memory**: Embedded devices have limited RAM/Flash
2. **LoRa Airtime**: EU regulations limit transmission duty cycle (1%)
3. **Packet Size**: LoRa packets are typically 255 bytes max
4. **Power**: Many nodes are battery/solar powered
5. **Real-time**: Network timing is critical for reliable routing

## Testing & Deployment

### Flashing Firmware
- Use the web flasher: https://flasher.meshcore.co.uk
- Or build and flash via PlatformIO

### Testing Tools
- Serial Monitor (Visual Studio Code)
- Serial USB Terminal (Android)
- Web app: https://app.meshcore.nz
- Mobile apps (iOS/Android)
- Config tool: https://config.meshcore.dev

## Contributing Workflow

1. **Base Branch**: Submit PRs against `dev` branch (NOT main)
2. **Discussion**: For impactful changes, open an Issue first
3. **Consensus**: Discuss approach before implementation
4. **Review**: Maintainer will review PRs

## Roadmap Awareness

Be aware of planned features (from README):
- Repeater/Bridge: Standardised Transport Codes for zoning/filtering
- Core: round-trip manual path support
- Multiple sub-meshes support
- LZW message compression
- Dynamic Coding Rate for weak vs strong hops
- V2 protocol specification

Avoid implementing features that conflict with planned architecture changes.

## Resources

- **Documentation**: `/docs` folder, especially `faq.md`
- **Intro Video**: https://www.youtube.com/watch?v=t1qne8uJBAc
- **Discord**: https://discord.gg/BMwCtwHj5V
- **Issues**: https://github.com/ripplebiz/MeshCore/issues

## Key Concepts

### Node Types
- **Companion**: Non-repeating nodes (end-user devices)
- **Repeater**: Relay nodes that extend network range
- **Room Server**: BBS-style message sharing nodes
- **Sensor**: Environmental sensor nodes with ACLs

### Network Behavior
- **Multi-hop Routing**: Messages relay through intermediate nodes
- **Self-healing**: Network adapts to node availability
- **Decentralized**: No central server required
- **Zero-hop Discovery**: Enhanced neighbor detection

### Security
- **Ed25519**: Public key cryptography
- **End-to-End Encryption**: Messages encrypted between endpoints
- **Identity**: Each node has unique cryptographic identity

## Tips for AI Assistants

1. **Embedded Context**: Always think about memory constraints
2. **No Malloc**: Avoid suggesting dynamic memory allocation
3. **Platform Aware**: Consider which platforms changes affect
4. **Protocol Impact**: Network protocol changes affect all nodes
5. **Read Examples**: Check example code for established patterns
6. **Conservative**: Prefer minimal, focused changes
7. **Hardware Reality**: Understand physical LoRa limitations

## Questions to Ask

When unsure about an implementation:
- Which platforms should this support?
- What's the memory impact?
- Does this affect the protocol?
- Should this be configurable via build flags?
- Is there existing code that does something similar?
- What's the impact on battery life?
- How does this affect routing behavior?
