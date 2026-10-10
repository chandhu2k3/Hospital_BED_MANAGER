# AI Usage Report — Class Activity 6: Architectural Documentation

**Course**: S26CS6.401 — Software Engineering  
**Activity**: Class Activity 6 — Architectural Documentation  
**Repository**: Hospital Bed Manager System (custom GitHub project)
**Repo Link**: https://github.com/piyukr2/Bed-Manager-updated.git 
**AI Tool Used**: Antigravity (AI coding assistant by Google DeepMind)

---

## 1. C1 Diagram — System Context

### Generation Process
- **Tool & Approach**: Used Antigravity AI assistant to analyze the full codebase — `server.js`, all route files (`beds.js`, `patients.js`, `bedRequests.js`, `wardTransfers.js`, `chatbot.js`, `alerts.js`, `systemSettings.js`, `seedRoutes.js`), all model files, and `package.json`. The AI generated a C4 Context diagram in PlantUML syntax using the `C4-PlantUML` library from the PlantUML stdlib.

### Initial Generated Version
The initial diagram correctly identified:
- The 4 user personas (ER Staff, ICU/Bed Manager, Ward Staff, Hospital Administrator)
- The Hospital Bed Manager as the central system
- MongoDB Atlas and Google Gemini AI API as external systems
- Basic directional relationships between all elements

However, the initial relationship labels were generic (e.g., "Uses the system") and the system description was brief without mentioning specific wards or capabilities.

### Changes Made
1. **Refined relationship labels per role**: Replaced generic "Uses" labels with role-specific actions — e.g., ER Staff: "Creates bed requests, views queue"; ICU Manager: "Approves requests, allocates beds, manages transfers, receives alerts."
2. **Added communication protocol annotations**: Added "HTTPS / WebSocket" for user-system links, "MongoDB Protocol / TLS" for the database, and "HTTPS / REST API" for Gemini — as recommended by the C4 model.
3. **Expanded system description**: Added the specific wards (Emergency, ICU, General Ward, Cardiology) and key system capabilities (smart allocation, occupancy analytics, AI-powered insights).
4. **Adjusted layout directionality**: Changed to `Rel_D` (downward) for user-system and `Rel_R` (rightward) for system-external to prevent crossing lines.

### Why Changes Were Necessary
- Role-specific labels demonstrate a clear understanding of system stakeholders and their distinct responsibilities — essential for a C1 diagram.
- Protocol annotations are a C4 best practice that adds architectural clarity.
- The system description needed sufficient detail for evaluators to understand scope at a glance.

---

## 2. C2 Diagram — Container

### Generation Process
- **Tool & Approach**: Used Antigravity AI to zoom into the system boundary from C1 and produce a Container diagram. The AI analyzed the project structure (separate `frontend/` and `backend/` directories), `package.json` dependencies, Socket.IO configuration in `server.js`, and the MongoDB connection setup.

### Initial Generated Version
The initial diagram showed:
- React SPA (Frontend)
- Express API Server (Backend)
- MongoDB Database
- Socket.IO was shown as part of the Backend container, not separately

### Changes Made
1. **Separated WebSocket server as a distinct container**: Although Socket.IO runs in the same Node.js process as Express, I separated it as a visually distinct container to emphasize the real-time architecture. The "In-process (shared io object)" relationship label clarifies that they share the same process.
2. **Added technology versions**: Replaced generic labels with specific versions from `package.json` — React 18.2, Express 4.18, MUI 7.3, Socket.IO 4.7, MongoDB 6.3, Mongoose 7.6, Axios 1.4.
3. **Added port numbers**: Frontend port 3000, Backend/WebSocket port 5000 — these are concrete deployment details.
4. **Added database collection count and indexes**: Specified "8 collections" and mentioned compound indexes on `(ward+status)` and `(status+createdAt)`.
5. **Detailed the Gemini interaction**: Added "6 function declarations, up to 5 rounds" to accurately represent the function calling architecture.

### Why Changes Were Necessary
- Separating the WebSocket server visually communicates the real-time architecture pattern (REST + WebSocket hybrid) — a key architectural decision in this system.
- Version numbers and port numbers prove familiarity with the actual deployment, not just conceptual understanding.
- Database details (collections, indexes) add the technical depth expected in a C2 diagram.

