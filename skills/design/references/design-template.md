# {Product Name} — Technical Design Document

**Author:** {author}
**Date:** {date}
**Status:** Draft
**Version:** 1.0
**Source PRD:** [{prd filename}]({relative path to prd})

---

## 1. Overview

### 1.1 Summary
{One-paragraph summary of what is being built and the high-level technical approach. A reader should be able to understand the scope and strategy from this paragraph alone.}

### 1.2 Goals
{Restate relevant goals from the PRD, framed as technical objectives.}
- {goal 1}
- {goal 2}

### 1.3 Non-Goals
{What this design explicitly does not address. Include future goals that are being deferred.}
- {non-goal 1}
- {non-goal 2}

### 1.4 PRD Open Items Resolved
| PRD Item | Resolution |
|----------|------------|
| {[TBD] or open question from PRD} | {Design decision made} |

---

## 2. Codebase Context & Technology Stack

### 2.1 Codebase Relationship
{How this feature maps to the existing codebase: modifying/extending an existing module, adding a new module, creating a standalone service, or greenfield. Reference specific directories and files.}

- **Approach:** {Modify existing / New module / Standalone service / Greenfield}
- **Affected areas:** {List specific directories, modules, or files that will be touched}

### 2.2 Technology Stack
| Layer | Technology | Version | Notes |
|-------|-----------|---------|-------|
| {e.g., Language} | {e.g., TypeScript} | {e.g., 5.x} | {existing / new, rationale if new} |
| {e.g., Framework} | {e.g., Express} | {e.g., 4.18} | {} |
| {e.g., ORM / DB Client} | {e.g., Drizzle ORM} | {} | {} |
| {e.g., Database} | {e.g., PostgreSQL} | {e.g., 16} | {} |
| {e.g., Validation} | {e.g., Zod} | {} | {} |
| {e.g., Testing} | {e.g., Vitest} | {} | {} |

### 2.3 External APIs & Services
| Service | Purpose | Auth Method | Docs |
|---------|---------|-------------|------|
| {e.g., Stripe API} | {payments} | {API key} | {link} |

---

## 3. System Context

### 3.1 Current Architecture
{Brief description of the existing system as relevant to this design. Reference actual project structure.}

### 3.2 Proposed Architecture

```mermaid
graph TD
    {architecture diagram showing components and their relationships}
```

### 3.3 Key Components
| Component | Responsibility | New / Modified |
|-----------|---------------|----------------|
| {component} | {what it does} | {New / Modified / Existing} |

---

## 4. Data Model

### 4.1 Entities

#### {Entity Name}
| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| {field} | {type} | {PK, FK, NOT NULL, UNIQUE, etc.} | {description} |

{Repeat for each entity.}

### 4.2 Entity Relationships

```mermaid
erDiagram
    {entity relationship diagram}
```

### 4.3 Data Migration
{Describe any schema changes to existing data, migration strategy, and rollback plan. Write "N/A — greenfield" if no migrations needed.}

### 4.4 Data Access Patterns
| Operation | Pattern | Expected Volume | Index / Optimization |
|-----------|---------|----------------|---------------------|
| {e.g., "List user's projects"} | {read, paginated} | {e.g., "~100 req/s"} | {index on user_id} |

---

## 5. API Design

### 5.1 New Endpoints / Interfaces

#### `{METHOD} {path}`
- **Description:** {what it does}
- **Auth:** {auth requirements}
- **Request:**
```json
{
  {request shape}
}
```
- **Response (success):**
```json
{
  {response shape}
}
```
- **Error responses:**
| Status | Code | Description |
|--------|------|-------------|
| {4xx/5xx} | {error_code} | {when this happens} |

{Repeat for each endpoint.}

### 5.2 Modified Endpoints
| Endpoint | Change | Backward Compatible |
|----------|--------|--------------------|
| {endpoint} | {what changes} | {Yes / No — migration plan} |

### 5.3 Event / Message Contracts
{If the design includes async messaging (events, queues, webhooks), define the contracts here. Write "N/A" if purely synchronous.}

| Event | Producer | Consumer(s) | Payload Shape |
|-------|----------|-------------|---------------|
| {event name} | {component} | {component(s)} | {brief shape or link} |

---

## 6. Key Sequences

### 6.1 {Primary User Workflow Name}

