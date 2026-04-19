# Implementation Plan: Hybrid AI Agent Platform

## Project Overview

**Project Name:** Hybrid AI Agent Platform (HAAP)
**Objective:** Build a production-grade AI-driven operations platform where the company owns the intelligence layer (Agent Brain, Prompt Engine, Rules Engine, Workflow Engine, Approvals, UI) and AWS provides managed infrastructure (Bedrock, Lambda, RDS, DynamoDB, Secrets Manager, CloudWatch).
**Scope:** Full-stack platform — React SPA frontend, Django DRF backend, AI agent orchestration, tool execution via Lambda, audit logging, approval workflows, RBAC, and observability.
**Delivery Phases:** Phase 1 MVP (Weeks 1–8) → Phase 2 Core Platform (Weeks 9–16) → Phase 3 Advanced Features (Weeks 17–24)
**Success Definition:** 100+ concurrent sessions supported; chat-to-tool-execution in under 5s; all production actions gated by approval; full audit trail; zero secrets in code.

---

## Tasks

- [ ] 1. Project Setup and Repository Structure
  - Initialize monorepo with `backend/` (Django) and `frontend/` (React) directories
  - Create `docker-compose.yml` for local dev: PostgreSQL, Redis, LocalStack (Lambda/DynamoDB/S3/Secrets)
  - Add `.env.example` with all required environment variable names (no values)
  - Configure `pre-commit` hooks: black, isort, flake8, mypy for backend; eslint, prettier for frontend
  - Add `Makefile` with targets: `make dev`, `make test`, `make migrate`, `make lint`
  - _Requirements: 16.7, 16.9_

  - [ ]* 1.1 Write smoke test confirming docker-compose stack starts and all services are healthy
    - Validate PostgreSQL, Redis, LocalStack containers respond to health checks
    - _Requirements: 16.3_

- [ ] 2. Database Schema and Migrations
  - Create Django models: User, Role, Permission, Project, ProjectRole, Environment, Config, ConfigVersion, Session, WorkflowRun, WorkflowStepRun, ApprovalRequest, ApprovalDecision, Rule, PromptTemplate
  - Write initial migration files
  - Add database indexes: user email (unique), session user_id+status, workflow_run session_id, approval_request workflow_run_id+status, rule project_id+priority+enabled
  - Create seed data management command: built-in roles (Admin, DevOps_Engineer, Developer, Support_Team, Approver, Auditor) with permission sets
  - _Requirements: 2.1, 9.1, 7.9_

  - [ ]* 2.1 Write property test for data model round-trip persistence
    - **Property 14: WorkflowRun persistence round-trip**
    - **Validates: Requirements 7.9**

- [ ] 3. DynamoDB Table Setup
  - Create DynamoDB tables via boto3/LocalStack: AuditLog, Message, ToolExecution
  - Define partition keys, sort keys, GSIs as per design Section 11.2
  - Create `aws_clients.py` boto3 factory with retry config and region injection
  - Write DynamoDB base repository class with `put_item`, `query`, `scan_with_filter` methods
  - _Requirements: 15.4, 11.5_

- [ ] 4. Authentication Service
  - Implement custom Django User model extending AbstractBaseUser: email, mfa_enabled, is_locked, failed_login_count, locked_until
  - Implement JWT auth views: `/api/v1/auth/login`, `/api/v1/auth/refresh`, `/api/v1/auth/logout`, `/api/v1/auth/me`
  - Use RS256 signing; private key loaded from Secrets Manager at startup
  - Implement account lockout: 5 failed attempts within 10 minutes locks account, emits CloudWatch alarm
  - Implement refresh token rotation: store hashed refresh token in Redis with 7-day TTL; invalidate on use
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6_

  - [ ]* 4.1 Write property test for authentication round-trip
    - **Property 1: Authentication round-trip validity**
    - **Validates: Requirements 1.1**

  - [ ]* 4.2 Write property test for invalid credential rejection
    - **Property 2: Invalid credentials are always rejected**
    - **Validates: Requirements 1.2**

  - [ ]* 4.3 Write unit test for account lockout edge case
    - Test: 5 consecutive failures within 10 min locks account; 6th attempt returns locked error
    - _Requirements: 1.6_

