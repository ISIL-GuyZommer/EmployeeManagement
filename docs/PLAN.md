# Employee Management System: Build Plan

## 1. Scope and conventions

**Modules:** Departments & Job Roles, Employee Management, Search & Filtering, Reports, User Permissions, React UI. Every module follows the rules in `CLAUDE.md`.

**Cross-cutting decisions** (the items marked with Q are decided in §6 "Decisions"):

| Topic | Proposal |
|---|---|
| Code layout | Feature folders in each project, e.g. `Application/Employees/` holds `EmployeeService`, `IEmployeeRepository` and DTOs |
| Mapping | Manual mapping. No AutoMapper. |
| Validation | DataAnnotations on request DTOs for shape (required, length). Domain invariants live in entity factory methods and constructors and throw `DomainValidationException`. |
| Errors | Exception middleware in Api maps Application exceptions to RFC 7807 `ProblemDetails` (400 / 404 / 409) |
| SP error signalling | SPs `THROW` custom error numbers: 50001 NotFound, 50002 Duplicate, 50003 InUse, 50004 ConcurrencyConflict. Infrastructure maps a `SqlException` to a typed Application exception, and also maps unique-index violations 2601/2627. |
| Concurrency | `ROWVERSION` column on editable tables. Clients send it back on update, and a mismatch returns 409. |
| Paging | `OFFSET/FETCH` in SPs. The total count comes from a second result set (Dapper `QueryMultiple`). |
| Sorting | A whitelisted sort column handled with `CASE` in `ORDER BY`. Never dynamic SQL. |
| SP scripts | `CREATE OR ALTER PROCEDURE`, so every script can be rerun safely |
| Manual API testing | `.http` files per controller (Q13) |
| Auth | JWT bearer tokens issued by the API. Users are stored in our own tables and passwords are hashed with ASP.NET's `PasswordHasher` (Q6). |

**Database folder layout**
```
/database
  /Setup             00_CreateDatabase.sql
  /Tables            001_Department.sql, 002_JobRole.sql, 003_Employee.sql, 010_AppUser.sql ...
  /Types             IntList.sql (table-valued parameter)
  /StoredProcedures  usp_<Entity>_<Action>.sql (one per procedure)
  /Seed              Roles.sql, DemoData.sql
  deploy.ps1         runs Setup → Tables → Types → StoredProcedures → Seed via sqlcmd (Q7)
```

**Out of scope for v1:** refresh tokens, employee photos and file uploads, email notifications, change-history audit log (Q14), Docker for deploying the app (local SQL Server in Docker is in scope) and deployment pipelines.

---

## 2. Features by module

Project abbreviations: **Dom** = Domain, **App** = Application, **Infra** = Infrastructure, **Api**, **DB** = `/database` scripts, **Tests**, **Client** = `/client`.

### Module 0: Foundations
| ID | Feature | Projects | Stored procedures |
|---|---|---|---|
| FND-1 | Database setup: create-database script, `Department` / `JobRole` / `Employee` tables, `EmployeeNumber` SEQUENCE, indexes, `deploy.ps1`, demo seed data | DB | none |
| FND-2 | Data-access plumbing: add Dapper and Microsoft.Data.SqlClient, `ISqlConnectionFactory`, `ConnectionStrings:Default` from User Secrets, `SqlException` → Application exception mapping, `AddInfrastructure()` DI extension | Infra, App (exception types), Api (DI call) | none |
| FND-3 | API conventions: exception-to-ProblemDetails middleware, `AddApplication()` DI extension | Api, App | none |
| FND-4 | Test projects: `Domain.Tests` and `Application.Tests` (xUnit), added to the solution (Q9) | Tests | none |

