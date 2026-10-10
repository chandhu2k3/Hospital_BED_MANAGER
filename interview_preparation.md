# 🏥 Hospital Bed Manager — Interview Preparation Guide

> **Course**: SSD (Software Systems Design) — Final Project, Sem-1, MTech PDM, IIIT-H  
> **Team**: Team 21

---

## 1. Project Overview (Elevator Pitch — 30 seconds)

> "We built a **real-time hospital bed management system** that tracks 60 beds across 4 wards — Emergency, ICU, General Ward, and Cardiology. It supports **4 distinct user roles** (ER Staff, ICU/Bed Manager, Ward Staff, and Admin) each with tailored dashboards. The system provides **real-time updates via WebSockets**, an **AI-powered analytics chatbot** using Google Gemini, **predictive occupancy forecasting** with exponential smoothing, and a **smart bed recommendation engine** that matches patients to beds based on ward preference and equipment needs."

---

## 2. Problem Statement

### The Real-World Problem
Hospitals face critical challenges in bed management:
- **Manual tracking** leads to delays in patient admission, especially in emergencies
- **No real-time visibility** of bed availability across wards
- **Inefficient allocation** — beds may sit empty in one ward while another is over capacity
- **No forecasting** — hospitals can't proactively plan for capacity spikes
- **Communication gaps** between ER staff requesting beds and managers approving them

### Our Solution
A full-stack web application that digitizes the entire bed lifecycle:
```
Available → Reserved → Occupied → Cleaning → Available
```
With role-based access, real-time updates, smart allocation, and predictive analytics.

---

## 3. Tech Stack — Deep Dive

### 3.1 Frontend

| Technology | Version | **Why We Chose It** |
|---|---|---|
| **React** | v18.2.0 | Component-based architecture ideal for complex role-based dashboards. React's virtual DOM ensures performant re-renders when bed states change frequently via WebSocket updates. |
| **Material-UI (MUI)** | v7.3.5 | Pre-built, accessible, production-ready UI components. Medical software demands consistent, professional UIs. MUI's theming system enabled dark/light mode toggle. |
| **Recharts** | v2.7.2 | Declarative charting library built on React + D3. Used for occupancy trend charts, ward-wise breakdowns, and forecasting visualizations on the Admin dashboard. |
| **Socket.IO Client** | v4.6.1 | Enables real-time bidirectional communication. When a bed status changes, all connected dashboards update instantly — critical for hospital operations. |
| **Axios** | v1.4.0 | Promise-based HTTP client with interceptor support. We used interceptors for automatic JWT token injection and 401/403 session-expiry handling (auto-logout). |
| **React Router DOM** | v7.9.5 | Client-side routing (though we primarily use conditional rendering for role-based dashboard switching). |
| **QRCode.react** | v4.2.0 | Generates QR codes for mobile access — staff can scan to access the app on their phones within the hospital WiFi network. |

### 3.2 Backend

| Technology | Version | **Why We Chose It** |
|---|---|---|
| **Node.js + Express** | v4.18.2 | Non-blocking I/O model ideal for a real-time system with multiple concurrent WebSocket connections. Express gives a lightweight, unopinionated routing framework. |
| **MongoDB** | v6.3.0 | NoSQL document database chosen because hospital data (bed states, patient records, ward stats) is inherently hierarchical/nested. Schema flexibility allowed rapid iteration. |
| **Mongoose** | v7.6.3 | ODM providing schema validation, middleware (pre-save hooks for auto-generating request IDs), indexing, and population of references. |
| **Socket.IO** | v4.7.2 | Full-duplex WebSocket server with fallback to long-polling. Supports room-based event routing (`ward-Emergency`, `bed-<id>`) for targeted updates. |
| **JWT (jsonwebtoken)** | v9.0.2 | Stateless authentication — each token carries user ID, role, and ward info. Tokens expire after 24h. No session storage needed on the server. |
| **bcrypt** | v6.0.0 | Password hashing with salt rounds of 10. Industry standard for secure credential storage. |
| **Google Generative AI (Gemini)** | v0.24.1 | Powers the AI analytics chatbot. Uses **Gemini 2.5 Flash** with **function calling** to query real hospital data and generate natural-language insights. |
| **QRCode** | v1.5.3 | Server-side QR code generation for the site URL, enabling mobile access. |
| **dotenv** | v16.3.1 | Environment variable management for secrets (DB password, JWT secret, Gemini API key). |