---

## 3. C3 Diagram — Component

### Generation Process
- **Tool & Approach**: Used Antigravity AI to zoom into the API Server container from C2 and identify all internal components. The AI analyzed each route file, model file, and middleware function in the `backend/src/` directory to map out the components and their interactions.

### Initial Generated Version
The initial diagram listed 8 components matching the Express route files:
- Authentication Module
- Bed Management
- Patient Management
- Bed Request System
- Ward Transfer System
- AI Chatbot
- Alert System
- System Settings

The initial version had flat layout (no grouping), generic descriptions, and missing components.

### Changes Made
1. **Added 5 additional components** that the initial version missed:
   - **RBAC Middleware**: Separated from the Auth Module since it's architecturally distinct — it sits between the frontend and every other component as a cross-cutting concern.
   - **User Management**: Inline routes in `server.js` for CRUD on users.
   - **Cleaning Workflow**: The `CleaningJob` model and its automated creation on bed discharge.
   - **Occupancy History**: The `OccupancyHistory` model as a time-series data component powering analytics.
   - **Seed Data Manager**: `seedRoutes.js` for importing historical data.
2. **Organized into 4 logical groups** using PlantUML boundaries:
   - Authentication & Authorization
   - Bed Operations & Lifecycle
   - Request & Transfer Workflows
   - Analytics, AI & Configuration
3. **Added comprehensive relationship mapping**:
   - Frontend→Component: All 9 API route paths
   - RBAC→Components: All 8 role guard connections
   - Inter-component: 8 dependencies (requests→beds, transfers→patients, beds→cleaning, etc.)
   - Component→Database: All 10 Mongoose model references
   - Component→WebSocket: All 12 Socket.IO event names across 5 emitting components
   - Chatbot→Gemini: Function calling details
4. **Added source file paths**: Each component references its actual source file (e.g., `routes/beds.js`, `routes/chatbot.js`).
5. **Added route-level detail**: Each component lists its specific REST endpoints, not just generic descriptions.

### Why Changes Were Necessary
- The RBAC Middleware is a critical cross-cutting concern — omitting it would misrepresent the security architecture.
- Logical grouping prevents the diagram from looking like a random collection of boxes; it shows intentional domain-driven design.
- Route-level detail and Socket.IO event names create direct traceability from diagram to source code.
- Missing components (Cleaning, History, Seed) would leave gaps in understanding the data flow and lifecycle management.

---

## 4. ADR 1 — MongoDB as Primary Database

### Generation Process
- **Tool & Approach**: Used Antigravity AI with all 8 Mongoose model files (`Bed.js`, `Patient.js`, `BedRequest.js`, `WardTransfer.js`, `OccupancyHistory.js`, `Alert.js`, `CleaningJob.js`, `SystemSettings.js`) as input. Generated following the Michael Nygard ADR format (Status, Context, Decision, Consequences).

### Initial Generated Version
The initial ADR stated: "MongoDB was chosen for flexible schema and document storage." It lacked alternatives, specific data model references, and balanced consequences.

### Changes Made
1. **Added 3 specific alternatives considered**: PostgreSQL, MySQL, Firebase/Firestore — each with pros and cons evaluated against our specific requirements (nested data, schema evolution, read-heavy workload).
2. **Added concrete data model references**: Cited nested structures from the actual codebase — `location {floor, section, roomNumber}` in Bed, `vitals[]` and `medications[]` arrays in Patient, `hourlyData[]` with 24 entries in OccupancyHistory.
3. **Added implementation-level decisions within MongoDB**: Compound indexes on `(ward, status)`, pre-save hooks for auto-generating `REQ-XXXXXX` IDs, static methods for the singleton pattern (`getSettings()`), and Mongoose `populate()` for cross-collection references.
4. **Expanded negative consequences with mitigations**: Added risks of no multi-document transactions (mitigated by careful operation ordering), no referential integrity (mitigated by application-level checks), and aggregation complexity.