- [ ] 5. RBAC System
  - Implement Role and Permission models with many-to-many through ProjectRole
  - Implement `HasPermission` DRF permission class: checks user's project-scoped role permissions
  - Implement `RoleGuard` utility: given user + action + project returns bool
  - Apply `HasPermission` to every API view with the required permission name
  - Implement Redis-cached permission lookup with 1-minute TTL; invalidate on role change
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

  - [ ]* 5.1 Write property test for RBAC enforcement
    - **Property 3: RBAC permission enforcement**
    - **Validates: Requirements 2.2, 2.3**

- [ ] 6. Middleware Stack
  - Implement `RequestIDMiddleware`: generate UUID, attach to request and response as `X-Request-ID`
  - Implement `RateLimitMiddleware`: Redis sliding window, 60 req/min per user; return 429 with `Retry-After` header
  - Implement `AuditContextMiddleware`: capture actor ID, IP, request ID in thread-local for audit writes
  - Implement `SecurityHeadersMiddleware`: HSTS, CSP, X-Frame-Options, X-Content-Type-Options
  - Register middleware in correct order in Django settings
  - _Requirements: 16.8, 17.1, 17.7_

- [ ] 7. Audit Layer
  - Implement `AuditLayer` service with `record(event_type, actor, resource, action, outcome, metadata)` method
  - Write to DynamoDB AuditLog table with TTL set to 90 days
  - Implement `AuditLogQueryView`: filter by date_from, date_to, event_type, actor_id; cursor-paginated
  - Implement `AuditLogExportView`: stream JSON/CSV export for filtered results
  - Ensure no DELETE or UPDATE operations are exposed on AuditLog table
  - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5, 11.6, 11.7_

  - [ ]* 7.1 Write property test for audit log immutability
    - **Property 20: Audit log immutability invariant**
    - **Validates: Requirements 11.3**

- [ ] 8. Config Manager
  - Implement Config and ConfigVersion models
  - Implement `ConfigManager` service: `get(project_id, env_id, key)` resolves secret-reference via Secrets Manager at runtime
  - Implement Config CRUD API views with audit logging on every write
  - Implement `ConfigVersionView`: return paginated version history for a key
  - Ensure secret-reference values stored as `secret:arn:...` strings, never the resolved value
  - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8_

  - [ ]* 8.1 Write property test for secret reference storage invariant
    - **Property 16: Secret reference storage invariant**
    - **Validates: Requirements 9.4**

  - [ ]* 8.2 Write property test for config version history growth
    - **Property 17: Config version history growth invariant**
    - **Validates: Requirements 9.5**

- [ ] 9. Checkpoint — Core Infrastructure
  - Ensure all tests pass, ask the user if questions arise.
  - Verify: auth endpoints return correct tokens, RBAC denies unauthorized requests, audit writes appear in DynamoDB, config CRUD works with secret references

- [ ] 10. Prompt Engine
  - Implement PromptTemplate model with versioning (template_id, version, is_active)
  - Implement `PromptEngine` service: `render(template_id, variables)` using Jinja2 with strict undefined mode
  - Implement token counting using `tiktoken`; enforce 8,000 token hard limit; raise `PromptTokenLimitError` if exceeded
  - Implement Redis caching of active templates with 5-minute TTL
  - Implement Bedrock client wrapper: `invoke_model(prompt, model_id, max_tokens, temperature)` with retry on throttling
  - Create initial prompt templates: `intent_classification_v1`, `plan_builder_v1`, `summarization_v1`
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 15.1_

  - [ ]* 10.1 Write property test for prompt token budget invariant
    - **Property 10: Prompt token budget invariant**
    - **Validates: Requirements 5.3**

