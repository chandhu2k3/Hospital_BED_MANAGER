# ADR 5: Server-Side Bed State Machine Enforcement

## Status
**Accepted**

## Date
2025-10-25

## Context

Hospital beds follow a strict lifecycle:

```
Available → Reserved → Occupied → Cleaning → Available
```

Each state transition has real-world implications:
- **Available → Reserved**: A bed request has been approved; the bed is held for an incoming patient.
- **Reserved → Occupied**: The patient has arrived and been admitted to the bed.
- **Occupied → Cleaning**: The patient has been discharged; the bed needs to be sanitized.
- **Cleaning → Available**: Cleaning is complete; the bed is ready for the next patient.

Invalid transitions would represent impossible or dangerous real-world scenarios:
- **Reserved → Cleaning**: A reserved bed can't need cleaning (no patient was in it).
- **Cleaning → Occupied**: A bed under cleaning can't accept a patient (hygiene violation).
- **Available → Cleaning**: An empty, already-clean bed doesn't need cleaning.

Without validation, any frontend bug, malicious API call, or race condition could put the system into an inconsistent state, visible to all users across all dashboards in real-time.

**Alternatives considered**:
- **Client-side validation only**: Disable invalid status buttons in the UI. Pros: simple to implement. Cons: easily bypassed via API calls (Postman, curl, or a buggy frontend), offers no actual guarantee.
- **Database-level constraints**: Use MongoDB validators or triggers. Pros: enforced at the persistence layer. Cons: MongoDB validators can't express "allowed transitions based on current value" — they can only validate the new value in isolation, not the transition.
- **Event sourcing**: Store every state change as an immutable event and derive current state. Pros: complete audit trail, replay capability. Cons: significant architectural complexity, overkill for this application.

## Decision

We will enforce the bed state machine **server-side in the Express route handler** (`PUT /api/beds/:id`) using a transitions map. Invalid transitions are rejected with descriptive HTTP 400 error messages.

### Implementation
```javascript
const validTransitions = {
  available: ['occupied', 'reserved', 'cleaning'],
  occupied:  ['cleaning', 'available'],
  cleaning:  ['available'],
  reserved:  ['occupied', 'available']
};

const allowedNextStates = validTransitions[previousStatus] || [];
if (!allowedNextStates.includes(newStatus)) {
  return res.status(400).json({
    error: `Invalid state transition: Cannot change from '${previousStatus}' to '${newStatus}'. Allowed transitions: ${allowedNextStates.join(', ')}`
  });
}
```

### Automated Side-Effects
State transitions trigger automatic side-effects:
- **→ cleaning**: Sets `lastCleaned` timestamp, creates a `CleaningJob` record, emits `new-cleaning-job` socket event.
- **→ available**: Clears `patientId` reference from the bed document.
- **Any transition**: Creates an info `Alert` documenting who changed what, emits `bed-updated` socket event.

## Consequences

### Positive
- **Data integrity guarantee**: Regardless of client implementation (React dashboard, Postman, mobile app), the bed can never reach an invalid state.
- **Descriptive error messages**: When an invalid transition is attempted, the error response lists allowed transitions, helping developers and users understand the lifecycle.
- **Automatic workflow**: Side-effects (cleaning job creation, patient detachment) are guaranteed to execute on the correct transitions — not dependent on client-side logic.
- **Audit trail**: Every status change generates an Alert record with the user's name, providing traceability.
- **Simple implementation**: A transitions map is easy to understand, test, and modify. Adding a new state (e.g., `maintenance`) requires only adding map entries.

### Negative
- **Tightly coupled to route handler**: The state machine logic is embedded in the `PUT /api/beds/:id` route handler rather than in a dedicated service/domain layer. For a larger system, this should be refactored into a `BedStateMachine` service class.
- **No compensation for partial failures**: If the state update succeeds but the CleaningJob creation fails, the system is in a partially inconsistent state. This could be addressed with MongoDB transactions.
- **Not enforced at the model layer**: The Mongoose schema allows any valid enum value; transition validation happens at the route level. A Mongoose pre-save hook could add an additional safety layer but would need access to the previous state, which isn't trivially available.