### 3.3 Database — MongoDB Atlas
- **Cloud-hosted** on MongoDB Atlas for reliability and zero-ops management
- **Server API v1** with strict mode and deprecation error reporting enabled
- **Compound indexes** on high-query fields (`ward + status`, `status + createdAt`)

---

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     FRONTEND (React SPA)                     │
│                                                              │
│  ┌───────────┐ ┌──────────────┐ ┌───────────┐ ┌──────────┐ │
│  │ ER Staff  │ │ ICU Manager  │ │Ward Staff │ │  Admin   │ │
│  │ Dashboard │ │  Dashboard   │ │Dashboard  │ │Dashboard │ │
│  └─────┬─────┘ └──────┬───────┘ └─────┬─────┘ └────┬─────┘ │
│        │               │               │            │       │
│        └───────┬───────┴───────┬───────┘            │       │
│                │               │                    │       │
│         Axios (REST)    Socket.IO Client      Axios + Chat  │
└────────────────┼───────────────┼────────────────────┼───────┘
                 │               │                    │
           ┌─────▼───────────────▼────────────────────▼───────┐
           │              BACKEND (Express + Node.js)          │
           │                                                   │
           │  ┌──────────────────────────────────────────────┐ │
           │  │            Authentication Layer               │ │
           │  │  JWT Verify → Role-Based Authorization (RBAC) │ │
           │  └──────────────────────────────────────────────┘ │
           │                                                   │
           │  ┌─────────┐ ┌──────────┐ ┌──────────┐          │
           │  │  REST   │ │ Socket.IO│ │ Gemini   │          │
           │  │  Routes │ │  Server  │ │ AI Chat  │          │
           │  │(8 route │ │ (rooms,  │ │(function │          │
           │  │ files)  │ │ events)  │ │ calling) │          │
           │  └────┬────┘ └────┬─────┘ └────┬─────┘          │
           │       │           │             │                 │
           │  ┌────▼───────────▼─────────────▼───────────────┐ │
           │  │         Mongoose ODM (8 Models)               │ │
           │  │  Bed | Patient | BedRequest | WardTransfer    │ │
           │  │  Alert | OccupancyHistory | SystemSettings    │ │
           │  │  CleaningJob | User (inline in server.js)     │ │
           │  └──────────────────┬────────────────────────────┘ │
           └─────────────────────┼─────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │    MongoDB Atlas         │
                    │  (Cloud Database)        │
                    └─────────────────────────┘
```

### Key Architectural Decisions

1. **Monolithic but Modular**: Single Express server with separated route files for each domain (beds, patients, bedRequests, wardTransfers, chatbot, alerts, systemSettings, seedRoutes). Easy to understand and deploy, while still maintainable.

2. **Real-time + REST Hybrid**: REST APIs for CRUD operations + Socket.IO for pushing live updates. When a bed status changes via REST, the route handler also emits a WebSocket event so all connected clients update without polling.

3. **Role-Based Dashboard Routing**: Instead of a traditional router, the frontend uses `currentUser.role` to conditionally render the appropriate dashboard component. Each dashboard is a self-contained component with its own data fetching and state management.

---

## 5. Database Design — 8 Collections

### 5.1 Bed Schema
```
Fields: bedNumber (unique), ward (indexed), status (enum: available|occupied|
        cleaning|reserved|maintenance), equipmentType (enum: Standard|Ventilator|
        ICU Monitor|Cardiac Monitor|VAD), patientId (ref→Patient), 
        reservedForRequestId (ref→BedRequest), location {floor, section, roomNumber},
        lastCleaned, notes, maintenanceSchedule
