# 🗺️ Remediation Plan — Borla Tracker WMS

Based on the detailed analysis, here is a prioritized, phased remediation plan. Each phase includes actionable tasks, estimated effort, and responsible team roles.

---

## 🚨 Phase 0: Immediate Patching (Next 24–48 Hours)

**Goal:** Stop current production failures and data corruption risks.

| # | Task | Effort | Action |
|---|---|---|---|
| 1 | **Fix `LiveCollectorsView` runtime crash** | 30 min | Replace `last_known_location` with `last_known_latitude`/`last_known_longitude` in `collector/views.py:489` and related queryset. |
| 2 | **Fix Go bridge race condition – targeted updates** | 2 hours | Change `UPDATE collector_collector SET is_online=false` to `UPDATE ... WHERE user_id = ANY(online_ids)` or use per-collector updates. |
| 3 | **Fix spatial distance – use geography** | 1 hour | Change `Point.distance()` to `ST_Distance(geography(loc1), geography(loc2))` in `routes/models.py:93` and `routes/services.py:131`. |
| 4 | **Fix `AutoAssignmentService` accepting null-location collectors** | 30 min | Add `collector.last_known_latitude IS NOT NULL AND collector.last_known_longitude IS NOT NULL` filter. |
| 5 | **Fix `auto_generate_stops` – reference ClientAddress** | 1 hour | Change `client.location` to `client.primary_address.location` (or iterate over all addresss). |
| 6 | **Fix pricing calculator waste type keys** | 30 min | Align `WASTE_TYPE_MULTIPLIERS` keys with actual choices in `on_demand/models.py` waste types. |
| 7 | **Fix `mark_completed`/`mark_failed` keyword arg** | 15 min | Ensure consistency: remove `provider_response` from calls or add it to model method signature. |

**Who:** Backend team lead  
**Verification:** Run all 346 existing tests + manual curl of `GET /api/collectors/live/`, `POST /api/on-demand/requests/dispatch/`, and OSRM route optimization.

---

## 🔐 Phase 1: Security Hardening (Week 1–2)

**Goal:** Eliminate critical security gaps & protect user data.

| # | Task | Effort | Details |
|---|---|---|---|
| 1 | **Encrypt `PaymentMethod.account_number`** | 4 hours | Use `django-cryptography` or `fernet` fields in Python. Mark existing data as encrypted via migration. |
| 2 | **Add rate limiting on password reset** | 1 hour | Apply `throttle_classes = [AnonRateThrottle]` or a custom `ResetRateThrottle` (e.g., 5/hour per IP/email). |
| 3 | **Move reset token out of URL** | 2 hours | Return token in response body and have frontend POST it; avoid query string exposure. |
| 4 | **Implement 2FA for admin users** | 2 days | Add `django-otp` or `pyotp` + QR code provisioning; enforce on `/admin/` and `/api/admin/*`. |
| 5 | **Audit `.env.example` and `accounts_credentials.json`** | 1 hour | Remove or encrypt `accounts_credentials.json`. Add all required env vars to `.env.example`. |

**Who:** Security specialist  
**Verification:** OWASP ZAP scan, penetration test of auth endpoints, static analysis (e.g., `bandit`).

---

## 🧪 Phase 2: Testing & Coverage (Week 2–3)

**Goal:** Achieve >85% coverage across all apps, especially notifications and services.