- [ ] 11. Memory Service
  - Implement `MemoryService`: `add_message(session_id, role, content)` writes to DynamoDB Message table
  - Implement `get_context(session_id, limit=50)` queries last N messages ordered by timestamp
  - Implement session lifecycle: `close_session(session_id)` sets status=closed; closed sessions excluded from context
  - Implement Celery task `close_inactive_sessions`: runs hourly, closes sessions inactive more than 24h
  - _Requirements: 13.1, 13.2, 13.3, 13.6_

  - [ ]* 11.1 Write property test for session context window invariant
    - **Property 5: Session context window invariant**
    - **Validates: Requirements 3.8**

- [ ] 12. Rules Engine
  - Implement Rule model: rule_id, name, type, priority, enabled, conditions (JSON), effect, approval_config, reason
  - Implement `RulesEngine` service: `evaluate(plan_step, user, environment)` returns ALLOW / DENY / APPROVAL_REQUIRED
  - Load active rules ordered by priority ASC; short-circuit on first DENY match
  - Apply default-deny for production, default-allow for non-production when no rules match
  - Implement Rule CRUD API (Admin only) with audit logging; changes take effect on next evaluation without restart
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8_

  - [ ]* 12.1 Write property test for rules evaluation ordering
    - **Property 11: Rules evaluated before tool execution ordering invariant**
    - **Validates: Requirements 6.1**

  - [ ]* 12.2 Write property test for rules short-circuit on DENY
    - **Property 12: Rules short-circuit on first DENY**
    - **Validates: Requirements 6.7**

- [ ] 13. Tool Execution Layer
  - Implement `TOOL_REGISTRY` with all 6 built-in tools: Deploy_ECS_Service, Fetch_CloudWatch_Logs, Scale_ECS_Service, Get_Cost_Analysis, Health_Check_Service, Describe_Resource
  - Implement `ToolExecutionLayer` service: `invoke_tool(tool_name, inputs, run_id)` with schema validation, IAM role assumption, Lambda invocation, timeout enforcement, structured result return
  - Implement JSON Schema validation using `jsonschema`; raise `ToolInputValidationError` before any Lambda call
  - Implement per-tool IAM role assumption via `sts:AssumeRole`
  - Implement `ToolResult` dataclass: status, output, error_message, execution_duration_ms, tool_name
  - Write audit record for every tool execution (success, failure, timeout)
  - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7, 10.8, 10.9_

  - [ ]* 13.1 Write property test for tool input validation pre-condition
    - **Property 18: Tool input validation pre-condition invariant**
    - **Validates: Requirements 10.2**

  - [ ]* 13.2 Write property test for tool result structural completeness
    - **Property 19: Tool result structural completeness invariant**
    - **Validates: Requirements 10.7**

  - [ ]* 13.3 Write property test for tool retry counting
    - **Property 8: Tool retry counting invariant**
    - **Validates: Requirements 4.7, 7.4**

- [ ] 14. Lambda Tool Functions
  - Create Lambda `haap-tool-deploy-ecs`: accepts service_name, image_tag, environment; calls ECS UpdateService; returns deployment_id, status, service_arn
  - Create Lambda `haap-tool-fetch-logs`: accepts log_group, start_time, end_time, filter_pattern; calls CloudWatch Logs FilterLogEvents; returns log_events array
  - Create Lambda `haap-tool-health-check`: accepts service_name, environment; calls ECS DescribeServices + ALB DescribeTargetHealth; returns health_status, healthy_count, unhealthy_count
  - Each Lambda has its own IAM execution role with least-privilege permissions
  - Each Lambda writes structured logs to its CloudWatch Log Group
  - _Requirements: 10.8, 15.2, 15.8_

