# Azure Static Site Delivery

Blob Storage · Azure Front Door Standard · Custom Domain · HTTPS · Cost Monitoring

*A professional, globally distributed website delivered without a single server.*

---

## 1. Architecture

![Architecture Diagram](Images/Architecture%20Diaghram.png)

---

## 2. Tools Used

### Azure Services

- **2 Resource Groups:** One for my Azure DNS zone and the other for every other resource. I'm separating the DNS zone I can't delete and deploy at will due to DNS propagation.

- **2 Azure Blob Storage (Canada Central and East US):** Hosts the static website files inside the $web container and for recovery where failover occurs in case one region fails

- **Azure Front Door Standard:** Delivers content from 100+ global edge locations. Handles caching, HTTPS termination, managed TLS certificates, custom domain binding, health probes, and routing rules.

- **Azure DNS Zone (mokcloud.site):** Manages DNS records for the custom domain. Nameservers delegated from Namecheap to Azure DNS so the domain is fully managed inside Azure.

- **Log Analytics Workspace:** Receives Front Door access logs, health probe logs, and all metrics via a diagnostic setting.

- **Azure Cost Management:** Monthly budget of CA$50 scoped to rg-staticsite-prod, with email alerts at 50% ($25), 80% (CA$40) and 100% (CA$50).

---

## 4. The Business Problem This Project Solves

### The Problem

Small businesses with simple online presences, a homepage, a menu, a contact page, an about us, face one of two website problems that silently cost them customers every day.

Situation 1: Using infrastructure that is bigger than the job requires. Small companies and startups running a simple marketing website on a cloud virtual machine pay for dedicated CPU, RAM, and storage 24/7, which sit idle the vast majority of the time. Research shows that the average VM runs at just 7–12% CPU utilization, meaning businesses are paying full price for capacity they almost never use.

Situation 2: Cheap hosting that destroys customer trust. They chose a $3/month hosting plan. The site is slow, especially for visitors far from the data centre. Worse, the SSL/TLS certificate expired and nobody noticed. The browser now shows "Your connection is not private" on their homepage. Studies show 82% of users immediately leave an insecure site and 60% are unlikely to return. The business is losing customers not because of their product, but because of a certificate nobody renewed.

13% of active websites globally still have no TLS certificate. Nearly 30% that have one are misconfigured. And small businesses are the most exposed group. They are the least likely to have an IT team managing any of this. The result is slow sites, browser security warnings, and customers who leave before seeing a single product.

**That is exactly what this project delivers to solve the situations.**

By hosting the website on Azure Blob Storage and delivering it through Azure Front Door Standard:

- The business pays near zero when traffic is low, there is no server sitting idle.

- Every visitor gets content delivered from the nearest edge location on earth. A customer in Vancouver and a customer in London both gets a fast experience.

- HTTPS is automatic and the certificate never expires. Azure Front Door manages and renews it silently.

- The custom domain works correctly and builds immediate trust.

- A budget alert means the business owner never opens their Azure bill and gets a surprise. They know the moment spend starts climbing abnormally.

- There is no web server to patch, no OS to update, no certificate to renew manually.

---

## 5. Prerequisites

Before starting, you need:

- **An active Azure subscription** with permission to create resource groups, storage accounts, Front Door profiles, and DNS zones

- **A registered domain** from any registrar that supports custom nameservers (this project used Namecheap with mokcloud.site)

- **Your static site files ready** — at minimum: index.html, 404.html, and any CSS/JS files

---

## 6. Problems Faced and Solutions

### Problems

- **Health probe failing silently:** After configuring Front Door, my site was not loading. The health probe was set to HTTP by default, but the storage account had "Require secure transfer" enabled, meaning it only accepts HTTPS.

**Solution:** I changed the health probe protocol from HTTP to HTTPS in the origin group settings in the Azure Portal.

---

- **"Your connection is not private" error on custom domain:** After adding my custom domains and enabling the route, visiting www.mokcloud.site showed a browser security warning. The certificate being served was a wildcard *.azureedge.net certificate and not the correct certificate for the custom domain.

**Solution:** This was a timing issue; the TLS certificate was still being provisioned by Azure Front Door. The certificate status showed deployed but still the name of didn't change until after 37mins.

---

- **DNS propagation delay after updating nameservers at Namecheap:** After updating the nameservers at Namecheap to point to Azure DNS, the custom domain did not resolve immediately. The site was unreachable for several hours after the change.

**Solution:** Waited for DNS propagation for approximately 8 hours. (Probably it was done earlier)

---

- **Trying to fully lock storage endpoint on Front Door Standard:** Tried lots of attempts like allow Azure Services (Microsoft.cdn/profile) under the Networking section which I still couldn't find and many other configurations.

**Solution:** I learned that this is a known architectural limitation of Front Door Standard with static website hosting. The content is public by design. The full fix requires upgrading to Front Door Premium with Private Link (which is a kill for this case). For a public marketing site, Standard remains the correct tier.

---

- **CSS not loading locally on 404.html:** The browser showed my website like a file meaning style.css wasn't functional.

**Solution:** I added "/site" to the path "/css/style.css" making it "/site/css/style.css" and style.css worked. The site loaded with the set design. "/site" is where all my website files are.
