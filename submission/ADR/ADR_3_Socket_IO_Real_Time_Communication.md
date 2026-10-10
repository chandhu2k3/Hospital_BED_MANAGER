# ADR 3: Socket.IO for Real-Time Communication

## Status
**Accepted**

## Date
2025-10-20

## Context

In a hospital bed management system, bed availability changes frequently — admissions, discharges, transfers, and cleaning can all happen within minutes. Multiple users across different roles view dashboard data simultaneously. Without real-time updates, users would see stale data, potentially leading to:

1. **Double-allocation**: Two ICU Managers approving the same bed for different patients.
2. **Outdated occupancy data**: Admin seeing 70% occupancy when it's actually 95% (critical threshold).
3. **Missed alerts**: Ward staff not seeing that a bed request has been approved until they manually refresh.

The system requires:
- **Bidirectional communication**: Server pushes updates to clients AND clients can emit events (e.g., joining a ward room).
- **Room-based routing**: Updates for a specific ward should only go to users viewing that ward.
- **Fallback transport**: Hospital networks may have restrictive firewalls that block WebSocket upgrades.
- **Connection management**: Track connected clients, handle disconnections gracefully.

**Alternatives considered**:
- **Server-Sent Events (SSE)**: Unidirectional (server → client only). Pros: simpler, native browser support. Cons: no client-to-server events (can't join rooms), limited to 6 concurrent connections per browser in HTTP/1.1.
- **Long polling**: Simulated real-time via repeated HTTP requests. Pros: works everywhere. Cons: high latency, wasted bandwidth, no true push.
- **Native WebSocket API (ws library)**: Raw WebSocket without abstraction. Pros: lightweight, no dependencies. Cons: no built-in rooms, no auto-reconnection, no fallback transport, no event namespacing.
- **Firebase Realtime Database**: Built-in real-time sync. Pros: zero server-side code. Cons: vendor lock-in, limited query flexibility, cost concerns with frequent updates.

## Decision

We will use **Socket.IO** (v4.7.2 server, v4.6.1 client) for real-time bidirectional communication between the backend and all connected frontend clients.

### Architecture
- Socket.IO server wraps the same HTTP server as Express (shared port 5000)
- The `io` object is attached to every Express request via middleware (`req.io = io`) so route handlers can emit events
- Clients connect on page load with WebSocket as primary transport, falling back to long-polling
- Room-based routing: clients join rooms like `ward-Emergency` or `bed-<id>` for targeted events

### Implemented Events
| Event | Emitter | Purpose |
|---|---|---|
| `bed-updated` | Server → All | Bed status changed |
| `new-bed-request` | Server → All | New bed request created |
| `bed-request-approved` | Server → All | Request approved by ICU Manager |
| `bed-request-denied` | Server → All | Request denied |
| `patient-admitted` | Server → All | New patient admission |
| `patient-discharged` | Server → All | Patient discharged |
| `ward-transfer-updated` | Server → All | Transfer status changed |
| `patient-transferred` | Server → All | Patient physically moved |
| `settings-updated` | Server → All | System settings modified |
| `new-cleaning-job` | Server → All | Cleaning job created |
| `join-ward` | Client → Server | Subscribe to ward events |
| `leave-ward` | Client → Server | Unsubscribe from ward events |
| `subscribe-bed` | Client → Server | Subscribe to specific bed |

### Connection Management
- Connected clients tracked in a `Map` (`connectedClients`)
- `client-count` event emitted on connect/disconnect
- Auto-reconnection configured (up to 10 attempts, 1s delay)

## Consequences

### Positive
- **Instant updates**: Bed status changes propagate to all dashboards within milliseconds. No polling latency.
- **Room-based targeting**: Ward staff only receive events for their ward, reducing unnecessary re-renders.
- **Automatic fallback**: If WebSocket upgrade fails (firewall), Socket.IO transparently falls back to long-polling.
- **Simple integration**: `req.io.emit()` in any Express route handler provides seamless REST + real-time hybrid.
- **Built-in reconnection**: Socket.IO client automatically reconnects after network interruptions — critical for hospital WiFi.

### Negative
- **Memory overhead**: Each connected client maintains a persistent WebSocket connection. With hundreds of concurrent users, this could strain server memory. Mitigated by the small expected user base (hospital staff).
- **No message persistence**: If a client is disconnected when an event occurs, they miss it. Mitigated by full data re-fetch on reconnection (`fetchData()` called on socket connect).
- **Scalability limitation**: Socket.IO connections are stateful and pinned to a server instance. For multi-instance deployment, a Redis adapter (`@socket.io/redis-adapter`) would be needed. Not implemented yet.
- **Debugging complexity**: Real-time event flows are harder to trace than sequential REST calls. Mitigated by console logging of all socket events with emoji prefixes (👤, 🛏️, 👥).
