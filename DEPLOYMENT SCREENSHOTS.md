# Deployment Steps

All steps below are performed through the Azure Portal unless stated otherwise. No CLI or Terraform required.

---

## Phase 1 — Build & Test the Site Locally

### Step 1 — Build and preview the site

Create your static site files in VS Code. This project used Contoso Bistro with the following structure:

```
site/
├── css/
│   └── style.css
├── js/
│   └── main.js
├── index.html
└── 404.html
```

Right-click `index.html` and choose **Open with Live Server** to launch the local preview. Confirm:

- The site loads at `127.0.0.1:5500`
- Styles render correctly
- The 404 page displays when you navigate to a non-existent path

> Debug all file path issues on Live Server **before** uploading to Azure.

---

## Phase 2 — Create Resource Groups & Storage Accounts

### Step 2 — Create the resource groups

In the Azure Portal go to **Resource groups → Create**. Create these two groups:

| Name | Region |
|---|---|
| `rg-mokcloudsite-prod` | Canada Central |
| `rg-dns-prod` | Canada Central |

> The DNS zone lives in a separate resource group because it cannot be deleted and redeployed freely — DNS propagation makes teardown and recreation very slow.

---

### Step 3 — Create the Storage Accounts

Create **two** storage accounts — one primary, one secondary for failover.

In the Azure Portal go to **Storage accounts → Create**. Configure each as follows:

| Setting | Value |
|---|---|
| Resource group | `rg-mokcloudsite-prod` |
| Primary region | Canada Central |
| Secondary region | East US |
| Performance | Standard |
| Redundancy | Locally Redundant Storage (LRS) |

**Data Protection tab — configure as follows:**

- ✅ Enable soft delete for blobs (7 days)
- ✅ Enable soft delete for containers (7 days)
- ✅ Enable blob versioning
- ❌ Uncheck blob change feed
- ❌ Uncheck point-in-time restore

> Soft delete and versioning are enabled because the site files are not backed by Git. The `$web` container is the only copy. These features act as a safety net against accidental deletion or bad uploads at minimal additional storage cost.
>
> If your project is backed by Git version control, uncheck everything — Git is your version control and Azure keeping old versions of files adds unnecessary storage cost.

**Advanced tab:**

- ✅ Enable blob anonymous access

Click **Review + Create → Create**.

---

### Step 4 — Enable static website hosting on both storage accounts

For **each** storage account:

1. Open the storage account
2. In the left menu under **Data management**, click **Static website**
3. Set to **Enabled**
4. Set **Index document name** to `index.html`
5. Set **Error document path** to `404.html`
6. Click **Save**

Azure creates the `$web` container automatically and generates the primary endpoint URL (e.g. `stmkstaticsiteprod01.z9.web.core.windows.net`). **Copy this URL** — you will need it when configuring Front Door.

---

### Step 5 — Upload site files to both $web containers

**Using Azure CLI (recommended):**

```powershell
az login
az account set --subscription "<your-subscription-id>"

# Assign yourself the Storage Blob Data Contributor role first
$USER_ID = az ad signed-in-user show --query id -o tsv
$SA_ID = az storage account show --name <storage-account-name> --resource-group rg-mokcloudsite-prod --query id -o tsv
az role assignment create --assignee $USER_ID --role "Storage Blob Data Contributor" --scope $SA_ID

# Upload site files
az storage blob upload-batch `
  --account-name <storage-account-name> `
  --source . `
  --destination '$web' `
  --auth-mode login `
  --overwrite true

# Fix MIME type for index.html
az storage blob update `
  --account-name <storage-account-name> `
  --container-name '$web' `
  --name index.html `
  --content-type "text/html" `
  --auth-mode login

# Remove accidentally uploaded .git folder
az storage blob delete-batch `
  --account-name <storage-account-name> `
  --source '$web' `
  --pattern ".git/*" `
  --auth-mode login
```

> **Important:** Use single quotes around `$web` in PowerShell to prevent variable expansion.

After uploading, open the primary endpoint URL in a browser and confirm the site renders correctly with all styles loaded before moving to Front Door.

---

## Phase 3 — Create the Azure DNS Zone

### Step 6 — Create a DNS Zone

In the Azure Portal go to **DNS zones → Create**. Configure:

| Setting | Value |
|---|---|
| Resource group | `rg-dns-prod` |
| Name | `mokcloud.site` (your domain) |

After creation, note the **four nameserver values** Azure assigns — they look like:
- `ns1-01.azure-dns.com`
- `ns2-01.azure-dns.net`
- `ns3-01.azure-dns.org`
- `ns4-01.azure-dns.info`

---

### Step 7 — Delegate nameservers at your registrar

Log into Namecheap (or your registrar). Go to:

**Domain List → Manage → Nameservers → Custom DNS**

Paste all four Azure nameservers and save.

> **DNS propagation** typically takes 1–4 hours but can take up to 48 hours. In this project it took approximately **8 hours**.

