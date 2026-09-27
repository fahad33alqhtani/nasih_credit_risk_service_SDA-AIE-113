# Nasih — Credit-Risk Scoring Service

A credit-risk scoring service for businesses. Given a business's
monthly cash flow and how long it has been trading, it returns a default
probability and one of three decisions: `auto_approve`, `manual_review`,
or `reject`.

Built as the capstone for SDA-AIE-113 (Software Engineering Practices for
AI Systems) at SDAIA Academy: layered architecture, a containerised
image, a three-level test suite, and a CI/CD pipeline.

## Architecture

```
src/nasih_service/
├── domain/     # entities.py, policies.py         pure business rules, no I/O
├── service/    # interfaces.py, scorer.py         use-case orchestration
├── adapters/   # linear_model.py, redis_audit.py  the only files touching model/Redis
├── api/        # schemas.py, app.py, routes.py    FastAPI wire contract
├── config.py                                      typed settings
├── logging_setup.py                               JSON logs, trace-id correlated
└── batch.py                                       offline scoring entrypoint
```

The model sits behind a `Model` Protocol (`service/interfaces.py`), injected via
`app.dependency_overrides` in tests and via the FastAPI `lifespan` in production.
Swapping `LinearModel` for XGBoost or a remote model server touches one adapter
file, not `service/` or `api/`. `AuditStore` follows the same pattern.

Layering is enforced by `import-linter` (`make check-arch`): domain cannot import
service, adapters or api; service cannot import adapters or api.

Training and serving are split at the dependency level. `scikit-learn`, `pandas`
and `numpy` live in the `training` extra and are used by the scripts in
`scripts/`; the service itself reads the fitted coefficients from
`models/credit_model.json` and computes the sigmoid with the standard library,
so none of the training stack is installed in the deployed image.

## Quick start

```bash
pip install -e ".[dev,api,training]"
python scripts/generate_baseline_assets.py   # trains the model, writes data + model
python scripts/generate_golden.py            # records the behavioural golden file
make test
make serve
```

```
$ curl -s localhost:8000/v1/predict -H "content-type: application/json" \
    -d '{"business_id":"BIZ-2026-00042","monthly_cash_flow_sar":20000,"business_age_months":24}'
{"business_id":"BIZ-2026-00042","default_probability":0.650092,"decision":"manual_review","model_version":"v1.0.0","trace_id":"a1b2c3d4e5f60718"}

$ curl -s localhost:8000/v1/decisions/BIZ-2026-00042
{"business_id":"BIZ-2026-00042","default_probability":0.650092,"decision":"manual_review","model_version":"v1.0.0"}
```

## Decision audit trail

Every `/v1/predict` call writes its result to Redis with a 24h TTL through the
`AuditStore` port. `GET /v1/decisions/{business_id}` reads a past decision back,
so a reviewer looking at a case does not have to resubmit the original figures.
It also gives the Redis service in `docker-compose.yml` a real job rather than
only demonstrating health gating.

## Docker

```bash
make build
make image-size
make up             # api + Redis, gated on Redis reporting healthy
docker compose ps
make smoke
make startup-time
```

Measured image size and startup numbers are recorded in `BENCHMARKS.md`.
On Apple Silicon, building for a different target architecture needs
`docker build --platform linux/amd64 .`

## Tests

```bash
make test-fast   # unit + integration, test doubles, no model or Redis I/O
make test-slow   # behavioural tests against the real model and real Redis
make lint        # ruff
make typecheck   # mypy
make check-arch  # import-linter layering contract
```

- **Unit** (`tests/unit/`) — decision bands and the feature definition.
- **Integration** (`tests/integration/`) — the FastAPI contract through
  `TestClient`, with the model and audit store replaced by test doubles via
  `app.dependency_overrides`. Includes a 30-file malformed-payload corpus
  (`payloads/malformed/`, every one must return 4xx) and a test proving a 500
  never leaks the internal exception name.
- **Behavioural** (`tests/behavioural/`, marked `slow`) — against the trained
  model: id-casing invariance, two directional properties (more cash flow and a
  longer history must never raise risk), and a 5,000-row golden file at
  `atol=1e-6`.

Redis-dependent tests skip when nothing is listening locally; CI provides a
Redis service container so they run there.

## Configuration

Settings are read once, typed, in `config.py` with the `NASIH_` prefix:
`NASIH_MODEL_PATH`, `NASIH_REJECT_THRESHOLD`, `NASIH_REDIS_URL`,
`NASIH_AUDIT_TTL_SECONDS`, `NASIH_LOG_LEVEL`. A malformed `NASIH_REDIS_URL`
fails at startup rather than on the first request that needs Redis.

No secrets are committed. `.dockerignore` and `.gitignore` keep `.env` files and
virtualenvs out of the build context and out of git, and CI runs a `gitleaks`
scan on every push.

## CI/CD

`.github/workflows/ci.yml` runs four stages: lint, typecheck and the
architecture contract; tests with a coverage gate and a Redis service container;
image build with a size check and a compose smoke test; then a publish to GHCR
tagged with the commit SHA, on pushes to `main` only.

Branch protection on `main` (required status checks, no force-push) is set in
the repository settings.

## Model

`scripts/generate_baseline_assets.py` trains a `LogisticRegression` on seeded
synthetic data where higher cash flow and a longer trading history both reduce
default risk. The script asserts both fitted coefficients come out negative
before saving, so the behavioural directional tests check a property the model
genuinely has. On the 5,000-row sample the scored book comes out at roughly 75%
`auto_approve`, 16% `manual_review`, 10% `reject`.

The artefact it writes is a readable JSON file, not a pickle:

```json
{
  "version": "v1.0.0",
  "intercept": 10.43827607182093,
  "weights": { "cash_flow_log": -0.9394287840992248, "age_months": -0.02146521249659913 }
}
```
 
`LinearModel` scores from those numbers directly. Scores match the scikit-learn
estimator to within 6e-16, and the golden file pins that.

The data is synthetic and the thresholds are illustrative; this is an
engineering exercise, not a validated credit model.
## conclusion
This project was completed as part of the SDA-AIE-113 — Software Engineering Practices for AI Systems training program at SDAIA Academy, under the supervision of Abdullah Khalid AlShahrani.

The portfolio demonstrates the practical application of software engineering practices for AI systems — building a production-style AI/ML service through clean architecture, a well-defined API contract, containerization, a layered automated testing suite, a CI/CD pipeline with branch protection, and safe configuration, secrets, and logging management.

Official SDAIA Academy GitHub:

https://github.com/SDAIAAcademy
