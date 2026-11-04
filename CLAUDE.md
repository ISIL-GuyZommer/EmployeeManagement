# CLAUDE.md

## Project overview
Employee Management System: a learning project for practicing Clean Architecture end to end.
The backend is an ASP.NET Core Web API (.NET 10) on SQL Server. A React frontend will be added later in `/client`.
All data access goes through SQL Server Stored Procedures.

## Architecture
Solution: `EmployeeManagement.sln`. Projects live under `/src`.

| Project | Contains | May reference |
|---|---|---|
| `EmployeeManagement.Domain` | Entities, value objects, domain rules | Nothing |
| `EmployeeManagement.Application` | Use cases/services, DTOs, interfaces (e.g. `IEmployeeRepository`) | Domain |
| `EmployeeManagement.Infrastructure` | Interface implementations: SQL Server, Stored Procedure calls | Application |
| `EmployeeManagement.Api` | Controllers, DI setup, middleware, config | Application, Infrastructure |

- Dependencies point inward only. Never add a reference that breaks the table above.
- Application defines interfaces and Infrastructure implements them. Api registers them with DI.
- Domain has no NuGet packages and knows nothing about ASP.NET, SQL or HTTP.

## Coding conventions
- Naming: `PascalCase` for types, methods and properties. `camelCase` for locals and parameters.
  `_camelCase` for private fields. Interfaces start with `I`.
- Async: use async all the way. Async methods end in `Async` and accept a `CancellationToken`.
  Never use `.Result`, `.Wait()` or `async void`.
- Controllers stay thin: validate input, call one Application service, map the result to an HTTP response.
  No business logic and no data access in controllers.
- Business rules live in Domain/Application, not in Infrastructure or Api.
- Return DTOs from the API, never Domain entities.
- Nullable reference types are on. Don't suppress warnings with `!` without a reason.

## Data access rules
- **Stored Procedures only.** No inline SQL strings in C#, and no EF Core for queries.
- Use **Dapper** over **Microsoft.Data.SqlClient** (`SqlConnection`) to call SPs. Both packages belong in Infrastructure only.
- Call SPs from Infrastructure with `CommandType.StoredProcedure` and typed parameters.
  Never build SQL by string concatenation.
- SP naming: `usp_<Entity>_<Action>`, e.g. `usp_Employee_GetById`. Never use the `sp_` prefix (reserved by SQL Server).
- SP scripts live in `/database/StoredProcedures`, one file per procedure, saved in source control.
- The connection string comes from configuration (`ConnectionStrings:Default`). Never hard-code it or commit secrets.
  Use User Secrets locally.

## Common commands
```bash
dotnet build EmployeeManagement.sln
dotnet run --project src/EmployeeManagement.Api --launch-profile https
dotnet test EmployeeManagement.sln        # once test projects exist
```

- In Development, the OpenAPI document is at `/openapi/v1.json`.
- `global.json` locks the SDK to 10.0.302 (rollForward: latestFeature).

## Working rules for the agent
- **Plan first:** before any change that touches more than one project or adds a feature,
  show a short plan (files and approach) and wait for approval.
- **One feature at a time:** finish it, build it, then stop for review before starting the next.
- **Never commit or push unless explicitly asked.** Never force-push.
- Run `dotnet build` after changes and report errors honestly.
- Stay within the request. No unrequested refactors, packages or "extra" features.
- If a rule here conflicts with a request, point out the conflict and ask.