Verify propagation is complete by running this in PowerShell:

```powershell
nslookup -type=NS mokcloud.site
```

You should see all four Azure nameservers in the response.

---

## Phase 4 — Deploy Azure Front Door Standard

### Step 8 — Create the Front Door profile

In the Azure Portal go to **Front Door and CDN profiles → Create → Quick create**. Configure:

| Setting | Value |
|---|---|
| Profile name | `afd-staticsite-prod` |
| Tier | Standard |
| Resource group | `rg-mokcloudsite-prod` |
| Endpoint name | `ep-staticsite` |
| Origin type | Azure Blob Storage static website |
| Origin host name | Your primary storage static website endpoint |
| Enable caching | Yes |

Click **Review + Create → Create**.

---

### Step 9 — Configure the Route, Origin Group, and Origins

In the Front Door profile go to **Front Door manager** and open the route. Configure:

**Route settings:**
- ✅ Enable route

**Origin group settings:**
- Add **both** storage account static website endpoints as origins
  - Primary (Canada Central) → Priority: **1**, Weight: 1000
  - Secondary (East US) → Priority: **2**, Weight: 1000
- Health probes:
  - Protocol: **HTTPS**
  - Method: HEAD
  - Path: /
  - Interval: 30 seconds

> **Why HTTPS for health probes?** When static website hosting is enabled, "Require secure transfer" is checked by default — the storage account rejects all plain HTTP requests. Setting probes to HTTP causes them to fail silently, Front Door marks the origin as unhealthy, and the site goes down even though nothing is actually wrong.

**Caching settings:**
- ✅ Enable caching
- Query string: **Ignore Query String**
- ✅ Enable compression (files between 1KB and 8MB)

Click **Update** and wait for provisioning state to show **Succeeded**.

---

### Step 10 — Add custom domains

In the Front Door profile go to **Domains → Add**. Add both:

| Domain | Type |
|---|---|
| `mokcloud.site` | Apex domain |
| `www.mokcloud.site` | WWW subdomain |

**Certificate settings:**
- Certificate type: **AFD managed** — Azure manages and auto-renews the TLS certificate

After adding each domain, click **Pending** under Validation state and click **Add** to automatically add the required TXT and CNAME validation record sets to your Azure DNS Zone.

Wait for the **Approved** validation state on both domains.

> **TLS certificate propagation:** After the portal shows the certificate as "Deployed," it can take up to **72 hours** for the certificate to fully propagate to all Front Door POPs worldwide. In this project the site loaded with the correct certificate after approximately **37 minutes** — but a "Your connection is not private" warning during this window is expected and resolves automatically.

---

### Step 11 — Update the route to include all domains

Go to **Front Door manager → route-default → Edit**. Under **Domains**, select all three:

- The AFD endpoint hostname
- `mokcloud.site`
- `www.mokcloud.site`

Click **Update**.

---

## Phase 5 — Monitoring & Cost Control

### Step 14 — Create a Log Analytics Workspace

In the Azure Portal go to **Log Analytics workspaces → Create**. Configure:

| Setting | Value |
|---|---|
| Name | `law-mokcloud-prod` |
| Region | Canada Central |
| Pricing tier | Pay-as-you-go |
| Resource group | `rg-mokcloudsite-prod` |

---

### Step 15 — Configure diagnostic settings on Front Door

Open the Front Door profile. Go to **Monitoring → Diagnostic settings → Add diagnostic setting**. Configure:

| Setting | Value |
|---|---|
| Name | `diag-afd-to-law` |
| Logs | ✅ FrontDoor Access Log, ✅ FrontDoor Health Probe Log |
| Metrics | ✅ AllMetrics |
| Destination | Send to Log Analytics workspace → `law-mokcloud-prod` |

Click **Save**.

---

### Step 16 — Create a Cost Management Budget

Go to **Cost Management → Budgets → Add**. Configure:

| Setting | Value |
|---|---|
| Scope | `rg-mokcloudsite-prod` resource group |
| Budget name | `budget-staticsite` |
| Amount | CA$50 monthly |
| Alert at 50% | CA$25 |
| Alert at 80% | CA$40 |
| Alert at 100% | CA$50 |
| Alert recipients | Your email address |

---

### Step 17 — Verify failover

To confirm the secondary origin takes over when the primary fails:

1. Go to your **primary storage account → Settings → Configuration → Static website → Disabled → Save**
2. Wait **5 minutes** for Front Door health probes to detect the failure
3. Visit `mokcloud.site` — the site should still load, now served from the East US secondary storage account
4. In the Azure Portal go to **Front Door → Origin groups** — the primary origin will show **Unhealthy**, the secondary will show **Healthy**
5. Re-enable static website on the primary storage account — Front Door automatically fails back within a few minutes

> In this project the site remained up after disabling the primary origin, confirming failover was working correctly.

---

*Built by Mokenyu · Cloud Engineering Student · Canada*
