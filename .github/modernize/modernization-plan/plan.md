# Modernization Plan: PhotoAlbum Azure Modernization

**Project**: PhotoAlbum

---

## Technical Framework

- **Language**: C#
- **Framework**: ASP.NET Core Razor Pages, .NET 9 (`net9.0`)
- **Build Tool**: .NET SDK / MSBuild
- **Database**: SQL Server LocalDB in local configuration; Azure SQL Database
  is already described in the repository's Bicep deployment
- **Key Dependencies**: Entity Framework Core 9.0.9, SQL Server EF provider
  9.0.9, ImageSharp 3.1.11, xUnit, and ASP.NET Core MVC testing

---

## Overview

> PhotoAlbum is a Razor Pages application for uploading and displaying photos.
> Its current source still writes uploaded photos to local disk and reads a
> LocalDB connection string, while its deployment definition already describes
> Azure Container Apps, Azure SQL Database, and Blob Storage. The modernization
> aligns the application with a supported .NET runtime and a secure, resilient
> Azure hosting model. The recommended target is Azure App Service, in keeping
> with the project's stated preference for a simple App Service deployment.
>
> - Move photo files from instance-local disk to private Azure Blob Storage so
>   uploads remain available across instances and restarts.
> - Use Azure SQL Database with managed identity and move secrets and
>   environment-specific settings out of source-controlled local configuration.
> - Replace the single local admin sign-in with Microsoft Entra ID and provide
>   centralized telemetry through Azure Monitor.
>
> Work proceeds from the .NET 10 and dependency upgrade, through hosting and
> service changes, then security remediation and post-migration verification.
> This document is a plan only: it does not provision resources or deploy the
> application.

---

## Current Architecture and Findings

- ASP.NET Core Razor Pages and EF Core run on .NET 9. The app and test projects
  both target `net9.0`; the Dockerfile uses .NET 9 images.
- `PhotoService` and the photo file page use local filesystem operations for
  uploads, retrieval, and deletion.
- The app's local configuration and legacy `Web.config` contain LocalDB
  connection strings. The Bicep deployment separately describes Azure SQL
  Database, a Storage Account, and a Container App with managed identity.
- `azure.yaml` selects Container Apps and Bicep, whereas the preferred target
  for this plan is App Service. Existing IaC is a starting point, not evidence
  that these resources are already provisioned.
- The storage resource currently allows public network access and sets
  `allowBlobPublicAccess` to `true` (the photo container itself is private).
  The ACR admin user is enabled, and the SQL server has an Azure-services
  firewall rule. Reassess these defaults and use least privilege and restricted
  network access in the target design.
- The SQL administrator password in Bicep is derived from resource-group and
  environment values. Treat this as a credential-management risk; avoid
  relying on it as a secret and use a secure bootstrap or managed-identity
  administration approach.
- The assessment's local/network I/O, local configuration, connection-string,
  SQL connection, and legacy connection-string findings are confirmed in the
  current source/configuration and are mapped to tasks below.
- The reported Windows-authentication finding is not confirmed: `Web.config`
  contains only a LocalDB connection string and no Windows-authentication
  setting. The reported hardcoded-sensitive-data finding is also not
  confirmed as a hardcoded password: `LoginModel` has a default admin
  *username*, but fails closed unless an admin password is supplied through
  configuration. The static-content finding reflects the app's normal web
  assets and is not treated as a defect.

---

## Recommended Azure Target Architecture

- **Web application**: Azure App Service for Linux, running the .NET 10
  application. Retire the Container Apps/ACR hosting definition for this
  application after the target is approved; retain the Dockerfile only if a
  later deployment choice requires it.
- **Photo storage**: Azure Blob Storage with a private container, application
  managed identity, and narrowly scoped Blob data permissions. Serve files
  through the application or a deliberately designed private/CDN access path;
  do not make the container public by default.
