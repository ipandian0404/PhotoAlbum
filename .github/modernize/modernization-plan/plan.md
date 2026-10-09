# Modernization Plan: PhotoAlbum Azure Modernization

**Project**: PhotoAlbum

## Technical Framework

- **Language**: C#
- **Framework**: ASP.NET Core Razor Pages, currently `net9.0`; target `net10.0`
- **Build Tool**: MSBuild / .NET SDK
- **Database**: SQL Server via EF Core; development currently uses LocalDB
- **Key Dependencies**: EF Core SQL Server and Design, ImageSharp, xUnit,
  ASP.NET Core MVC Testing, and EF Core InMemory

## Overview

PhotoAlbum is a server-rendered photo gallery with database-backed photo
metadata and local-disk image storage. The repository already contains Azure
Container Apps, Azure SQL, Blob Storage, Container Registry, and Log Analytics
provisioning, but the application still stores files on disk and does not
consume the configured Blob Storage endpoint.

The modernization will upgrade the runtime and packages, move photo files and
database access to managed Azure services, protect runtime secrets, and provide
cloud monitoring. The target hosting preference is Azure App Service rather
than AKS. Existing Container Apps infrastructure must be reconciled with that
target before deployment; no new environment provisioning is assumed.

The work is phased: establish a baseline, upgrade the runtime and packages,
migrate storage/database/configuration and monitoring, verify the changes,
complete security review, and then deploy to an approved App Service.

## Current Architecture and Assessment

- ASP.NET Core 9 Razor Pages, EF Core SQL Server, ImageSharp, and xUnit.
- `PhotoService` reads and writes uploaded photos under `wwwroot/uploads`;
  metadata is stored in SQL Server. The default connection string is LocalDB.
- `infra/main.bicep` currently defines Container Apps, SQL Database, Blob
  Storage, ACR, and Log Analytics. It configures managed identity for Azure
  resources, but the application has no Blob Storage integration.
- The local `Web.config` contains a legacy LocalDB connection string and no
  Windows-authentication setting. `Login.cshtml.cs` reads the password from
  configuration and has no hard-coded password; the assessment's Windows
  authentication and hard-coded-sensitive-data findings were not reproduced.
- Confirmed assessment findings include local/network file I/O, local
  application configuration, SQL connection strings, and static content.
  Static assets are expected for this Razor Pages application. The assessment's
  connection-string-without-configuration-builder finding refers to the
  legacy `Web.config` entry, which should be removed or kept out of runtime use.
- Additional infrastructure risks to review include the SQL administrator
  password expression in Bicep, enabled ACR admin credentials, broadly
  accessible storage networking, and the SQL firewall rule allowing Azure
  services.

## Target Azure Architecture

- **Azure App Service** for the existing Razor Pages monolith, honoring the
  project's preference for a simple App Service deployment. Do not introduce
  AKS, Functions, or API Management: the current app has no separate
  event-driven workload or API gateway requirement.
- **Azure SQL Database** for photo metadata, using App Service managed identity
  and Microsoft Entra authentication instead of LocalDB or SQL credentials.
- **Azure Blob Storage** for uploaded images, using managed identity and
  private container access. Preserve the existing upload validation and
  database/file consistency behavior while migrating any existing image files.
- **Azure Key Vault** for the configured admin password and any remaining
  runtime secrets, accessed with managed identity. Keep non-secret settings in
  App Service configuration.
- **Azure Monitor and Application Insights** with OpenTelemetry for application
  logs, metrics, traces, and availability diagnostics.
- Retain static assets in the web application. Adapt the existing IaC and
  identity assignments to the approved App Service target and apply least
  privilege and appropriate network restrictions.

## Prioritized Phases

1. **Baseline and platform update** — capture existing behavior, move the
   application and test projects to .NET 10 LTS, and update NuGet dependencies.
2. **Data and configuration modernization** — migrate local image storage to
   Blob Storage, use Azure SQL with managed identity, and protect the admin
   secret with Key Vault.
3. **Operations and verification** — add Azure Monitor/OpenTelemetry
   instrumentation, verify the migrated services with mocked integration
   dependencies, and scan/remediate dependency vulnerabilities.
4. **Hosting transition** — reconcile the existing Container Apps Bicep
   resources with the App Service target and deploy only after an App Service
   environment and resource ownership are approved.

Task-level requirements, affected files, dependencies, and success criteria
are in `.metadata/tasks.json`.

## Affected Files and Components

The work is expected to affect `PhotoAlbum/PhotoAlbum.csproj`,
`PhotoAlbum.Tests/PhotoAlbum.Tests.csproj`, `PhotoAlbum/Program.cs`,
`PhotoAlbum/Services/PhotoService.cs`, `PhotoAlbum/Services/IPhotoService.cs`,
`PhotoAlbum/Pages/PhotoFile.cshtml.cs`, `PhotoAlbum/Data/PhotoAlbumContext.cs`,
`PhotoAlbum/Models/Photo.cs`, `PhotoAlbum/appsettings*.json`,
`PhotoAlbum/Web.config`, test files under `PhotoAlbum.Tests/`, and
`infra/main.bicep` plus deployment configuration.

## Dependencies and Risks

- Upgrade the app and test projects before Azure-service changes; run existing
  tests after each phase and preserve compatibility with existing photo
  metadata and URLs.
- Blob migration must account for existing uploaded files and avoid losing
  image data if database updates fail or are retried.
- The current Bicep targets Container Apps while the requested project
  preference is App Service. Validate the App Service plan, deployment
  environment, network access, identity permissions, and rollback approach
  before replacing or deploying infrastructure.
- Removing connection strings and changing managed identity assignments can
  affect local development and deployment; retain a safe local development
  path without committing secrets.
- Review exposure and secret handling in Bicep and enforce private Blob Storage
  access, least-privilege identities, and restricted SQL/network access.
- Updating ImageSharp and EF Core may introduce behavioral or migration
  compatibility changes; validate image formats, database migrations, and
  baseline tests.

## Open Questions & Questionnaire

- [x] Q: Should infrastructure be provisioned? → A: No new provisioning is
  assumed; use the repository's existing IaC as the starting point and
  reconcile its Container Apps target with App Service before deployment.
- [x] Q: Should integration testing be included? → A: Yes, use mocked Azure
  dependencies until an approved App Service environment is available.
- [x] Q: Should security/CVE remediation be included? → A: Yes.
- [x] Q: Which Azure deployment target should be used? → A: Azure App Service,
  consistent with the project's stated preference; do not use AKS.
- [x] Q: Is separate containerization required? → A: No separate
  containerization task; use the App Service deployment path.
- [ ] Which App Service environment, resource group, and deployment region are
  approved?