### Module 2: Departments & Job Roles
| ID | Feature | Projects | Stored procedures |
|---|---|---|---|
| DEP-1 | Read departments: `GET /api/departments`, `GET /api/departments/{id}`. Includes the active employee count. | Dom, App, Infra, Api, DB | `usp_Department_GetAll`, `usp_Department_GetById` |
| DEP-2 | Write departments: `POST`, `PUT /{id}`, `DELETE /{id}`. Duplicate name → 409. A department with employees cannot be deleted → 409. | Dom, App, Infra, Api, DB | `usp_Department_Create`, `usp_Department_Update`, `usp_Department_Delete` |
| JOB-1 | Read job roles: `GET /api/job-roles`, `GET /api/job-roles/{id}` | Dom, App, Infra, Api, DB | `usp_JobRole_GetAll`, `usp_JobRole_GetById` |
| JOB-2 | Write job roles: create, update and delete, with the same rules as DEP-2 | Dom, App, Infra, Api, DB | `usp_JobRole_Create`, `usp_JobRole_Update`, `usp_JobRole_Delete` |

### Module 1: Employee Management
| ID | Feature | Projects | Stored procedures |
|---|---|---|---|
| EMP-1 | Employee domain model and get by id. `Employee` entity, `EmploymentStatus` enum, invariants (names required, valid email, hire date not in the future). `GET /api/employees/{id}` returns the department and job role names. | Dom, App, Infra, Api, DB, Tests | `usp_Employee_GetById` |
| EMP-2 | Create employee: `POST /api/employees` → 201. The SP checks that the department and job role exist, rejects a duplicate email with 409, and generates `EmployeeNumber` from the SEQUENCE (Q12). | Dom, App, Infra, Api, DB, Tests | `usp_Employee_Create` |
| EMP-3 | Update employee: `PUT /api/employees/{id}` with `rowVersion`. A stale version → 409. | Dom, App, Infra, Api, DB, Tests | `usp_Employee_Update` |
| EMP-4 | Delete employee: `DELETE /api/employees/{id}`. As a soft delete it sets `Status = Terminated` and `TerminationDate` (Q1). | Dom, App, Infra, Api, DB, Tests | `usp_Employee_Delete` |

### Module 3: Search & Filtering
| ID | Feature | Projects | Stored procedures |
|---|---|---|---|
| SRCH-1 | Paged, sorted employee list: `GET /api/employees?page&pageSize&sortBy&sortDir`. Adds a `PagedResult<T>` DTO. | App, Infra, Api, DB, Tests | `usp_Employee_Search` |
| SRCH-2 | Filters: free text (name, email, employee number), `departmentId`, `jobRoleId`, `status`, `hireDateFrom` / `hireDateTo`. Terminated employees are excluded unless the status filter asks for them. Uses `OPTION (RECOMPILE)` and supporting indexes. | App, Infra, Api, DB, Tests | `usp_Employee_Search` (altered) |

### Module 5: User Permissions
| ID | Feature | Projects | Stored procedures |
|---|---|---|---|
| AUTH-1 | Users and roles schema: `AppUser`, `AppRole`, `AppUserRole` tables, the `IntList` type, and a seed of the roles Admin, HR and Viewer (Q3) | DB | `usp_Role_GetAll` |
| AUTH-2 | Password hashing and first-admin bootstrap. `IPasswordHasher` lives in App, with an Infra implementation using `Microsoft.Extensions.Identity.Core`. On startup, if no users exist, the API creates an admin from the User Secrets value `Bootstrap:AdminPassword`. | App, Infra, Api, DB | `usp_User_Create`, `usp_User_GetCount` |
| AUTH-3 | Login → JWT: `POST /api/auth/login` and `GET /api/auth/me`. Adds `ITokenService` (App) and `JwtTokenService` (Infra), plus JwtBearer setup in Api. The signing key comes from User Secrets. | App, Infra, Api, DB, Tests | `usp_User_GetByUserName` (user + roles), `usp_User_UpdateLastLogin` |
| AUTH-4 | Authorization policies. A fallback policy requires an authenticated user. Policies: `CanManageEmployees` (Admin, HR), `CanViewSalary` (Admin, HR), `CanManageUsers` (Admin). They are applied to every existing controller. `ICurrentUser` (App) is implemented in Api, so App can hide salary from readers who lack the role (Q2). | Api, App, Tests | none |
| AUTH-5 | User management, part 1: `GET /api/users`, `GET /api/users/{id}`, `POST /api/users` (Admin only) | App, Infra, Api, DB, Tests | `usp_User_GetAll`, `usp_User_GetById` (reuses `usp_User_Create`) |
| AUTH-6 | User management, part 2: set roles, enable or disable a user, reset a password. Roles are passed as a table-valued parameter. | App, Infra, Api, DB, Tests | `usp_User_SetRoles`, `usp_User_SetActive`, `usp_User_UpdatePassword` |

