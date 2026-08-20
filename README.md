<div align="center">

<img src="assets/app-store-connect-mcp-hero.jpg" alt="App Store Connect MCP server — the full App Store Connect API connected to Claude and other AI agents over the Model Context Protocol" width="100%">

# App Store Connect MCP Server

**The complete App Store Connect API, as an MCP server.**
1,263 operations generated straight from Apple's official OpenAPI specification, exposed to Claude Code, Claude Desktop, Cursor and every other Model Context Protocol client through just **4 token-efficient tools**.

[![CI](https://img.shields.io/github/actions/workflow/status/imfaisii/asc-mcp/ci.yml?branch=main&style=flat-square&label=CI)](https://github.com/imfaisii/asc-mcp/actions/workflows/ci.yml)
[![Model Context Protocol](https://img.shields.io/badge/Model_Context_Protocol-server-0A7CFF?style=flat-square)](https://modelcontextprotocol.io)
[![App Store Connect API 4.4.1](https://img.shields.io/badge/App_Store_Connect_API-4.4.1-000000?style=flat-square&logo=apple&logoColor=white)](https://developer.apple.com/documentation/appstoreconnectapi)
[![1263 operations](https://img.shields.io/badge/operations-1263-1f6feb?style=flat-square)](#coverage-every-app-store-connect-resource)
[![4 MCP tools](https://img.shields.io/badge/MCP_tools-4-2ea043?style=flat-square)](#the-four-mcp-tools)
[![Bun](https://img.shields.io/badge/Bun-1.1+-f472b6?style=flat-square&logo=bun&logoColor=white)](https://bun.sh)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.8-3178c6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

</div>

---

## What is the App Store Connect MCP server?

`app-store-connect-mcp` is an open source **Model Context Protocol (MCP) server for the App Store Connect API**. Install it once and your AI agent can read and write everything Apple exposes for your apps and your developer account:

App Store listings and metadata · TestFlight builds, beta groups and testers · in-app purchases · auto-renewable subscriptions, offers and win-back offers · Game Center leaderboards, achievements and matchmaking · Xcode Cloud workflows and build runs · provisioning profiles, certificates, bundle IDs and devices · App Review submissions and review details · customer reviews and responses · pricing, availability and territories · sales, finance and analytics reports · users, roles and invitations · App Clips, background assets and alternative distribution.

Most App Store Connect MCP servers hand-wrap two or three dozen endpoints and then drift out of date. This one is **generated from Apple's published OpenAPI specification**, so coverage is the entire documented surface and it stays current with a single `bun run generate`.

> **No credentials live in this repository.** You supply your own App Store Connect API key at runtime through environment variables. Nothing is bundled, logged or transmitted anywhere except Apple.

### Table of contents

- [At a glance](#at-a-glance)
- [Why 4 tools instead of 1,263](#why-4-tools-instead-of-1263)
- [The four MCP tools](#the-four-mcp-tools)
- [Coverage: every App Store Connect resource](#coverage-every-app-store-connect-resource)
- [Requirements](#requirements)
- [Get your App Store Connect API key](#get-your-app-store-connect-api-key)
- [Install in Claude Code](#install-in-claude-code)
- [Install in Claude Desktop](#install-in-claude-desktop)
- [Install in Cursor, Windsurf, VS Code and other MCP clients](#install-in-cursor-windsurf-vs-code-and-other-mcp-clients)
- [Environment variables](#environment-variables)
- [How an agent uses it](#how-an-agent-uses-it)
- [Example prompts](#example-prompts)
- [Scripts](#scripts)
- [Refresh Apple's OpenAPI specification](#refresh-apples-openapi-specification)
- [Project layout](#project-layout)
- [Security model](#security-model)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)

---

## At a glance

| | |
| --- | --- |
| **What it is** | MCP server for the full App Store Connect API |
| **API version** | App Store Connect API **4.4.1** (OpenAPI 3.0.1) |
| **Operations in catalog** | **1,263** — one per path + method |
| **API paths** | **966** |
| **Resource tags** | **195** |
| **MCP tools registered** | **4** meta-tools: `asc_search`, `asc_schema`, `asc_call`, `asc_tags` |
| **Method split** | 797 `GET` · 176 `POST` · 158 `PATCH` · 132 `DELETE` |
| **Source of truth** | Apple's OpenAPI zip, vendored at `openapi/app-store-connect.openapi.json` |
| **Auth** | ES256 JWT built at runtime from Issuer ID + Key ID + `.p8` private key |
| **Base URL** | `https://api.appstoreconnect.apple.com` (overridable) |
| **Transport** | stdio |
| **Runtime** | [Bun](https://bun.sh) 1.1+ and [`@modelcontextprotocol/sdk`](https://www.npmjs.com/package/@modelcontextprotocol/sdk) |
| **License** | MIT |

---

## Why 4 tools instead of 1,263

<img src="assets/app-store-connect-mcp-token-efficiency.jpg" alt="Diagram: 1,263 App Store Connect API operations compressed through a search-and-schema layer down to 4 MCP tools, saving hundreds of thousands of context tokens" width="100%">

Registering every OpenAPI operation as its own MCP tool dumps 1,263 full JSON Schemas into the model's context window. That is on the order of **hundreds of thousands of tokens burned before the agent does any real work** — and many clients simply refuse or truncate a tool list that large.

This server keeps the full catalog **internal** and exposes a small, stable surface on top of it:

```text
asc_search  →  find operations           (names, methods, paths, tags — no schemas)
asc_schema  →  describe one operation    (full input JSON Schema + call hint)
asc_call    →  execute one operation     (real request, or _dryRun)
asc_tags    →  list resource tags        (with operation counts, to narrow search)
```

The agent pays for exactly the schema it is about to use, and nothing else. A typical three-step flow costs a few thousand tokens instead of a few hundred thousand.

### Escape hatch (debugging only)

```bash
export ASC_EXPOSE_ALL_TOOLS=1   # also register all 1,263 operations as individual MCP tools
```

Leave this **unset** in normal use. It exists for spec debugging and it is extremely expensive in context.

---

## The four MCP tools

### `asc_search` — find the right operation

Free-text search over operationId, name, path, tags, method and description. Returns ranked hits without the JSON Schemas.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `query` | string | optional* | Free text, e.g. `"list apps"`, `"beta testers"`, `"sales reports"`, `"/v1/builds"` |
| `tag` | string | optional* | Exact OpenAPI tag filter, e.g. `Apps`, `Builds`, `BetaGroups` |
| `method` | string | optional | `GET`, `POST`, `PATCH`, `DELETE`, `PUT` |
| `limit` | integer | optional | Max hits, default `15`, max `50` |

\* At least one of `query`, `tag` or `method`.

```jsonc
{ "query": "list apps", "method": "GET", "limit": 5 }
```

### `asc_schema` — describe one operation

Returns the operation's full input JSON Schema, its path parameter list, its query parameter list, and a `callHint` showing how to invoke it.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `operation` | string | **yes** | OperationId or generated tool name, e.g. `apps_getCollection` |

### `asc_call` — execute the operation

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `operation` | string | **yes** | OperationId or generated tool name |
| `args` | object | optional | Path params, query params, `body`, `_dryRun`. Additional properties allowed |
| `_dryRun` | boolean | optional | Resolve method/path/query/body **without** calling Apple |

Path and query fields work either nested inside `args` or as top-level keys next to `operation`. JSON:API writes go in `body`.

```jsonc
// read
{ "operation": "apps_getCollection", "args": { "limit": 10, "fields[apps]": "name,bundleId,sku" } }

// dry run — no network call, shows the resolved request
{ "operation": "apps_getCollection", "_dryRun": true, "args": { "limit": 10 } }

// write (JSON:API document)
{
  "operation": "betaGroups_createInstance",
  "args": {
    "body": {
      "data": {
        "type": "betaGroups",
        "attributes": { "name": "Internal QA" },
        "relationships": { "app": { "data": { "type": "apps", "id": "1234567890" } } }
      }
    }
  }
}
```

### `asc_tags` — orient inside the API

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `limit` | integer | optional | Max tags, default all 195, sorted by operation count |

### Response handling

- JSON responses are returned parsed.
- CSV, XML and text responses are returned as text.
- Gzip and binary downloads (finance reports, sales reports) come back as base64 with `encoding`, `contentType`, `byteLength` and a decode note.
- Responses over ~120,000 characters are truncated with a hint to narrow `filter[…]`, `fields[…]` or `limit`.
- HTTP 4xx/5xx are surfaced as MCP errors with Apple's own error body intact, so the agent can read Apple's `detail` string and self-correct.

---

## Coverage: every App Store Connect resource

All 1,263 operations across 195 resource tags. Counts are operations per tag.

### Apps, listings and App Store metadata — ~250 operations

`Apps` (86) · `AppStoreVersions` (30) · `AppInfos` (20) · `AppCustomProductPageLocalizations` (12) · `AppStoreVersionExperiments` (12) · `AppStoreVersionLocalizations` (12) · `AppEncryptionDeclarations` (8) · `AppEventLocalizations` (8) · `AppStoreVersionExperimentTreatmentLocalizations` (7) · `AppCategories` (6) · `AppCustomProductPages` (6) · `AppEvents` (6) · `AppPreviewSets` (6) · `AppScreenshotSets` (6) · `AppStoreVersionExperimentTreatments` (6) · `EndUserLicenseAgreements` (6) · `AppCustomProductPageVersions` (5) · `AppInfoLocalizations` (4) · `AppPreviews` (4) · `AppScreenshots` (4) · `AppEventScreenshots` (4) · `AppEventVideoClips` (4) · `AccessibilityDeclarations` (4) · `AndroidToIosAppMappingDetails` (4) · `AppTags` (3) · `AppStoreVersionPhasedReleases` (3) · `AgeRatingDeclarations` (1) · `AppStoreVersionPromotions` (1) · `AppStoreVersionReleaseRequests` (1) · `RoutingAppCoverages` (4)

### TestFlight and beta testing — ~90 operations

`BetaGroups` (21) · `BetaTesters` (16) · `BetaAppLocalizations` (7) · `BetaBuildLocalizations` (7) · `PreReleaseVersions` (6) · `BetaAppReviewDetails` (5) · `BetaAppReviewSubmissions` (5) · `BetaLicenseAgreements` (5) · `BuildBetaDetails` (5) · `BetaAppClipInvocations` (4) · `BetaFeedbackCrashSubmissions` (4) · `BetaRecruitmentCriteria` (3) · `BetaAppClipInvocationLocalizations` (3) · `BetaFeedbackScreenshotSubmissions` (2) · `BetaCrashLogs` (1) · `BetaRecruitmentCriterionOptions` (1) · `BetaTesterInvitations` (1) · `BuildBetaNotifications` (1)

### Builds and uploads — ~50 operations

`Builds` (30) · `BuildBundles` (8) · `BuildUploads` (5) · `BuildUploadFiles` (3) · `DiagnosticSignatures` (1)

### In-app purchases and subscriptions — ~200 operations

`Subscriptions` (32) · `InAppPurchases` (25) · `SubscriptionGroups` (10) · `InAppPurchaseOfferCodes` (9) · `SubscriptionOfferCodes` (9) · `InAppPurchaseImages` (8) · `InAppPurchaseLocalizations` (8) · `InAppPurchasePriceSchedules` (8) · `InAppPurchaseVersions` (8) · `SubscriptionGroupLocalizations` (8) · `SubscriptionImages` (8) · `SubscriptionLocalizations` (8) · `SubscriptionVersions` (8) · `SubscriptionPlanAvailabilities` (6) · `SubscriptionPromotionalOffers` (6) · `WinBackOffers` (6) · `InAppPurchaseAppStoreReviewScreenshots` (4) · `InAppPurchaseAvailabilities` (4) · `InAppPurchaseOfferCodeOneTimeUseCodes` (4) · `SubscriptionAppStoreReviewScreenshots` (4) · `SubscriptionAvailabilities` (4) · `SubscriptionGroupVersions` (4) · `SubscriptionOfferCodeOneTimeUseCodes` (4) · `SubscriptionPricePoints` (4) · `InAppPurchaseOfferCodeCustomCodes` (3) · `SubscriptionIntroductoryOffers` (3) · `SubscriptionOfferCodeCustomCodes` (3) · `PromotedPurchases` (4) · `InAppPurchasePricePoints` (2) · `SubscriptionGracePeriods` (2) · `SubscriptionPrices` (2) · `InAppPurchaseContents` (1) · `InAppPurchaseSubmissions` (1) · `SubscriptionGroupSubmissions` (1) · `SubscriptionSubmissions` (1) · `SandboxTesters` (2) · `SandboxTestersClearPurchaseHistoryRequest` (1)

### Game Center — ~330 operations

`GameCenterDetails` (42) · `GameCenterGroups` (29) · `GameCenterLeaderboardSets` (27) · `GameCenterLeaderboards` (21) · `GameCenterAchievements` (19) · `GameCenterAchievementLocalizations` (14) · `GameCenterActivities` (14) · `GameCenterLeaderboardLocalizations` (12) · `GameCenterLeaderboardSetLocalizations` (12) · `GameCenterMatchmakingRuleSets` (11) · `GameCenterMatchmakingQueues` (10) · `GameCenterAppVersions` (9) · `GameCenterAchievementImages` (8) · `GameCenterChallenges` (8) · `GameCenterLeaderboardImages` (8) · `GameCenterLeaderboardSetImages` (8) · `GameCenterLeaderboardSetMemberLocalizations` (8) · `GameCenterActivityVersions` (7) · `GameCenterActivityLocalizations` (6) · `GameCenterChallengeLocalizations` (6) · `GameCenterChallengeVersions` (6) · `GameCenterMatchmakingRules` (6) · `GameCenterEnabledVersions` (5) · plus achievement/leaderboard/activity/challenge versions, images, releases, matchmaking teams, rule-set tests, entry submissions and player achievement submissions

### Xcode Cloud (continuous integration) — ~50 operations

`CiProducts` (13) · `CiBuildActions` (9) · `CiWorkflows` (8) · `CiBuildRuns` (6) · `ScmRepositories` (6) · `CiMacOsVersions` (4) · `CiXcodeVersions` (4) · `ScmProviders` (4) · `CiArtifacts` (1) · `CiIssues` (1) · `CiTestResults` (1) · `ScmGitReferences` (1) · `ScmPullRequests` (1)

### Certificates, identifiers and profiles — ~45 operations

`BundleIds` (11) · `Profiles` (10) · `Certificates` (7) · `MerchantIds` (7) · `PassTypeIds` (7) · `Devices` (4) · `BundleIdCapabilities` (3)

### App Review and submissions — ~20 operations

`ReviewSubmissions` (6) · `AppStoreReviewDetails` (5) · `AppStoreReviewAttachments` (4) · `ReviewSubmissionItems` (3) · `AppStoreVersionSubmissions` (1)

### Pricing, availability and territories — ~25 operations

`AppPriceSchedules` (8) · `AppAvailabilities` (4) · `AppPricePoints` (3) · `Nominations` (5) · `Territories` (1) · `TerritoryAvailabilities` (1) · `EndAppAvailabilityPreOrders` (1)

### Reports and analytics — ~15 operations

`AnalyticsReportRequests` (5) · `AnalyticsReportInstances` (3) · `AnalyticsReports` (3) · `AnalyticsReportSegments` (1) · `SalesReports` (1) · `FinanceReports` (1)

### Customer reviews — 6 operations

`CustomerReviews` (3) · `CustomerReviewResponses` (3)

### Users and access — ~15 operations

`Users` (9) · `UserInvitations` (6) · `Actors` (2)

### App Clips — ~30 operations

`AppClipDefaultExperiences` (11) · `AppClipDefaultExperienceLocalizations` (6) · `AppClips` (5) · `AppClipHeaderImages` (4) · `AppClipAdvancedExperienceImages` (3) · `AppClipAdvancedExperiences` (3) · `AppClipAppStoreReviewDetails` (3)

### Background assets — ~15 operations

`BackgroundAssets` (5) · `BackgroundAssetVersions` (4) · `BackgroundAssetUploadFiles` (3) · plus App Store, internal beta and external beta release endpoints

### Alternative distribution (EU DMA) — ~15 operations

`AlternativeDistributionPackageVersions` (5) · `AlternativeDistributionDomains` (4) · `AlternativeDistributionKeys` (4) · `AlternativeDistributionPackages` (4) · `MarketplaceWebhooks` (4) · `MarketplaceSearchDetails` (3) · `AlternativeDistributionPackageDeltas` (1) · `AlternativeDistributionPackageVariants` (1)

### Webhooks — 8 operations

`Webhooks` (6) · `WebhookDeliveries` (1) · `WebhookPings` (1)

Run `asc_tags` at any time for the live, exact list.

---

## Requirements

- [**Bun**](https://bun.sh) 1.1 or newer
- An **App Store Connect API key**: Issuer ID, Key ID and the `.p8` private key file
- An MCP client that speaks stdio: Claude Code, Claude Desktop, Cursor, Windsurf, VS Code with an MCP extension, Zed, or your own agent

---

## Get your App Store Connect API key

1. Sign in to [App Store Connect](https://appstoreconnect.apple.com).
2. Go to **Users and Access → Integrations → App Store Connect API**.
3. Click **+** to generate a key. Pick the smallest role that covers your use case (`Developer`, `App Manager`, or `Admin` for user management).
4. Copy the **Issuer ID** (a UUID, shown once at the top of the page).
5. Copy the **Key ID** (10 characters).
6. Download the `.p8` private key file. **Apple lets you download it exactly once.** Store it outside your repository, for example `~/.appstoreconnect/AuthKey_XXXXXXXXXX.p8`, and `chmod 600` it.

Apple's own walkthrough: [Creating API keys for App Store Connect API](https://developer.apple.com/documentation/appstoreconnectapi/creating-api-keys-for-app-store-connect-api).

---

## Install in Claude Code

```bash
git clone https://github.com/imfaisii/asc-mcp.git
cd asc-mcp
bun install
bun run smoke     # offline catalog check + server boot — no credentials needed
```

Register the server globally:

```bash
claude mcp add app-store-connect \
  -s user \
  -t stdio \
  -e ASC_ISSUER_ID=your-issuer-uuid \
  -e ASC_KEY_ID=your-key-id \
  -e ASC_PRIVATE_KEY_PATH=$HOME/.appstoreconnect/AuthKey_XXXXXXXXXX.p8 \
  -- bun run /ABS/PATH/to/asc-mcp/src/index.ts
```

Verify and restart:

```bash
claude mcp get app-store-connect
claude mcp list
```

Start a new Claude Code session. The tools appear as `mcp__app-store-connect__asc_search`, `…asc_schema`, `…asc_call` and `…asc_tags`.

---

## Install in Claude Desktop

Add this to `claude_desktop_config.json`:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "app-store-connect": {
      "command": "bun",
      "args": ["run", "/ABS/PATH/to/asc-mcp/src/index.ts"],
      "env": {
        "ASC_ISSUER_ID": "your-issuer-uuid",
        "ASC_KEY_ID": "your-key-id",
        "ASC_PRIVATE_KEY_PATH": "/ABS/PATH/to/AuthKey_XXXXXXXXXX.p8"
      }
    }
  }
}
```

Restart Claude Desktop.

---

## Install in Cursor, Windsurf, VS Code and other MCP clients

Any client that launches a stdio MCP server works. The shape is always the same — `mcp.example.json` in this repo is a ready-to-copy template.

**Cursor** — `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (per project):

```json
{
  "mcpServers": {
    "app-store-connect": {
      "command": "bun",
      "args": ["run", "/ABS/PATH/to/asc-mcp/src/index.ts"],
      "env": {
        "ASC_ISSUER_ID": "your-issuer-uuid",
        "ASC_KEY_ID": "your-key-id",
        "ASC_PRIVATE_KEY_PATH": "/ABS/PATH/to/AuthKey_XXXXXXXXXX.p8"
      }
    }
  }
}
```

**Windsurf** — `~/.codeium/windsurf/mcp_config.json`, same `mcpServers` block.

**VS Code** — `.vscode/mcp.json` with a `servers` block using the same `command`, `args` and `env`.

**Your own agent** — run `bun run src/index.ts` and speak MCP over stdio. It will look like it is hanging in a raw terminal; that is a stdio server waiting for a client, and it is correct.

---

## Environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `ASC_ISSUER_ID` | yes | Issuer ID (UUID) from App Store Connect |
| `ASC_KEY_ID` | yes | Key ID (10 characters) |
| `ASC_PRIVATE_KEY_PATH` | yes* | Absolute path to the `.p8` private key file |
| `ASC_PRIVATE_KEY` | yes* | Inline PEM contents, instead of a file path (useful in CI) |
| `ASC_BASE_URL` | no | Defaults to `https://api.appstoreconnect.apple.com` |
| `ASC_EXPOSE_ALL_TOOLS` | no | `1` registers all 1,263 operations as individual MCP tools. Debug only |

\* Supply exactly one of `ASC_PRIVATE_KEY_PATH` or `ASC_PRIVATE_KEY`.

These aliases are also accepted: `APP_STORE_CONNECT_ISSUER_ID`, `APP_STORE_CONNECT_KEY_ID`, `APP_STORE_CONNECT_PRIVATE_KEY_PATH`, `APP_STORE_CONNECT_PRIVATE_KEY`.

Copy `.env.example` to `.env` for local shells. `.env*` and `*.p8` are already in `.gitignore`.

---

## How an agent uses it

```text
1. asc_tags                                     # optional: orient inside 195 resource tags
2. asc_search { query: "list apps" }            # find candidate operations
3. asc_schema { operation: "apps_getCollection" }  # read the exact input schema
4. asc_call   { operation: "apps_getCollection", args: { limit: 10 } }
```

For anything that writes, run it as a dry run first:

```jsonc
{ "operation": "appStoreVersions_updateInstance", "_dryRun": true, "args": { "id": "…", "body": { } } }
```

`_dryRun` returns the resolved method, path, path params, query and body without touching Apple, so the agent can check its own request before it commits.

---

## Example prompts

Once the server is connected, these all work in plain language:

- "List my apps with their bundle IDs and SKUs."
- "Show the last 5 TestFlight builds for MyApp and who they were distributed to."
- "Create a TestFlight beta group called Internal QA for MyApp and add these three testers."
- "What is the current App Store version of MyApp in each localization, and which ones are missing a description?"
- "Pull this month's sales report and summarise units by territory."
- "List all auto-renewable subscriptions in my subscription group with their price points."
- "Which provisioning profiles expire in the next 30 days?"
- "Show my latest App Review submission and its rejection reason."
- "Fetch the newest customer reviews with 1 or 2 stars and draft responses."
- "Trigger the Xcode Cloud workflow named Release and report the build run status."
- "Which of my in-app purchases are missing a review screenshot?"
- "Add this device UDID to my account and regenerate the development profile."

---

## Scripts

| Command | Purpose |
| --- | --- |
| `bun install` | Install dependencies |
| `bun run generate` | Rebuild `generated/tools.json` and `generated/manifest.json` from the vendored OpenAPI spec |
| `bun run start` | Start the stdio MCP server |
| `bun run smoke` | Generate, validate the catalog inventory, boot the server. Runs a live `GET /v1/apps` only if credentials are in the environment |
| `bun run typecheck` | `tsc --noEmit` |
| `bun run live` | Live credential check against `GET /v1/apps` |

---

## Refresh Apple's OpenAPI specification

Apple ships a zip. Vendor the JSON, regenerate, verify:

```bash
curl -fsSL -o /tmp/asc-openapi.zip \
  "https://developer.apple.com/sample-code/app-store-connect/app-store-connect-openapi-specification.zip"
unzip -p /tmp/asc-openapi.zip '*.json' > openapi/app-store-connect.openapi.json
bun run generate
bun run smoke
```

`generated/` is build output. Never hand-edit it — regenerate instead.

**Coverage note:** "full surface" means every operation in Apple's published OpenAPI zip for the vendored version. If an endpoint is described in Apple's human documentation but missing from the zip, it appears here after Apple updates the zip and you regenerate.

---

## Project layout

```text
asc-mcp/
├── openapi/
│   └── app-store-connect.openapi.json   # Apple's spec, vendored (source of truth)
├── generated/
│   ├── tools.json                       # 1263-operation catalog (build output)
│   └── manifest.json                    # counts, tags, spec version
├── src/
│   ├── index.ts                         # MCP stdio server, 4 meta-tools
│   ├── catalog.ts                       # search / resolve over the catalog
│   ├── client.ts                        # fetch wrapper, response shaping
│   ├── auth.ts                          # ES256 JWT, in-memory cache
│   └── types.ts
├── scripts/
│   ├── generate-tools.ts                # OpenAPI → catalog
│   ├── smoke.ts                         # offline inventory + boot check
│   └── live-check.ts                    # live GET /v1/apps
├── assets/                              # README graphics
├── mcp.example.json
├── .env.example
└── package.json
```

---

## Security model

- **Nothing secret is stored in this repository.** Credentials come from environment variables only.
- `.gitignore` blocks `*.p8`, `AuthKey_*.p8` and `.env*`.
- JWTs are ES256, signed locally with your `.p8`, scoped to audience `appstoreconnect-v1`, valid for **20 minutes** (Apple's maximum), cached in memory only, and refreshed 60 seconds before expiry.
- Tool results never echo credentials back to the model.
- The only network destination is `ASC_BASE_URL`, which defaults to Apple's API host.
- If you ever commit a `.p8` by accident, **revoke that key in App Store Connect immediately** and generate a new one.

See [SECURITY.md](SECURITY.md) for reporting a vulnerability.

---

## Troubleshooting

**`Missing credentials` / `ASC_ISSUER_ID not set`**
The MCP client did not pass the environment through. Put the variables in the client's `env` block rather than in your shell profile — GUI apps like Claude Desktop and Cursor do not inherit your shell environment.

**HTTP 401 `NOT_AUTHORIZED`**
Issuer ID, Key ID and `.p8` must all belong to the same key. Confirm the Key ID matches the filename (`AuthKey_<KeyID>.p8`) and that the key has not been revoked.

**HTTP 403 `FORBIDDEN_ERROR`**
The key's role is too narrow for that operation. User management and finance reports need higher roles than `Developer`.

**HTTP 409 with a JSON:API `detail` string**
Apple is rejecting the document shape. Run the same call with `_dryRun: true`, compare against `asc_schema` output, then fix the `body`.

**The server "hangs" when I run `bun run start`**
That is correct. A stdio MCP server blocks waiting for a client on stdin. Use `bun run smoke` if you want a check that exits.

**Tools do not show up in my client**
Restart the client after editing its MCP config, and use an absolute path to `src/index.ts`. Confirm `bun` is on the PATH the client sees (`which bun`), or use the absolute path to the `bun` binary as `command`.

**Responses get truncated**
Payloads over ~120,000 characters are cut. Narrow with `fields[…]`, `filter[…]` and `limit` instead of pulling whole collections.

---

## FAQ

**Is this an official Apple product?**
No. It is an independent open source project. "App Store Connect", "TestFlight", "Xcode" and "Game Center" are trademarks of Apple Inc.

**How many tools does this MCP server expose?**
Four by default — `asc_search`, `asc_schema`, `asc_call` and `asc_tags` — sitting on an internal catalog of 1,263 App Store Connect API operations. Setting `ASC_EXPOSE_ALL_TOOLS=1` registers all 1,263 individually, which is only useful for debugging.

**Does it support TestFlight?**
Yes. Roughly 90 operations cover beta groups, beta testers, invitations, beta build localizations, beta app review submissions, crash feedback and recruitment criteria.

**Can an agent submit my app for review?**
Yes. `ReviewSubmissions`, `ReviewSubmissionItems`, `AppStoreVersionSubmissions` and the review detail resources are all in the catalog. Use `_dryRun` first, and keep a human in the loop for anything that reaches the App Store.

**Does it work with Cursor, Windsurf and VS Code?**
Yes. It is a plain stdio MCP server, so any Model Context Protocol client can launch it.

**Does it store or transmit my App Store Connect API key?**
No. The key is read from the environment at call time, used to sign a short-lived JWT locally, and the JWT goes only to Apple's API host.

**How is this different from other App Store Connect MCP servers?**
Most are hand-written wrappers around a small subset of endpoints. This one generates its catalog from Apple's official OpenAPI specification, so coverage is the whole published surface, and refreshing it is one command instead of a rewrite.

**Why not just give the model 1,263 tools?**
Full JSON Schemas for 1,263 operations cost hundreds of thousands of context tokens and many clients truncate or reject lists that long. Search-then-schema-then-call costs a few thousand tokens per task.

**Can I use it in CI?**
Yes. Set `ASC_PRIVATE_KEY` with the inline PEM instead of a file path, so no `.p8` needs to touch the runner's disk.

**Does it handle App Store Connect API rate limits?**
It does not retry on your behalf. Apple's rate limit responses and headers are passed straight through so the agent (or your code) can decide how to back off.

**Which App Store Connect API version is vendored?**
4.4.1, OpenAPI 3.0.1. Run `bun run generate` after dropping in a newer spec to move up.

---

## Contributing

Issues and pull requests are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md). Two rules matter most: `generated/` is regenerated rather than edited by hand, and no credentials ever appear in a pull request.

---

## License

[MIT](LICENSE) © imfaisii

Not affiliated with or endorsed by Apple Inc.

---

<div align="center">

**Keywords:** App Store Connect MCP · App Store Connect API MCP server · MCP server for App Store Connect · Model Context Protocol Apple · TestFlight MCP · Xcode Cloud MCP · in-app purchase MCP · App Store automation · Claude Code App Store Connect · Cursor MCP App Store Connect · iOS release automation with AI agents

</div>
