# Open CS Labs — API Contracts

**Status:** Initial contract proposal. Endpoint names and payloads become binding only after reviewed implementation and OpenAPI validation. Do not treat this document as proof that routes already exist.

## 1. General rules

- Base path: `/api/v1`.
- JSON for normal request/response bodies; UTF-8.
- Dates/times are ISO 8601 UTC timestamps with timezone, e.g. `2026-10-22T08:30:00Z`.
- IDs are opaque strings (UUIDs recommended). Clients must not infer permissions from IDs.
- Authentication and authorization are enforced by the backend for every protected operation.
- Use a consistent error envelope:
  ```json
  {
    "error": {
      "code": "VALIDATION_ERROR",
      "message": "Request validation failed.",
      "request_id": "opaque-request-id",
      "details": []
    }
  }
  ```
- Never include stack traces, secrets, hidden tests, private documents, or internal credentials in public errors.
- Pagination should use opaque cursors for potentially large or changing collections. Document cursor semantics per endpoint.
- Mutating requests that may be retried should support idempotency keys where duplicate effects would be harmful.
- Generate/validate the actual OpenAPI document from the implementation. Do not manually claim that an endpoint exists before it is implemented.
- Contract changes require updating callers, tests, and this document in the same change.

## 2. Authentication model

Preferred initial model: server-managed session using a Secure, HttpOnly cookie in production. Configure SameSite and CSRF protection according to deployment. If the implementation chooses bearer tokens instead, record an ADR and document token storage/rotation; never put long-lived secrets in browser localStorage.

Suggested endpoints:
- `POST /auth/register` — create an account subject to validation and any enrollment policy.
- `POST /auth/login` — authenticate and establish session.
- `POST /auth/logout` — revoke session.
- `GET /auth/me` — return current user and effective roles.
- `POST /auth/csrf` — only if the selected CSRF strategy requires an explicit token endpoint.

Registration must not allow a client to self-assign professor/admin roles. Professor provisioning is controlled by an authorized process.

## 3. Courses and content

Suggested endpoints:
- `GET /courses` — list courses current user may view.
- `GET /courses/{course_id}` — course details after membership/visibility check.
- `GET /courses/{course_id}/announcements`
- `POST /courses/{course_id}/announcements` — authorized professor/course staff only.
- `GET /courses/{course_id}/resources`
- `POST /courses/{course_id}/resources` — authorized staff; upload flow must enforce file limits and access policy.
- `GET /courses/{course_id}/assignments`
- `GET /assignments/{assignment_id}` — access check before returning details.

Membership/enrollment management routes must be defined before implementation and must not allow arbitrary users to enroll into private courses without the configured policy.

## 4. Problems and submissions

Suggested endpoints:
- `GET /problems` — filterable problem catalog; never returns hidden test content.
- `GET /problems/{problem_id}` — statement and permitted examples.
- `POST /submissions` — validate request, persist `QUEUED` state, enqueue safely, return a submission ID.
- `GET /submissions/{submission_id}` — only owner or authorized course staff.
- `GET /submissions?problem_id=...` — current user's submissions by default; staff access is policy-controlled.
- `GET /submissions/{submission_id}/results` — safe normalized result details.

Example submission request (illustrative contract only):
```json
{
  "problem_id": "opaque-problem-id",
  "language": "python",
  "source_code": "print('hello')",
  "idempotency_key": "client-generated-unique-value"
}
```

Example response:
```json
{
  "id": "opaque-submission-id",
  "status": "QUEUED",
  "created_at": "2026-10-22T08:30:00Z"
}
```

Candidate statuses: `QUEUED`, `RUNNING`, `ACCEPTED`, `WRONG_ANSWER`, `TIME_LIMIT_EXCEEDED`, `MEMORY_LIMIT_EXCEEDED`, `RUNTIME_ERROR`, `COMPILE_ERROR`, `JUDGE_ERROR`, `CANCELLED`. Exact states and transitions must be enforced in backend code and tests.

Never trust a client-supplied verdict, score, execution time, or test result. Hidden tests are run only by the judge side.

## 5. Drafts

Suggested endpoints:
- `GET /drafts?problem_id=...`
- `PUT /drafts/{draft_id}` or an agreed upsert route.

Define conflict behavior, ownership, maximum size, and save timestamps before implementation. Browser localStorage may provide a small local cache, but it is not the authoritative durable server copy.

## 6. Chat — WebSocket contract proposal

Suggested endpoint: `GET /api/v1/chat/ws` upgraded to WebSocket, with session authentication and origin checks.

Illustrative client messages:
```json
{"type":"join_room","room_id":"opaque-room-id"}
{"type":"send_message","room_id":"opaque-room-id","client_message_id":"unique-client-id","content":"Hello"}
{"type":"ack","message_id":"server-message-id"}
```

Illustrative server messages:
```json
{"type":"message_created","message":{"id":"server-message-id","room_id":"opaque-room-id","sender_id":"opaque-user-id","content":"Hello","created_at":"2026-10-22T08:30:00Z"}}
{"type":"error","code":"FORBIDDEN","message":"You cannot access this room."}
```

These event names are proposed and must be kept consistent with the actual implementation. Requirements:
- Authenticate connection and authorize each room join/send.
- Enforce payload-size and message-rate limits.
- Assign stable server message IDs; deduplicate retries using client message IDs where appropriate.
- Define persistence-before-broadcast behavior.
- Support history pagination via REST or a documented WebSocket event.
- Reconnect must fetch messages after a cursor/last acknowledged ID.
- Multi-instance deployments need tested cross-instance fan-out; in-memory sockets alone are insufficient.
- Do not assume Cassandra is approved until its decision gate is complete.

## 7. RAG doubts

Suggested endpoints:
- `POST /courses/{course_id}/questions` — question answered using only documents the user is allowed to access.
- `POST /courses/{course_id}/documents` — staff-authorized ingestion.
- `GET /rag/feedback` or `POST /rag/feedback` — define precise method/path before implementation; this line is a reminder, not a final route.

Answers should include source references when available. Enforce document permissions before retrieval and do not expose provider keys or private source content outside the user's access rights. Add provider timeouts, quotas, rate limits, and a safe fallback when the model is unavailable.

## 8. Professor grading and reports

Suggested endpoints:
- `GET /courses/{course_id}/analytics`
- `POST /courses/{course_id}/grading-runs`
- `GET /grading-runs/{run_id}`
- `POST /grading-runs/{run_id}/approve` — authorized professor only, with audit record.
- `GET /courses/{course_id}/reports`
- `POST /courses/{course_id}/reports` — generate a report job for an authorized scope.
- `GET /reports/{report_id}` — metadata after access check.
- `GET /reports/{report_id}/download` — secure download or short-lived signed URL after access check.

Relative grading is configurable analysis, not a universal IIT policy. A run records input scores, policy/version, recommendations, approver, and audit events. Manual overrides require reason and identity.

## 9. Health and operational endpoints

- `GET /health/live` — process liveness only.
- `GET /health/ready` — readiness and essential dependency checks, without exposing secrets or detailed infrastructure internals.
- Metrics endpoint, if enabled, must be restricted to the monitoring network/authorized systems.

## 10. OpenAPI acceptance criteria

- Every implemented endpoint has schema, auth requirement, status codes, and error behavior.
- OpenAPI is generated from the running code or checked-in source and validated in CI.
- Contract tests verify representative success, validation failure, unauthenticated, and forbidden cases.
- No endpoint in this document should be described as implemented until tests and route inspection confirm it.
