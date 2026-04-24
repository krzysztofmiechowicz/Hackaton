# Business Background & Modernization Log

This document captures the business context driving modernization and serves as a living record of every significant change, decision, and milestone as the project evolves. Update it whenever a new phase starts, a key decision is made, or a change is delivered.

---

## Why This System Exists

CarRetailSystem was built in the early 2000s to digitize the sales process for a car retail dealership. Before its introduction, inventory tracking was done in spreadsheets and sales orders were paper-based. The system automated:

- Vehicle inventory management and availability tracking
- Sales order creation with automatic stock updates
- Customer record-keeping
- Basic financial reporting for management

For many years the system fulfilled its purpose. Over time the dealership grew, the market changed, and the technology around it advanced — but the system did not.

---

## Business Drivers for Modernization

### 1. Operational Risk

The system runs on technology that Microsoft stopped supporting over a decade ago (Windows Server 2003, IIS 5, VB6). There is no vendor support, no security patches, and a shrinking pool of engineers who know how to maintain it. A single server failure or a Windows update incompatibility could take the entire business offline with no fast path to recovery.

### 2. Security Exposure

An independent review identified 17 security vulnerabilities, including SQL injection on the login page that allows an attacker to bypass authentication entirely without knowing any password. Customer personal data (names, addresses, phone numbers) and financial transaction history are at risk. Regulatory exposure under data protection laws is significant and growing.

### 3. Lost Revenue from Downtime and Manual Workarounds

Because the system cannot be accessed remotely and has no mobile interface, sales staff work around it — keeping notes on paper or in personal spreadsheets and entering data later. This creates duplicate records, reporting inaccuracies, and lost leads. Estimated impact: several lost sales per month due to delayed or incorrect inventory information.

### 4. Inability to Integrate

The system has no API. Every integration with external tools (financing calculators, manufacturer inventory feeds, CRM platforms, accounting software) requires manual data re-entry. Staff spend significant time on data reconciliation that could be automated.

### 5. Reporting Limitations

Management reports are generated manually from the system or exported to Excel for further processing. There is no real-time dashboard, no trend analysis, and no way to track salesperson performance over time. Strategic decisions are made on data that is days or weeks old.

### 6. Talent Risk

The two engineers who built and maintain the system are approaching retirement. VB6 and Classic ASP expertise is essentially non-existent in the current hiring market. If either engineer leaves before the system is replaced, the business would have no one who can modify or fix it.

---

## Expected Business Outcomes

| Outcome | How Modernization Delivers It |
|---------|-------------------------------|
| Eliminate security risk | Parameterized queries, bcrypt passwords, HTTPS, JWT auth — all 17 vulnerabilities resolved |
| 99.9% uptime | Containerized deployment on cloud infrastructure with health checks and auto-restart |
| Remote and mobile access | React SPA accessible from any browser; responsive design for tablets and phones |
| API-first integrations | REST API enables connection to financing, CRM, and accounting tools without manual data entry |
| Real-time reporting | Cached dashboard with live sales metrics; no more end-of-day export to Excel |
| Onboarding new developers | Standard .NET + React stack — available in every hiring market |
| Compliance readiness | Password hashing, audit trail, data-at-rest encryption — foundation for GDPR/PCI posture |

---

## Modernization Scope

**In scope:**
- Full replacement of all Classic ASP pages with a React SPA
- Full replacement of VB6 COM components with ASP.NET Core service layer
- Database upgrade from SQL Server 7.0 to SQL Server 2022 (schema preserved)
- Authentication replaced with JWT + bcrypt (existing user accounts migrated)
- All 17 security vulnerabilities resolved
- Docker-based deployment replacing manual IIS/COM registration

**Out of scope (current phase):**
- Redesign of the business data model (new fields, new tables)
- Integration with external systems (CRM, financing, accounting) — enabled by the new API but not built in this phase
- Mobile native application — responsive web is sufficient for Phase 1
- Multi-tenancy or multi-branch support

---

## Modernization Change Log

This section is updated continuously. Each entry records what changed, why, and what business impact it has.

---

### 2026-04-24 — Project Initiated

**What:** Full analysis of the legacy codebase completed. Modernization plan documented.

**Context:** Management approved the modernization initiative following an internal security review that flagged the SQL injection vulnerability on the login page as an immediate risk.

**Decisions made:**
- Architecture: Modular Monolith (not microservices) — system size does not justify microservices complexity
- Migration pattern: Strangler Fig — legacy stays live throughout, no big-bang cutover
- Technology: ASP.NET Core 8 + React 18 + SQL Server 2022
- Timeline: 14 weeks across 5 phases

**Artifacts created:**
- `Documentation/01_CURRENT_STATE.md` — full inventory of legacy code, DB schema, component analysis
- `Documentation/02_SECURITY_VULNERABILITIES.md` — 17 vulnerabilities with exact file/line locations
- `Documentation/03_TARGET_ARCHITECTURE.md` — target stack, module structure, API contracts
- `Documentation/04_MIGRATION_STRATEGY.md` — Strangler Fig phases, risk register, rollback plan
- `Documentation/05_QUICK_WINS.md` — 7 immediate fixes for the legacy system (~10 hours of work)
- `CLAUDE.md` (project root) — consolidated migration reference for AI-assisted development

**Business impact:** No change to the running system yet. Risk identified and documented. Quick wins can reduce critical exposure immediately.

---

<!-- TEMPLATE FOR NEW ENTRIES — copy and fill in when changes are made

### YYYY-MM-DD — [Short title of change]

**Phase:** [1 / 2 / 3 / 4 / 5 / Post-cutover]

**What:** [One paragraph — what was built, changed, or decided]

**Why:** [Business or technical reason driving this change]

**Decisions made:**
- [Key choice and rationale]

**Vulnerabilities resolved:** [List from 02_SECURITY_VULNERABILITIES.md, if any]

**Business impact:** [What this means for the business — risk reduced, capability added, workflow changed]

**Artifacts changed:**
- [File or system changed]

-->

---

## Open Questions & Pending Decisions

| Question | Owner | Target Date | Notes |
|----------|-------|-------------|-------|
| Which cloud provider for deployment — Azure vs. on-premise? | Architecture | Week 2 | Azure preferred; on-premise possible if compliance requires it |
| Should the existing `Users` table be kept or replaced with ASP.NET Identity schema? | Development | Week 3 | Keeping existing table is simpler; Identity schema is more standard |
| Three VB6 DLLs have no source (`CustomerMgmt`, `ReportGenerator`, `Utilities`). Reverse-engineer from docs or rewrite from scratch? | Development | Week 5 | `COM_COMPONENTS.md` documents the interfaces; legacy system can be used for behavior verification |
| Tax rate (10%) is hardcoded. Should it become configurable per-region? | Business | Week 4 | Currently out of scope but `appsettings.json` is the right place to put it |

---

## Stakeholders

| Role | Responsibility in This Project |
|------|-------------------------------|
| Business Owner | Approves scope, priorities, and go/no-go at each phase cutover |
| Development Team | Builds and tests the new system |
| QA | Validates that new system behavior matches legacy for all workflows |
| Operations / DevOps | Sets up CI/CD, Docker, cloud infrastructure, monitoring |
| End Users (Sales Staff) | UAT during Phase 4; primary source of feedback on new UI |
