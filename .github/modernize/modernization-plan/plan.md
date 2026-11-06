# Modernization Plan: Azure Modernization

**Project**: PhotoAlbum

---

## Technical Framework

- **Language**: C# targeting .NET 9 (`net9.0`)
- **Framework**: ASP.NET Core Razor Pages 9
- **Build Tool**: .NET SDK / MSBuild
- **Database**: SQL Server LocalDB through Entity Framework Core 9.0.9
- **Key Dependencies**: EF Core SQL Server and Design 9.0.9, ImageSharp 3.1.11, xUnit
- **Hosting today**: Docker image and Azure Container Apps deployment via azd/Bicep

As of November 6, 2026, .NET 9 is still supported but reaches end of support on
November 10, 2026. .NET 10 is the current LTS release. Reassess the runtime
target before implementation or production rollout; this plan does not add a
runtime-upgrade task because an upgrade was not explicitly requested.

---

## Overview

PhotoAlbum is a single ASP.NET Core application for uploading, browsing, and
deleting photos. It currently stores image files on the local filesystem and
photo metadata in SQL Server LocalDB. The repository already contains a
containerized Azure Container Apps deployment, Azure SQL and Blob Storage
resources, and a service interface that can support storage replacement.

The recommended target keeps the Razor Pages application as a single web
application and moves state and operations to managed Azure services:

- Host the application on Azure App Service with a system-assigned managed
  identity.
- Store photo objects in private Azure Blob Storage and metadata in Azure SQL.
- Keep non-secret settings in Azure App Configuration and secrets in Key Vault.
- Send application telemetry to Azure Monitor using OpenTelemetry.
- Reassess .NET 9 support before implementation; it reaches end of support
  shortly after this plan's creation date.

App Service is preferred for this web application over introducing AKS or
splitting the existing UI into Functions. API Management is not required for
the current Razor Pages interface; reconsider it if a separately consumed API,
central API policy, or external partner access becomes a requirement. The
repository's current Container Apps infrastructure is an alternative already
implemented in code, but it does not match the simpler App Service target.

---

## Current Architecture and Findings

### Architecture

- Browser requests are handled by Razor Pages in `PhotoAlbum/Pages`.
- `IPhotoService` abstracts photo operations, while `PhotoService` currently
  writes and reads files under `wwwroot/uploads`.
- `PhotoAlbumContext` persists photo metadata through EF Core and SQL Server.
- Authentication uses a cookie for administrative deletion. The admin password
  is read from configuration and login fails closed if it is absent.
- `Program.cs` creates the local upload directory and applies EF migrations at
  application startup outside the test environment.
- `Dockerfile`, `azure.yaml`, and `infra/main.bicep` currently configure
  deployment to Azure Container Apps, with SQL, Blob Storage, ACR, and Log
  Analytics resources.

### Technical debt and operational concerns

- Instance-local photo files are not durable or shared across scaled instances;
  photo reads load the whole file into memory.
- Startup-triggered database migrations can race during scale-out and require
  production runtime permissions to modify the schema.
- `appsettings.json` contains LocalDB and local upload-path configuration;
  `Web.config` also contains a legacy LocalDB connection string.
- Deployment configuration is duplicated across Bicep, azd, scripts, and
  documentation. Some documentation and infrastructure variable names differ.
- Current container infrastructure and the recommended App Service target
  differ; the deployment/IaC transition needs separate approval and planning.
- The upload handler accepts anonymous requests. Confirm the intended public
  upload policy and set suitable abuse, size, and cost controls before release.

### Security and assessment verification

- The assessment's **Local or network IO operations detected** finding is
  confirmed in startup, upload, deletion, and photo-serving code.
- **Local application configuration detected**, **SQL database connection
  detected**, and **Connection string is detected** are confirmed by the
  LocalDB and upload settings in `appsettings.json`.
- **Connection strings without configuration builders detected** refers to the
  LocalDB string in `Web.config`. The current app uses
  `WebApplication.CreateBuilder`; confirm the legacy file is not needed before
  removing it.
- The reported **Windows authentication detected** finding was not confirmed:
  the current `Web.config` only contains a connection string and has no Windows
  authentication setting.
