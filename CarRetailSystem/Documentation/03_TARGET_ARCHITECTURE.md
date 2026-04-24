# Target Architecture

## Recommendation: Modular Monolith + API-First

### Why Not Microservices?

The system has 5 database tables and ~2,000 lines of code. Microservices would introduce significant operational overhead (service discovery, inter-service communication, distributed tracing, multiple deployments) that is disproportionate to the system's size and complexity.

A **modular monolith** delivers the same clean separation of concerns while remaining simple to deploy, debug, and operate. The API-first design means individual modules can be extracted into separate services later if traffic or team size demands it.

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Frontend** | React 18 + TypeScript + Tailwind CSS | Component-based, strongly typed, natural SPA for CRUD workflows |
| **API** | ASP.NET Core 8 (Web API) | Natural migration from VB6/.NET, mature ecosystem, excellent performance |
| **ORM** | Entity Framework Core 8 | Code-first migrations, LINQ queries replace ADO string SQL |
| **Authentication** | ASP.NET Identity + JWT | Replaces plain-text session auth; bcrypt password hashing built-in |
| **Database** | SQL Server 2022 | Preserve existing schema, supported by EF Core migrations |
| **Caching** | Redis (or IMemoryCache for start) | Dashboard stats, inventory lists — eliminate per-request DB hits |
| **Logging** | Serilog → Azure Application Insights | Structured logging, replaces `On Error Resume Next` + file logs |
| **Secrets** | Azure Key Vault (or env vars) | Eliminates hardcoded credentials |
| **Containerization** | Docker + Docker Compose | Reproducible environments, replaces manual IIS setup |
| **CI/CD** | GitHub Actions | Automated build, test, deploy pipeline |
| **API Docs** | OpenAPI / Swagger | Auto-generated from controller annotations |

---

## System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                        FRONTEND                              │
│              React 18 + TypeScript + Tailwind CSS            │
│         (React Query for caching, React Router v6)           │
│                                                              │
│  Pages: Login │ Dashboard │ Inventory │ Sales │ Customers    │
│               │ Reports   │ User Management                  │
└─────────────────────────┬────────────────────────────────────┘
                          │ HTTPS + JWT Bearer token
┌─────────────────────────▼────────────────────────────────────┐
│                      BACKEND (ASP.NET Core 8)                │
│                                                              │
│  ┌────────────┐ ┌───────────┐ ┌─────────┐ ┌─────────────┐  │
│  │    Auth    │ │ Inventory │ │  Sales  │ │   Reports   │  │
│  │   Module   │ │  Module   │ │  Module │ │   Module    │  │
│  │            │ │           │ │         │ │             │  │
│  │ /auth/     │ │ /cars/    │ │ /sales/ │ │ /reports/   │  │
│  │ login      │ │ GET list  │ │ POST    │ │ monthly     │  │
│  │ refresh    │ │ GET :id   │ │ GET :id │ │ inventory   │  │
│  │ logout     │ │ POST      │ │ history │ │ top-sellers │  │
│  │ register   │ │ PUT :id   │ │         │ │             │  │
│  │            │ │ DELETE    │ │         │ │             │  │
│  └────────────┘ └───────────┘ └─────────┘ └─────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                   Customers Module                     │  │
│  │  /customers/ — GET list, GET :id, POST, PUT, DELETE   │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              Cross-Cutting Infrastructure              │  │
│  │  EF Core DbContext │ JWT Middleware │ Audit Middleware  │  │
│  │  FluentValidation  │ Serilog        │ Exception Handler │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────┬────────────────────────────────────┘
                          │
         ┌────────────────┼─────────────────┐
         │                │                 │
┌────────▼───────┐ ┌──────▼──────┐ ┌───────▼──────┐
│  SQL Server    │ │    Redis    │ │  Azure Blob  │
│  2022          │ │  (Cache)    │ │  (Files)     │
│                │ │             │ │              │
│  CarRetailDB   │ │  Dashboard  │ │  Reports     │
│  (migrated     │ │  stats      │ │  exports     │
│  schema)       │ │  Inventory  │ │              │
└────────────────┘ └─────────────┘ └──────────────┘
```

---

## Project Structure (Backend)

```
CarRetailSystem.Api/
├── Modules/
│   ├── Auth/
│   │   ├── AuthController.cs
│   │   ├── AuthService.cs
│   │   ├── Dtos/           (LoginRequest, TokenResponse, etc.)
│   │   └── Models/         (ApplicationUser)
│   ├── Inventory/
│   │   ├── CarsController.cs
│   │   ├── InventoryService.cs
│   │   ├── Dtos/
│   │   └── Models/
│   ├── Sales/
│   │   ├── SalesController.cs
│   │   ├── SalesService.cs
│   │   ├── Dtos/
│   │   └── Models/
│   ├── Customers/
│   │   ├── CustomersController.cs
│   │   ├── CustomerService.cs
│   │   ├── Dtos/
│   │   └── Models/
│   └── Reports/
│       ├── ReportsController.cs
│       ├── ReportService.cs
│       └── Dtos/
├── Infrastructure/
│   ├── Data/
│   │   ├── AppDbContext.cs
│   │   └── Migrations/
│   ├── Auth/
│   │   └── JwtTokenService.cs
│   └── Caching/
│       └── CacheService.cs
├── Common/
│   ├── Middleware/
│   │   ├── ExceptionHandlerMiddleware.cs
│   │   └── AuditMiddleware.cs
│   ├── Exceptions/
│   └── Extensions/
├── Program.cs
└── appsettings.json (no secrets — all from Key Vault / env vars)
```

---

## Project Structure (Frontend)

```
CarRetailSystem.Web/
├── src/
│   ├── pages/
│   │   ├── Login.tsx
│   │   ├── Dashboard.tsx
│   │   ├── Inventory.tsx
│   │   ├── Sales.tsx
│   │   ├── Customers.tsx
│   │   └── Reports.tsx
│   ├── components/
│   │   ├── common/       (Table, Modal, Form, Button, etc.)
│   │   └── domain/       (CarCard, SaleRow, CustomerForm, etc.)
│   ├── api/              (axios clients per module)
│   ├── hooks/            (useAuth, useInventory, useSales, etc.)
│   ├── store/            (Zustand or React Context for auth state)
│   └── types/            (TypeScript interfaces matching API DTOs)
└── package.json
```

---

## API Design

### Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/login` | Returns access token + refresh token |
| POST | `/auth/refresh` | Exchanges refresh token for new access token |
| POST | `/auth/logout` | Revokes refresh token |