### Module 4: Reports
| ID | Feature | Projects | Stored procedures |
|---|---|---|---|
| RPT-1 | Headcount by department and by job role: `GET /api/reports/headcount?groupBy=department\|jobRole` | App, Infra, Api, DB | `usp_Employee_GetHeadcountByDepartment`, `usp_Employee_GetHeadcountByJobRole` |
| RPT-2 | Hires and exits by month: `GET /api/reports/hires-exits?from&to` | App, Infra, Api, DB | `usp_Employee_GetHiresAndExitsByMonth` |
| RPT-3 | Salary summary (min, average and max) by department. Requires `CanViewSalary` and depends on Q2. | App, Infra, Api, DB | `usp_Employee_GetSalarySummaryByDepartment` |
| RPT-4 | CSV export of search results: `GET /api/employees/export?{same filters as search}`. Data comes from `EmployeeService`. The CSV is formatted in Api, with no package (Q11). | App, Api | reuses `usp_Employee_Search` |

Report SPs read employee data, so they use the `usp_Employee_*` prefix. That keeps the `usp_<Entity>_<Action>` rule instead of inventing a "Report" entity.

### Module 6: React UI (`/client`)
| ID | Feature | Projects | Stored procedures |
|---|---|---|---|
| UI-1 | Scaffold: Vite + React + TypeScript, React Router, TanStack Query, the UI library (Q10), a typed `fetch` wrapper that parses ProblemDetails, a Vite dev proxy `/api` → API (so no CORS config is needed), and the app shell with navigation | Client | none |
| UI-2 | Authentication: login page, `AuthContext`, token storage (Q6), bearer header, protected routes, role-aware navigation, logout on 401 | Client | none |
| UI-3 | Departments and job roles pages: list, plus dialogs to create, edit and delete | Client | none |
| UI-4 | Employee list: search box, filters, sorting and paging, with state kept in the URL query string | Client | none |
| UI-5 | Employee details and form: create, edit and terminate. Shows server validation errors and a concurrency-conflict message. | Client | none |
| UI-6 | Reports dashboard: charts (Q11) and a CSV export button | Client | none |
| UI-7 | User administration page (Admin only) | Client | none |

---

## 3. Build order

| # | Features | Why at this point |
|---|---|---|
| 1 | FND-1 | Every operation goes through SPs, and SPs need tables to exist first. |
| 2 | FND-2 | Every repository needs the connection factory and the SQL error mapping. It is verified by a clean build here and by its first real use in DEP-1. |
| 3 | FND-3 | The first endpoint should already return consistent ProblemDetails errors. |
| 4 | FND-4 | Each later feature then ships with its unit tests. |
| 5 | DEP-1 → DEP-2 → JOB-1 → JOB-2 | `Employee` has foreign keys to both tables. These are also the simplest entities, so the full Dom → App → Infra → Api → DB pattern gets set on easy ground. |
| 6 | EMP-1 → EMP-2 → EMP-3 → EMP-4 | This is the core module, and search and reports read its data. Get by id comes first so create can return a `Location` and update and delete have something to test against. |
| 7 | SRCH-1 → SRCH-2 | They need employees to exist. The UI list page and the RPT-4 export reuse this SP. |
| 8 | AUTH-1 → … → AUTH-6 | This comes once there are endpoints to protect. AUTH-4 is a small retrofit because controllers are thin. It must come before RPT-3 (salary policy) and UI-2 (login). |
| 9 | RPT-1 → RPT-4 | They need employee data, the search SP (for the export) and the salary policy. |
| 10 | UI-1 → UI-7 | The UI is built against a stable, secured API contract. UI-2 needs AUTH-3. Each page needs its backend module. |

## 4. Proposed database schema
All tables use the `dbo` schema. `CreatedAt` and `UpdatedAt` are `DATETIME2` UTC values defaulting to `SYSUTCDATETIME()`.

