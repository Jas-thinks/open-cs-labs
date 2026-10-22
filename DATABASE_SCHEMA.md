# Open CS Labs — Database Ownership and Logical Schema

**Status:** Logical model proposal. It is not executable DDL. Actual schema is defined by versioned migrations and must remain synchronized with this document.

## 1. Ownership principles

- PostgreSQL is the authoritative store for application identity, course access, problem metadata, submissions, scores, grading, document metadata, and report metadata.
- Cassandra is conditional and, if approved, stores chat-message history only.
- Redis is optional ephemeral infrastructure for queues/cache/rate limiting/coordination; it is not the only copy of business data.
- Object storage holds binary files; PostgreSQL stores metadata and access control.
- Do not duplicate a source of truth without an explicit consistency/reconciliation design.

## 2. PostgreSQL logical entities

### Identity and access
- `users`: id, display_name, email (unique/normalized), password_hash, status, created_at, updated_at.
- `roles` or constrained role mapping: role definitions.
- `user_roles` if multiple roles are supported; avoid client-controlled role assignment.
- `sessions` if sessions are persisted in PostgreSQL.
- `courses`: id, code, title, description, owner/created_by, visibility, status, timestamps.
- `course_memberships`: course_id, user_id, membership_role/status, joined_at; unique course/user pair.

### Course content
- `announcements`: id, course_id, author_id, title, body, published_at, status.
- `file_assets`: id, owner/course, storage_key, filename, content_type, size_bytes, checksum, visibility, created_at.
- `learning_resources`: id, course_id, file_asset_id or external URL, title, type, visibility, created_by.
- Upload metadata must not contain raw secrets or public storage credentials.

### Problems and assignments
- `problems`: id, slug, title, statement, difficulty, supported_languages, visibility, created_by, timestamps.
- `problem_examples`: problem_id, input, expected_output, explanation, display_order. These are public examples, not hidden tests.
- `test_case_sets` / `hidden_test_case_refs`: protected metadata and storage references; hidden contents must not be returned by student APIs.
- `assignments`: id, course_id, title, instructions, due_at, status, created_by.
- `assignment_problems`: assignment_id, problem_id, points, display_order.
- `submissions`: id, user_id, problem_id, assignment_id nullable, language, source reference/content policy, status, idempotency key, created_at, started_at, finished_at.
- `submission_results`: submission_id, verdict, score, execution_ms, memory_kb, safe_public_feedback, judge_reference, completed_at.
- `drafts`: id, user_id, problem_id, language, source_code or source reference, updated_at, version/conflict metadata.
- `assignment_scores`: id, assignment_id, user_id, score, status, graded_by, timestamps; define uniqueness and regrade history explicitly.

### Grading and audit
- `grading_policies`: id, course_id, version, configuration, created_by, created_at.
- `grading_runs`: id, course_id, assessment scope, policy_version, status, summary, created_by, created_at, approved_by, approved_at.
- `grade_recommendations`: grading_run_id, user_id, calculated metrics, recommended_grade, explanation.
- `final_grades`: course_id, user_id, grade, source_run_id nullable, approved_by, approved_at, updated_at.
- `grade_audit_events`: actor_id, action, target, old/new values or safe diff, reason, created_at.
- Never overwrite final grades silently. Keep auditable history.

### RAG
- `knowledge_documents`: id, course_id, file_asset_id or source, title, ingestion_status, visibility, uploaded_by, checksum, timestamps.
- `document_chunks`: id, document_id, chunk_index, text, source locator/page, token count, metadata, embedding vector (pgvector) and relevant indexes.
- `rag_feedback`: id, user_id, document/course scope, feedback rating, optional safe note, created_at.
- Check course membership and document visibility before retrieving chunks. Treat content as untrusted.

### Reports and operations
- `report_jobs`: id, scope/course, period start/end, status, requested_by, attempts, next_retry_at, error_code, timestamps.
- `student_reports`: id, job_id, student_id, storage_key, checksum, generated_at, visibility.
- `report_deliveries`: id, report_id/job_id, channel, recipient reference, status, attempts, sent_at, safe error metadata.
- `audit_events`: actor, action, resource, request_id, timestamp, safe metadata.
- Add indexes based on actual query patterns and inspect query plans.

## 3. Relationship summary

- A user belongs to a course through `course_memberships`.
- A course owns announcements, resources, assignments, and course-scoped documents.
- An assignment links to problems through `assignment_problems`.
- A user creates submissions; submissions link to a problem and optionally an assignment.
- A submission has zero or more result events, but the current normalized result must be unambiguous.
- Grading runs produce recommendations; an authorized professor approves final grades.
- Report jobs produce private report artifacts and delivery records.
- RAG chunks belong to a document and inherit its access policy.

## 4. Constraints and data integrity

- Use primary keys and unique constraints for identity and natural uniqueness.
- Normalize email consistently before unique comparison.
- Use foreign keys for relational integrity and define delete behavior deliberately.
- Use transactions for multi-row state changes that must be atomic.
- Define enum/status transitions centrally and test invalid transitions.
- Use UTC timestamps and document timestamp semantics.
- Add indexes for common lookups, foreign keys, filtering, sorting, and pagination; validate with query plans rather than adding every imaginable index.
- Keep hidden test data separate from public problem examples and restrict database roles.
- Avoid storing duplicate totals that can drift unless there is an explicit cache/recalculation strategy.
- Add retention/deletion policy for source code, chat messages, documents, reports, and logs.

## 5. Cassandra chat model — only after decision approval

Target query: retrieve newest messages for a room in a bounded time bucket and page backward/forward predictably.

Candidate table shape:
- Partition key: `(room_id, bucket_date)` or another validated bounded bucket.
- Clustering key: time-sortable message ID.
- Fields: sender ID, content, server-created timestamp, client message ID, optional reply/thread reference, safe metadata.
- Query model: room + bucket + cursor; no arbitrary full-table search.

Before implementation, document:
- expected messages per room per time bucket;
- maximum message size and retention;
- partition sizing assumptions;
- ordering semantics and timestamp source;
- retry/idempotency/deduplication strategy;
- consistency level and replication topology;
- history pagination across bucket boundaries;
- backup/restore, schema migration, and operational owner.

Do not assume a relational foreign key from Cassandra to PostgreSQL. Validate sender/room permissions in the backend against authoritative membership data.

## 6. Redis and object storage

Redis keys require documented purpose, TTL, memory policy, and behavior if Redis is unavailable. Queue jobs must be durable enough for the chosen workload and retries must be idempotent.

Object storage keys must be opaque/non-guessable and access-controlled. Keep checksums, size, content type, owner/course, and visibility metadata in PostgreSQL. Never store private report artifacts in public buckets.

## 7. Migration policy

- All PostgreSQL schema changes use Alembic migrations.
- Do not edit migrations already applied to shared/staging/production databases; add a new migration.
- Review destructive changes and use expand/contract rollout for incompatible changes.
- Test migrations on a clean database and test upgrade paths from the previous schema.
- Cassandra schema changes (if used) require a documented deployment sequence compatible with running code.
- Backups are not verified until a restore test succeeds.
- Never run destructive migrations against production without explicit approval and a tested recovery plan.

## 8. Schema acceptance criteria

- Every implemented table/column/index exists in a migration.
- Constraints and ownership rules are covered by tests.
- API responses do not leak hidden tests or unauthorized document/report data.
- Seed data is synthetic and clearly separated from production.
- Documentation is updated in the same PR as schema changes.