```mermaid
sequenceDiagram
    {sequence diagram for the primary happy-path flow}
```

### 6.2 {Error / Edge Case Flow Name}

```mermaid
sequenceDiagram
    {sequence diagram for a critical error or edge case flow}
```

{Add additional sequence diagrams as needed for complex flows.}

---

## 7. Infrastructure & Operations

### 7.1 Infrastructure Requirements
| Resource | Purpose | Specification |
|----------|---------|---------------|
| {e.g., PostgreSQL} | {primary datastore} | {version, sizing, managed/self-hosted} |

### 7.2 Scaling Strategy
{How the system scales. Horizontal, vertical, auto-scaling triggers, queue-based backpressure, etc.}

### 7.3 Failure Modes & Resilience
| Failure | Impact | Mitigation |
|---------|--------|------------|
| {e.g., "Database unavailable"} | {what breaks} | {retry, circuit breaker, fallback, etc.} |

### 7.4 Observability
| Signal | Tool | Details |
|--------|------|---------|
| Metrics | {e.g., Prometheus} | {key metrics to track} |
| Logs | {e.g., structured JSON} | {key log events} |
| Traces | {e.g., OpenTelemetry} | {key spans} |
| Alerts | {e.g., PagerDuty} | {alert conditions and thresholds} |

---

## 8. Security & Privacy

### 8.1 Authentication & Authorization
{How users/services authenticate. What authorization model is used (RBAC, ABAC, etc.). Reference existing patterns if extending them.}

### 8.2 Data Protection
| Data | Classification | Protection |
|------|---------------|------------|
| {e.g., "user email"} | {PII / sensitive / public} | {encryption at rest, in transit, access controls} |

### 8.3 Compliance
{Relevant regulatory or compliance requirements and how this design satisfies them. Write "No specific compliance requirements" if none.}

---

## 9. Testing Strategy

| Level | Scope | Approach |
|-------|-------|----------|
| Unit | {what's unit-tested} | {framework, coverage targets} |
| Integration | {what's integration-tested} | {approach, test environment} |
| E2E | {critical user paths} | {framework, approach} |
| Load / Performance | {what's load-tested} | {tool, target thresholds} |

---

## 10. Alternatives Considered

### 10.1 {Alternative Approach Name}
- **Description:** {what it is}
- **Pros:** {advantages}
- **Cons:** {disadvantages}
- **Why rejected:** {decision rationale}

{Repeat for each alternative.}

### 10.2 Key Trade-offs in Chosen Design
| Decision | Trade-off | Rationale |
|----------|-----------|-----------|
| {e.g., "SQL over NoSQL"} | {what you gain vs. lose} | {why this is the right call} |

---

## 11. Implementation Guidance

This section provides enough detail to derive a task-level work breakdown.

### 11.1 Component Work Breakdown

#### {Component / Layer 1}
| Work Item | Description | Dependencies | Complexity |
|-----------|-------------|--------------|------------|
| {item} | {what needs to be built/changed} | {blockers or prereqs} | {S / M / L} |

{Repeat for each component or layer.}

### 11.2 Suggested Implementation Order
{Recommended sequencing of work. What should be built first to unblock other work? What can be parallelized?}

1. **Phase 1:** {description — foundational work}
2. **Phase 2:** {description — core feature work}
3. **Phase 3:** {description — integration, polish, hardening}

### 11.3 Dependencies & Blockers
| Dependency | Owner | Status | Impact if Delayed |
|------------|-------|--------|-------------------|
| {e.g., "API gateway config"} | {team/person} | {Ready / In Progress / Blocked} | {what can't proceed} |

---

## 12. Rollout Plan

### 12.1 Rollout Phases
| Phase | Audience | Criteria to Advance |
|-------|----------|--------------------|
| {e.g., "Internal dogfood"} | {who} | {what must be true to move to next phase} |

### 12.2 Feature Flags
| Flag | Purpose | Default |
|------|---------|--------|
| {flag name} | {what it controls} | {off / on} |

### 12.3 Rollback Plan
{How to revert if something goes wrong. Include data rollback considerations.}

---

## 13. Open Questions & Risks
| Item | Type | Impact | Owner | Status |
|------|------|--------|-------|--------|
| {item} | Risk / Question | {H/M/L} | {who will resolve} | Open |