### Why Changes Were Necessary
- An ADR without alternatives is just a statement, not a decision record — showing PostgreSQL and Firebase were evaluated demonstrates deliberate engineering judgment.
- Tying the MongoDB choice to actual schema structures (vitals arrays, hourly ward stats) grounds the decision in real code, not theory.
- Honest negative consequences show architectural maturity; ignoring downsides would weaken the ADR's credibility.

---

## 5. ADR 2 — JWT-Based Stateless Authentication

### Generation Process
- **Tool & Approach**: Used Antigravity AI to analyze the authentication implementation in `server.js` — login route, `authenticateToken` middleware, `authorize()` function, JWT signing/verification code, and Axios interceptors in the frontend `App.js`.

### Initial Generated Version
Covered JWT basics: token issuance on login, verification middleware, and 24-hour expiry.

### Changes Made
1. **Added JWT payload structure**: Documented the exact fields — `{id, username, role, ward, name, iat, exp}` — showing what information travels with each request.
2. **Added WebSocket authentication context**: Socket.IO connections use the JWT token via the `auth: { token }` option during WebSocket handshake — this was a key architectural decision that JWT enables cleanly.
3. **Added auto-logout mechanism**: Documented the Axios response interceptor that detects 401/403 errors and automatically triggers logout with localStorage cleanup.
4. **Added security tradeoff discussion**: Acknowledged the XSS risk of storing JWT in localStorage and justified the decision for an internal hospital tool (not public-facing) with Content Security Policy as mitigation.
5. **Added alternatives**: Session-based auth (express-session + Redis), OAuth 2.0/OIDC, and API keys — each evaluated against our requirements (stateless, WebSocket-compatible, cross-device).

### Why Changes Were Necessary
- The token payload structure shows the reader exactly how role-based authorization works at the data level.
- WebSocket authentication is often overlooked and is a genuine architectural concern for real-time systems.
- The security tradeoff discussion (localStorage + XSS) demonstrates awareness of security implications, not just feature implementation.

---

## 6. ADR 3 — Socket.IO for Real-Time Communication

### Generation Process
- **Tool & Approach**: Used Antigravity AI to analyze the Socket.IO configuration in `server.js` and traced every `req.io.emit()` call across all route files to compile a complete event inventory.

### Initial Generated Version
Covered basic Socket.IO usage: bidirectional events and room-based routing.

### Changes Made
1. **Added a complete event table**: Compiled all 13+ events with their emitter (server/client), direction, and purpose — derived from actual `io.emit()` calls traced through `beds.js`, `bedRequests.js`, `wardTransfers.js`, `patients.js` (inside `server.js`), and `systemSettings.js`.
2. **Added connection management details**: Documented the `connectedClients` Map for tracking connections, `client-count` event for live user counts, and auto-reconnection configuration (10 attempts, 1s delay).
3. **Added scalability limitation**: Noted that Socket.IO connections are stateful and pinned to a server instance — multi-instance deployment would require `@socket.io/redis-adapter`, which is not yet implemented.
4. **Added the REST + WebSocket hybrid pattern**: Explained how `req.io = io` middleware bridges REST handlers with WebSocket emissions, enabling atomic state changes with real-time broadcast.
5. **Added alternatives**: Server-Sent Events, Long Polling, native WebSocket (ws library), Firebase Realtime Database — each evaluated against our bidirectional, room-based routing requirements.

### Why Changes Were Necessary
- A complete event table transforms this ADR from a design justification into practical documentation that future developers can reference.
- The scalability limitation is an honest assessment — omitting it would misrepresent the system's production readiness.
- The REST + WebSocket hybrid pattern (`req.io`) is the most architecturally significant aspect of the real-time design and was initially underemphasized.

---

## 7. ADR 4 — Google Gemini AI with Function Calling for Analytics Chatbot

### Generation Process
- **Tool & Approach**: Used Antigravity AI to analyze the entire `chatbot.js` route file (946 lines of code), including all 6 function implementations, the caching mechanism, parameter normalization logic, response validation, and system prompt construction.

### Initial Generated Version
Covered the basic function calling concept: user sends query, Gemini calls a function, backend executes and returns result.

