# Migration Strategy

## Approach: Strangler Fig Pattern

Rather than a "big bang" rewrite (shut down the old, launch the new), the Strangler Fig pattern incrementally replaces the legacy system piece by piece while keeping it running. This minimizes risk and allows business continuity throughout the migration.

```
          Week 1                         Week 14
Legacy:  ████████████████████████░░░░░░░░░░░░░░
New:     ░░░░░░░░▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
Traffic: 100% legacy → gradually shifts → 100% new
```

---

## Phase Overview

| Phase | Duration | Goal |
|-------|----------|------|
| Phase 1: Foundation | Weeks 1–2 | Project setup, CI/CD, DB migrations, legacy hotfixes |
| Phase 2: Authentication | Weeks 3–4 | New auth service with JWT, password re-hashing |
| Phase 3: Core API | Weeks 5–8 | All business logic as REST API |
| Phase 4: Frontend | Weeks 9–12 | React SPA replacing all ASP pages |
| Phase 5: Cutover | Weeks 13–14 | Traffic migration, decommission legacy |

---

## Phase 1: Foundation (Weeks 1–2)

### Goals
- New solution running in CI/CD
- Database schema under migration control
- Critical legacy vulnerabilities patched (buys time while migration runs)

### Tasks

**New project setup:**
- [ ] Create `CarRetailSystem.Api` (ASP.NET Core 8 Web API)
- [ ] Create `CarRetailSystem.Web` (React + TypeScript + Vite)
- [ ] Configure Docker Compose (api, web, sqlserver, redis)
- [ ] Set up GitHub Actions: build → test → Docker push → deploy

**Database:**
- [ ] Add EF Core to API project
- [ ] Scaffold entities from existing `CarRetailDB` schema
- [ ] Generate initial EF Core migration (baseline — matches current schema exactly)
- [ ] Run migration against development SQL Server instance

**Legacy hotfixes (applied to existing ASP code):**
- [ ] Replace string-concat SQL in `login.asp` and `inventory.asp` with calls to existing stored procedures
- [ ] Move `DB_CONNECTION_STRING` and `SMTP_PASSWORD` from `config.asp` to environment variables / web.config encrypted section
- [ ] Add `HTMLEncode()` to all `Response.Write` outputs in ASP pages (function already exists in `utils.asp`)
- [ ] Configure IIS to redirect HTTP → HTTPS

### Deliverable
Running skeleton API (health check endpoint only) + patched legacy system

---

## Phase 2: Authentication (Weeks 3–4)

### Goals
- New auth system operational with secure password storage
- Legacy users migrated on first login (no forced password reset)

### Tasks

**API:**
- [ ] Add ASP.NET Identity to DbContext
- [ ] Add migration: new `AspNetUsers` table (or adapt existing `Users` table)
- [ ] Implement `POST /auth/login` — validates credentials, returns JWT + refresh token
- [ ] Implement `POST /auth/refresh` — exchanges refresh token
- [ ] Implement `POST /auth/logout` — revokes refresh token
- [ ] Add rate limiting middleware on `/auth/login` (max 5 attempts per IP per minute)
- [ ] Add JWT middleware to validate Bearer tokens on all protected endpoints
- [ ] Map legacy roles (Admin, Salesperson, Manager) to ASP.NET Identity roles

**Password migration strategy (no user disruption):**
```
1. Add column: Users.PasswordHash (bcrypt), Users.PasswordMigrated (bit)
2. On login attempt:
   a. If PasswordMigrated = 1 → validate against bcrypt hash (new flow)
   b. If PasswordMigrated = 0 → validate against plain text (legacy)
      → if valid: hash the password with bcrypt, set PasswordMigrated = 1
      → if invalid: return 401
3. After all users have logged in once, drop plain-text Password column
```

**Frontend:**
- [ ] Build Login page (email + password form, JWT stored in memory + HttpOnly cookie for refresh)

### Deliverable
Secure login working via new API. Legacy ASP login still operational in parallel.

---

## Phase 3: Core API (Weeks 5–8)

### Goals
- All business logic reimplemented as REST API endpoints
- All legacy stored procedure logic ported to service layer
- Full test coverage

### Tasks

**Inventory module (Week 5):**
- [ ] `GET /cars` — paginated list, filterable by make/price/availability
- [ ] `GET /cars/{id}` — car details
- [ ] `POST /cars` — add car (Admin role required)
- [ ] `PUT /cars/{id}` — update car (Admin role required)
- [ ] `DELETE /cars/{id}` — soft delete (Admin role required)
- [ ] `GET /cars/{id}/stock` — current stock level
- [ ] Port `sp_GetAvailableCars` → `InventoryService.GetAvailableCarsAsync()`
- [ ] Port `TR_Cars_Audit` trigger → `AuditMiddleware` writing to Inventory_Log

**Sales module (Week 6):**
- [ ] `POST /sales` — create order (atomic: sale + stock update + audit log)
- [ ] `GET /sales/{id}` — sale details
- [ ] Port `sp_CreateSalesOrder` → `SalesService.CreateOrderAsync()` with `IDbContextTransaction`
- [ ] Port tax calculation (10%) to `SalesService.CalculateTax()`