| # | Task | Effort | Details |
|---|---|---|---|
| 1 | **Write full tests for `notifications` app** | 2 days | Cover Notification model, preferences, services (SMS/Email/Push), tasks, and API views. |
| 2 | **Add tests for `collector/tasks.py`** | 1 day | `calculate_daily_incentives`, `validate_gps`, `update_collector_rating`, `update_collector_stats`. |
| 3 | **Add tests for `routes/services.py`** | 1 day | `RouteOptimizationService`, `OSRMService`, `auto_generate_stops`. |
| 4 | **Add integration test for Live Tracking flow** | 1 day | Mock Go WebSocket or test via Redis → Django bridge → API response. |
| 5 | **Add tests for `notifications/services.py`** | 1 day | Each provider path (console, africa's talking, twilio) with monkey-patched calls. |
| 6 | **Add tests for `payment/services.py` edge cases** | 1 day | Missing keys, network timeouts, duplicate webhooks. |

**Who:** QA engineer + backend devs  
**Verification:** `coverage run --source='.' manage.py test; coverage report` ≥85%. Run `make test` green.

---

## 🧹 Phase 3: Code Quality & Refactoring (Week 3–4)

**Goal:** Eliminate technical debt, improve maintainability.

| # | Task | Effort | Details |
|---|---|---|---|
| 1 | **Deduplicate waste-type color maps** | 1 hour | Extract `WASTE_COLORS` to `borla-frontend/src/constants.ts`. |
| 2 | **Refactor singleton pricing calculator** | 1 hour | Use class-based instantiation or factory for test isolation. |
| 3 | **Configurable freshness cutoff** | 30 min | Add `DISPATCH_FRESHNESS_MINUTES` to settings, use in `on_demand/services.py`. |
| 4 | **Route frontend OSRM calls through backend proxy** | 3 hours | Add endpoint like `/api/osrm/proxy/` that caches results and enforces rate limits. |
| 5 | **Fix float/Decimal consistency** | 1 hour | Change `zone_fee_multiplier` default from `1.0` to `Decimal('1.0')`. |
| 6 | **Remove duplicate URL aliases** | 1 hour | Decide canonical URLs (e.g., keep `/api/collectors/` only) and redirect or consolidate. |

**Who:** All developers  
**Verification:** Code review, linter (flake8, prettier) pass, no regression.

---

## 🚀 Phase 4: Infrastructure & CI/CD (Month 2)

**Goal:** Enable reliable, automatable deployments.

| # | Task | Effort | Details |
|---|---|---|---|
| 1 | **Set up GitHub Actions CI** | 1 day | Lint → test → coverage → build (Django static, Go binary, React dist). |
| 2 | **Add database migration checks** | 1 hour | `python manage.py makemigrations --check` in CI. |
| 3 | **Containerize all services** | 3 days | Docker Compose for local dev, production Dockerfiles. |
| 4 | **Add PgBouncer connection pooling** | 1 day | Docker sidecar, configure Django to use `127.0.0.1:6432`. |
| 5 | **Implement separate JWT keys for Go service** | 2 hours | Use asymmetric keys (RS256) or separate HS256 keys with a trust endpoint. |
| 6 | **Add health checks & monitoring** | 1 day | Prometheus metrics endpoints for Django, Celery, Go service. |

**Who:** DevOps engineer  
**Verification:** `docker compose up` runs all services; CI pipeline green; deployment to staging.

---

## 📊 Timeline & Ownership

| Phase | Timeline | Owner | Key Deliverable |
|---|---|---|---|
| Phase 0 (Critical) | Day 1–2 | Backend lead | 7 bugfixes deployed |
| Phase 1 (Security) | Week 1–2 | Security specialist | Encryption + rate limiting + 2FA |
| Phase 2 (Testing) | Week 2–3 | QA + devs | 200+ new tests, 90% coverage |
| Phase 3 (Quality) | Week 3–4 | All devs | Cleaned codebase, no duplications |
| Phase 4 (Infra) | Month 2 | DevOps | CI/CD, Docker, monitoring |

---

## ✅ Success Criteria

- **All 7 critical bugs** closed and verified in production.
- **346 existing tests continue passing** (no regressions).
- **Notification app test coverage** >85%.
- **Security scan** passes with no high/critical findings.
- **`/api/collectors/live/`** returns correct data.
- **Distance calculations** within ±5% of real-world measurements.
- **CI pipeline** green on every push.

---

## 📝 Notes

- Every bugfix must include a **regression test**.
- All secrets (API keys, DB passwords) must be rotated after encryption implementation.
- Consider a **feature flag** for the live tracking fix to allow gradual rollout.
- The `accounts_credentials.json` file should be **gitignored and removed** immediately.

This plan is aggressive but realistic given the project's current maturity. Let's start with Phase 0 — I can draft the pull request for each fix if needed.
