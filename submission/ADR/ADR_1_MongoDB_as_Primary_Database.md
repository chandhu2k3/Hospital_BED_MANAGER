# ADR 1: Use MongoDB as the Primary Database

## Status
**Accepted**

## Date
2025-10-15

## Context

The Hospital Bed Manager system requires persistent storage for beds (60 records across 4 wards), patients, bed requests, ward transfers, alerts, occupancy history (60-day time-series with hourly granularity per ward), system settings, and cleaning jobs. The data model has several characteristics:

1. **Hierarchical/nested data**: Bed records contain nested `location` objects (`{floor, section, roomNumber}`), patient records contain arrays of `vitals` and `medications`, and occupancy history records contain nested `wardStats` arrays with further nested `hourlyData` arrays of 24 entries each.

2. **Schema evolution**: During development, the schema changed frequently — new fields like `reservedForRequestId` on Bed, `isDeleted`/`deletedBy` for soft-delete on BedRequest, and the entire `CleaningJob` collection were added iteratively.

3. **Read-heavy workload**: Dashboards for all 4 roles continuously poll for bed stats, occupancy data, and alerts. Write operations (bed status changes, admissions, discharges) are less frequent but must propagate in real-time.

4. **No complex JOIN requirements**: While there are references between collections (e.g., Bed → Patient, BedRequest → Bed), most queries operate on single collections. Cross-collection queries are limited to `populate()` calls on specific endpoints.

**Alternatives considered**:
- **PostgreSQL**: Strong ACID guarantees, mature JOIN support, better for relational data. However, the nested/array data (vitals, medications, hourly ward stats) would require separate tables with frequent multi-table JOINs, increasing complexity.
- **MySQL**: Similar relational advantages and drawbacks as PostgreSQL.
- **Firebase/Firestore**: Real-time database with built-in WebSocket support. Rejected due to vendor lock-in, limited query flexibility, and cost unpredictability at scale.

## Decision

We will use **MongoDB** (v6.3) hosted on **MongoDB Atlas** (cloud) with **Mongoose** (v7.6) as the ODM layer.

Key implementation details:
- **8 Mongoose schemas** with built-in validation (enums, min/max, required fields)
- **Compound indexes** on frequently queried field combinations: `(ward, status)` on Bed, `(status, createdAt)` on BedRequest
- **Mongoose populate()** for cross-collection references instead of application-level JOINs
- **Pre-save middleware** on BedRequest for auto-generating sequential request IDs (`REQ-XXXXXX`)
- **Static methods** on SystemSettings for the singleton pattern (`getSettings()`)
- **MongoDB Atlas** for managed hosting with automatic backups, monitoring, and TLS encryption

## Consequences

### Positive
- **Natural document mapping**: Nested objects (location, vitals, hourlyData) map directly to MongoDB documents without normalization
- **Schema flexibility**: Adding fields like `isDeleted`, `cancelledBy`, and `reservedForRequestId` required zero migrations
- **Mongoose validation**: Enum constraints on `status` fields (`available|occupied|cleaning|reserved|maintenance`) catch invalid data at the ODM layer
- **Development speed**: Schema definition in JavaScript alongside application code accelerated prototyping
- **Atlas managed service**: Zero operational overhead for backups, scaling, and monitoring

### Negative
- **No multi-document transactions** (initially): Ward transfers update 3 documents (old bed, new bed, patient). Without transactions, a crash mid-operation could leave inconsistent state. Mitigated by careful ordering (release old bed → assign new bed → update patient).
- **No referential integrity enforcement**: MongoDB doesn't prevent orphaned references (e.g., a Bed referencing a deleted Patient). Mitigated by application-level checks.
- **Aggregation complexity**: The ward-wise statistics endpoint uses MongoDB aggregation pipeline (`$group`, `$sum`, `$cond`), which is less intuitive than SQL GROUP BY for team members familiar with relational databases.
- **No built-in audit trail**: Unlike some relational databases, MongoDB doesn't natively track document changes. Mitigated by the Alert collection and soft-delete pattern on BedRequest.
