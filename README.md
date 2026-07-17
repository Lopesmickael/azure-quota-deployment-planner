# Azure Quota & Deployment Planner

A single-file, zero-install web tool that pulls **quota limits vs. current usage** for one or more Azure subscriptions straight from the **public Azure Resource Manager (ARM) API** — the same numbers the Azure Portal shows — and turns them into a shareable, self-contained HTML report.

> **Live tool:** https://aseukunian.github.io/azure-quota-deployment-planner/

Everything runs in your browser. No backend, no data leaves your machine, and the exported report is a standalone `.html` file you can email or archive.

## Features

- **Live fetch** from ARM using a bearer token you generate yourself (`Microsoft.Compute`, `Microsoft.Network`, and the unified `Microsoft.Quota` multi-RP provider).
- **At-risk view** — surfaces quotas approaching their limit so you can request increases before deployments fail.
- **Pending quota-increase requests** — see in-flight `Microsoft.Quota/quotaRequests` at a glance.
- **Deployment planner** — pick one subscription and any region to browse **every VM size** offered there (available *and* restricted), one row per SKU with vCPUs, memory, availability-zone coverage, family quota headroom, and a deployable/blocked status. Filter by vCPUs/memory to find a working alternative when a size is restricted.
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

### Offline / CORS-blocked

Use the **Script generator** tab to produce an `az` collector, run it in Azure Cloud Shell, then bring the combined `quota-report.json` back via the **Import** tab. Fully offline — no token in the browser.

## Data sources

`Microsoft.Compute` / `Microsoft.Network` usages, `Microsoft.Quota` (quotas, usages, quotaRequests), `Microsoft.Compute/skus` for VM availability & restrictions, and `Microsoft.ResourceGraph` for region detection — all on `management.azure.com`.

## Security & privacy

This tool is designed to keep your Azure credential safe:

- **No server, no telemetry.** Every API call goes **directly from your browser to `management.azure.com`** over HTTPS. Nothing is collected, stored remotely, or sent anywhere else.
- **No third-party requests.** There are **no external scripts, fonts, or CDNs** — even the country-flag images are bundled inline as base64. The page loads and runs fully offline once opened.
- **The token is never persisted.** It lives only in the input field / memory while the page is open and is gone on refresh. It is **never written to `localStorage`** and **never included in the exported HTML/CSV**.
- **Strict Content-Security-Policy.** A `<meta>` CSP restricts network access to `connect-src https://management.azure.com` and blocks all external origins, so even in the unlikely event of injected markup, the token cannot be exfiltrated to another host.
- **Use a least-privilege token.** The tool only needs **Reader**. Prefer a Reader-scoped identity, and treat the token like a password — it is a live Azure credential valid for ~60–75 minutes.

The exported report is a standalone HTML file containing only quota/usage data you chose to include — never the token.

## License

[MIT](LICENSE)
