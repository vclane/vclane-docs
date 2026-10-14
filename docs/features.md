# System Key Features

## AI-Powered Traffic Intelligence

- Real-time object detection (YOLO)
- Multi-object tracking with persistent IDs (ByteTrack, opt-in)
- Per-vehicle detection events with full annotated frames (via REST)
- Automated traffic analysis
- Congestion detection
- Traffic pattern monitoring

## Edge Computing Architecture

- Local AI processing
- Reduced latency
- Reduced cloud dependency
- Lower bandwidth usage

## Real-Time Monitoring

- Live camera streaming (annotated with boxes + track IDs)
- Device telemetry (signal state, emergency flag) + per-vehicle detections history
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