- **Relational data**: Azure SQL Database with EF Core 10 and App Service
  managed identity. Keep the current photo metadata model and migrations unless
  schema changes are demonstrated as necessary.
- **Identity and configuration**: Microsoft Entra ID for administrator
  authentication; Azure Key Vault for any remaining secrets; Azure App
  Configuration for non-secret environment settings. Access these through
  managed identity and remove stale LocalDB settings from deployed
  configuration.
- **Operations**: Azure Monitor and Application Insights using OpenTelemetry
  for application logs, metrics, and traces, with actionable availability and
  failure alerts.
- **Not recommended for this app's current shape**: Azure Functions would add
  an unnecessary second compute model without a background/event workload;
  API Management is not justified for a Razor Pages-only application without
  an independently consumed API. Revisit either if those product needs arise.

The existing Bicep and `azure.yaml` describe Container Apps, SQL, and Blob
resources, not an already-provisioned App Service environment. The plan
recommends aligning that hosting configuration, but does not include a separate
resource-provisioning or deployment task.

---

## Migration Impact Summary

| Application | Original Service | New Azure Service | Authentication | Comments |
|-------------|------------------|-------------------|----------------|----------|
| PhotoAlbum | .NET 9 | .NET 10 | N/A | Upgrade app, tests, and image |
| PhotoAlbum | Local disk | Azure Blob Storage | Managed identity | Keep container private |
| PhotoAlbum | LocalDB/configured SQL | Azure SQL Database | Managed identity | Existing SQL target in Bicep |
| PhotoAlbum | Container Apps definition | Azure App Service | Managed identity | Preferred simple hosting target |
| PhotoAlbum | Local admin sign-in | Microsoft Entra ID | Entra ID | Protect admin operations |
| PhotoAlbum | Framework logging | Azure Monitor | Managed identity | OpenTelemetry telemetry |

---

## Modernization Phases

1. **Runtime and dependency foundation**: upgrade both application and test
   projects to .NET 10, align compatible framework dependencies, and update
   runtime/container definitions.
2. **Azure hosting and persistence alignment**: align the hosting definition
   with App Service; replace local photo-file operations with Blob Storage;
   configure Azure SQL and application settings for managed identity.
3. **Identity and operations**: use Microsoft Entra ID for admin access,
   separate secrets from non-secret configuration, and add Azure Monitor
   telemetry.
4. **Security and verification**: scan and remediate dependency CVEs, then run
   the planned integration verification against mocked Azure dependencies.

The detailed, dependency-ordered implementation work and affected files are in
`.metadata/tasks.json`.

---

## Risks and Decisions

- .NET 10 and EF Core 10 may expose package or behavior incompatibilities;
  validate the full test suite and review database provider compatibility
  before rollout.
- Moving existing photo records to Blob Storage requires a migration strategy
  for previously uploaded local files and a rollback plan; the database
  metadata and blob object must remain consistent on upload and deletion.
- App Service and the current Container Apps definition differ in deployment,
  configuration, networking, and scaling. Confirm the hosting preference and
  required network boundaries before changing or provisioning resources.
- Azure service integration depends on correct managed-identity role
  assignments and SQL database permissions. Avoid storage account keys and
  long-lived credentials.
- Integration verification is planned in mock mode because no provisioned
  target environment or resource identifiers were provided. Real Azure
  end-to-end verification remains a release decision.

---

## Open Questions & Questionnaire

- [x] Environment setup → Reuse repository-defined Azure configuration as
  context; do not provision resources in this planning-only request.
- [x] Integration testing → Include verification in mock mode; no provisioned
  environment was identified.
- [x] Security/CVE remediation → Include the required dependency scan and
  remediation phase.
- [x] Deployment target → Azure App Service, consistent with the project's
  previously stated preference; no deployment is requested in this task.
- [x] Containerization → No separate containerization task for the App Service
  target; retain/revisit the Dockerfile only if the final deployment approach
  requires it.
