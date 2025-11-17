# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a specialized version of SRS (Simple Realtime Server) that focuses on GB28181 support. GB28181 is China's national standard for IP-based video surveillance systems. The implementation provides a complete solution for receiving GB28181 streams from surveillance devices and making them available through standard protocols like RTMP, HTTP-FLV, and WebRTC.

## Key Components

### Core GB28181 Implementation Files
- Main GB28181 Module: `trunk/src/app/srs_app_gb28181.hpp` and `srs_app_gb28181.cpp`
- SIP Stack: `trunk/src/app/srs_app_gb28181_sip.hpp` and `srs_app_gb28181_sip.cpp` for handling SIP signaling
- Jitter Buffer: `trunk/src/app/srs_app_gb28181_jitter.hpp` and `srs_app_gb28181_jitter.cpp`
- Protocol Stack: `trunk/src/app/srs_app_gb28181_stack.hpp` and `srs_app_gb28181_stack.cpp`

### Major Classes
1. **SrsGb28181Manger**: The main manager class that handles RTMP multiplexer management, RTP listener allocation, session management, and SIP service integration
2. **SrsGb28181RtmpMuxer**: Responsible for processing raw H.264/AAC streams, publishing to RTMP server, PS demuxing, and RTP packet handling
3. **SIP Implementation**: The system implements SIP signaling for device registration, invitation, and control commands

## Build System

### Building the Project
```bash
cd trunk
./configure --gb28181=on
make
```

The configure script supports various options including:
- `--gb28181=on/off` to enable/disable GB28181 support
- Other standard SRS options for enabling/disabling features

### Running the Server
```bash
./objs/srs -c conf/push.gb28181.conf
```

Configuration files for GB28181 are located in `trunk/conf/`:
- `push.gb28181.conf`: Main configuration with UDP transport
- `push.gb28181.tcp.conf`: Configuration with TCP transport

## Testing

### Unit Tests
Unit tests can be built and run with:
```bash
./configure --gb28181=on --utest=on
make utest
./objs/srs_utest
```

### Testing with Real Devices
The system can be tested with real GB28181 surveillance devices that support the standard. Configuration parameters like SIP server ID, realm, and ports need to match between the server and device configurations.

## API and Control Interface

The system provides a comprehensive HTTP API for:
- Session management
- Channel creation/deletion
- Device control (PTZ commands)
- Stream querying
- SIP session management

## Architecture Integration

The GB28181 module is integrated into SRS as a stream caster component:
- Works alongside other input methods (RTMP, SRT, etc.)
- Uses SRS's core infrastructure (RTMP server, HTTP API, etc.)
- Follows SRS's coroutine-based architecture for high performance

## Key Features Implemented
1. Full GB28181 Protocol Stack: Complete implementation of China's national standard for IP-based video surveillance
2. SIP Signaling: Device registration, keepalive, and control command handling
3. RTP Media Transport: Support for both TCP and UDP transport modes
4. PS Stream Processing: Demuxing of Program Stream containers to extract elementary streams
5. RTMP Integration: Seamless conversion from GB28181 streams to RTMP for redistribution
6. WebRTC Support: Integration with SRS's WebRTC capabilities
7. HTTP API: Comprehensive RESTful API for system control and monitoring