Indexes: Compound index on (ward, status) for fast ward-filtered queries
```

### 5.2 Patient Schema
```
Fields: patientId (unique, auto-generated PAT-XXXXXX), name, age, gender,
        contactNumber, emergencyContact {name, relation, phone}, department,
        reasonForAdmission, estimatedStay (hours), admissionDate, expectedDischarge,
        actualDischarge, bedId (ref→Bed), vitals[], medications[], status 
        (enum: admitted|critical|stable|recovering|discharged), transferHistory[]
```

### 5.3 BedRequest Schema
```
Fields: requestId (auto-generated REQ-XXXXXX via pre-save hook), createdBy {userId, 
        username, name}, patientDetails {name, age, gender, contactNumber, 
        reasonForAdmission, requiredEquipment, estimatedStay}, preferredWard, eta,
        status (enum: pending|approved|denied), assignedBed {bedId, bedNumber, ward},
        reviewedBy, denialReason, priority (1-5), cancelledAt, cancelledBy, 
        isDeleted (soft delete), deletedAt, deletedBy
Indexes: (status, createdAt), (createdBy.userId), (assignedBed.bedId)
Pre-save Hook: Auto-generates sequential requestId
```

### 5.4 WardTransfer Schema
```
Fields: bedId (ref→Bed), newBedId (ref→Bed), patientId (ref→Patient), 
        currentWard, targetWard, reason, status (enum: pending|approved|denied|
        completed), requestedBy (ref→User), reviewedBy (ref→User), denyReason,
        completedAt, notes
Business Rule: Cannot transfer TO Emergency ward
```

### 5.5 OccupancyHistory Schema (Time-Series Data)
```
Fields: timestamp (indexed DESC), totalBeds, occupied, available, cleaning, 
        reserved, occupancyRate, wardStats[] (per-ward breakdown), peakHour (bool),
        hourlyData[] (24-entry array with per-hour per-ward stats)
Purpose: 60-day historical tracking with hourly granularity, powers forecasting
```

### 5.6 SystemSettings Schema (Singleton Pattern)
```
Fields: singleton ('settings', unique), thresholds {warningThreshold: 80, 
        criticalThreshold: 90}, reporting {defaultPeriod, autoRefreshInterval},
        exportOptions {includePHI, defaultFormat}, wardCapacity {Emergency: 15, 
        ICU: 15, General Ward: 15, Cardiology: 15}, lastUpdatedBy
Static Method: getSettings() — finds or creates the singleton document
```

### 5.7 Alert Schema
```
Fields: type (enum: critical|warning|info|success|emergency), message, ward,
        bedId (ref→Bed), acknowledged (bool), acknowledgedBy, acknowledgedAt,
        priority (1-5)
```

### 5.8 CleaningJob Schema
```
Fields: bedId (ref→Bed), bedNumber, ward, floor, section, roomNumber,
        status (enum: pending|active|completed), assignedToStaffId, assignedToName,
        startedAt, completedAt
Purpose: Tracks bed cleaning workflow after patient discharge
```

---

## 6. Role-Based Access Control (RBAC)

### Authentication Flow
```
Client → POST /api/auth/login {username, password}
Server → bcrypt.compare(password, hashedPassword)
       → jwt.sign({id, username, role, ward, name}, secret, {expiresIn: '24h'})
       → Response: {token, user}
Client → Stores token in localStorage
       → Axios interceptor attaches `Authorization: Bearer <token>` to every request
       → authenticateToken middleware verifies on every protected route
```

### Authorization Middleware (Two Layers)
```javascript
// Layer 1: Token verification (applied globally to /api/* routes)
const authenticateToken = (req, res, next) => {
  jwt.verify(token, JWT_SECRET, (err, user) => {
    req.user = user; // {id, username, role, ward, name}
    next();
  });
};

