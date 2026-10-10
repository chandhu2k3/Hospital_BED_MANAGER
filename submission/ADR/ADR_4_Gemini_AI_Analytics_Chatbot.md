# ADR 4: Google Gemini AI with Function Calling for Analytics Chatbot

## Status
**Accepted**

## Date
2025-11-05

## Context

The Hospital Administrator role requires an analytics capability to answer ad-hoc questions about hospital operations, such as:
- "What was the average ICU occupancy last week?"
- "Compare this month's admission rate to last month."
- "How many bed requests were denied in the last 30 days?"

Building a traditional analytics dashboard for every possible query would be impractical. The system stores 60 days of occupancy history with hourly ward-wise breakdowns, bed request records, patient admission/discharge data, and real-time bed status — sufficient data for rich analytical insights, but the query interface needs to be flexible and natural.

**Key requirements**:
1. **Natural language input**: Admins should ask questions in plain English, not construct database queries.
2. **Grounded in real data**: Responses must be based on actual hospital data, not AI-generated fabrications.
3. **Multiple data sources**: A single question may need data from multiple collections (occupancy history + bed requests + patients).
4. **Conversation history**: Follow-up questions (e.g., "What about ICU specifically?") should maintain context.

**Alternatives considered**:
- **Pre-built analytics dashboards (Recharts only)**: Static charts for common metrics. Pros: fast, predictable. Cons: can't answer ad-hoc questions, requires developer effort for each new metric.
- **OpenAI GPT API**: Industry-leading LLM. Pros: excellent function calling support, mature tooling. Cons: higher cost per token, usage policy restrictions for healthcare data.
- **Custom NLP + SQL/MongoDB query generator**: Build a query parser that converts natural language to database queries. Pros: no third-party AI dependency. Cons: enormous development effort, limited to pre-defined query patterns, poor natural language understanding.
- **LangChain with local LLM**: Open-source LLM with LangChain agent. Pros: no API cost, data stays local. Cons: requires significant compute resources, lower quality responses for complex questions.

## Decision

We will use **Google Gemini AI** (Gemini 2.5 Flash model, via `@google/generative-ai` v0.24.1) with **function calling** to create an analytics chatbot that queries real hospital data.

### Function Calling Architecture
Instead of letting the AI generate database queries directly, we define **6 typed functions** that the AI can invoke:

| Function | Parameters | Returns |
|---|---|---|
| `getOccupancyData` | startDate, endDate, ward? | Time-series occupancy data |
| `calculateOccupancyStats` | startDate, endDate, ward? | Average, peak, lowest, median, current |
| `getBedRequestStats` | startDate, endDate, status?, ward? | Request counts by status and ward |
| `getPatientStats` | startDate, endDate, ward? | Admissions, discharges, avg stay |
| `getCurrentBedStatus` | ward? | Real-time bed counts by status |
| `compareTimePeriods` | period1Start/End, period2Start/End | Comparative analysis |

### Safeguards
1. **Response caching**: 5-minute TTL cache keyed by `message + today_date` to reduce API calls and cost.
2. **Parameter normalization**: Maps informal ward names to exact DB values (e.g., "ER" → "Emergency", "intensive care" → "ICU").
3. **Input validation**: Checks date formats (YYYY-MM-DD), valid ward names, date range consistency (start < end).
4. **Response validation**: Verifies percentages are 0-100%, peak ≥ lowest, status counts sum to total.
5. **Execution logging**: Tracks function name, parameters, execution time, success/failure, and warnings.
6. **Calculation transparency**: Every response includes the formula used and sample size.
7. **Maximum function call rounds**: Capped at 5 rounds to prevent infinite loops.
8. **Admin-only access**: Middleware restricts chatbot endpoint to `admin` role.

### System Prompt
The system prompt provides:
- Current date and time
- Available data range (last 60 days)
- Exact valid ward names with aliases
- Data schema and calculation formulas
- Function calling guidelines with date defaults
- Accuracy requirements

## Consequences

### Positive
- **No hallucination**: AI cannot fabricate data — it must call our functions to get real database values. Functions execute Mongoose queries against the actual MongoDB collections.
- **Natural language flexibility**: Admins can ask any question about occupancy, admissions, requests, or beds without learning a query language.
- **Conversation continuity**: Gemini's chat API maintains conversation history, enabling follow-up questions.
- **Low development effort**: 6 functions + system prompt vs. building a full analytics dashboard for every possible metric.
- **Cost-effective**: Gemini 2.5 Flash is optimized for speed and cost. Response caching further reduces API calls.

### Negative
- **External API dependency**: If Google Gemini is unavailable, the chatbot returns a 503 error. Mitigated by a fallback error message and graceful degradation (analytics dashboards still work independently).
- **API cost**: Each conversation round costs tokens. Mitigated by 5-minute response caching and max 5 function call rounds.
- **Non-deterministic responses**: Same question may produce slightly different natural language responses. Function calls are deterministic but AI's choice of which function to call can vary.
- **Latency**: Multi-round function calling adds latency (typically 2-5 seconds per question). Acceptable for analytics queries that aren't time-critical.
- **Data privacy consideration**: User messages and database summaries are sent to Google's API. Mitigated by sending only aggregated statistics, not individual patient records (PHI).
