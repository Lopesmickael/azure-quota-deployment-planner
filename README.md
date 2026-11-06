# Azure Quota & Deployment Planner

A single-file, zero-install web tool that pulls **quota limits vs. current usage** for one or more Azure subscriptions straight from the **public Azure Resource Manager (ARM) API** — the same numbers the Azure Portal shows — and turns them into a shareable, self-contained HTML report.

> **Live tool:** https://aseukunian.github.io/azure-quota-deployment-planner/

Everything runs in your browser. No backend, no data leaves your machine, and the exported report is a standalone `.html` file you can email or archive.

## Features

- **Live fetch** from ARM using a bearer token you generate yourself (`Microsoft.Compute`, `Microsoft.Network`, the unified `Microsoft.Quota` multi-RP provider, and App Service regional quotas via `Microsoft.Web` in Preview).
- **At-risk view** — surfaces quotas approaching their limit so you can request increases before deployments fail.
- **Pending quota-increase requests** — see in-flight `Microsoft.Quota/quotaRequests` at a glance.
- **Deployment planner** — pick one subscription and any region to browse **every VM size** offered there (available *and* restricted), one row per SKU with vCPUs, memory, availability-zone coverage, family quota headroom, and a deployable/blocked status. Filter by vCPUs/memory to find a working alternative when a size is restricted.
- **Isolated platform-service report** — a separate live-only tab for App Service Plans, Azure SQL / Managed Instance, PostgreSQL Flexible Server, Storage, and Automation Accounts. It has independent inputs, results, and CSV/HTML exports, so it does not modify the existing quota report.
- **Zone mapping** — Azure assigns logical zones (1/2/3) independently per subscription; this shows each subscription's logical→physical zone mapping.
- **Adaptive throttling** — Live fetch auto-tunes concurrency: each HTTP 429 halves the request rate and ramps back up as ARM recovers, so large subscription lists still finish.
- **Offline path** — can't use Live fetch (e.g., CORS)? Generate an `az` collector script, run it in Cloud Shell, and import the resulting `quota-report.json`. No token ever touches the browser.
- **Exportable report** — self-contained HTML plus CSV export.

## Usage