**Customers module (Week 6):**
- [ ] `GET /customers` — paginated list
- [ ] `GET /customers/{id}` — customer details
- [ ] `POST /customers` — create customer
- [ ] `PUT /customers/{id}` — update customer
- [ ] `DELETE /customers/{id}` — soft delete
- [ ] `GET /customers/{id}/sales` — purchase history (port `sp_GetCustomerSalesHistory`)

**Reports module (Week 7):**
- [ ] `GET /reports/monthly-sales` — port `sp_GenerateMonthlySalesReport`
- [ ] `GET /reports/inventory` — current stock summary
- [ ] `GET /reports/top-sellers` — top-selling cars by period
- [ ] Add Redis caching (5-minute TTL) on all report endpoints

**Quality (Week 8):**
- [ ] Unit tests for all service methods (xUnit + Moq)
- [ ] Integration tests with test SQL Server instance (Testcontainers)
- [ ] Add FluentValidation for all request DTOs
- [ ] Add global exception handler middleware
- [ ] Verify Swagger / OpenAPI documentation is complete

### Deliverable
Complete REST API with test coverage. Frontend not yet connected.

---

## Phase 4: Frontend (Weeks 9–12)

### Goals
- React SPA replacing all ASP pages
- All user workflows functional through new UI
- E2E tests passing

### Tasks

**Week 9 — Core pages:**
- [ ] Login page (connect to `POST /auth/login`)
- [ ] Dashboard (stats: total cars, customers, sales this month — cached endpoint)
- [ ] Navigation layout

**Week 10 — Inventory:**
- [ ] Inventory list page (paginated table, search/filter by make and price)
- [ ] Add car form (Admin only)
- [ ] Edit car form (Admin only)
- [ ] Delete car confirmation (Admin only)
- [ ] Stock level display

**Week 11 — Sales & Customers:**
- [ ] Create sales order form (select customer, select car, show tax + total)
- [ ] Sales confirmation and receipt view
- [ ] Customer list page
- [ ] Customer detail page with purchase history
- [ ] Add / edit customer forms

**Week 12 — Reports & Polish:**
- [ ] Monthly sales report page (chart + table)
- [ ] Inventory status report
- [ ] Error handling (toast notifications, form validation feedback)
- [ ] Loading states and empty states
- [ ] Responsive design (mobile-friendly)
- [ ] E2E tests with Playwright: login, create sale, view report

### Deliverable
Fully functional React application. Legacy ASP still accessible during parallel run period.

---

## Phase 5: Cutover (Weeks 13–14)

### Goals
- All traffic migrated to new system
- Legacy system decommissioned
- Monitoring in place

### Tasks

**Week 13 — Blue-green deployment:**
- [ ] Deploy new system alongside legacy (different URL or subdomain)
- [ ] Run both systems in parallel against the same database
- [ ] Beta users test new system → collect feedback
- [ ] Verify all data mutations are consistent (no dual-write bugs)
- [ ] Configure Application Insights alerts (error rate, response time, failed logins)

**Week 14 — Final cutover:**
- [ ] Switch DNS / load balancer to route 100% traffic to new system
- [ ] Monitor error rates and performance for 48 hours
- [ ] Remove `On Error Resume Next` and legacy COM DLL registrations
- [ ] Archive legacy ASP source files (do not delete — keep for reference)
- [ ] Decommission IIS Classic ASP application pool
- [ ] Remove VB6 COM DLL registrations (`regsvr32 /u`)
- [ ] Update documentation

### Deliverable
Legacy system decommissioned. New system serving all traffic with monitoring active.

---

## Risk Register

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Password migration misses a user | Low | Medium | Keep legacy flow for 30 days post-cutover; monitor login failures |
| Data integrity issue during parallel run | Low | High | Write integration tests that compare query results across both systems |
| Missing business logic in VB6 DLLs (no source for CustomerMgmt, ReportGenerator) | Medium | Medium | Reverse-engineer from COM documentation; test against legacy behavior |
| Legacy session tokens still valid after cutover | Low | Low | Session timeout (30 min); old sessions expire naturally |
| Performance regression in new system | Low | Medium | Load test before cutover; Redis caching on hot endpoints |

---

## Rollback Plan

At each phase, the legacy system remains operational. Rollback means:

- **Phases 1–3:** Revert to 100% legacy traffic — no user impact
- **Phase 4:** Legacy ASP pages still deployed — point DNS back
- **Phase 5 (first 48h):** Legacy system on standby — DNS switchback takes minutes

After the 48-hour monitoring window in Week 14, legacy is archived and rollback is no longer straightforward.

---

## Definition of Done

The migration is complete when:
- [ ] All 5 ASP pages have React equivalents
- [ ] All 5 stored procedures have service layer equivalents
- [ ] All 17 identified vulnerabilities are resolved
- [ ] Unit test coverage ≥ 80% on service layer
- [ ] E2E tests cover all primary workflows (login, add car, create sale, view report)
- [ ] Zero `On Error Resume Next` in codebase
- [ ] Zero hardcoded credentials
- [ ] Application Insights monitoring active with alert policies
- [ ] IIS Classic ASP application pool decommissioned
