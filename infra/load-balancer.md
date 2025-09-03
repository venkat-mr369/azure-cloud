

## 🌐 Types of Load Balancers in Azure

## 1. **Azure Load Balancer** (Layer 4, TCP/UDP)

* Works at the **transport layer** (OSI Layer 4).
* Routes traffic based on **IP address and port** (not URL/HTTP headers).
* Two SKUs:

  * **Public Load Balancer** → balances internet traffic into Azure VMs.
  * **Internal Load Balancer** → balances traffic **inside a VNet** (no internet).

🔹 **Example**:

* You have 3 VMs hosting a web app (IIS servers) in `10.10.1.0/24`.
* You create a **Public Load Balancer** with frontend IP `20.x.x.x`.
* Clients hitting `http://20.x.x.x` → requests are distributed to all 3 backend VMs.

---

## 2. **Azure Application Gateway** (Layer 7, HTTP/HTTPS)

* Works at the **application layer** (OSI Layer 7).
* Supports **URL-based routing, path-based routing, SSL termination, WAF (Web Application Firewall)**.
* Best for **web applications**.

🔹 **Example**:

* You run:

  * `/api` → backend API servers.
  * `/app` → frontend web servers.
* Application Gateway routes `/api` traffic to API pool, `/app` traffic to frontend pool.
* It can also handle **HTTPS offloading** (clients connect via SSL, Gateway decrypts, forwards HTTP internally).

---

## 3. **Azure Traffic Manager** (DNS-based, Global)

* Works at **DNS level**.
* Routes users to **different endpoints/regions** based on rules.
* Supports **geographic routing, performance routing (closest region), priority routing**.
* Does **not** directly load balance TCP/HTTP — it only provides DNS redirection.

🔹 **Example**:

* You have two deployments of a website:

  * West Europe
  * East US
* Users in Europe → DNS resolves to West Europe.
* Users in US → DNS resolves to East US.
* If West Europe goes down, DNS automatically routes everyone to East US.

---

## 4. **Azure Front Door** (Global, Layer 7 CDN + Load Balancer)

* Microsoft’s **global entry point** for applications.
* Works at **Layer 7** with **content delivery network (CDN), caching, SSL offload, WAF**.
* Provides **global failover + acceleration** (anycast routing).

🔹 **Example**:

* You run an e-commerce website in multiple Azure regions.
* Customers across the globe hit **one global URL** (e.g., `shop.contoso.com`).
* Front Door routes users to the **closest healthy region** and caches static content at edge POPs for speed.

---

# 📊 Quick Comparison

| Service                 | OSI Layer    | Scope           | Best For                                               |
| ----------------------- | ------------ | --------------- | ------------------------------------------------------ |
| **Azure Load Balancer** | L4 (TCP/UDP) | Regional (VNet) | VM traffic, database HA (Always On), internal services |
| **Application Gateway** | L7 (HTTP/S)  | Regional        | Web apps, WAF, URL/path routing                        |
| **Traffic Manager**     | DNS-based    | Global          | Multi-region routing, disaster recovery                |
| **Azure Front Door**    | L7 (HTTP/S)  | Global          | Web apps with global users, CDN + WAF                  |

---

# ✅ Which to Choose (with Example Use Cases)

1. **If you only need TCP/UDP balancing** inside VNet:
   → Use **Azure Load Balancer** (example: SQL Always On cluster across 3 VMs).

2. **If you need HTTP routing, SSL, or WAF**:
   → Use **Application Gateway** (example: Web App with `/api` and `/frontend`).

3. **If you need global DNS-level failover between regions**:
   → Use **Traffic Manager** (example: Active-Passive deployment in East US + West Europe).

4. **If you need global web acceleration + WAF**:
   → Use **Azure Front Door** (example: Global e-commerce website serving users worldwide).

---

👉 Pro tip: In **real-world deployments**, many companies **combine them**:

* **Front Door (global)** → **Application Gateway (regional)** → **Internal Load Balancer (VNet backend)**.

---