### Changes Made
1. **Added complete function table**: Listed all 6 callable functions (`getOccupancyData`, `calculateOccupancyStats`, `getBedRequestStats`, `getPatientStats`, `getCurrentBedStatus`, `compareTimePeriods`) with their parameters and return types.
2. **Added 7 safeguards**: Response caching (5-minute TTL), parameter normalization (ER→Emergency, intensive care→ICU), input validation (date format, ward names), response validation (percentages 0-100%, peak≥lowest), execution logging, calculation transparency, and max 5 function call rounds.
3. **Added data privacy consideration**: Noted that only aggregated statistics are sent to Google's API — never individual patient records (PHI).
4. **Added system prompt documentation**: Described the contextual information provided to Gemini — current date, available data range, exact ward names with aliases, data schema, and calculation formulas.
5. **Added alternatives**: Pre-built dashboards (Recharts only), OpenAI GPT, custom NLP query parser, LangChain with local LLM.

### Why Changes Were Necessary
- The 7 safeguards represent the most substantial engineering effort in the chatbot module — omitting them would undervalue the implementation.
- Data privacy is non-negotiable in healthcare software and must be explicitly addressed.
- Function calling (not plain prompting) is the key architectural choice that prevents hallucination — this distinction is critical for evaluators to understand.

---

## 8. ADR 5 — Server-Side Bed State Machine Enforcement

### Generation Process
- **Tool & Approach**: Used Antigravity AI to analyze the state transition validation code in `routes/beds.js` (`PUT /api/beds/:id` handler) and the side-effects triggered on each transition.

### Initial Generated Version
Covered the basic transitions map concept: a dictionary mapping current status to allowed next statuses.

### Changes Made
1. **Added real-world justification for each invalid transition**: Explained why "reserved→cleaning" is invalid (no patient was in the bed), why "cleaning→occupied" is invalid (hygiene violation), and why "available→cleaning" is unnecessary.
2. **Added automated side-effects documentation**: `occupied→cleaning` creates a CleaningJob record and emits `new-cleaning-job` event; any transition to `available` clears the `patientId` reference; every transition creates an info Alert for the audit trail.
3. **Added the actual code snippet**: Included the `validTransitions` map and error response format from the codebase to show the concrete implementation.
4. **Added 3 alternative approaches evaluated**: Client-side validation only (easily bypassed), database-level constraints (MongoDB can't express conditional transitions), and event sourcing (overkill for this use case).

### Why Changes Were Necessary
- Connecting state transitions to physical hospital scenarios (hygiene violations, ghost patients) demonstrates domain understanding beyond code.
- Automated side-effects are the architectural rationale for server-side enforcement — without documenting them, the "why not client-side?" question remains unanswered.
- Alternative approaches show the decision was deliberate and considered multiple options.

---

## Tools Used Summary

| Artifact | AI Tool | Diagram Tool | Rendering |
|---|---|---|---|
| C1 Diagram (Context) | Antigravity AI | PlantUML (C4-PlantUML library) | PlantUML Web Server → PNG |
| C2 Diagram (Container) | Antigravity AI | PlantUML (C4-PlantUML library) | PlantUML Web Server → PNG |
| C3 Diagram (Component) | Antigravity AI | PlantUML (C4-PlantUML library) | PlantUML Web Server → PNG |
| ADR 1 — MongoDB | Antigravity AI | — | — |
| ADR 2 — JWT Auth | Antigravity AI | — | — |
| ADR 3 — Socket.IO | Antigravity AI | — | — |
| ADR 4 — Gemini AI | Antigravity AI | — | — |
| ADR 5 — State Machine | Antigravity AI | — | — |
| This Report | Manual authoring | — | — |

### Process Summary
1. **Generation**: Each artifact was initially generated by the AI assistant based on analysis of the actual codebase source files.
2. **Review**: Each generated artifact was reviewed against the codebase for accuracy, completeness, and adherence to C4/ADR standards.
3. **Refinement**: Modifications were made to add missing details, improve clarity, fix inaccuracies, and ensure the artifacts reflect the true architecture of the system.
4. **Final Output**: Diagrams were rendered as PNG images via the PlantUML Web Server. ADRs were finalized as Markdown files.
