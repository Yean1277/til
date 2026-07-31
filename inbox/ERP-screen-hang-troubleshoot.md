# Troubleshooting Report: MySoft ERP - Good Receive Detail Screen Hang (Not Responding)

**Document Type:** Troubleshooting Article
**Affected Module:** MySoft ERP300 - Good Receive Note (GRN) Detail Screen
**Status:** Resolved
**Environment:** Windows Server (IE ESC enabled), SQL Server (SQLEXPRESS instance)

---

## 1. Issue Summary

Every time the user logs into MySoft ERP300 and opens the **Good Receive Detail (GRN Detail)** screen on this specific PC, the application **consistently hangs**, with Windows showing **Not Responding**. The issue only occurs on this PC — other PCs opening the same screen work normally.

**Key symptoms:**
- Task Manager shows CPU usage steady at **~25%** (this PC has 4 cores / 4 logical processors, so 25% ≈ one core fully saturated)
- Memory usage stabilizes at around **16180K** and stops growing (ruling out a continuously growing memory leak)
- The process must be force-killed to exit

---

## 2. Diagnostic Steps

### 2.1 Resource Usage Analysis (Task Manager / Resource Monitor)
Using Task Manager and Resource Monitor confirmed:
- CPU pinned at one full core → indicates a **single-thread infinite loop**, not waiting on an external resource (network / lock)
- Memory not growing further → rules out a data-loading stage issue; the problem is in **subsequent processing logic**

### 2.2 Handle Inspection
In Resource Monitor's **Associated Handles**, found:
- A mutex named `ZonesLockedCacheCounterMutex` (related to WinINet / IE security zones)
- This PC has **Internet Explorer Enhanced Security Configuration (IE ESC)** enabled

This initially suggested an embedded WebBrowser control being blocked by IE ESC, but this direction was **later ruled out** after further investigation.

### 2.3 Process Monitor (procmon) Investigation
Using Sysinternals **Process Monitor** to capture process behavior revealed:
- Over **1.3 million** `RegOpenKey` operations in a short time
- Target path: `HKCU\Software\Classes\CLSID\{0000051A-0000-0010-8000-00AA006D2EA4}\ExtendedErrors`
- All results: `NAME NOT FOUND`

Analyzing the **Stack (call stack)** confirmed the call originated from:
```
oledb32.dll → DllGetClassObject
msado15.dll (ADO 15.0)
```

**Conclusion:** This CLSID belongs to the standard **ADO / OLE DB Rich Error Info lookup mechanism**, not a missing COM component (confirmed by comparing registry structure against a working PC — they matched). This mechanism was triggered over a million times, indicating the program was **repeatedly hitting the same ADO/OLE DB error**, with each failure triggering another lookup, forming an infinite loop.

### 2.4 SQL Server Profiler Capture (Root Cause Confirmed)
Using **SQL Server Profiler** connected to the `APPSRV\SQLEXPRESS` instance, with **Errors and Warnings** (Exception / User Error Message) and **SQL:BatchCompleted / RPC:Completed** events enabled, the issue was reproduced and the following key error was captured:

```
EventClass: Exception / User Error Message
TextData: Invalid object name 'SSTImportK1'.

SQL statement:
SELECT DISTINCT GRNItemNo FROM SSTImportK1 WHERE GRNVoucherNo='HQ03263'
```

**Confirmed Root Cause:**
`SSTImportK1` is a table that **does not exist** (likely intended as a dynamically created / use-then-drop staging table). When loading the GRN Detail screen for a specific GRN Voucher No. (e.g. `HQ03263`), the program executes a SELECT against this table. Since the table doesn't exist, SQL Server immediately returns `Invalid object name`. The program has **no exception handling or loop-break logic** for this error, resulting in:

```
Execute query → Error (Invalid object name)
             → Triggers ADO error-detail lookup (ExtendedErrors CLSID)
             → Still errors → Executes query again
             → ... (infinite loop)
```

Massive repeated execution → CPU saturates one core → UI thread blocked → screen becomes Not Responding.

---

## 3. Why Doesn't It Hang at Login — Only When Entering This Screen?

The login step only validates the database connection and account credentials, and **does not involve the `SSTImportK1` table**, so login completes normally. The query against `SSTImportK1` is only triggered when entering the **GRN Detail** screen and loading detail data for a specific voucher — which is why the hang only occurs at that point.

---

## 4. Resolution

**Fix applied:** Created the missing table `SSTImportK1` in the database.

Once the table existed, the SELECT statement (`SELECT DISTINCT GRNItemNo FROM SSTImportK1 WHERE GRNVoucherNo='...'`) no longer returned `Invalid object name`, the ADO error-detail lookup loop stopped triggering, and the Good Receive Detail screen loaded normally.

**Notes for future occurrences / other PCs:**
- This confirms `SSTImportK1` is expected to exist as a standing table (or at minimum needs to exist before this screen is used) rather than being created per-transaction by a working import flow — otherwise creating it manually would only be a temporary patch.
- If this table gets dropped again (e.g. by a cleanup script, a failed deployment, or a maintenance job), the same hang will recur. Worth checking with the vendor/DBA whether `SSTImportK1` should be part of the standard schema/deployment scripts so it isn't accidentally missing on any given environment.
- Recommend still logging this with MySoft support so they can add proper error handling (table existence check) on their end — that would prevent a full application hang even if the table goes missing again in the future.

---

## 5. Ruled-Out Directions

| Investigation direction | Outcome |
|---|---|
| IE Enhanced Security Configuration (IE ESC) blocking an embedded browser control | Not confirmed, ruled out |
| Missing COM component CLSID / corrupted registry | Registry structure matched a working PC — not the root cause |
| SQL Server instance name mismatch / connection failure | Login succeeds normally, ruling out a connection-layer issue |
| Re-registering `oleaut32.dll` via `regsvr32` | Ineffective attempt (and registration itself failed with Error 0x80070005 Access Denied) |

---

## 6. Tools Used

- **Task Manager** — Initial check of CPU / Memory usage
- **Resource Monitor** — Inspected Associated Handles (mutex, CLSID references)
- **Process Monitor (procmon)** — Captured registry/file operations, identified the infinite-loop behavior and call stack
- **SQL Server Profiler** — Captured the actual SQL statement executed and the database-level error, ultimately confirming the root cause

---

*This document was compiled based on an actual troubleshooting session, for reference in resolving similar application hang / Not Responding issues.*