- [ ] 15. Agent Brain
  - Implement `AgentBrain` service: context building → intent classification → plan building → rules evaluation → execution
  - Implement `classify_intent(message, context)`: calls Prompt Engine with `intent_classification_v1`; parses response into IntentResult
  - Implement `build_plan(intent, message, context)`: calls Prompt Engine with `plan_builder_v1`; parses response into ExecutionPlan with steps
  - Implement guardrail checks: tool allowlist validation, max 10 tool calls per message, destructive action detection
  - Implement retry logic: up to 2 retries with exponential backoff (5s, 10s) on tool failure
  - Implement `generate_summary(run_result)`: calls Prompt Engine with `summarization_v1`; returns structured summary
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 14.1, 14.3, 14.4, 14.5, 14.6_

  - [ ]* 15.1 Write property test for intent classification output domain
    - **Property 6: Intent classification output domain invariant**
    - **Validates: Requirements 4.1**

  - [ ]* 15.2 Write property test for execution plan structural invariant
    - **Property 7: Execution plan structural invariant**
    - **Validates: Requirements 4.3, 14.1**

  - [ ]* 15.3 Write property test for destructive action safety invariant
    - **Property 9: Destructive action safety invariant**
    - **Validates: Requirements 4.8**

  - [ ]* 15.4 Write property test for tool invocation count limit
    - **Property 21: Tool invocation count limit invariant**
    - **Validates: Requirements 14.6**

- [ ] 16. Checkpoint — AI and Rules Layer
  - Ensure all tests pass, ask the user if questions arise.
  - Verify: intent classification returns valid categories, plans contain valid tool names, rules engine blocks production actions, tool execution validates inputs before Lambda call

- [ ] 17. Workflow Engine
  - Implement WorkflowRun and WorkflowStepRun models with full state machine (PENDING → RUNNING → COMPLETED/FAILED/ROLLED_BACK/TIMED_OUT/CANCELLED)
  - Implement `WorkflowEngine` service: `create_run`, `execute_run`, `rollback_run`
  - Implement sequential step execution: execute steps in dependency order, persist state after each step
  - Implement retry policy per step: read `retry.max_attempts` and `retry.backoff_seconds` from step definition
  - Implement rollback: on step failure with `on_failure=rollback`, execute rollback steps in reverse order
  - Implement 30-minute timeout: Celery task monitors in-progress runs; triggers rollback on timeout
  - Emit SSE events for each state transition: step_started, step_completed, step_failed, run_completed, run_rolled_back
  - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 7.9_

  - [ ]* 17.1 Write property test for rollback step ordering
    - **Property 13: Rollback step ordering invariant**
    - **Validates: Requirements 7.5**

  - [ ]* 17.2 Write property test for WorkflowRun persistence round-trip
    - **Property 14: WorkflowRun persistence round-trip**
    - **Validates: Requirements 7.9**

- [ ] 18. Approval System
  - Implement ApprovalRequest and ApprovalDecision models
  - Implement `ApprovalSystem` service: `create_request`, `approve(request_id, approver_id, comment)`, `reject(request_id, approver_id, comment)`
  - Enforce self-approval prevention: compare approver_id to initiator_id; raise `SelfApprovalError` if equal
  - Implement multi-approval: increment approval_count; resume workflow only when approval_count >= required_approvals
  - Implement Celery task `expire_approval_requests`: runs every 5 minutes; transitions PENDING requests past expires_at to EXPIRED; cancels WorkflowRun
  - Implement approval notification Celery task: SSE push + email on ApprovalRequest creation
  - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8_

  - [ ]* 18.1 Write property test for self-approval prevention
    - **Property 4: Self-approval prevention invariant**
    - **Validates: Requirements 2.7, 8.4**

  - [ ]* 18.2 Write property test for approval expiry state transition
    - **Property 15: ApprovalRequest expiry state transition**
    - **Validates: Requirements 8.5**

- [ ] 19. Chat Controller and SSE Streaming
  - Implement Session and Message models (RDS for session metadata; DynamoDB for message content)
  - Implement Chat API views: create session, send message, confirm plan, delete session
  - Implement SSE streaming endpoint `GET /api/v1/chat/sessions/{id}/stream`: Django StreamingHttpResponse with `text/event-stream`; replay missed events using `Last-Event-ID`
  - Wire Chat Controller → Agent Brain → Workflow Engine → SSE event emission
  - Implement plan confirmation flow: store pending plan in Redis with 10-minute TTL; confirm endpoint retrieves and executes
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7_

