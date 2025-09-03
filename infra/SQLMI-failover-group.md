Here’s a refined **step‑by‑step, non‑scripted setup guide** for configuring an Azure SQL Managed Instance failover group using the IP ranges you specified (`10.10.1.0/24` and `20.20.1.0/24`) in East US—based on the official Microsoft documentation and best practices cited below:

---

## General Steps to Set Up a Failover Group for Azure SQL Managed Instance (MI)

### 1. **Plan and Prepare Azure Resources**

* **Resource Groups & Regions**
  Create resource group(s) in **East US**. If you plan to deploy both instances in the same region (East US), be aware: failover groups **cannot** be created within the same Azure region ([GitHub][1], [hcmarque.github.io][2]).
  → **Recommendation**: Use a **paired region** (e.g., East US 2) for the secondary instance to comply with constraints and performance recommendations ([GitHub][1], [hcmarque.github.io][2]).

### 2. **Configure Virtual Networks (VNets)**

* **Primary VNet**

  * Assign the IP range **10.10.1.0/24** for the VNet and subnet used by the primary MI.
* **Secondary VNet**

  * Assign the IP range **20.20.1.0/24** for the secondary MI’s VNet/subnet.
* **Ensure**: The IP ranges **must not overlap** ([GitHub][1], [Medium][3], [Go2Share][4]).
* **Connectivity**

  * Establish connectivity between the two VNets using either **global VNet peering**, **ExpressRoute**, or **VPN gateways** ([All About Tech][5], [hcmarque.github.io][2], [Medium][3]).

### 3. **Set Up Network Security**

* On both primary and secondary subnets, configure **Network Security Group (NSG) rules** to allow **inbound and outbound TCP traffic** on ports:

  * **5022**
  * **11000–11999**
    These ports are required for replication and failover traffic ([GitHub][1], [Microsoft Tech Community][6], [Go2Share][4]).

### 4. **Create the Primary SQL Managed Instance**

* Deploy your primary MI in East US, on your primary VNet/subnet with 10.10.1.0/24.
* Choose your service tier, vCores, and storage as required.

### 5. **Create the Secondary SQL Managed Instance**

* Deploy your secondary MI in the paired region (instead of East US), placed on the secondary VNet/subnet using 20.20.1.0/24.
* **Important configuration checks**:

  * It **must be empty** (no user databases) at time of failover group creation ([All About Tech][5], [Medium][3]).
  * It must match the **service tier** and **storage size** of the primary MI. Matching compute size is strongly recommended ([Medium][3], [GitHub][1], [All About Tech][7]).
  * Both MIs must be in the **same DNS zone**—when creating the secondary, you must specify the primary’s DNS zone ID. Once set, it cannot be changed ([GitHub][1], [Medium][3]).
  * Both should use the **same update policy** ([GitHub][1]).

### 6. **Create the Failover Group**

* In the Azure Portal:

  1. Navigate to the **primary MI**, go to **Data Management → Failover groups**.
  2. Select **Add group**.
  3. Configure:

     * **Failover group name**
     * Choose the **secondary instance**
     * Set a **failover policy** (Manual is recommended for controlled failovers)
     * Optionally disable **failover write rights** if it’s for disaster recovery only ([GitHub][1]).
  4. Create the group; the initial **seeding** of data begins (this can take time depending on data size and network, with typical speeds around \~360 GB/hour) ([All About Tech][7], [Go2Share][4]).

### 7. **Locate & Use Listener Endpoints**

* Once created, get the **Read/Write listener endpoint** (routes to the current primary MI) and the optional **Read-only endpoint** (routes to secondary) via the portal under the Failover group blade ([GitHub][1], [Go2Share][4]).
* Update your application connection strings to use these endpoints so failovers are seamless.

### 8. **Test Connectivity & Failover**

* Test network reachability between MI instances, specifically on ports 5022 and 11000–11999 ([Microsoft Tech Community][6]).
* Perform a **manual failover** via portal or supported tools (requires appropriate RBAC roles) ([Microsoft Tech Community][8], [GitHub][1]).

### 9. **Ongoing Management & Limitations**

* **Synchronize configurations**: Changes to the primary (e.g., backup settings) must be manually mirrored to the secondary ([GitHub][1]).
* **Scaling guidance**:

  * On upgrades: scale the **secondary first**, then the primary.
  * On downgrades: scale the **primary first**, then the secondary ([GitHub][1]).
* **Key limitations**:

  * Failover group cannot span same region.
  * One MI can belong to only one failover group.
  * Instancing across subscriptions may require PowerShell or REST API; portal doesn’t support cross-subscription creation ([GitHub][1]).
  * If seeding fails to complete within 7 days, the operation rolls back ([GitHub][1]).
  * User objects from system databases (e.g., logins, jobs) do not replicate automatically—must synchronize manually ([GitHub][1]).

---

## Summary Table

| Step | Activity                                                                       |
| ---- | ------------------------------------------------------------------------------ |
| 1    | Plan resource group(s), choose paired region (avoid same-region)               |
| 2    | Create non-overlapping VNets: 10.10.1.0/24 (primary), 20.20.1.0/24 (secondary) |
| 3    | Configure NSG rules: open ports 5022 & 11000–11999 inbound/outbound            |
| 4    | Deploy primary MI in East US                                                   |
| 5    | Deploy secondary MI in paired region with matching specs & empty state         |
| 6    | Create Failover Group via portal, start seeding                                |
| 7    | Obtain and use listener endpoints for app connectivity                         |
| 8    | Validate network connectivity and test manual failover                         |
| 9    | Keep configurations synchronized; be mindful of limitations                    |

---

Let me know if you'd like help choosing a paired region (like East US 2), or want guidance on testing failover behavior or connection string configurations!

[1]: https://github.com/MicrosoftDocs/sql-docs/blob/live/azure-sql/managed-instance/failover-group-configure-sql-mi.md?utm_source=chatgpt.com "sql-docs/azure-sql/managed-instance/failover-group-configure-sql-mi.md at live · MicrosoftDocs/sql-docs · GitHub"
[2]: https://hcmarque.github.io/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/?utm_source=chatgpt.com "SQL-Managed Instance: Disaster Recovery Reference Architecture | SQL-Managed-Instance-Disaster-Recovery-Architecture-Design"
[3]: https://medium.com/%40rashisingh202020/business-continuity-and-disaster-recovery-setup-along-with-sql-mi-in-replication-83ac9c3181ce?utm_source=chatgpt.com "Business Continuity and Disaster Recovery setup along with SQL MI in replication | by Rashisingh | May, 2025 | Medium"
[4]: https://www.go2share.net/article/azure-sql-failover-group?utm_source=chatgpt.com "Configuring Azure SQL Failover Group for High Availability"
[5]: https://blobeater.blog/2021/11/22/failover-groups-for-managed-instances/?utm_source=chatgpt.com "Failover Groups for Managed Instances | All About Tech"
[6]: https://techcommunity.microsoft.com/blog/azuresqlblog/how-to-test-failover-group-connectivity-between-primary-and-secondary-sql-manage/3058443?utm_source=chatgpt.com "How-to test failover group connectivity between primary and secondary SQL Managed Instances"
[7]: https://blobeater.blog/2021/12/09/sql-managed-instance-failover-groups-quick-tips/?utm_source=chatgpt.com "SQL Managed Instance – Failover Groups Quick Tips | All About Tech"
[8]: https://techcommunity.microsoft.com/t5/azure-sql-blog/user-initiated-manual-failover-on-sql-managed-instance/ba-p/1538803?utm_source=chatgpt.com "User initiated manual failover on SQL Managed Instance - Microsoft Community Hub"