**Department**
- `DepartmentId` INT IDENTITY, primary key
- `Name` NVARCHAR(100), unique
- `Description` NVARCHAR(500) NULL
- `CreatedAt`, `UpdatedAt`, `RowVersion` ROWVERSION

**JobRole**
- `JobRoleId` INT IDENTITY, primary key
- `Title` NVARCHAR(100), unique
- `Description` NVARCHAR(500) NULL
- `CreatedAt`, `UpdatedAt`, `RowVersion`

**Employee**
- `EmployeeId` INT IDENTITY, primary key
- `EmployeeNumber` NVARCHAR(20), unique, generated from `seq_EmployeeNumber`, e.g. `EMP-000123`
- `FirstName`, `LastName` NVARCHAR(100)
- `Email` NVARCHAR(256), unique
- `Phone` NVARCHAR(30) NULL
- `HireDate` DATE
- `TerminationDate` DATE NULL
- `Status` TINYINT with a CHECK constraint: 1 Active, 2 OnLeave, 3 Terminated
- `DepartmentId` foreign key → Department
- `JobRoleId` foreign key → JobRole
- `Salary` DECIMAL(12,2) NULL (Q2)
- `CreatedAt`, `UpdatedAt`, `RowVersion`
- Indexes: (`LastName`, `FirstName`), `DepartmentId`, `JobRoleId`, `Status`, `HireDate`

**AppUser** (the name avoids the reserved word `User`)
- `UserId` INT IDENTITY, primary key
- `UserName` NVARCHAR(100), unique
- `Email` NVARCHAR(256)
- `PasswordHash` NVARCHAR(500)
- `IsActive` BIT
- `LastLoginAt` DATETIME2 NULL
- `CreatedAt`, `UpdatedAt`, `RowVersion`

**AppRole**
- `RoleId` INT, primary key
- `Name` NVARCHAR(50), unique; seeded with Admin, HR, Viewer

**AppUserRole**
- `UserId` foreign key → AppUser, `RoleId` foreign key → AppRole; the pair is the composite primary key

**Type `IntList`**
- One `Id INT` column. It passes role IDs to `usp_User_SetRoles`.

## 5. Definition of done (every feature)
1. `dotnet build EmployeeManagement.sln` finishes with no errors or warnings. For the client, `npm run build` and lint pass.
2. New SP and table scripts are in `/database` and deployed with `deploy.ps1`.
3. Unit tests for the new domain rules and service logic pass (`dotnet test`).
4. The endpoint has been exercised through its `.http` file, including the error paths (400 / 404 / 409, plus 401 / 403 once auth exists).
5. Work stops for review before the next feature starts.

## 6. Decisions

| # | Topic | Decision |
|---|---|---|
| Q1 | Deleting an employee | Soft delete: set `Status = Terminated` and `TerminationDate`. This keeps history for the hires and exits report. |
| Q2 | Salary | Stored in a nullable `Salary` column. Only Admin and HR can see it. RPT-3 stays in scope. |
| Q3 | Roles | Admin, HR and Viewer. No department-level restrictions in v1. |
| Q4 | Manager hierarchy | No `ManagerId` in v1. |
| Q5 | Linking a login account to an employee | Not linked in v1. |
| Q6 | Authentication | JWT with a 60-minute token stored in `sessionStorage`. No refresh token in v1. |
| Q7 | Database deployment | Plain scripts run by `deploy.ps1` through sqlcmd. |
| Q8 | Local SQL Server | SQL Server in Docker: container `sql-ems` at `localhost,1433` with SQL authentication. The password and the `ConnectionStrings:Default` value live in User Secrets and are never committed. |
| Q9 | Tests | xUnit unit test projects from FND-4. Integration tests can come later as a separate feature. |
| Q10 | React UI library | MUI. |
| Q11 | Charts and export | Recharts, and CSV export only. |
| Q12 | Employee number | Generated automatically from a SEQUENCE. |
| Q13 | Manual API testing | `.http` files. No extra package. |
| Q14 | Recording who changed a record | Not in v1. It could be added after AUTH-4. |