- [ ] 20. Checkpoint — Backend Complete
  - Ensure all tests pass, ask the user if questions arise.
  - Verify end-to-end: send chat message → intent classified → plan built → rules evaluated → workflow executed → SSE events received → audit log written

- [ ] 21. React Frontend Bootstrap
  - Initialize React app with Vite + TypeScript
  - Install: React Query (TanStack), React Router v6, Axios, Tailwind CSS, Radix UI primitives, Recharts
  - Configure Axios client: base URL, JWT Bearer header injection, 401 interceptor (redirect to login), request ID header
  - Implement `AppShell` layout: sidebar navigation, top nav with user menu, main content area
  - Implement `RoleGuard` component: renders children only if current user has required role
  - Implement theme toggle (light/dark) using Tailwind dark mode class strategy
  - _Requirements: 18.1, 18.2, 18.7_

- [ ] 22. Login Page
  - Implement Login page: email + password form, MFA code field (shown conditionally), submit handler
  - Call `POST /api/v1/auth/login`; store access token in memory; store refresh token in httpOnly cookie
  - Implement token refresh interceptor: on 401, call `/auth/refresh`, retry original request
  - Implement loading state, error banner for invalid credentials, account locked message
  - _Requirements: 1.1, 1.2, 1.7, 18.3, 18.4_

- [ ] 23. Chat Assistant Page
  - Implement `MessageList`: renders user and assistant messages with timestamps, role badges, markdown rendering
  - Implement `InputBar`: textarea with Enter-to-send, Shift+Enter for newline, character count
  - Implement `PlanConfirmModal`: displays structured execution plan steps; Confirm/Cancel buttons
  - Implement `StatusStream`: connects to SSE endpoint; renders real-time step progress with status badges
  - Implement `WorkflowSummary`: renders completion summary with step outcomes
  - Wire to React Query for session management and message history
  - _Requirements: 3.1, 3.4, 3.5, 3.6, 18.3, 18.6_

- [ ] 24. Dashboard Page
  - Implement metrics cards: active sessions, workflows in progress, pending approvals, recent tool executions
  - Implement `WorkflowRunTable`: paginated table of recent runs with status badges and detail links
  - Implement `ToolExecutionChart`: Recharts time-series chart of tool execution counts and error rates over 24h
  - Implement auto-refresh using React Query `refetchInterval: 30000`
  - Restrict to Admin and DevOps_Engineer roles via `RoleGuard`
  - _Requirements: 12.1, 12.2, 12.3, 12.5, 12.6_

- [ ] 25. Approval Center Page
  - Implement `ApprovalCard`: shows workflow name, initiator, plan summary, created time, expiry countdown
  - Implement `ApprovalDetail` modal: full plan snapshot, approve/reject buttons, comment textarea
  - Call approve and reject API endpoints; show SSE-driven real-time updates for new approvals
  - Restrict to Approver and Admin roles
  - _Requirements: 8.1, 8.7, 18.1, 18.2_

- [ ] 26. Config Management Page
  - Implement `ConfigTable`: list configs for selected project+environment with key, value type, last updated
  - Implement `ConfigEditor` modal: create/edit config with value type selector; secret-reference shows ARN input
  - Implement `ConfigVersionHistory` drawer: version history for a key with diff view
  - Restrict create/update/delete to Admin and DevOps_Engineer; read-only for others
  - _Requirements: 9.1, 9.3, 9.5, 9.7_

- [ ] 27. Audit Logs Page
  - Implement `LogTable`: paginated table with columns: timestamp, event type, actor, resource, action, outcome
  - Implement `LogFilters` panel: date range picker, event type multi-select, actor search, outcome filter
  - Implement `LogExportButton`: triggers CSV/JSON export download
  - Restrict to Auditor and Admin roles
  - _Requirements: 11.4, 11.6, 11.7_

- [ ] 28. User Management Page
  - Implement `UserTable`: list users with email, roles, status (active/locked)
  - Implement `RoleAssignModal`: assign/revoke project-scoped roles for a user
  - Implement user deactivation action with confirmation dialog
  - Restrict to Admin role only
  - _Requirements: 2.4, 2.6_

