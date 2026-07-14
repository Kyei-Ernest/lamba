# Borla Master Django Project — Full App Audit

> Scanned: 2026-07-14  
> Command: `python manage.py test` → **FAILED** (65 tests, 7 errors)  
> 7 errors are all the same root cause: `company_username` field removed from Supervisor model but tests still reference it.

---

## Table of Contents
1. [accounts](#1-accounts)
2. [borla_master (project root)](#2-borla_master)
3. [client](#3-client)
4. [collection_management](#4-collection_management)
5. [collector](#5-collector)
6. [notifications](#6-notifications)
7. [on_demand](#7-on_demand)
8. [payment](#8-payment)
9. [routes](#9-routes)
10. [scheduled_request](#10-scheduled_request)
11. [supervisor](#11-supervisor)
12. [wallet](#12-wallet)
13. [waste_management_company](#13-waste_management_company)
14. [zones](#14-zones)
15. [Overall Test Status](#15-overall-test-status)

---

## 1. accounts

**Directory:** `accounts/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `User` (custom auth model extending `AbstractBaseUser`, `PermissionsMixin`), `CustomUserManager` |
| **Serializers** | `serializers.py` — `LoginSerializer`, `TokenRefreshCustomSerializer`, `UserProfileSerializer` |
| **Views** | `views.py` — `LoginView`, `TokenRefreshView`, `LogoutView`, `UserProfileView`, `AdminUserListView`, `SupervisorDashboardStatsView`, `AdminDashboardView`, `AdminCollectorStatsView`, `AdminZoneStatsView`, `AdminTrendsView` |
| **URLs** | `urls.py` — `token/refresh/`, `login/`, `logout/`, `me/`, `geocode/search/`, `geocode/reverse/` |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 3 test classes: `UserRegistrationTests` (2 tests), `AuthenticationFlowTests` (6 tests), `LogoutTests` (1 test) |
| **Other** | `permissions.py`, `services.py` (`AdminStatisticsService`), `tasks.py`, `geocode.py`, `management/commands/seed_all_data.py` |

**Test Result:** All pass ✓

---

## 2. borla_master

**Directory:** `borla_master/` (project root)

| Artifact | Files |
|----------|-------|
| **Models** | None |
| **Serializers** | None |
| **Views** | None (urls.py wires everything together) |
| **URLs** | `urls.py` — Root URLConf including all app routes + Swagger/ReDoc + admin endpoints |
| **Templates** | None |
| **Static Files** | None (Swagger UI in `drf-yasg` auto-generates) |
| **Tests** | None |
| **Other** | `settings.py`, `production.py`, `celery.py`, `asgi.py`, `wsgi.py`, `health.py` (health check endpoint) |

**Installed Apps** (all 13 custom apps): accounts, client, waste_management_company, supervisor, collector, zones, routes, collection_management, on_demand, scheduled_request, wallet, payment, notifications

**Test Result:** N/A (no tests) ✓

---

## 3. client

**Directory:** `client/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `Client` (OneToOne to User), `ClientAddress` (with PostGIS PointField) |
| **Serializers** | `serializers.py` — `UserSerializer`, `ClientSerializer`, `ClientCreateSerializer`, `ClientListSerializer`, `ClientUpdateSerializer` |
| **Views** | `views.py` — `ClientCreateView`, `ClientListView`, `ClientProfileView`, `ClientStatisticsView`, `ClientListCreateView` |
| **URLs** | `urls.py` — `""` (list+create), `list/`, `register/`, `profile/` |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 2 test classes: `ClientWorkflowTests` (5 tests), `OnDemandRequestTests` (6 tests) |

**Test Result:** All pass ✓

---

## 4. collection_management

**Directory:** `collection_management/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `CollectionRecord` (immutable audit log with payment, GPS, photos, segregation score) |
| **Serializers** | `serializers.py` — `CollectionRecordSerializer`, `CollectionRecordCreateSerializer`, `CollectionRecordSummarySerializer` |
| **Views** | `views.py` — `CollectionRecordViewSet` (ReadOnlyModelViewSet with `my_summary`, `my_records`, `update_record` actions) |
| **URLs** | `urls.py` — DefaultRouter at root |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 2 test classes: `CollectionRecordTests` (4 tests), `CollectionRecordModelTests` (4 tests) |

**Test Result:** All pass ✓

---

## 5. collector

**Directory:** `collector/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `Collector` (3 types: individual/business/company_driver, PostGIS service_area, live location), `CollectorRating` |
| **Serializers** | `serializers.py` — `CollectorSerializer`, `CollectorCreateSerializer`, `CollectorListSerializer`, `CollectorUpdateSerializer` |
| **Views** | `views.py` — `CollectorCreateView`, `CollectorListView`, `CollectorProfileView`, `CollectorsByCompanyView`, `CollectorsBySupervisorView`, `PrivateCollectorsListView`, `CollectorsByZoneView`, `CollectorApprovalView`, `CollectorListCreateView`, `LiveCollectorsView` |
| **URLs** | `urls.py` — `register/`, `""` (list+create), `private/`, `company/<id>/`, `supervisor/<id>/`, `zone/`, `me/`, `<id>/approval/`, `live/` |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 3 test classes: `CollectorRegistrationTests` (2 tests), `CollectorWorkflowTests` (4 tests), `IncentivePenaltyTests` (6 tests) |
| **Other** | `services.py` (IncentiveCalculator, PenaltyCalculator), `signals.py`, `tasks.py` |

**Test Result:** All pass ✓

---

## 6. notifications

**Directory:** `notifications/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `Notification` (SMS/email/push with retry, delivery tracking), `NotificationPreference` |
| **Serializers** | `serializers.py` — `NotificationSerializer`, `NotificationPreferenceSerializer` |
| **Views** | `views.py` — `NotificationViewSet` (unread, mark_read, mark_all_read), `NotificationPreferenceViewSet` (my_preferences, update_preferences) |
| **URLs** | `urls.py` — DefaultRouter with `notifications/` and `preferences/` |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | None |
| **Other** | `services.py`, `tasks.py` |

**Test Result:** N/A (no tests) ✓

---

## 7. on_demand

**Directory:** `on_demand/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `OnDemandRequest` (one-time pickup, PostGIS PointField, pricing with zone+type multipliers) |
| **Serializers** | `serializers.py` — `OnDemandRequestListSerializer`, `OnDemandRequestDetailSerializer`, `OnDemandRequestCreateSerializer`, `OnDemandRequestUpdateSerializer` (with GPS validation) |
| **Views** | `views.py` — `OnDemandRequestViewSet` (ModelViewSet with assign/auto_assign/start/complete/accept/cancel + list_pending/list_today/list_by_collector/list_by_company + summary endpoints) |
| **URLs** | `urls.py` — DefaultRouter |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 2 test classes: `OnDemandLifecycleTests` (5 tests), `OnDemandModelTests` (3 tests) |
| **Other** | `services.py` (auto_assign_request), `pricing_calculator.py` |

**Test Result:** All pass ✓

---

## 8. payment

**Directory:** `payment/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `PaymentMethod`, `Payment` (with refund support), `PaymentWebhook` |
| **Serializers** | `serializers.py` — `PaymentMethodSerializer`, `PaymentMethodCreateSerializer`, `PaymentSerializer`, `PaymentInitiateSerializer`, `PaymentWebhookSerializer` |
| **Views** | `views.py` — `PaymentMethodViewSet` (CRUD + set_default), `PaymentViewSet` (initiate/history/details/verify), `WebhookViewSet` (momo_webhook/paystack_webhook), `GetBanksView`, `GetMobileMoneyProvidersView`, `CreateBankRecipientView`, `CreateMobileMoneyRecipientView`, `InitiateTransferView` |
| **URLs** | `urls.py` — DefaultRouter (methods/, payments/, webhooks/) + banks/, momo-providers/, recipients/bank/, recipients/momo/, transfers/ |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — **Empty** (only pass statement) |
| **Other** | `services.py` (PaymentService, PaystackService), `tasks.py` |

**Test Result:** N/A (no real tests) ✓

---

## 9. routes

**Directory:** `routes/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `Route` (company/zone/supervisor/collector assignments, completion tracking), `RouteStop` (links to OnDemandRequest or ScheduledRequest, PostGIS PointField) |
| **Serializers** | `serializers.py` — `RouteStopSerializer`, `RouteSerializer` (nested stops, derived fields) |
| **Views** | `views.py` — `RouteViewSet` (CRUD + start/complete/my_route_today/set_current_stop/summary_timebound), `RouteStopViewSet` (CRUD + start/complete/skip/fail with CollectionRecord creation) |
| **URLs** | `urls.py` — DefaultRouter at root + route-stops/ |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 2 test classes: `RouteWorkflowTests` (3 tests), `RouteStopTests` (2 tests) |
| **Other** | `services.py`, `signals.py`, `tasks.py` |

**Test Result:** All pass ✓

---

## 10. scheduled_request

**Directory:** `scheduled_request/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `ScheduledRequest` (recurring/subscription-style, PostGIS PointField, auto-pricing) |
| **Serializers** | `serializers.py` — `ScheduledRequestDetailSerializer`, `ScheduledRequestCreateSerializer`, `ScheduledRequestUpdateSerializer` |
| **Views** | `views.py` — `ScheduledRequestViewSet` (ModelViewSet with assign/start/complete/cancel + list_pending/list_today/list_by_collector/list_by_company + summary endpoints) |
| **URLs** | `urls.py` — DefaultRouter |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 2 test classes: `ScheduledRequestTests` (1 test), `ScheduledRequestModelTests` (4 tests) |

**Test Result:** All pass ✓

---

## 11. supervisor

**Directory:** `supervisor/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `Supervisor` (OneToOne to User, M2M assigned_zones, company FK, role_level, permission flags, reports_to self-FK) |
| **Serializers** | `serializers.py` — `SupervisorSerializer`, `SupervisorCreateSerializer`, `SupervisorListSerializer`, `SupervisorUpdateSerializer` |
| **Views** | `views.py` — `SupervisorCreateView`, `SupervisorListView`, `CompanySupervisorListView`, `SupervisorProfileView`, `CompanySupervisorDetailView` |
| **URLs** | `urls.py` — `create/`, `list/`, `profile/`, `company/`, `company/<pk>/` |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 1 class: `SupervisorTests` (4 tests) |
| **Other** | `admin.py` |

**Test Result:** ❌ **FAILS** — All 4 tests fail with `TypeError: Supervisor() got unexpected keyword arguments: 'company_username'`. The `company_username` field was removed from the model but tests still reference it.

---

## 12. wallet

**Directory:** `wallet/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `Wallet` (balance, pending credits/debits, credit/debit methods), `Transaction` (balance tracking), `WalletSettlement` |
| **Serializers** | `serializers.py` — `WalletSerializer`, `TransactionSerializer`, `WalletTransferSerializer`, `WalletSettlementSerializer` |
| **Views** | `views.py` — `WalletViewSet` (my_wallet/history/summary/transfer), `SettlementViewSet` (ReadOnly + summary) |
| **URLs** | `urls.py` — DefaultRouter (wallets/, settlements/) |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — **Empty** (only pass statement) |
| **Other** | `tasks.py` |

**Test Result:** N/A (no real tests) ✓

---

## 13. waste_management_company

**Directory:** `waste_management_company/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `Company` (OneToOne to User, RGD number, EPA/MMDA permits, M2M service_zones, commission_tier, fleet tracking) |
| **Serializers** | `serializers.py` — `CompanySerializer`, `CompanyCreateSerializer` |
| **Views** | `views.py` — `CompanyRegisterView`, `CompanyListView`, `CompanyProfileView`, `CompanyDashboardView`, `CompanyFleetView`, `CompanyBillingView` |
| **URLs** | `urls.py` — `register/`, `profile/`, `list/`, `dashboard/`, `fleet/`, `billing/` |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 3 test classes: `CompanyWorkflowTests` (2 tests), `SupervisorWorkflowTests` (4 tests), `AutoAssignmentTests` (4 tests) |
| **Other** | `accounts/` sub-dir |

**Test Result:** ❌ **FAILS** — `SupervisorWorkflowTests.setUpTestData` fails with the same `company_username` error. 1 class-level error.

---

## 14. zones

**Directory:** `zones/`

| Artifact | Files |
|----------|-------|
| **Models** | `models.py` — `Zone` (PostGIS Polygon boundary + Point center, zone_code, city/district/region, service_fee_multiplier, population_density) |
| **Serializers** | `serializers.py` — `ZoneCreateSerializer`, `ZoneUpdateSerializer`, `ZoneListSerializer` (all GeoFeatureModelSerializer) |
| **Views** | `views.py` — `ZoneCreateView`, `ZoneUpdateView`, `ZoneListView`, `ZoneDetailView`, `PointInZoneView` |
| **URLs** | `urls.py` — `list/`, `create/`, `<id>/`, `<id>/update/`, `check-point/` |
| **Templates** | None |
| **Static Files** | None |
| **Tests** | `tests.py` — 2 test classes: `ZoneTests` (3 tests), `PricingTests` (8 tests) |

**Test Result:** All pass ✓

---

## 15. Overall Test Status

### Summary

| Category | Count |
|----------|-------|
| **Total tests** | 65 |
| **Passed** | 58 |
| **Failed/Errored** | 7 |
| **App test files** | 11 (out of 13 apps) |
| **Apps with no tests** | notifications, payment, wallet |

### Failed Tests (7 errors)

All 7 errors share the same root cause:

```
TypeError: Supervisor() got unexpected keyword arguments: 'company_username'
```

The `company_username` field was removed from the `Supervisor` model (now uses `company` FK to `waste_management_company.Company`), but the following test files still reference it:

| File | Affected Tests |
|------|---------------|
| `supervisor/tests.py` | `test_supervisor_profile_view` (line 99), `test_supervisor_profile_update` (line 121), `test_list_supervisors` (line 148) |
| `waste_management_company/tests.py` | `SupervisorWorkflowTests.setUpTestData` (line 120) |
| `on_demand/tests.py` | `OnDemandLifecycleTests.setUpTestData` (line 64) |
| `routes/tests.py` | `RouteWorkflowTests.setUpTestData` (line 63), `RouteStopTests.setUpTestData` (line 236) |

### Checklist

- [x] **accounts** — 3 models (User, CustomUserManager), 3 serializers, 8 views, 6 URL routes, 9 tests ✓
- [x] **borla_master** — Project config, URL routing, health check, no tests ✓
- [x] **client** — 2 models (Client, ClientAddress), 6 serializers, 5 views, 4 URL routes, 11 tests ✓
- [x] **collection_management** — 1 model (CollectionRecord), 3 serializers, 1 ViewSet, 1 URL, 8 tests ✓
- [x] **collector** — 2 models (Collector, CollectorRating), 4 serializers, 10 views, 8 URL routes, 12 tests ✓
- [x] **notifications** — 2 models (Notification, NotificationPreference), 2 serializers, 2 ViewSets, 1 URL, **0 tests**
- [x] **on_demand** — 1 model (OnDemandRequest), 4 serializers, 1 ViewSet, 1 URL, 8 tests ✓
- [x] **payment** — 3 models (PaymentMethod, Payment, PaymentWebhook), 5 serializers, 7 views, 6 URL routes, **0 tests** (empty file)
- [x] **routes** — 2 models (Route, RouteStop), 2 serializers, 2 ViewSets, 1 URL, 5 tests ✓
- [x] **scheduled_request** — 1 model (ScheduledRequest), 3 serializers, 1 ViewSet, 1 URL, 5 tests ✓
- [x] **supervisor** — 1 model (Supervisor), 4 serializers, 5 views, 4 URL routes, **4 tests FAILING** ❌
- [x] **wallet** — 3 models (Wallet, Transaction, WalletSettlement), 4 serializers, 2 ViewSets, 1 URL, **0 tests** (empty file)
- [x] **waste_management_company** — 1 model (Company), 2 serializers, 6 views, 6 URL routes, **1 test FAILING** ❌
- [x] **zones** — 1 model (Zone), 3 serializers, 5 views, 5 URL routes, 11 tests ✓
- [x] **`python manage.py test`** — ❌ **FAILED** (65 tests, 7 errors — all from `company_username` issue)

### Key Findings

1. **3 apps have zero tests:** `notifications`, `payment`, `wallet` (their `tests.py` files are empty stubs)
2. **7 tests fail** all from the same cause: `Supervisor.model` no longer has `company_username` field (replaced by `company` FK), but tests still pass it in `Supervisor.objects.create()`
3. **Test count** is 65 total; no coverage for payment flows, wallet operations, or notification delivery
4. **No templates or static files** are shipped within any Django app — the frontend is a separate React (Vite/TypeScript) app in `borla-frontend/`
5. All autodiscovered tasks/signals/services files exist but their actual test coverage is unknown