### Inventory

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/cars` | List all cars (paginated, filterable by make/price) |
| GET | `/cars/{id}` | Get single car details |
| POST | `/cars` | Add new car (Admin only) |
| PUT | `/cars/{id}` | Update car (Admin only) |
| DELETE | `/cars/{id}` | Soft-delete car (Admin only) |
| GET | `/cars/{id}/stock` | Get current stock level |

### Sales

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/sales` | Create sales order (atomic: sale + stock update + audit log) |
| GET | `/sales/{id}` | Get sale details |
| GET | `/customers/{id}/sales` | Customer purchase history |

### Customers

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/customers` | List customers (paginated) |
| GET | `/customers/{id}` | Get customer |
| POST | `/customers` | Create customer |
| PUT | `/customers/{id}` | Update customer |
| DELETE | `/customers/{id}` | Soft-delete customer |

### Reports

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/reports/monthly-sales` | Monthly sales summary |
| GET | `/reports/inventory` | Inventory status |
| GET | `/reports/top-sellers` | Top-selling cars by period |

---

## Security Design

### Authentication Flow

```
User → POST /auth/login (email + password)
     → ASP.NET Identity validates against hashed password (bcrypt)
     → Returns: { accessToken (15 min), refreshToken (7 days) }

Subsequent requests:
     → Authorization: Bearer <accessToken>
     → JWT Middleware validates signature, expiry, claims
     → ClaimsPrincipal injected into controllers
```

### What Replaces Each Legacy Vulnerability

| Legacy Vulnerability | Modern Solution |
|---------------------|----------------|
| SQL injection (string concatenation) | EF Core parameterized queries — string SQL impossible by default |
| Hardcoded `sa` credentials | Azure Key Vault / environment variables |
| Plain text passwords | bcrypt via ASP.NET Identity (cost factor 12) |
| No HTTPS | HTTPS enforced at ASP.NET Core + HSTS header |
| XSS (unencoded output) | React escapes all output by default |
| No CSRF | JWT in Authorization header (not cookie) — CSRF not applicable |
| Session hijacking | Short-lived JWT (15 min) + HttpOnly refresh token cookie |
| `On Error Resume Next` | Global exception handler middleware → structured log |
| No rate limiting | ASP.NET Core rate limiting middleware on `/auth/login` |

---

## Infrastructure Diagram

```
┌─────────────────────────────────────────────────┐
│                  GitHub Repo                    │
│  main branch push → GitHub Actions CI/CD        │
└────────────────────┬────────────────────────────┘
                     │
         ┌───────────▼───────────┐
         │    Docker Build       │
         │  api image + web image│
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │   Azure Container     │
         │   Apps (or AKS)       │
         │                       │
         │  api-service          │
         │  web-service          │
         └───────────┬───────────┘
                     │
    ┌────────────────┼───────────────┐
    │                │               │
┌───▼────┐  ┌───────▼───┐  ┌────────▼──────┐
│ Azure  │  │  Azure    │  │  Azure        │
│ SQL DB │  │  Cache    │  │  Key Vault    │
│        │  │  (Redis)  │  │  (secrets)    │
└────────┘  └───────────┘  └───────────────┘
```

---

## What Is Preserved from the Legacy System

| Legacy Element | Modern Equivalent |
|---------------|-------------------|
| Database schema (5 tables) | EF Core entities + migrations (schema preserved, passwords column replaced) |
| Stored procedure logic | Service layer methods + EF Core LINQ |
| `sp_CreateSalesOrder` transaction | `SalesService.CreateOrderAsync()` with `IDbContextTransaction` |
| `sp_GenerateMonthlySalesReport` | `ReportService.GetMonthlySummaryAsync()` |
| Inventory_Log audit trail | `AuditMiddleware` + dedicated `AuditLogService` |
| Tax calculation (10%) | `SalesService.CalculateTax()` — same rule, configurable via appsettings |
| Role-based access (Admin, Salesperson, Manager) | `[Authorize(Roles = "Admin")]` attributes on controllers |
| Sample data | EF Core seed data in `AppDbContext.OnModelCreating` |