- [ ] 29. Checkpoint — Frontend Complete
  - Ensure all tests pass, ask the user if questions arise.
  - Verify: login flow works end-to-end, chat sends messages and receives SSE events, approval center shows pending requests, role-based navigation hides unauthorized pages

- [ ] 30. Additional Lambda Tool Functions (Phase 2)
  - Create Lambda `haap-tool-scale-ecs`: accepts service_name, environment, desired_count; calls ECS UpdateService; returns new and previous desired_count
  - Create Lambda `haap-tool-describe`: accepts resource_type, resource_id, environment; calls appropriate Describe* API; returns resource metadata JSON
  - Create Lambda `haap-tool-cost-analysis` (Phase 3): accepts start_date, end_date, granularity, group_by; calls Cost Explorer GetCostAndUsage; returns cost_by_service array
  - _Requirements: 10.8, 15.2_

- [ ] 31. Observability Setup
  - Configure `structlog` in Django: JSON output with request_id, user_id, level, message, duration_ms
  - Implement CloudWatch Logs handler: stream Django logs to `/haap/backend/{env}` log group
  - Implement CloudWatch Metrics publisher: emit custom metrics via `boto3.client("cloudwatch").put_metric_data`
  - Create CloudWatch Alarms for: p99 latency > 3s, error rate > 1%, workflow timeout rate > 5%, failed login rate > 10/min
  - Enable AWS X-Ray tracing on Django (via `aws-xray-sdk-python`) and all Lambda functions
  - _Requirements: 15.6, 16.9_

- [ ] 32. Security Hardening
  - Run OWASP ZAP automated scan against staging; fix all High and Critical findings
  - Implement input sanitization: Django serializer validators + `bleach` for any HTML content fields
  - Verify no secrets appear in CloudWatch logs (log scrubbing regex for ARNs, tokens, passwords)
  - Implement password complexity validators: min 12 chars, uppercase, lowercase, digit, special char
  - Verify all API endpoints have `HasPermission` class applied via automated URL pattern enumeration test
  - _Requirements: 16.4, 16.5, 16.10, 17.2, 17.3, 17.8_

- [ ] 33. CI/CD Pipeline
  - Create GitHub Actions `ci.yml`: on PR → lint → type check → unit tests → property tests → Docker build
  - Create `deploy-dev.yml`: on merge to main → build → push to ECR → deploy to ECS dev → smoke tests
  - Create `deploy-staging.yml`: on passing dev smoke tests → deploy to staging → integration tests
  - Create `deploy-prod.yml`: manual trigger with approval gate → blue/green ECS deploy → health check → auto-rollback on failure
  - Add DB migration step in deploy workflows: `migrate --check` before deploy, `migrate` after
  - _Requirements: 16.7_

- [ ] 34. Final Checkpoint — All Tests Pass
  - Ensure all tests pass, ask the user if questions arise.
  - Run full test suite: unit tests, property tests, integration tests, security scan
  - Verify all 21 correctness properties have corresponding passing property-based tests
  - Verify end-to-end workflow scenarios from Requirements 20.1–20.5 pass as integration tests

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Checkpoints (tasks 9, 16, 20, 29, 34) ensure incremental validation before proceeding
- Property tests use `hypothesis` (Python backend) with minimum 100 iterations per property
- Tag format for property tests: `# Feature: hybrid-ai-agent-platform, Property {N}: {title}`
- Lambda functions in tasks 14 and 30 can be developed and tested locally using LocalStack

### Critical Path (blocks everything else)
Tasks 1 → 2 → 3 → 4 → 5 → 6 → 7 → 10 → 12 → 13 → 15 → 17 → 18 → 19

### Quick Wins (show progress fast)
Tasks 1, 4 (login works), 22 (login UI), 23 (chat UI renders), 14 (first Lambda tool deployed)

### MVP Cut Scope (minimum for first demo)
Tasks 1, 2, 3, 4, 5, 6, 7, 10, 12, 13, 14 (Deploy + Fetch Logs + Health Check only), 15, 17 (sequential only), 19, 21, 22, 23, 24