### SQL Managed Instance (MI)
## 📘 Azure SQL Database Ledger 

### 🔹 What is Ledger in Azure SQL Database?

The **Ledger** feature in Azure SQL Database provides **cryptographic proof of data integrity**.

* It ensures data hasn’t been tampered with, even by privileged users like DBAs.
* It uses **blockchain-like cryptographic hashing** to build a verifiable chain of transactions.
* It allows external parties (auditors, regulators, partners) to confirm that the database content is trustworthy.

👉 In short: Ledger is a **tamper-proof, blockchain-style audit trail** for SQL data.

---

### 🔑 Why Use Ledger?

* **Data Integrity** → Prevent undetected tampering.
* **Compliance** → Meet regulatory/audit requirements (finance, healthcare, govt).
* **Trust** → Provide verifiable proof to external entities.

---

### 📌 Types of Ledger Tables

1. **Updatable Ledger Table**

   * Supports `INSERT`, `UPDATE`, `DELETE`.
   * Keeps a **history table** automatically.

2. **Append-only Ledger Table**

   * Only `INSERT` allowed.
   * Ideal for logs, financial transactions, or immutable records.

---

### ✅ Example Scenario

Banking system with a `Transactions` table:

* Without ledger → a rogue DBA could change balances.
* With ledger → every change is hashed & verifiable, tampering is detectable.

SQL Example:

```sql
CREATE TABLE Transactions (
    TransactionID INT PRIMARY KEY,
    AccountID INT,
    Amount DECIMAL(10,2),
    TransactionDate DATETIME2
)
WITH (LEDGER = ON);
```

---

### 🔹 Ledger Configuration in Azure Portal

When creating a database, you can configure **Ledger settings** at the **database level**:

### 1. Ledger Database Option

* If enabled, **all tables** in the database will automatically be **ledger tables** (updatable).
* This choice is **irreversible** once the DB is created.
* If not enabled, you can still create specific ledger tables later using T-SQL.

**Option in Portal:**
✔️ *Enable for all future tables in this database*

---

### 2. Digest Storage

Ledger generates **digest files** (cryptographic proofs of data state).
These digests can be stored securely and used later for verification.

You can store digests in:

* **Azure Storage** → Simple, cost-effective.
* **Azure Confidential Ledger (ACL)** → A dedicated, tamper-proof, immutable log service. Stronger guarantees, but incurs extra cost.

**Portal Options:**

* ✔️ *Enable automatic digest storage*
* Storage type:

  * Azure Storage
  * Azure Confidential Ledger (recommended for compliance-critical data)

Example (from your screenshot):

* Storage type = **Azure Confidential Ledger**
* ACL created: `newConfidentialLedger`
* Region = **(US) West US 2**

---

## 🔒 How Verification Works

* Every transaction is hashed and linked (chain of hashes).
* Digest files are generated and stored securely (in ACL or Storage).
* Later, auditors can verify the DB state matches the digest.
* If any row was altered/deleted, verification fails.

---

## ⚡ Summary

* **Ledger** in Azure SQL Database = tamper-proof audit trail.
* Configure at DB creation or table level.
* Use **Azure Confidential Ledger** for strongest protection.
* Ideal for industries requiring **compliance, transparency, and trust**.

---
