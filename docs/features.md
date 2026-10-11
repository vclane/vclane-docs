# System Key Features

## AI-Powered Traffic Intelligence

- Real-time object detection (YOLO)
- Multi-object tracking with persistent IDs (ByteTrack, opt-in)
- Vehicle counting: live occupancy + cumulative line crossings
- Per-direction and per-class traffic breakdowns
- Automated traffic analysis
- Congestion detection
- Traffic pattern monitoring

## Edge Computing Architecture

- Local AI processing
- Reduced latency
- Reduced cloud dependency
- Lower bandwidth usage

## Real-Time Monitoring

- Live camera streaming (annotated with boxes, track IDs, counting lines)
- Device telemetry (live/total counts, density, signal state)
- Traffic analytics
- Intersection status

## Distributed Traffic Management

- Independent intersection operation
- Centralized monitoring
- Multi-device coordination
- Per-group scheduling mode: fixed turns loop or dynamic on-demand queue (all-RED idle, GREEN on vehicle detection)

## Secure IoT Communication

- MQTT over TLS
- Device authentication
- Encrypted communication
- Topic authorization

## Fault Tolerant Operation

- Local fallback logic
- Offline operation capability
- Device health monitoring
- Automatic recovery support