- The reported **Hardcoded sensitive data detected** finding was not confirmed
  in `Login.cshtml.cs`: the password is read from configuration and a missing
  password rejects login. The default username is `admin`, not a password.
- **Static content detected** is present and expected for a Razor Pages
  application; it is not treated as a cloud-readiness defect in this plan.
- Additional infrastructure risks not listed by the assessment exist in
  `infra/main.bicep`: ACR admin access is enabled, Blob public access and broad
  network access are enabled, SQL allows Azure-service firewall access, and a
  deterministic SQL bootstrap password is composed in the template. Restrict
  access and move credential handling to a managed, secret-safe mechanism.

---

## Target Azure Architecture

1. **Web tier — Azure App Service**: Run the ASP.NET Core Razor Pages app on
   App Service. Assign a system-managed identity and use HTTPS.
2. **Photo objects — Azure Blob Storage**: Store objects in a private container.
   The application identity receives only the Blob data-plane role it needs.
3. **Photo metadata — Azure SQL Database**: Keep EF Core metadata persistence
   and authenticate the application with Microsoft Entra ID/managed identity.
4. **Configuration and secrets**: Use Azure App Configuration for non-secret
   settings and Key Vault for any required secrets. Access them with managed
   identity; do not place credentials in source files or deployment outputs.
5. **Observability — Azure Monitor**: Instrument the application with
   OpenTelemetry and export logs, metrics, and traces to Azure Monitor.
6. **Networking and supply chain**: Disable public Blob access, restrict
   network paths where practical, remove unnecessary ACR admin credentials if
   the container deployment is retained, and scan dependencies for CVEs.

The target does not require Azure Functions, AKS, or API Management for the
current application shape. The existing Container Apps deployment can remain
an interim option, but changing its Bicep/azd deployment to App Service is
outside the requested planning-only task list until deployment scope is
approved.

---

## Migration Impact Summary

| Application | Original Service | New Azure Service | Authentication | Comments |
|---|---|---|---|---|
| PhotoAlbum | Local photo files | Azure Blob Storage | Managed identity | Private photo container |
| PhotoAlbum | SQL Server LocalDB | Azure SQL Database | Managed identity | Retain EF Core data access |
| PhotoAlbum | Local app settings | App Configuration / Key Vault | Managed identity | Separate settings and secrets |
| PhotoAlbum | Console/app logs | Azure Monitor | Managed identity | OpenTelemetry signals |
| PhotoAlbum | Container Apps | Azure App Service | Managed identity | Recommended hosting target |

---

## Prioritized Phases, Dependencies, and Risks

| Priority | Phase | Affected areas | Dependency | Key risks |
|---|---|---|---|---|
| P0 | Baseline and verification | `PhotoAlbum/`, `PhotoAlbum.Tests/` | None | Existing behavior may rely on local storage |
| P1 | Photo storage modernization | `PhotoService`, `IPhotoService`, `PhotoFile` | Baseline | Blob access, metadata/file consistency, cutover |
| P1 | Database modernization | `PhotoAlbumContext`, configuration, startup | Storage design | Identity permissions, schema migration timing |
| P1 | Configuration and secrets | `Program.cs`, appsettings, `Web.config` | Azure identities | Missing settings or accidental secret exposure |
| P2 | Observability | `Program.cs`, project dependencies | Configuration | Telemetry volume and sensitive data in logs |
| P2 | Security and verification | NuGet dependencies, `infra/main.bicep`, tests | Modernization changes | Breaking package updates and Azure-only behavior |
| P3 | Hosting transition decision | `Dockerfile`, `azure.yaml`, `infra/`, deployment docs | Target approval | App Service and current Container Apps setup differ |

The P3 row records the required follow-up for a complete rollout. Resource
provisioning, IaC generation, and application deployment are not included as
executable tasks because this request asks for a plan, not for infrastructure
or deployment changes.

---

## Task Scope and Decisions

The task list captures baseline verification, the requested Azure service
modernizations, security/CVE review, and post-migration verification. Existing
assessment findings are named in the relevant task descriptions. Assessment
findings that could not be verified against current files are documented above
and are not represented as confirmed remediation work.

No separate .NET upgrade, infrastructure provisioning, or deployment task is
included. The immediate .NET 9 support deadline and the App Service transition
should be revisited before implementation begins.
