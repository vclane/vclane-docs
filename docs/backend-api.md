# Backend API

The backend acts as the centralized management and coordination layer between users, edge devices, and cloud services.

## Responsibilities

- User authentication
- Device registration
- Device management
- Traffic monitoring
- Telemetry processing
- Signal override handling
- Intersection grouping
- Traffic synchronization
- MQTT command publishing
- Firebase synchronization

## Technology Stack

| Component               | Technology                          |
| ----------------------- | ----------------------------------- |
| Programming Language    | Python                              |
| Backend Framework       | FastAPI                             |
| API Architecture        | REST API                            |
| Real-Time Communication | WebSocket                           |
| MQTT Integration        | MQTT Client                         |
| Authentication          | Firebase Admin SDK                  |
| Database                | PostgreSQL / Firestore              |
| Cache                   | Redis                               |
| Background Processing   | Celery / Background Tasks           |
| API Documentation       | OpenAPI / Swagger                   |
| Containerization        | Docker                              |
| Deployment              | Linux Server / Cloud Infrastructure |

## Key Features

**Device Management** — Register edge devices, manage device configurations, track device status, monitor device health, assign devices to intersections.

**Traffic Monitoring** — Receive real-time telemetry, monitor traffic conditions, store traffic information, generate traffic statistics, detect traffic events.

**Traffic Control** — Send traffic signal commands, trigger manual overrides, manage intersection schedules, coordinate multiple intersections, process command responses.

**Real-Time Synchronization** — MQTT command publishing, device heartbeat processing, WebSocket client updates, event notifications.