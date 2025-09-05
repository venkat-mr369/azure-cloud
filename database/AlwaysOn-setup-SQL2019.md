### Before Configuring **SQL Server 2019 Always On Availability Groups (AGs)** across **two Azure regions (East US and West US)**. need to set up [Network Configuration](network-setup-for-always.on.md)
### Configuring **SQL Server 2019 Always On Availability Groups (AGs)** across **two Azure regions (East US and West US)**.

### 📘 Step-by-Step process: Always On AG in Azure (East US + West US, SQL 2019)

---

#### 1. 📋 Prerequisites

1. **Azure Resources**

   * Two Azure regions: **East US** (Primary) and **West US** (DR).
   * Minimum 2 VMs in East US (for synchronous HA).
   * 1–2 VMs in West US (for DR, asynchronous), below i mentioned only 1 vm
   * A **Domain Controller VM** (or Azure AD DS) replicated to both regions.

   👉 Example:

   ```
   Region East US:
     DC1  -> 10.10.1.4
     SQL1 -> 10.10.1.5
     SQL2 -> 10.10.1.6

   Region West US:
     DC2  -> 10.20.1.4
     SQL3 -> 10.20.1.5
   ```

2. **Networking**

   * Create **VNet in each region**.
   * Set up **VNet Peering** (East US ↔ West US).
   * Configure **subnets** for DC and SQL separately.
   * Ensure DNS from DC is used by all SQL VMs.

3. **Windows Server Failover Clustering (WSFC)**

   * All SQL VMs must be domain-joined.
   * Cluster nodes = SQL1, SQL2, SQL3.

4. **SQL Server 2019 Enterprise** installed.

   * Enable **AlwaysOn Availability Groups feature**.

---

## 2. ⚙️ Windows Setup

### a) Join all VMs to the domain

On each SQL VM:

```powershell
Add-Computer -DomainName corp.local -Credential corp\admin -Restart
```

### b) Configure WSFC

On SQL1 (primary node):

```powershell
New-Cluster -Name SQLCluster -Node SQL1,SQL2,SQL3 -StaticAddress 10.10.1.10
```

* This creates a cluster with 3 SQL nodes.
* Use **Cloud Witness** for quorum (recommended in Azure).

```powershell
Set-ClusterQuorum -CloudWitness -AccountName mystorageacct -AccessKey "StorageKeyHere"
```

---

#### 3. 🗄️ SQL Server Configuration

##### a) Enable Always On

On each SQL VM:

1. Open **SQL Server Configuration Manager**.
2. Go to **SQL Server Services → SQL Server (MSSQLSERVER) → Properties → AlwaysOn High Availability**.
3. Check **Enable AlwaysOn Availability Groups**.
4. Restart SQL service.

##### b) Create AG Database

On SQL1:

```sql
CREATE DATABASE SalesDB;
ALTER DATABASE SalesDB SET RECOVERY FULL;
BACKUP DATABASE SalesDB TO DISK = 'C:\Backup\SalesDB.bak';
BACKUP LOG SalesDB TO DISK = 'C:\Backup\SalesDB.trn';
```

Restore backups on SQL2 and SQL3:

```sql
RESTORE DATABASE SalesDB FROM DISK = 'C:\Backup\SalesDB.bak' WITH NORECOVERY;
RESTORE LOG SalesDB FROM DISK = 'C:\Backup\SalesDB.trn' WITH NORECOVERY;
```

---

#### 4. 🔄 Create Always On Availability Group

On SQL1 (Primary):

```sql
CREATE AVAILABILITY GROUP [AG1]
WITH (AUTOMATED_BACKUP_PREFERENCE = PRIMARY)
FOR DATABASE [SalesDB]
REPLICA ON 
    N'SQL1' WITH (
        ENDPOINT_URL = 'TCP://SQL1.corp.local:5022', 
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT, 
        FAILOVER_MODE = AUTOMATIC),
    N'SQL2' WITH (
        ENDPOINT_URL = 'TCP://SQL2.corp.local:5022', 
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT, 
        FAILOVER_MODE = AUTOMATIC),
    N'SQL3' WITH (
        ENDPOINT_URL = 'TCP://SQL3.corp.local:5022', 
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT, 
        FAILOVER_MODE = MANUAL);
```

On SQL2 and SQL3:

```sql
ALTER AVAILABILITY GROUP [AG1] JOIN;
ALTER DATABASE [SalesDB] SET HADR AVAILABILITY GROUP = [AG1];
```

---

#### 5. 🌐 Configure Listener (Multi-Region)

1. **Create an Internal Load Balancer (ILB)** in each region:

   * East US ILB → 10.10.1.20
   * West US ILB → 10.20.1.20

2. **Cluster Resource Configuration**

   * Add a new cluster IP resource (per ILB).
   * Map ILB frontends to SQL listener DNS.

   Example:

   ```powershell
   Add-ClusterResource -Name AG1_Listener -ResourceType "IP Address" -Group "AG1"
   Set-ClusterParameter -Resource AG1_Listener -Name Address -Value 10.10.1.20
   ```

3. **SQL Listener**

   ```sql
   ALTER AVAILABILITY GROUP [AG1]
   ADD LISTENER 'AG1Listener' (WITH IP (('10.10.1.20','255.255.255.0')));
   ```

   * Clients connect to `AG1Listener.corp.local`, automatically routed.

---

#### 6. ✅ Testing Failover

* **Within East US (SQL1 → SQL2):**

  ```sql
  ALTER AVAILABILITY GROUP [AG1] FAILOVER;
  ```

* **To West US (SQL3):**

  ```sql
  ALTER AVAILABILITY GROUP [AG1] FAILOVER;
  ```

* Check:

  ```sql
  SELECT ag.name, ar.replica_server_name, ar.availability_mode_desc, ar.failover_mode_desc, drs.synchronization_state_desc
  FROM sys.availability_groups ag
  JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
  JOIN sys.dm_hadr_database_replica_states drs ON ar.replica_id = drs.replica_id;
  ```

---

#### 7. 📊 Monitoring

* Use **SQL Server Management Studio (SSMS) → AlwaysOn Dashboard**.
* Use `sys.dm_hadr_*` DMV views for monitoring sync health.
* Enable **Azure Monitor + Log Analytics** for VM health.

---

#### 8. 🎯 Best Practices

* Use **Premium SSDs** for SQL data/logs.
* Place replicas in **Availability Zones** within a region for extra HA.
* Keep East US replicas in **synchronous** and West US replica in **asynchronous**.
* Use **Azure Traffic Manager / Front Door** for global DNS failover.

---