// Layer 2: Role-based authorization (applied per-route)
const authorize = (...roles) => (req, res, next) => {
  if (!roles.includes(req.user.role)) return res.status(403);
  next();
};
```

### Permissions Matrix

| Action | ER Staff | ICU Manager | Ward Staff | Admin |
|---|:---:|:---:|:---:|:---:|
| Create bed request | ✅ | ✅ | ❌ | ❌ |
| Approve/deny request | ❌ | ✅ | ❌ | ✅ |
| Fulfill request (admit patient) | ❌ | ✅ | ✅ | ✅ |
| Update bed status | ❌ | ✅ | ✅ | ✅ |
| Initiate ward transfer | ❌ | ✅ | ✅ | ❌ |
| Approve ward transfer | ❌ | ✅ | ❌ | ❌ |
| View analytics | ❌ | ❌ | ❌ | ✅ |
| Use AI chatbot | ❌ | ❌ | ❌ | ✅ |
| Manage system settings | ❌ | ❌ | ❌ | ✅ |
| Manage users (CRUD) | ❌ | ❌ | ❌ | ✅ |

---

## 7. Key Feature Implementations

### 7.1 Real-Time Updates (Socket.IO)

**Architecture**: When any REST API modifies data, the route handler also emits WebSocket events:

```javascript
// Example: When a bed status is updated via PUT /api/beds/:id
const updatedBed = await Bed.findByIdAndUpdate(id, updateFields, {new: true});
req.io.emit('bed-updated', updatedBed);                    // Broadcast to ALL
req.io.to(`bed-${id}`).emit('bed-status-changed', updatedBed); // To subscribers
req.io.to(`ward-${bed.ward}`).emit('ward-bed-updated', updatedBed); // To ward
```

**Room-based routing**: Clients join rooms like `ward-Emergency` or `bed-<id>` for targeted updates.

**Events emitted**: `bed-updated`, `new-bed-request`, `bed-request-approved`, `patient-admitted`, `patient-discharged`, `ward-transfer-updated`, `settings-updated`, `beds-synced`, `new-cleaning-job`

### 7.2 Smart Bed Recommendation Engine

The `POST /api/beds/recommend` endpoint implements a **multi-tier matching algorithm**:

```
Tier 1 (Perfect Match): Ward + Equipment match
Tier 2 (Ward Match):    Ward match, different equipment
Tier 3 (Equipment Match): Equipment match, any ward
Tier 4 (Any Available):  Fallback to any available bed
```

**Smart allocation logic**:
- If ≥3 perfect matches → Return 2 perfect + 1 different-equipment (for variety)
- If 2 perfect matches → Return 2 perfect + 1 different-equipment
- If 1 perfect match → Return 1 perfect + 2 different-equipment
- If 0 perfect matches → Return up to 3 with any equipment
- If 0 available → Suggest beds under cleaning (coming available soon)

### 7.3 AI-Powered Analytics Chatbot (Google Gemini)

**How it works**:
1. Admin sends a natural language query (e.g., "What was ICU occupancy last week?")
2. Backend passes the query to **Gemini 2.5 Flash** with:
   - A detailed system prompt with hospital context, valid ward names, and date references
   - **6 function declarations** that Gemini can call to fetch real data
3. Gemini decides which function(s) to call and with what parameters
4. Backend executes the function(s) against MongoDB and returns results to Gemini
5. Gemini synthesizes a natural-language response with the data
6. Loop continues for up to 5 function call rounds

**Available Functions** (Gemini can call these):
| Function | Purpose |
|---|---|
| `getOccupancyData` | Get raw occupancy data for a date range |
| `calculateOccupancyStats` | Calculate avg/peak/lowest/median occupancy |
| `getBedRequestStats` | Get bed request approval/denial stats |
| `getPatientStats` | Get admission/discharge/avg-stay stats |
| `getCurrentBedStatus` | Get real-time bed counts by ward |
| `compareTimePeriods` | Compare metrics between two periods |

**Enhancements implemented**:
- **Response caching** (5-minute TTL) to reduce API calls
- **Parameter normalization** ("ER" → "Emergency", "intensive care" → "ICU")
- **Data validation** (date format, ward names, logical consistency)
- **Response validation** (checking percentages are 0-100, totals match)
- **Execution logging** (tracking function call times and success/failure)
- **Calculation transparency** (formula + sample size returned with every result)

### 7.4 Bed State Lifecycle with Validation

```
Available → Reserved → Occupied → Cleaning → Available
```

**Server-side enforcement**:
```javascript
const validTransitions = {
  available: ['occupied', 'reserved', 'cleaning'],
  occupied:  ['cleaning', 'available'],
  cleaning:  ['available'],
  reserved:  ['occupied', 'available']
};
// Invalid transitions return 400 with descriptive error message
```

**Automated side-effects**:
- `occupied → cleaning`: Creates a `CleaningJob` record, emits `new-cleaning-job` event
- `* → available`: Clears `patientId` from the bed
- Status change → Creates an `Alert` record for audit trail

### 7.5 Threshold-Based Alerting System

**Backend**: Generates alerts when `GET /api/beds/stats` calculates occupancy:
- ≥95% → Critical (Priority 5) — "Immediate action required"
- ≥90% → Critical (Priority 4)
- ≥80% → Warning (Priority 3)

**Frontend** (ICU Manager dashboard): 
- **Debounced** (1.5s delay) to prevent alert storms during rapid bed changes
- **Smart threshold tracking**: Only alerts when crossing INTO a new threshold level (not repeatedly at the same level)
- **Per-ward + overall** threshold monitoring
- **Dismissable** critical alert modals

### 7.6 Ward Transfer Workflow

1. Ward Staff or ICU Manager creates transfer request
2. Business rule: **Cannot transfer TO Emergency ward**
3. Checks for duplicate pending transfers
4. ICU Manager approves → System automatically:
   - Finds an available bed in the target ward (matching equipment type)
   - Releases old bed (status → `cleaning`)
   - Assigns patient to new bed (status → `occupied`)
   - Updates patient record
   - Transfer marked as `completed` immediately (no intermediate "approved" state)
   - Emits 5 different socket events for real-time UI updates

### 7.7 Occupancy History & Forecasting

- **60-day historical data** with hourly granularity (24 data points per day, per ward)
- **Today's live data** — upserts the current day's entry on each stats fetch
- **Predictive forecasting**: Exponential smoothing algorithm on the Admin dashboard
- **Ward-wise breakdown**: Each historical record contains per-ward stats
- **Seed data generation**: Realistic occupancy patterns with:
  - Weekly cycles, weekend dips, Monday spikes
  - Hourly patterns (morning peak, evening peak, night lull)
  - Ward-specific adjustments (ICU busier, General Ward quieter)
  - Random surge events

---

## 8. Design Patterns Used

| Pattern | Where | Details |
|---|---|---|
| **Singleton** | `SystemSettings` model | `getSettings()` static method ensures exactly one config document exists |
| **Middleware Chain** | Express auth pipeline | `authenticateToken → authorize(...roles) → route handler` |
| **Observer** | Socket.IO events | Pub/sub pattern — bed changes publish events, dashboards subscribe |
| **Repository** | Mongoose models | Data access abstracted behind model methods |
| **Strategy** | Bed recommendation | Different matching strategies (perfect/ward/equipment/any) based on input |
| **Soft Delete** | BedRequest model | `isDeleted` flag preserves audit trail while hiding from queries |
| **Debounce** | Frontend alert checking | 1.5s delay prevents rapid-fire threshold alerts |

---

## 9. Challenges Faced & How We Solved Them

### Challenge 1: Alert Storm Prevention
**Problem**: Every time bed stats refreshed, threshold alerts would fire repeatedly, flooding the ICU Manager with the same alert.  
**Solution**: Implemented a **state-tracking system** (`exceededThresholds` map) that records the current threshold level per ward. Alerts only fire when the threshold LEVEL changes (e.g., from "warning" → "critical"), not when it's already at that level. Combined with 1.5s debouncing.

### Challenge 2: Concurrent Bed Reservation Conflicts
**Problem**: Two managers could try to reserve the same bed simultaneously.  
**Solution**: The approve workflow first checks `bed.status === 'available'` before reserving. If two requests race, the second one gets a clear error: "Bed is not available (current status: reserved)".

### Challenge 3: Invalid State Transitions
**Problem**: Without enforcement, beds could go from "reserved" directly to "cleaning" (skipping patient admission).  
**Solution**: Implemented a **state machine** with a valid transitions map on the server. Invalid transitions return 400 errors with descriptive messages showing allowed states.

### Challenge 4: AI Chatbot Hallucination
**Problem**: Gemini would sometimes fabricate hospital data instead of querying the actual database.  
**Solution**: Used **function calling** (not plain prompting). Gemini MUST call our predefined functions to get data — it can't make up numbers. Added parameter normalization, response validation, and calculation transparency.

### Challenge 5: Mobile Access Within Hospital
**Problem**: Hospital staff needed access from phones on the hospital network.  
**Solution**: Server auto-detects local IP address and exposes URLs. CORS supports private network IP ranges (192.168.x.x, 10.x.x.x, 172.16-31.x.x). QR code generation for easy mobile onboarding.

---

## 10. What I Would Improve (Self-Awareness for Interviews)

1. **Testing**: Add unit tests (Jest) and integration tests. Currently no automated test suite.
2. **Rate limiting**: Add express-rate-limit to prevent API abuse.
3. **Database transactions**: Use MongoDB transactions for multi-document operations (e.g., ward transfers updating 3 documents).
4. **Microservices**: For production scale, split into separate services (auth, beds, patients, analytics).
5. **Caching layer**: Add Redis for bed status caching to reduce DB reads.
6. **Audit logging**: More comprehensive logging with a dedicated audit service.
7. **Pagination**: Currently most list endpoints return all records. Need cursor-based pagination.
8. **TypeScript**: Migrating to TypeScript for better type safety and developer experience.

---

## 11. Interview Questions & Answers

### Architecture & Design

**Q1: Why did you choose MongoDB over a relational database?**
> Hospital bed data is hierarchical — a bed has nested location info, patients have nested vitals arrays, occupancy history has nested wardStats with nested hourlyData. MongoDB's document model maps naturally to this. Also, the schema evolved rapidly during development — adding fields like `reservedForRequestId` or `CleaningJob` was trivial with MongoDB. That said, for a production system with strict ACID requirements (financial transactions), a relational DB might be better. But for operational tracking, MongoDB's flexibility was ideal.

**Q2: How does real-time communication work in your system?**
> We use Socket.IO for bidirectional WebSocket communication. The backend creates an HTTP server, wraps it with Socket.IO, and attaches the `io` object to every Express request via middleware (`req.io = io`). When any REST API endpoint modifies data (e.g., bed status update), the route handler emits the appropriate event. Clients subscribe to events on connection and update their local state. We also use room-based routing — clients join rooms like `ward-Emergency` so we can broadcast ward-specific events without flooding all clients.

**Q3: Explain your authentication and authorization design.**
> Two-layer approach. **Layer 1 (Authentication)**: JWT tokens signed with a secret on login, containing user ID, role, ward, and name. Tokens expire in 24 hours. A global middleware (`authenticateToken`) verifies tokens on all `/api/*` routes. **Layer 2 (Authorization)**: A higher-order middleware function `authorize(...roles)` checks if the decoded token's role is in the allowed list. Applied per-route — e.g., `authorize('icu_manager', 'admin')` on bed approval. The frontend also conditionally renders different dashboard components based on `currentUser.role`.

**Q4: What design patterns did you use and why?**
> **Singleton** for SystemSettings — only one config document should exist, enforced by a unique `singleton` field. **Observer/Pub-Sub** through Socket.IO — loose coupling between data changes and UI updates. **Middleware Chain** in Express for composable auth: token verification → role check → business logic. **Strategy Pattern** in bed recommendation — different matching strategies (perfect, ward, equipment, fallback) compose into a priority-ordered recommendation list. **Soft Delete** on BedRequest for audit trail compliance.

### Technical Deep-Dive

**Q5: How does the AI chatbot work? Isn't it just calling an API?**
> It's more sophisticated than a simple API call. We use Gemini's **function calling** feature. We define 6 functions (getOccupancyData, calculateOccupancyStats, etc.) with their parameter schemas. When a user asks "What was ICU occupancy last week?", we send the question to Gemini WITH the function declarations. Gemini determines it needs to call `calculateOccupancyStats` with `ward: "ICU"` and `startDate/endDate` for last week. We execute that function against our MongoDB, return the results to Gemini, and Gemini synthesizes a human-readable response. This prevents hallucination — Gemini can ONLY work with real data from our database. We also added caching (5-min TTL), parameter normalization ("ER" → "Emergency"), and response validation.

**Q6: How does the bed recommendation algorithm work?**
> It's a **multi-tier priority matching system**. When a bed request specifies a preferred ward and required equipment, we search in tiers:
> 1. **Perfect match**: Same ward AND same equipment
> 2. **Ward match**: Same ward, different equipment
> 3. **Equipment match**: Different ward, same equipment
> 4. **Any available**: Fallback
> 
> The smart allocation also balances variety — even if 5 beds perfectly match, we return 2 perfect + 1 alternative to give the ICU manager options. If zero beds are available, we suggest beds under cleaning (coming available soon). This mimics how real hospital staff mentally prioritize.

**Q7: How do you prevent the alert system from spamming the ICU Manager?**
> Three mechanisms: (1) **Debouncing** — alerts are checked 1.5 seconds AFTER the last stats update, not on every change. (2) **Threshold level tracking** — we maintain a map of `{wardName: 'none'|'warning'|'critical'}`. Only when the level CHANGES (e.g., 'none' → 'warning' or 'warning' → 'critical') does an alert fire. If ICU is already at 'critical', subsequent stats updates don't re-trigger. (3) **Ward-specific deduplication** — new alerts for a ward replace old ones, preventing accumulation.

**Q8: Explain the bed state machine and why you enforce transitions server-side.**
> The lifecycle is: `Available → Reserved → Occupied → Cleaning → Available`. We enforce valid transitions server-side with a transitions map. For example, you can't go from `reserved → cleaning` (you must admit the patient first). This prevents invalid states caused by buggy frontends or malicious API calls. Each transition also triggers side-effects: `occupied → cleaning` creates a CleaningJob, `* → available` clears the patient reference. Without server-side enforcement, we'd have ghost patients in unavailable beds.

**Q9: How do you handle concurrent bed reservations?**
> Currently, we use an **optimistic concurrency** approach. The approve endpoint checks `bed.status === 'available'` before updating to `reserved`. If two concurrent approve requests hit, one will find the bed already reserved and return a 400 error. For a production system, I'd improve this with MongoDB transactions or atomic `findOneAndUpdate` with a status condition: `findOneAndUpdate({_id: bedId, status: 'available'}, {status: 'reserved'})`. If the condition fails, it returns null, and we know someone else reserved it.

**Q10: How does your frontend manage state with so many real-time updates?**
> The main `App.js` component is the state authority. It holds beds, patients, alerts, settings, and stats in React state. Socket.IO events trigger `setState` updates (e.g., `setBeds(prev => prev.map(b => b._id === bed._id ? bed : b))` for in-place updates). Axios interceptors handle token injection and auto-logout. We use `useCallback` for memoized handler functions and `useEffect` with dependency arrays for lifecycle management. Each role-specific dashboard receives data via props and manages its own local state for UI-specific concerns.

### Process & Teamwork

**Q11: How did you divide work in the team?**
> We split by feature areas: ER Staff workflow (request creation), ICU Manager dashboard (approval, allocation, transfers), Ward Staff operations (status updates, cleaning), and Admin analytics (occupancy history, chatbot, settings). The backend models and server setup were done collaboratively. We used Git for version control and had regular integration sessions.

**Q12: What was the most technically challenging part?**
> The AI chatbot integration. Getting Gemini's function calling to work correctly with our database required careful prompt engineering: we had to provide exact date references (today, yesterday, last week), valid ward names, and clear function descriptions. We also discovered Gemini would use incorrect date formats or ward names, so we added a normalization layer. Debugging was hard because the AI's function call decisions were non-deterministic — same prompt could call different functions on different runs.

**Q13: If you had to scale this system to handle 1000+ beds across multiple hospitals, what would you change?**
> (1) **Microservices** — split into bed-service, patient-service, auth-service, analytics-service. (2) **Message queue** (RabbitMQ/Kafka) between services for event-driven communication. (3) **Redis** for caching bed status (frequently read, occasionally written). (4) **Database sharding** by hospital ID. (5) **Load balancer** with sticky sessions for Socket.IO. (6) **Kubernetes** for orchestration and auto-scaling. (7) **Replace Gemini function calling** with a purpose-built analytics engine for predictable query performance.

### Scenario-Based

**Q14: A nurse says a patient was discharged but the bed still shows "occupied." What might have gone wrong?**
> Several possibilities: (1) The discharge API call failed silently (network error) and the client didn't retry. (2) The bed state transition validation blocked an invalid state change. (3) The Socket.IO event was emitted but the nurse's browser wasn't connected (disconnected state). I'd check: server logs for the discharge API call, verify the patient record has `actualDischarge` set, check if a `bed-updated` event was emitted. Fix: Retry logic on the frontend, display clear error messages, and add a manual bed status override for ward staff.

**Q15: The hospital admin says the occupancy chart shows 120% for ICU. How do you debug this?**
> This would indicate a data integrity issue. (1) Check if `occupied` count exceeds `total` in the `OccupancyHistory` record. (2) Check if beds were deleted/added without updating totals. (3) Verify the `calculateOccupancyStats` function — we have validation that flags percentages outside 0-100%. (4) Check the chatbot response validation layer which catches `peak < lowest` and similar logical inconsistencies. (5) Root cause is likely multiple patients assigned to the same bed (violated uniqueness). Fix: Add a database constraint ensuring one patient per bed.

---

## 12. Quick Stats to Remember

| Metric | Value |
|---|---|
| Total beds | 60 (4 wards × 15 beds) |
| User roles | 4 (ER Staff, ICU Manager, Ward Staff, Admin) |
| Backend routes | 8 route files |
| Mongoose models | 8 schemas |
| Socket.IO events | 10+ distinct events |
| AI chatbot functions | 6 callable functions |
| Equipment types | 5 (Standard, Ventilator, ICU Monitor, Cardiac Monitor, VAD) |
| Auth token expiry | 24 hours |
| API version | v2.0 |
| Occupancy history | 60 days, hourly granularity |
| Alert thresholds | Warning: 80%, Critical: 90% (configurable) |
| Frontend port | 3000 |
| Backend port | 5000 |

---

## 13. Code Snippets to Reference

### Mongoose Pre-Save Hook (Auto-ID Generation)
```javascript
bedRequestSchema.pre('save', async function(next) {
  if (!this.requestId) {
    const count = await mongoose.model('BedRequest').countDocuments();
    this.requestId = `REQ-${String(count + 1).padStart(6, '0')}`;
  }
  next();
});
```

### Singleton Pattern (SystemSettings)
```javascript
systemSettingsSchema.statics.getSettings = async function() {
  let settings = await this.findOne({ singleton: 'settings' });
  if (!settings) {
    settings = await this.create({ singleton: 'settings' });
  }
  return settings;
};
```

### Socket.IO Room Routing
```javascript
socket.on('join-ward', (ward) => { socket.join(`ward-${ward}`); });
// Later in route handler:
req.io.to(`ward-${bed.ward}`).emit('ward-bed-updated', updatedBed);
```

### Axios Interceptor (Auto Auth + Session Expiry)
```javascript
axios.interceptors.request.use((config) => {
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
axios.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) handleLogout(); // Auto-logout
    return Promise.reject(error);
  }
);
```

---

> **💡 Tip**: When answering interview questions, always structure your response as:
> 1. **What** — State what you did
> 2. **Why** — Explain the reasoning/tradeoff
> 3. **How** — Give technical details with specific tech names/patterns
> 4. **Results** — What was the outcome or impact
