# Azure Static Site Delivery

**Blob Storage · Azure Front Door Standard · Custom Domain · HTTPS · Cost Monitoring**

*A professional, globally distributed website delivered without a single server.*

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [Tools Used](#2-tools-used)
3. [The Business Problem This Project Solves](#3-the-business-problem-this-project-solves)
4. [Prerequisites](#4-prerequisites)
5. [Deployment Steps](#5-deployment-steps)
6. [Problems Faced and Solutions](#6-problems-faced-and-solutions)

---

## 1. Architecture

> Architecture diagram — insert your diagram here.

**Traffic flow:**

1. Visitor types `www.mokcloud.site` into their browser
2. Azure DNS Zone resolves the domain and points the request to the Azure Front Door endpoint
3. Front Door checks its edge cache — if content is cached (cache hit), it serves it instantly from the nearest edge location worldwide
4. On a cache miss, Front Door fetches the files from the Azure Blob Storage `$web` container (the origin)
5. The response travels back to the visitor. Data moving from Azure origin to Front Door edge is free — no egress charge on that leg
6. Front Door access logs and metrics flow into the Log Analytics Workspace in real time
7. A Cost Management budget watches monthly spend and fires alerts at 50%, 80%, and 100% of the monthly limit

> **Security note:** After Front Door is configured, the Storage Account's direct public endpoint is locked down as much as possible on Standard tier. All real users access the site only through Front Door. This reduces egress costs and adds a DDoS protection layer at the edge.

---

## 2. Tools Used

### Azure Services

| Service | Purpose |
|---|---|
| **2 Resource Groups** | One for the Azure DNS Zone (`rg-dns-prod`) and one for all other resources (`rg-mokcloudsite-prod`). The DNS zone is separated because it cannot be deleted and redeployed freely due to DNS propagation delays |
| **2 Azure Blob Storage accounts (Canada Central + East US)** | Hosts static website files inside the `$web` container. The second storage account in East US serves as a failover origin — if the primary goes down, Front Door automatically routes traffic to the secondary |
| **Azure Front Door Standard** | Delivers content from 100+ global edge locations. Handles caching, HTTPS termination, managed TLS certificates, custom domain binding, health probes, and routing rules |
| **Azure DNS Zone (mokcloud.site)** | Manages DNS records for the custom domain. Nameservers delegated from Namecheap to Azure DNS so the domain is fully managed inside Azure |
| **Log Analytics Workspace** | Receives Front Door access logs, health probe logs, and all metrics via a diagnostic setting |
| **Azure Cost Management** | Monthly budget of CA$50 scoped to `rg-mokcloudsite-prod`, with email alerts at 50% (CA$25), 80% (CA$40), and 100% (CA$50) |

---

## 3. The Business Problem This Project Solves

Small businesses with simple online presences — a homepage, a menu, a contact page, an about us — face one of two website problems that silently cost them customers every day.

**Situation 1 — Using infrastructure bigger than the job requires.**
Small companies and startups running a simple marketing website on a cloud virtual machine pay for dedicated CPU, RAM, and storage 24/7, which sit idle the vast majority of the time. Research shows that the average VM runs at just 7–12% CPU utilization, meaning businesses are paying full price for capacity they almost never use.

**Situation 2 — Cheap hosting that destroys customer trust.**
They chose a $3/month hosting plan. The site is slow, especially for visitors far from the data centre. Worse, the SSL/TLS certificate expired and nobody noticed. The browser now shows "Your connection is not private" on their homepage. Studies show 82% of users immediately leave an insecure site and 60% are unlikely to return. The business is losing customers not because of their product — but because of a certificate nobody renewed.

13% of active websites globally still have no TLS certificate. Nearly 30% that have one are misconfigured. And small businesses are the most exposed group — they are the least likely to have an IT team managing any of this. The result is slow sites, browser security warnings, and customers who leave before seeing a single product.

**That is exactly what this project delivers to solve both situations.**

By hosting the website on Azure Blob Storage and delivering it through Azure Front Door Standard:

- The business pays near zero when traffic is low — there is no server sitting idle
- Every visitor gets content delivered from the nearest edge location on earth. A customer in Vancouver and a customer in London both get a fast experience
- HTTPS is automatic and the certificate never expires — Azure Front Door manages and renews it silently
- The custom domain works correctly and builds immediate trust
- A budget alert means the business owner never opens their Azure bill and gets a surprise — they know the moment spend starts climbing abnormally
- There is no web server to patch, no OS to update, no certificate to renew manually

---

## 4. Prerequisites

Before starting, you need:

- **An active Azure subscription** with permission to create resource groups, storage accounts, Front Door profiles, and DNS zones
- **A registered domain** from any registrar that supports custom nameservers (this project used Namecheap with `mokcloud.site`)
- **Your static site files ready** — at minimum: `index.html`, `404.html`, and any CSS/JS files

---

## 5. Deployment Steps

See [DEPLOYMENT.md](DEPLOYMENT.md) for the full step-by-step deployment guide.

---

## 6. Problems Faced and Solutions

### Health probe failing silently
**Problem:** After configuring Front Door, the site was not loading. The health probe was set to HTTP by default, but the storage account had "Require secure transfer" enabled — meaning it only accepts HTTPS. Front Door was sending HTTP probes, the origin was rejecting them, Front Door marked the origin as unhealthy, and returned errors.

**Solution:** Changed the health probe protocol from HTTP to **HTTPS** in the origin group settings in the Azure Portal.

---

### "Your connection is not private" error on custom domain
**Problem:** After adding custom domains and enabling the route, visiting `https://www.mokcloud.site` showed a browser security warning. The certificate being served was a wildcard `*.azureedge.net` certificate — not the correct certificate for the custom domain.

**Solution:** This was a timing issue — the TLS certificate was still being provisioned by Azure Front Door. The certificate status showed "Deployed" in the portal but the certificate had not yet propagated to all Front Door POPs. Waited approximately **37 minutes** before the site loaded correctly with the custom domain certificate. Also opened the site in an **incognito window** to bypass the browser's cached bad certificate.

---

### DNS propagation delay after updating nameservers at Namecheap
**Problem:** After updating the nameservers at Namecheap to point to Azure DNS, the custom domain did not resolve immediately. The site was unreachable for several hours after the change.

**Solution:** Waited for DNS propagation — approximately **8 hours** in this case. Confirmed propagation using `nslookup -type=NS mokcloud.site` in PowerShell and looking for the Azure nameservers (`ns1-01.azure-dns.com` etc.) in the response. No action required — propagation completes on its own.

---

### Trying to fully lock the storage endpoint on Front Door Standard
**Problem:** Attempted many configurations to block the raw storage endpoint — including trying to allow only Azure services (`Microsoft.cdn/profile`) under the Networking section — but none of them fully worked.

**Solution:** This is a known architectural limitation of Front Door Standard with static website hosting. The static website endpoint is publicly accessible by design. A complete origin lockdown requires **Front Door Premium with Private Link** — but for a public marketing site serving content that is intentionally public, Front Door Standard is the correct and cost-appropriate tier. Understanding the threat model before choosing a solution is what led to this conclusion.

---

### CSS not loading locally on 404.html
**Problem:** When opening the 404 page locally via Live Server, the browser showed the page as unstyled — `style.css` was not loading.

**Solution:** Added `/site` to the CSS path — changed `/css/style.css` to `/site/css/style.css`. The `/site` folder is where all website files are stored in the project. Live Server was serving from the project root, so the path needed to include the subfolder name.

---

*Built by Mokenyu · Cloud Engineering Student · Canada*
