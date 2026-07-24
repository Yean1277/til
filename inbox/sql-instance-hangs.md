# Troubleshooting: SQL Server (SQLEXPRESS) Instance Hangs and Won't Restart — Non-Yielding Scheduler

## Quick Facts

| | |
|---|---|
| **Instance** | `SQLEXPRESS` (named instance) |
| **Product** | SQL Server 2019 Express Edition, build `15.0.2000.5` (RTM — **no cumulative updates installed**) |
| **OS / Hardware** | Windows 10 Home Single Language, Dell Inspiron 5559, 8083 MB RAM, 4 logical processors (1 socket × 2 cores × HT) |
| **Affected process** | `sqlservr.exe`, PID **4908** |
| **Symptom** | Service stuck in `Stop Pending` / `Change pending...`; restart never completes |
| **Root cause category** | Non-yielding scheduler — internal engine hang, not OS/CPU resource exhaustion |
| **Databases active at time of hang** | `UserCheck`, `AccDatabase1` |

---

## 1. Symptom

The `SQLEXPRESS` instance would not restart. Two things confirmed this wasn't a normal stop/start delay:

- `Get-Service` showed `MSSQL$SQLEXPRESS` sitting in **`StopPending`**, while the separate default instance (`MSSQLSERVER`) was running normally.
- SQL Server Configuration Manager showed the same service stuck in **`Change pending...`**, with Process ID `4908`.
<img width="630" height="323" alt="image" src="https://github.com/user-attachments/assets/9e7a0fa8-f034-4ad2-be38-7af7ae7f5031" />

```powershell
Get-Service | Where-Object {$_.DisplayName -like "SQL Server*"}
```

A hung/stuck service state like this — as opposed to a slow stop — is the first clue that the engine itself is unresponsive rather than just busy.

## 2. Investigation

The instance's `ERRORLOG` (`...\MSSQL15.SQLEXPRESS\MSSQL\Log\ERRORLOG`) was pulled and reviewed. Note: SQL Server writes this file as UTF-16LE, so it needs converting before it's readable as plain text (`iconv -f UTF-16LE -t UTF-8` or equivalent).

### 2.1 Timeline reconstructed from the log

| Time (local) | Event |
|---|---|
| 22/7/2026 21:53:01 | Last known run of this instance (PID 5380) — noted by SQL Server as the "last reported" process, implying the prior shutdown was **not** clean |
| 2026-07-23 09:26:24 | Instance starts up |
| 2026-07-23 09:26:26 | `master` database recovery: **23 transactions rolled forward** — confirms the previous shutdown was a crash/kill, not a graceful stop |
| 2026-07-23 09:26:31 | Startup and recovery complete normally, no errors |
| 2026-07-23 09:31:49 | `xplog70.dll` loads; databases **`UserCheck`** and **`AccDatabase1`** come online (redo starts and finishes almost instantly for both) |
| 2026-07-23 09:33:06 | **First stack dump** — "Non-yielding Scheduler" reported on **Scheduler 0** |
| 2026-07-23 09:33:06 → 10:51:45 | Scheduler 0 continues reporting non-yielding roughly every 20–60 seconds, without recovering |
| 2026-07-23 10:51:45 | **Scheduler 2** *also* begins reporting non-yielding — a second stuck worker thread |
| 2026-07-23 10:51:45 → 14:07:22 | Both schedulers report non-yielding continuously; **274 events on Scheduler 0, 195 on Scheduler 2** |
| 2026-07-23 14:07:22 | Last line in the log — still hung, mid-event, no shutdown message ever recorded |

Total observed hang duration in the log: **~4 hours 34 minutes**, with no self-recovery.

### 2.2 Was this a CPU-exhaustion problem?

It's tempting to assume "the CPU hit 100% and that's why it hung," so this was checked directly against the log rather than assumed. Every non-yielding report includes a `Process Utilization` (SQL Server's own CPU use) and `System Idle` (whole-machine CPU use) reading. Aggregating all 469 samples across the incident:

| Metric | Min | Max | Average |
|---|---|---|---|
| System Idle (whole machine) | 38% | 75% | **57.9%** |
| SQL Server Process Utilization | 19% | 39% | **30.9%** |

**System Idle never dropped below 38%.** The machine had substantial spare CPU capacity for the entire 4.5-hour hang — this was not a system starved of CPU.

Looking at the CPU-time deltas *per stuck thread* instead tells a different story: each of the two non-yielding threads was individually consuming **~75–80% of one CPU core, continuously**, for the whole duration. That's a thread actively spinning/retrying without making progress — not a thread quietly blocked and idle waiting on I/O, and not a system-wide CPU shortage either.

**Conclusion:** the causality is the reverse of "high CPU caused the hang." Something inside the engine caused two specific worker threads to stop cooperating with SQL Server's internal scheduler (SQLOS); the CPU those threads burned while spinning is a *symptom* of that, not the trigger. The rest of the machine — and even SQL Server's other threads — had plenty of room the entire time.

