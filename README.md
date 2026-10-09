# Agentic URL Shortener — Engineering Assignment Submission

## Executive summary
This prototype converts a high-level software requirement into a reviewable engineering outcome through a **stateful, governed orchestration DAG**. It includes a URL shortener API, analytics, three required execution scenarios, explicit dependency sequencing, approval gates, bounded retry, rollback, safe-stop behavior, audit lineage, and reliability metrics.

The implementation intentionally keeps the runtime dependency footprint small so a reviewer can run it locally without cloud credentials or an external LLM. AI-agent behavior is represented by explicit stage agents and a governance/orchestration layer; an LLM adapter can be introduced behind the same stage boundaries without changing the control plane.

## Requirements mapped to implementation
| Assignment requirement | Implementation |
|---|---|
| Requirement understanding | `requirements` / `clarify` stages normalize requirements and ambiguity |
| Task decomposition | Explicit `Stage(name, deps, action)` DAG |
| Brownfield reasoning | `codebase_reasoning` identifies impacted modules |
| Agentic orchestration | `AgenticOrchestrator` owns lifecycle execution and state |
| Sequential + parallel-ready DAG | Dependency graph determines readiness; independent stages form a ready batch |
| Entry/exit gates | Policy evaluated before every stage; completion emitted after successful execution |
| Cross-stage context | `WorkflowRun.context` persists decisions and outputs |
| Decision lineage | Audit events + run decisions |
| Human approval | High-impact release stage safe-stops until `human_approved=true` |
| Retry / fallback / rollback | Bounded retries and context snapshot restoration on failure |
| Safe stop | Approval gate and policy denial stop execution without release |
| Security/compliance/change control | `PolicyGuard` checks security findings, test failures, and approval |
| Observability | Audit endpoint + run status + reliability metrics |
| Dynamic re-plan | `replan()` updates context and records a re-plan decision |
| Engineering outputs | REST API, tests, documentation, setup and scenario scripts |

## Architecture
```text
                         +----------------------+
                         | Requirement / User   |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | Agentic Orchestrator  |
                         | - stateful DAG        |
                         | - gates               |
                         | - retries             |
                         | - rollback            |
                         | - safe-stop           |
                         | - re-plan             |
                         +---+---------+----------+
                             |         |
                   +---------+         +----------+
                   v                              v
          +------------------+          +------------------+
          | Policy / Human   |          | Audit + Metrics  |
          | approval gates   |          | traceability     |
          +------------------+          +------------------+
                             |
                             v
                    +------------------+
                    | Engineering DAG  |
                    | Req -> Design ->  |
                    | Build -> Test ->  |
                    | Docs -> Release   |
                    +---------+---------+
                              |
                              v
                    +------------------+
                    | URL Shortener    |
                    | API + Store      |
                    +------------------+
```

## API
- `POST /api/v1/urls` — create a short URL
- `GET /r/{code}` — resolve and count a click
- `GET /api/v1/urls/{code}/analytics` — analytics
- `POST /api/v1/workflows/{greenfield|brownfield|ambiguous}?approved=true|false` — execute scenario
- `GET /api/v1/workflows/{run_id}` — workflow state
- `GET /api/v1/audit` — audit-grade event stream
- `GET /api/v1/metrics` — success/retry/rollback/safe-stop/latency metrics
- `GET /health` — health check

## Run locally
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Open `http://127.0.0.1:8000/docs` for Swagger/OpenAPI.

Run tests:
```bash
pytest -q
```

Run the scenario demonstration:
```bash
PYTHONPATH=. python scripts/demo.py
```

## Example workflow behavior
A greenfield run without approval intentionally ends in `WAITING_FOR_APPROVAL` at the release gate. Re-running with approval permits release. This demonstrates that agents are autonomous inside bounded permissions while humans retain ownership of high-impact changes.

## Three required scenarios
### 1. Greenfield
Requirement: build the URL shortener from scratch.
- Normalize requirements
- Design architecture
- Generate implementation
- Test
- Document
- Gate release behind human approval

### 2. Brownfield
Requirement: enhance an existing shortener with analytics retention without breaking redirect behavior.
- Identify impacted modules
- Design backward-compatible change
- Implement
- Regression test
- Gate release

### 3. Ambiguous
Requirement: “Add custom aliases and analytics.”
The workflow explicitly surfaces ambiguity around alias collision policy, retention, and abuse controls before implementation. It records safe assumptions and validates risk before proceeding.

## Failure handling
- **Transient failure:** retry up to the stage's bounded retry limit.
- **Persistent failure:** restore the stage's context snapshot and mark the run failed.
- **High-impact action:** safe-stop for human approval.
- **Policy violation:** deny the stage and preserve the audit trail.
- **Upstream change:** call `replan()` to update context and record lineage; downstream gates are re-evaluated on subsequent execution.

## Production hardening roadmap
For production deployment, replace the in-memory store with PostgreSQL/Redis, persist workflow state in a durable state store, add Kafka/OpenTelemetry integration, introduce an authenticated human-approval service, add rate limiting/abuse detection, implement URL reputation checks, add distributed locks/idempotency keys, and connect an LLM provider through a sandboxed tool adapter. These are deliberate boundaries rather than hidden dependencies in the prototype.

## Design trade-offs
1. **Deterministic prototype over mandatory external LLM:** repeatable evaluation and no credentials required; an LLM adapter can be plugged into stage actions.
2. **In-memory persistence:** simple local demonstration; production requires durable storage.
3. **Deterministic batch execution:** makes audit ordering reproducible; the dependency model is ready for parallel worker execution.
4. **Context snapshot rollback:** demonstrates rollback semantics without pretending to provide transactional infrastructure across arbitrary external systems.

## Submission artifacts
- `app/` — API, URL domain logic, governance and orchestration
- `tests/` — unit tests for URL behavior and orchestration controls
- `docs/ARCHITECTURE.md` — architecture and control-flow explanation
- `docs/ENGINEERING_SUMMARY.md` — rationale, assumptions, risks and validation
- `docs/SCENARIOS.md` — greenfield/brownfield/ambiguous walkthroughs
- `scripts/demo.py` — reproducible scenario execution
