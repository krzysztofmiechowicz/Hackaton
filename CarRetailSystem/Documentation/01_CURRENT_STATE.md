# Current State Analysis

## Business Domain

**System purpose:** Monolithic enterprise application for managing retail car sales — inventory, sales transactions, customer records, and business reporting.

### Core Business Entities

| Entity | Table | Description |
|--------|-------|-------------|
| Cars / Inventory | `Cars` | Vehicles available for sale (make, model, year, VIN, price, stock) |
| Customers | `Customers` | Buyer database with contact information |
| Sales Transactions | `Sales` | Purchase orders linking customers to vehicles |
| Users / Employees | `Users` | Staff with role-based access (Admin, Salesperson, Manager) |
| Audit Trail | `Inventory_Log` | Full change history for compliance |

### Core Business Workflows

1. **Login** → session creation with role-based authorization
2. **Inventory Browse** → query available cars with optional Make/Price filter
3. **Sales Order** → atomic transaction updating Sales, Cars stock, and Audit Log
4. **Customer Management** → CRUD operations on customer records
5. **Reporting** → monthly sales reports, inventory summaries, top sellers

---

## Technology Stack

| Component | Technology | Version | Status |
|-----------|-----------|---------|--------|
| Web / Presentation | Classic ASP (VBScript) | IIS 5.0+ | Deprecated |
| Business Logic | VB6 COM Components | 6.0 | Deprecated |
| Data Access | ADO 2.8 (ADODB) | 2.8 | Outdated |
| Database | SQL Server | 7.0 / 2000 | End of Life |
| Server | Windows IIS | 5.0 / 6.0 | Legacy |
| OS | Windows Server | 2003 / 2008 | Unsupported |

---

## Architecture

**Type:** Monolithic, tightly-coupled, stateful

```
┌──────────────────────────────────────────────────┐
│          ASP Pages (IIS)                         │
│  login.asp │ inventory.asp │ index.asp            │
│  ────────────────────────────────────────────    │
│  includes: config.asp, security.asp,             │
│            session.asp, utils.asp                │
└──────────────────┬───────────────────────────────┘
                   │ CreateObject()
┌──────────────────▼───────────────────────────────┐
│          VB6 COM Components                      │
│  CarInventory.dll │ SalesOrder.dll               │
│  CustomerMgmt.dll │ ReportGenerator.dll          │
│  Utilities.dll                                   │
└──────────────────┬───────────────────────────────┘
                   │ ADODB / string-concatenated SQL
┌──────────────────▼───────────────────────────────┐
│          SQL Server 7.0                          │
│  Tables, Stored Procedures, Triggers             │
└──────────────────────────────────────────────────┘
```

### Key Architectural Problems

- No separation of concerns — business logic split between ASP and COM with no clear boundary
- Tight coupling — direct database connections embedded in every layer
- Stateful monolith — session affinity required, cannot scale horizontally
- Synchronous only — all operations block until completion
- No API layer — HTML rendered server-side, no REST or SOAP interface

---

## File Inventory

### ASP Pages

| File | Lines | Purpose |
|------|-------|---------|
| `ASP/pages/login.asp` | 94 | Authentication, session creation |
| `ASP/pages/inventory.asp` | 130 | Inventory CRUD (admin only) |
| `ASP/pages/index.asp` | 107 | Landing page, quick stats dashboard |

Missing (referenced but not implemented): `dashboard.asp`, `sales.asp`, `customers.asp`, `reports.asp`, `logout.asp`

### ASP Includes

| File | Lines | Purpose |
|------|-------|---------|
| `ASP/includes/config.asp` | 62 | DB connection string, app settings, credentials |
| `ASP/includes/security.asp` | 114 | Auth checks, input sanitization, event logging |
| `ASP/includes/session.asp` | 84 | Session lifecycle (init, validate, expire, clear) |
| `ASP/includes/utils.asp` | 246 | 40+ helpers: strings, dates, numbers, validation, HTML |

### VB6 Components

| File | Lines | ProgID | Purpose |
|------|-------|--------|---------|
| `VB6Components/CarInventory.vb` | 132 | `CarRetailSystem.CarInventory` | Stock queries, availability, stock updates |
| `VB6Components/SalesOrder.vb` | 133 | `CarRetailSystem.SalesOrder` | Transaction creation, tax calculation |

Missing source: `CustomerMgmt.dll`, `ReportGenerator.dll`, `Utilities.dll`

### Database

