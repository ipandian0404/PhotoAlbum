# Modernization Plan: PhotoAlbum Azure Modernization

**Project**: PhotoAlbum

---

## Technical Framework

- **Language**: C#
- **Framework**: ASP.NET Core Razor Pages, .NET 9 (`net9.0`)
- **Build Tool**: MSBuild / .NET SDK
- **Database**: SQL Server LocalDB with Entity Framework Core 9
- **Key Dependencies**: EF Core SQL Server and Design 9.0.9, ImageSharp 3.1.11; xUnit and ASP.NET Core MVC Testing 9.0.9

---

## Overview

> PhotoAlbum is a Razor Pages photo gallery that currently stores photo metadata in SQL Server and image files on the local filesystem. The repository already contains a Docker image and Bicep/Azure Developer CLI configuration for Azure Container Apps, Azure SQL, and Blob Storage, but the application still performs photo-file operations locally and its infrastructure does not target the preferred App Service hosting model.
>
> The modernization will:
>
> - Move photo files to Azure Blob Storage so uploads survive application restarts and scale-out.
> - Connect the application to Azure SQL using managed identity and move runtime secrets to Key Vault.
> - Host the existing web application on Azure App Service and add Azure Monitor visibility.
>
> The plan starts with a frozen behavior baseline, makes the application changes in dependent phases, verifies them with mocked integration tests, reviews dependencies and secrets, and then deploys to App Service.

---

## Migration Impact Summary

| Application | Original Service | New Azure Service | Authentication | Comments |
|-------------|------------------|-------------------|----------------|----------|
| PhotoAlbum | Local file storage | Azure Blob Storage | Managed identity | Persist uploaded photos |
| PhotoAlbum | SQL Server LocalDB | Azure SQL Database | Managed identity | Keep EF Core data model |
| PhotoAlbum | Local configuration | App Service + Key Vault | Managed identity | Settings and admin secret |
| PhotoAlbum | Container Apps | Azure App Service | Managed identity | Preferred simple web hosting |
| PhotoAlbum | Console/application logs | Azure Monitor | Managed identity | Centralized telemetry |

---

## Current Architecture and Findings

- ASP.NET Core Razor Pages serves the gallery, login, upload, and photo-file routes. `PhotoService` currently writes and deletes files under `wwwroot/uploads`; the photo endpoint reads them back from the local filesystem.
- EF Core stores photo metadata in SQL Server. `appsettings.json` and the legacy `Web.config` contain LocalDB connection strings; `Program.cs` reads `ConnectionStrings:DefaultConnection` and applies migrations during startup.
- The repository has a Dockerfile, `azure.yaml`, and Bicep templates for Container Apps, Azure SQL, Blob Storage, a registry, and Log Analytics. This is useful existing deployment work, but it is not an App Service deployment configuration and the application does not yet use Blob Storage.
- The app targets .NET 9. It is supported on the plan date but reaches end of support on November 10, 2026; deployment timing should account for the short remaining support window. No .NET upgrade task is included because the current framework is not yet out of support and an upgrade was not explicitly requested.
- Technical debt includes coupling photo persistence and retrieval to the local filesystem, legacy LocalDB configuration, and a deployment topology that differs from the preferred App Service target.
- The assessment findings confirmed in the current tree are local/network I/O, local application configuration, a connection string, SQL database configuration, a trusted LocalDB connection in `Web.config`, and static assets. The reported hard-coded sensitive data in `Login.cshtml.cs` was not confirmed: the current code reads the admin password from configuration and fails closed when it is absent. Retain a security review to recheck secrets and dependencies without assuming that report finding is valid.
- Bicep currently enables public blob access and Azure-wide SQL firewall access; these should be tightened during deployment review. Its generated SQL administrator credential should not become an application secret.

### Target Architecture

```text
Browser
  └─ HTTPS → Azure App Service (ASP.NET Core Razor Pages)
                ├─ Managed identity → Azure SQL Database
                ├─ Managed identity → Azure Blob Storage (private container)
                ├─ Managed identity → Azure Key Vault (admin credential)
                └─ OpenTelemetry → Azure Monitor / Application Insights
```

Azure Functions and API Management are not proposed: the current app has no independent background workload or separate API gateway requirement. Container Apps is also not the recommended host for this plan; the repository’s current Container Apps configuration must be adapted or replaced for App Service. Keep the gallery’s static assets served by the web app unless future requirements call for a CDN.

---

## Prioritized Phases

1. **Baseline and application transformation** — Capture existing behavior, then migrate photo persistence and runtime configuration to Azure services. Main files: `PhotoAlbum/Services/PhotoService.cs`, `PhotoAlbum/Pages/PhotoFile.cshtml.cs`, `PhotoAlbum/Program.cs`, `PhotoAlbum/appsettings.json`, `PhotoAlbum/Web.config`, and relevant tests. Risks: preserving upload validation, cleanup behavior, file metadata, and existing photo URLs while changing storage.
2. **Operational visibility and verification** — Add Azure Monitor instrumentation and run mocked integration verification after all application transforms. Main files: `PhotoAlbum/Program.cs`, `PhotoAlbum/PhotoAlbum.csproj`, and `PhotoAlbum.Tests/`. Dependency: phase 1. Risk: telemetry configuration must avoid recording credentials or sensitive upload data.
3. **Security and dependency review** — Scan dependencies for CVEs and confirm application/deployment configuration has no exposed secrets or overly broad access. Main files: project files, `infra/main.bicep`, and application configuration. Dependency: phase 2. Risk: remediation may require package updates or infrastructure access-policy changes.
4. **App Service deployment** — Adapt the current `infra/`, `azure.yaml`, and deployment documentation/configuration for App Service, managed identity, private Blob Storage, Azure SQL, Key Vault, and monitoring. Dependency: phase 3. Risk: current Container Apps assumptions and container-only hooks do not directly match App Service; validate migration and rollback before cutover.

The task-level files and dependencies are listed in `.metadata/tasks.json`.

---

## Open Questions & Questionnaire

- [x] Q: Should the plan provision new infrastructure? → A: No separate infrastructure-generation task; adapt the repository’s existing deployment configuration for the selected host.
- [x] Q: Should integration testing be included? → A: Yes, mocked-dependency mode (inferred default because no test infrastructure was specified).
- [x] Q: Should security/CVE remediation be included? → A: Yes, include the required dependency and secret review.
- [x] Q: Which Azure deployment target should be used? → A: Azure App Service, matching the project preference for simple App Service deployment over AKS.
- [x] Q: Should containerization be included? → A: No; use the App Service .NET hosting model rather than retaining a container-only deployment.