## 3. Root Cause

A **non-yielding scheduler** condition: one worker thread (Scheduler 0) stopped yielding control back to SQLOS shortly after two user databases came online, and ~78 minutes later a second thread (Scheduler 2) did the same. Because SQL Server's core engine threads were wedged, the instance could never process the stop command from the Service Control Manager — which is why the service sat in `Stop Pending` / `Change pending` indefinitely rather than actually stopping.

Likely contributing factors, in rough order of suspicion:

1. **Unpatched engine.** Build `15.0.2000.5` is the original SQL Server 2019 RTM release with **zero cumulative updates** ever applied. Multiple CUs since release have fixed scheduler/spinlock stability issues.
2. **Antivirus / Windows Defender interference** with `sqlservr.exe` or the data/log files at the moment `UserCheck` and `AccDatabase1` came online — a very common cause of this exact pattern on consumer Windows editions running SQL Express.
3. **A wedged lock/latch cascading to a second thread.** The 78-minute gap between the first and second stuck scheduler is consistent with one thread holding a resource that a second thread later blocked on and also got stuck retrying for.
4. **History of unclean shutdowns.** The 23-transaction rollforward on this startup indicates the *previous* run of this instance also ended in a crash/kill rather than a graceful stop — suggesting this may be a recurring condition rather than a one-off.

The exact code path can only be confirmed from the `SQLDump0001.txt` file SQL Server generates alongside `ERRORLOG` at the moment of the first dump (09:33:06) — see Recommendations.

## 4. Steps to Resolve

Because the engine is internally deadlocked, a graceful stop/restart will never complete. The frozen process has to be force-terminated:

```powershell
# Confirm the PID is still current before killing it
Get-Process -Id 4908

# Force-terminate the hung SQL Server process
Stop-Process -Id 4908 -Force

# Confirm the service drops out of Stop Pending
Get-Service 'MSSQL$SQLEXPRESS'
```

If the service does not clear `Stop Pending` within a minute or two of the process dying, a full reboot is the reliable fallback — Windows can occasionally leave the Service Control Manager state stuck even after the underlying process is gone.

Then bring the instance back up:

```powershell
Start-Service 'MSSQL$SQLEXPRESS'
```

## 5. Recommendations / Follow-Up Actions

- [ ] **Retrieve Log Files `ERRORLOG.txt`** from `...\Program Files\Microsoft SQL Server\MSSQL15.SQLEXPRESS\MSSQL\Log\` (generated at 09:33:06) to identify the exact stuck call stack.
- [ ] **Apply the latest SQL Server 2019 Cumulative Update.** Running an unpatched RTM build in production is the single largest outstanding risk found here.
- [ ] **Add antivirus/Defender exclusions** for `sqlservr.exe` and the instance's `DATA`/`Log` folders (including wherever `UserCheck` and `AccDatabase1`'s `.mdf`/`.ldf` files live, if different).
- [ ] **Run `DBCC CHECKDB`** on `UserCheck` and `AccDatabase1` once the instance is stable — they were the two databases actively coming online immediately before the freeze.
- [ ] **Check disk health** (`chkdsk`, SMART status) and the Windows System event log around 09:31–09:33 for any correlating hardware/storage warnings.
- [ ] **Set up alerting** for non-yielding scheduler events going forward (e.g., a SQL Agent alert on error severity, or a monitoring tool watching the `ERRORLOG`) so a recurrence is caught within minutes instead of discovered 4+ hours later.

## Appendix: Key Log Evidence

```
2026-07-23 09:26:25.92 Server   This instance of SQL Server last reported using a process ID
                                  of 5380 at 22/7/2026 9:53:01 PM (local). This is an
                                  informational message only; no user action is required.

2026-07-23 09:26:26.53 spid10s  Starting up database 'master'.
2026-07-23 09:26:26.86 spid10s  23 transactions rolled forward in database 'master' (1:0).
2026-07-23 09:26:26.98 spid10s  0 transactions rolled back in database 'master' (1:0).

2026-07-23 09:33:06.xx Server   * Non-yielding Scheduler
                                 * BEGIN STACK DUMP: 07/23/26 09:33:06 ...
                                 Process Utilization 19%. System Idle 74%.

2026-07-23 10:51:45.xx Server   * Non-yielding Scheduler (Scheduler 2)
                                 Process Utilization ~24%. System Idle ~66%.

2026-07-23 14:07:22.xx Server   Process 0:0:0 (0x...) Worker 0x... appears to be
                                 non-yielding on Scheduler 0/2.
                                 Process Utilization 33-39%. System Idle 51-58%.
```

*(Timestamps and utilization figures condensed from the full log; PID, build number, and database names are exact.)*