| File | Lines | Purpose |
|------|-------|---------|
| `SQL/database_setup.sql` | 272 | Full schema, stored procedures, triggers, sample data |

---

## Database Schema

### Tables

**Users**
```
UserID (PK) | UserName (UNIQUE) | Password (plain text) | FirstName | LastName
Email | Role | IsActive | CreatedDate | LastLoginDate
```
Sample accounts: `admin/admin123`, `john/pass123`, `mary/pass123`, `bob/pass123`

**Cars**
```
CarID (PK) | Make | Model | Year | Color | VIN (UNIQUE) | Price | Stock
Description | CreatedDate | LastModifiedDate | IsActive
```
Indexes: `IX_Cars_Make`, `IX_Cars_Stock`

**Customers**
```
CustomerID (PK) | FirstName | LastName | Email | Phone | Address
City | State | ZipCode | RegisteredDate | IsActive
```
Index: `IX_Customers_Email`

**Sales**
```
SalesID (PK) | CustomerID (FK) | CarID (FK) | SalesPrice | SalesDate
SalespersonID (FK→Users) | PaymentMethod | Notes
```
Indexes: `IX_Sales_Date`, `IX_Sales_CustomerID`

**Inventory_Log**
```
LogID (PK) | CarID (FK) | Action (ADD/SALE/UPDATE/DELETE/AUDIT)
Quantity | ChangedDate | ChangedBy | Notes
```

### Stored Procedures

| Name | Purpose |
|------|---------|
| `sp_GetAvailableCars` | Parameterized query — cars with Stock > 0, optional Make filter |
| `sp_CreateSalesOrder` | Full transaction: insert Sale → update Stock → log → commit/rollback |
| `sp_UpdateInventory` | Update stock level, log change |
| `sp_GetCustomerSalesHistory` | 4-table join ordered by date DESC |
| `sp_GenerateMonthlySalesReport` | GROUP BY month, count sales, sum revenue |

### Triggers

| Name | Event | Action |
|------|-------|--------|
| `TR_Cars_Audit` | AFTER UPDATE/DELETE on Cars | Insert audit record to Inventory_Log |

---

## VB6 Component Details

### CarInventory

| Method | Description | Issue |
|--------|-------------|-------|
| `Initialize()` | Opens ADODB connection | Hardcoded connection string (line 25) |
| `GetAvailableCars(make, maxPrice)` | SELECT with optional filters | SQL injection (lines 50, 54) |
| `GetCarDetails(carID)` | Single-car query | SQL injection (line 70) |
| `UpdateStock(carID, quantity)` | UPDATE stock | SQL injection (line 83) |
| `GetStockLevel(carID)` | Simple SELECT | SQL injection |
| `LogInventoryChange(carID, quantity)` | Insert to Inventory_Log | SQL injection (line 120) |

### SalesOrder

| Method | Description | Issue |
|--------|-------------|-------|
| `Initialize()` | Opens connection, sets 30s timeout | Hardcoded connection string (line 23) |
| `CreateSalesOrder(...)` | INSERT Sale + UPDATE stock + INSERT log | SQL injection (lines 59-60, 78, 82) |
| `GetSalesHistory(customerID)` | 4-table JOIN | SQL injection (line 105) |
| `CalculateTax(salePrice)` | Returns salePrice × 10% | Hardcoded tax rate |
| `CalculateTotal(salePrice)` | salePrice + tax | — |

---

## Performance Characteristics

| Problem | Location | Impact |
|---------|----------|--------|
| No connection pooling | All ASP pages, all VB6 components | Connection pool exhaustion under load |
| No pagination | `inventory.asp:96` | Slow for large datasets |
| No caching | `index.asp:77-95` | DB hit on every page load |
| No async operations | Entire codebase | Blocking UI on slow queries |
| Session affinity required | `session.asp` | Cannot load-balance |

---

## Deployment Architecture

```
Windows Server 2003/2008
├── IIS 5.0/6.0
│   ├── Classic ASP Engine
│   └── COM Components (registered via regsvr32)
└── SQL Server 7.0/2000/2005
    └── CarRetailDB
```

**Deployment process:** Manual — copy files, `regsvr32` DLLs, edit config.asp, `iisreset`. No automation, no rollback strategy.

---

## Statistics

| Metric | Value |
|--------|-------|
| Total source files | 15+ |
| Lines of code | ~2,000 |
| ASP pages | 8 (3 implemented) |
| VB6 components | 5 (2 with source) |
| Database tables | 5 |
| Stored procedures | 5 |
| Sample cars | 8 |
| Sample customers | 5 |