1. Open the [live tool](https://aseukunian.github.io/azure-quota-deployment-planner/) (or download `index.html` and open it locally).
2. Get a token (you need at least **Reader** on the target subscription):
   ```bash
   az account get-access-token --resource https://management.azure.com --query accessToken -o tsv
   ```
3. Paste the token and one or more subscription IDs, pick regions (or auto-detect), and click **Generate report**.
4. Export the report as HTML or CSV to share.

The token is used only from your browser to call `management.azure.com` directly and is valid for ~60–75 minutes.

### Preview regions

**Southeast Asia 3 (`southeastasia3`)** and **Taiwan North (`taiwannorth`)** are selectable in the Asia Pacific group with a **Preview** label. Region selectors and HTML reports retain that label; API requests, collector scripts, imports, and CSV data keep the unmodified Azure region names.

Preview status is maintained in `PREVIEW_REGIONS` in `index.html`. Once Microsoft confirms general availability, remove the region from that array; keep it in `ALL_REGIONS` and its geography/country mappings. Newly generated reports use the updated status; existing downloaded reports retain their original labels.

Microsoft's [region list](https://learn.microsoft.com/en-us/azure/reliability/regions-list) identifies both as Preview and directs customers to their Microsoft representative for available services and access. Selecting a region here does not enable it for your subscription. The [subscription locations API](https://learn.microsoft.com/en-us/rest/api/resources/subscriptions/list-locations?view=rest-resources-2022-12-01) lists locations, but individual resource providers can support only a subset. The documented [App Service regional usages API](https://learn.microsoft.com/en-us/rest/api/appservice/get-usages-in-location/list?view=rest-appservice-2024-11-01) accepts a region name; documentation alone does not establish whether either preview region currently returns counters for your subscription.

To check actual App Service quota access, select either region and **Microsoft.Web — App Service (Preview)**, then generate a live report with your subscription and ARM token. Successful counters confirm that endpoint is accessible for that subscription, not deployment readiness. Endpoint failures remain visible; an error or empty App Service response is not treated as zero usage or unlimited capacity. In Script generator, no region selection still queries all listed regions, now including these two; select specific regions to avoid preview endpoint attempts.

### App Service Quota (Preview)

In **Live fetch → Resource providers**, enable **Microsoft.Quota**, then select **Microsoft.Web — App Service (Preview)** under **Quota scopes to query**. Use the existing subscription, token, and region inputs and select **Generate report**. App Service counters appear alongside the other providers in the existing report, filters, risk view, and HTML/CSV exports.

The correct Azure namespace is **Microsoft.Web**, not `Microsoft.AppServices`. Although selected in the scope picker, App Service uses its documented regional `Microsoft.Web/locations/{region}/usages` endpoint, not the unified `Microsoft.Quota` endpoints. No App Service quota-request API calls or VM deployment analysis are performed. Quota headroom does not guarantee App Service deployment/scaling capacity.

For offline collection, select **App Service Quota (Microsoft.Web)** in **Script generator**, then import the combined JSON as usual. The generated Bash/PowerShell collectors preserve the App Service provider identity. Scope selections are saved with the existing preferences; tokens are never saved. The separate **Platform services** workflow remains unchanged.

### Platform services

Use the **Platform services** tab when you only need quota/current-limit counters for App Service, Azure SQL, PostgreSQL Flexible Server, Storage, or Automation Accounts. Enter its subscription IDs, ARM token, and regions, select the services, then click **Generate service report**.

This workflow is intentionally independent from **Live fetch**:

- It maintains separate inputs, state, failures, results, and CSV/HTML exports.
- It is live-only in this version; the Script generator and Import tabs continue to serve the original report.
- Automation Accounts are discovered per subscription and then queried individually for account-scoped usage counters.
- The values are quota counters, not Azure Monitor operational metrics. CPU percentage, database storage consumed, transactions, IOPS, bandwidth, and job health are out of scope.

### Offline / CORS-blocked

Use the **Script generator** tab to produce an `az` collector, run it in Azure Cloud Shell, then bring the combined `quota-report.json` back via the **Import** tab. Fully offline — no token in the browser.

## Data sources

`Microsoft.Compute` / `Microsoft.Network` usages, `Microsoft.Quota` (quotas, usages, quotaRequests), `Microsoft.Compute/skus` for VM availability & restrictions, `Microsoft.ResourceGraph` for region detection, and service-specific usage APIs for `Microsoft.Web`, `Microsoft.Sql`, `Microsoft.DBforPostgreSQL`, `Microsoft.Storage`, and `Microsoft.Automation` — all on `management.azure.com`.

## API calls & permissions (read-only)

Every call this tool makes is **read-only** — it never creates, modifies, or deletes anything in Azure. There are **no `PUT` / `PATCH` / `DELETE`** operations and no writes of any kind; the only `POST`s are Azure Resource Graph **queries** (KQL reads). A **Reader** role is sufficient, which by Azure RBAC makes it *incapable* of changing your environment.

| # | HTTP | Endpoint | Reads | When |
|---|------|----------|-------|------|
| 1 | GET | `/subscriptions/{sub}` | Subscription display name | Always (once per sub) |
| 2 | GET | `…/Microsoft.Compute/locations/{region}/usages` | Compute usage vs. limits | If "Compute usages" is enabled |
| 3 | GET | `…/Microsoft.Network/locations/{region}/usages` | Network usage vs. limits | If "Network usages" is enabled |
| 4 | POST | `…/Microsoft.ResourceGraph/resources` | KQL query listing which regions have resources | Only when regions are left unselected (auto-detect) |
| 5 | GET | `…/Microsoft.Quota/quotas` | Quota limits | If "Microsoft.Quota" is enabled |
| 6 | GET | `…/Microsoft.Quota/usages` | Quota usages | If "Microsoft.Quota" is enabled |
| 7 | GET | `…/Microsoft.Quota/quotaRequests` | Existing quota-increase requests | If "Microsoft.Quota" is enabled |
| 8 | GET | `…/Microsoft.Compute/skus?$filter=location eq '{region}'` | VM sizes + restrictions in a region | If "SKU availability" is enabled |
| 9 | GET | `…/Microsoft.Compute/skus` | VM sizes across all regions | If "SKU availability" is enabled |
| 10 | GET | `/subscriptions/{sub}/locations` | Logical→physical availability-zone mapping | Always (once per sub) |
| 11 | POST | `…/Microsoft.ResourceGraph/resources` | KQL query listing deployed VMs / disks / public IPs | Only if "Deployed inventory & waste" is enabled |
| 12 | GET | `…/Microsoft.Web/locations/{region}/usages` | Regional App Service core usage and limit | Live fetch App Service Preview scope; also Platform services if App Service is selected |
| 13 | GET | `…/Microsoft.Sql/locations/{region}/usages` | Regional Azure SQL / Managed Instance quotas and usage | Platform services tab; if Azure SQL is selected |
| 14 | GET | `…/Microsoft.DBforPostgreSQL/locations/{region}/resourceType/flexibleServers/usages` | PostgreSQL Flexible Server family quota and usage | Platform services tab; if PostgreSQL is selected |
| 15 | GET | `…/Microsoft.Storage/locations/{region}/usages` | Regional Storage resource-count quotas and usage | Platform services tab; if Storage is selected |
| 16 | GET | `…/Microsoft.Automation/automationAccounts` | Automation Accounts in the subscription | Platform services tab; once per selected subscription |
| 17 | GET | `…/Microsoft.Automation/automationAccounts/{account}/usages` | Per-account and subscription Automation usage counters | Platform services tab; for accounts in selected regions |

**Why this is safe for your Azure resources**

- **Read-only, always.** No endpoint accepts a body that changes state. The Resource Graph `POST` is Azure's standard way to *run a read query* — it cannot write.
- **No resource is touched.** It reads quota/usage counters, the SKU catalog, and zone metadata. It never reads resource *contents* or secrets, and never starts/stops/tags/deletes anything.
- **Least privilege.** Works with **Reader**; hand it a Reader-scoped token and it is unable to modify anything by RBAC.
- **Same data as the Portal.** These are the identical ARM endpoints the Azure Portal's Quota and Usage blades use.
- **Gentle on ARM.** Calls are throttle-aware (adaptive concurrency backs off on HTTP 429), so large runs won't hammer the management plane.
- **Scoped to your choices.** Nothing runs for providers you leave disabled; the two Resource Graph queries only fire on region auto-detect or when you opt into inventory.
- **Worst case is harmless.** A call can be denied (403) or throttled (429) — there is no code path that can alter, create, or delete a resource.

For a browser that makes **zero** calls, use the **Script generator + Import** tabs: you run the same read-only `az` commands yourself and import the resulting JSON.

## Security & privacy


This tool is designed to keep your Azure credential safe:

- **No server, no telemetry.** Every API call goes **directly from your browser to `management.azure.com`** over HTTPS. Nothing is collected, stored remotely, or sent anywhere else.
- **No third-party requests.** There are **no external scripts, fonts, or CDNs** — even the country-flag images are bundled inline as base64. The page loads and runs fully offline once opened.
- **The token is never persisted.** It lives only in the input field / memory while the page is open and is gone on refresh. It is **never written to `localStorage`** and **never included in the exported HTML/CSV**.
- **Strict Content-Security-Policy.** A `<meta>` CSP restricts network access to `connect-src https://management.azure.com` and blocks all external origins, so even in the unlikely event of injected markup, the token cannot be exfiltrated to another host.
- **Use a least-privilege token.** The tool only needs **Reader**. Prefer a Reader-scoped identity, and treat the token like a password — it is a live Azure credential valid for ~60–75 minutes.

The exported report is a standalone HTML file containing only quota/usage data you chose to include — never the token.

## Versioning

The app footer and both downloaded HTML report types display the tool version. `APP_VERSION` in [index.html](index.html) is the single source of truth; downloaded reports retain the version that generated them.

Update this value manually when a set of merged changes is ready for release, not on every branch merge. Verify the app and both HTML exports show the new version, then commit and push the release changes. No automatic version bumps or changelog are required.

## License

[MIT](LICENSE)
