## 🔁 Redo Log Multiplexing

---

### 📘 What Are Redo Logs?

Redo logs store **all changes made to the database**, helping in **instance recovery** in case of crash or failure. These are critical for **data durability and integrity**.

---

### 🧾 What Redo Logs Contain

Redo logs capture **every change made to data** at a **low level**, such as:

* 📝 **DML changes** — `INSERT`, `UPDATE`, `DELETE`
* 🔧 **DDL operations** — `CREATE TABLE`, `ALTER INDEX`, etc.
* 🧱 **Undo data** — rollback data for uncommitted transactions
* 📦 **Transaction control** — markers for `COMMIT` and `ROLLBACK`
* 🔄 **Data dictionary changes** — system metadata updates
* 📚 **Temporary tablespace activity** (during recovery if used)

> ✅ Redo logs **do NOT record `SELECT` statements** — only **modifications** to data

---

### 🧩 Redo Log Structure

* A **Redo Log Group** = one or more **Redo Log Members**
* Each **group** is written **in parallel** to all its members
* Oracle reuses groups in **circular fashion** 🔄

> ❗Redo logs are **vital for crash recovery**. Keeping them on a single disk = single point of failure!

---

### 🔄 Circular Working Mechanism

```mermaid
flowchart LR
    A[Group 1] --> B[Group 2] --> C[Group 3] --> A
    style A fill:#fffae6,stroke:#d35400,stroke-width:2px
    style B fill:#e8f8f5,stroke:#1abc9c,stroke-width:2px
    style C fill:#fbeee6,stroke:#d35400,stroke-width:2px
```

🌀 Oracle continuously **cycles through redo log groups** — it writes to the `CURRENT` group, then switches in sequence.

---

### 🔒 Why Multiplex Redo Logs?

To avoid **data loss or corruption** in case of disk or file failures.

### ✅ Benefits of Multiplexing:

* Reduces chance of losing redo data
* Continues operation even if one member is corrupted
* Recommended by Oracle in **all production environments**

---

### 🧠 Redo Log Statuses (`V$LOG`)

| Status     | Meaning                                        |
| ---------- | ---------------------------------------------- |
| `CURRENT`  | Group actively being written to 🟢             |
| `ACTIVE`   | Still needed for recovery, not reusable yet 🟡 |
| `INACTIVE` | Archived and **safe to drop** ⚫                |

🔍 Check log group statuses:

```sql
SELECT GROUP#, STATUS FROM V$LOG ORDER BY GROUP#;
```

> ⚠️ Only `INACTIVE` groups/members can be dropped safely.

---

### 📏 How to Check Redo Log Size (MB)

```sql
SELECT GROUP#, BYTES/1024/1024 AS SIZE_MB FROM V$LOG ORDER BY GROUP#;
```

---

## ⚙️ Redo Log Multiplexing – Practical Steps (for ORADB)

---

### 🔍 Step 1: Validate Existing Redo Log Members

```bash
sqlplus / as sysdba
```

```sql
SELECT GROUP#, MEMBER FROM V$LOGFILE ORDER BY GROUP#;
```

👁️ Confirm redo logs exist under `/u01/oradata/ORADB`.

```sql
EXIT;
```

---

### 📁 Step 2: Create Directory for New Members

```bash
mkdir -p /u02/oradata/ORADB
```

🏗️ This will act as the **second disk location** for new members.

---

### ➕ Step 3: Add New Members to Each Group

```bash
sqlplus / as sysdba
```

```sql
ALTER DATABASE ADD LOGFILE MEMBER '/u02/oradata/ORADB/redo01b.log' TO GROUP 1;
ALTER DATABASE ADD LOGFILE MEMBER '/u02/oradata/ORADB/redo02b.log' TO GROUP 2;
ALTER DATABASE ADD LOGFILE MEMBER '/u02/oradata/ORADB/redo03b.log' TO GROUP 3;
```

✅ Now each group is **mirrored**: 1 file in `/u01`, 1 in `/u02`

---

### 🧾 Step 4: Verify All Redo Members Are Added

```sql
SELECT GROUP#, MEMBER FROM V$LOGFILE ORDER BY GROUP#;
```

Confirm that each group has members in both `/u01` and `/u02`.

```sql
EXIT;
```

---

## 🗑️ Drop Old Redo Log Members

---

### 🔍 Step 5.1: Check Log Group Statuses

```bash
sqlplus / as sysdba
```

```sql
SELECT GROUP#, STATUS FROM V$LOG ORDER BY GROUP#;
```

🎯 Identify groups with `INACTIVE` status.

---

### 🔃 Step 5.2: Switch Logs to Make Groups INACTIVE (If Needed)

```sql
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM CHECKPOINT;
```

🔁 Re-check until the **targeted group becomes `INACTIVE`**:

```sql
SELECT GROUP#, STATUS FROM V$LOG ORDER BY GROUP#;
```

---

### 🧠 Step 5.3: Identify Which Members to Drop

Run:

```sql
SELECT L.GROUP#, L.MEMBER, LG.STATUS
FROM V$LOGFILE L
JOIN V$LOG LG ON L.GROUP# = LG.GROUP#
WHERE L.MEMBER LIKE '/u01/oradata/ORADB/%';
```

✅ Only members from `INACTIVE` groups can be dropped.

---

### ❌ Step 5.4: Drop Old Redo Log Members from `/u01`

```sql
ALTER DATABASE DROP LOGFILE MEMBER '/u01/oradata/ORADB/redo01a.log';
ALTER DATABASE DROP LOGFILE MEMBER '/u01/oradata/ORADB/redo02a.log';
ALTER DATABASE DROP LOGFILE MEMBER '/u01/oradata/ORADB/redo03a.log';
```

> 🚫 Skip any member that still belongs to a `CURRENT` or `ACTIVE` group.

```sql
EXIT;
```

---

### 🧹 Step 5.5: Remove Physical Files at OS Level

```bash
rm /u01/oradata/ORADB/redo01a.log
rm /u01/oradata/ORADB/redo02a.log
rm /u01/oradata/ORADB/redo03a.log
```

📁 Oracle **does not** delete physical redo log files automatically.

---

## 🧾 Summary Table

| ✅ Step | Description                                       |
| ------ | ------------------------------------------------- |
| 1      | Check current redo members via `V$LOGFILE`        |
| 2      | Create new directory `/u02/oradata/ORADB`         |
| 3      | Add members to each group                         |
| 4      | Verify new members are added correctly            |
| 5.1    | Check if log groups are `INACTIVE`                |
| 5.2    | Force switches + checkpoint to make them inactive |
| 5.3    | Query members that belong to `INACTIVE` groups    |
| 5.4    | Drop old members from `/u01`                      |
| 5.5    | Manually remove old redo files from disk          |

---
