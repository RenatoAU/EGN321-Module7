# Module 7 — Unattended Sensor Monitor
Renato Jacinto · EGN321 · Assignment 7.1

This extends the existing Module 6 history builder. It adds a Flask dashboard, persistent replay position and alert state, stale-data checks, structured logs, and a process supervisor. The supplied instructor repository was reviewed as a reference: https://github.com/ivansuarez-AU/EGN321_Module7_Assignment7_1 . Its example evidence was not reused.

## Start on Windows / VS Code
Extract the ZIP, open this folder in VS Code, and run:

```powershell
py -m pip install -r requirements.txt
py -m pytest -q
py supervisor.py
```

Open http://127.0.0.1:5000 . Use Ctrl+C in the terminal to stop the supervisor and application. Run only one supervisor for each runtime folder. On Linux/macOS use `python3` instead of `py`.

The worker and dashboard run as one supervised process; you do not launch separate Python files. The SQLite database is generated on first startup from the unchanged CSV. This path does not require matplotlib, pandas, or the Module 6 plot command.

## Input source, history, and timing
Source: `data/module6_sample_sensor_history.csv`, the five-channel classroom CSV used in Module 6, not the earlier 103-reading Arduino dataset. Its 1,756 rows become 8,780 archive records using the original `history.py` importer. Original timestamps (timezone unspecified), raw values, accepted values and source/validation statuses are preserved in `readings`. All five channels remain available in SQLite; the web historical table displays temperature.

Original sampling interval: 60 seconds. Runtime replay delivery: 2 seconds by default. This is accelerated classroom replay, not a live physical sensor. Each replay record has its unchanged source timestamp plus a separate delivery timestamp in UTC. Delivery time indicates replay activity and must not be interpreted as a new field measurement. Historical gaps remain absent; no readings are filled. The archive has 45 missing minute slots per channel. Replay advances across archive gaps without pretending those slots were measured.

Replay is finite: at the end of 1,756 rows it stops delivering and correctly becomes stale/failed. It does not silently loop. The cursor is stored transactionally with each replay record. Restart resumes at the next row without deleting history. To intentionally begin another replay, choose a new runtime folder, for example `py supervisor.py --runtime another_run`.

## Alerts and validation
Classroom temperature acceptance: finite 0–50°C and source status OK; out-of-range or source-error readings are rejected. Empty/source-MISSING inputs remain MISSING. Malformed/nonfinite values are INVALID. All unusable accepted values are SQL NULL.

Classroom high-temperature warning: three consecutive accepted delivery readings ≥30°C activate ALERT. An intervening accepted low value or unusable reading resets the activation count. A delivery interruption exceeding the stale threshold also resets that count. An active alert stays active until an accepted reading ≤28.5°C clears it (hysteresis). Rejected, missing, or stale input never clears an existing alert. This is a classroom warning, not a calibrated safety limit. No email/SMS is sent.

## Stale data and system health
Stale means no replay delivery for **more than 3 delivery intervals** (default >6 seconds). The latest timestamp remains visible, but the current value becomes Unavailable. No usable accepted reading for >3 intervals also degrades health, even when invalid input continues arriving.

| State | Meaning |
|---|---|
| RECOVERING | No replay reading yet during startup grace period |
| HEALTHY | Timely accepted replay input and no active threshold alert |
| DEGRADED | Active alert, unusable latest input, stale delivery, or no recent accepted value |
| FAILED | Delivery or accepted-value age exceeds 10 intervals (default >20 seconds) |

A dashboard/database request failure returns HTTP 503 with FAILED. A worker error is logged and exits the process so the supervisor can restart it. During a process outage the dashboard is unavailable; the supervisor log records the exit and restart. Health is recomputed from persisted readings and time after restart.

## Automatic startup and restart
`supervisor.py` starts the combined app and waits for its exit. Any child exit is logged, then the supervisor restarts it after one second. The source cursor, alert state/count, and readings survive child crashes in SQLite. This mechanism works while the supervisor and host remain running; it is not itself an OS boot service.

Windows automatic logon startup: press Win+R, enter `shell:startup`, and place a shortcut to `start_monitor.cmd` there. Configure this yourself on the computer where you run the project; it has not been enabled here. Python must be installed and `py` available. This starts at user logon, not before logon.

Linux optional boot service: copy the project to `/opt/module7`, create `.venv`, install requirements into it, edit the username in `module7-monitor.service.example`, install as `/etc/systemd/system/module7-monitor.service`, then run `sudo systemctl daemon-reload` and `sudo systemctl enable --now module7-monitor`. Check `systemctl status module7-monitor`. This example has not been installed/tested as an OS service. Use either systemd directly or the supervisor, not two services writing the same runtime.

## Logs and recovery evidence
Default log: `runtime/application.jsonl`, rotated at 2 MB with three backups. Events cover STARTUP, READING_PROCESSED, ALERT_ACTIVATED, ALERT_CLEARED, MISSING_OR_STALE_DATA, APPLICATION_ERROR, PROCESS_INTERRUPTED, RESTART_SCHEDULED, PROCESS_STARTED and RECOVERY_READY. A forced kill cannot log from inside the killed child; the surviving supervisor records it.

`evidence/` contains this assistant environment's actual HTTP snapshots, state JSON, application log, tested SQLite database and `verification.json`. These show a **synthetic fault test at 0.5-second delivery intervals on Linux**, not the student's own computer or physical sensor. The test deliberately kills the app child, verifies a changed PID, checks HTTP availability and final HEALTHY state, and verifies SQLite integrity and archive preservation. Saved HTML is dashboard output, not a screenshot. The packaged database is a historical test snapshot; start normally to create your own runtime database.

Repeat the test without browser screenshots:

```powershell
py capture_evidence.py --no-browser
```

`test_runtime` must not exist before a fresh run; rename an old test directory if needed. `capture_evidence.py` uses port 5057 and does not touch the normal runtime. It enables local-file-only synthetic controls under `--demo`; those controls are ignored during normal replay. The synthetic controls are test fixtures and do not modify the original CSV or archive.

To generate actual dashboard PNG screenshots automatically on your computer:

```powershell
py -m pip install playwright
py -m playwright install chromium
py capture_evidence.py
```

The script saves healthy dashboard, original history, alert active, alert cleared, stale, failed, explicit missing, and recovered healthy screenshots alongside logs and output. Rename `test_runtime` before rerunning. Review the screenshots before submission. Browser installation was blocked in the assistant environment, so PNG screenshots are still required. You may also take screenshots manually while a demo runs. The instructor should approve the supervised virtual process-recovery approach if required by course policy.

## Limitations and human intervention
Automatic recovery covers app process crashes while the supervisor survives, resumption from committed SQLite records, and health recovery when accepted input resumes. It does not repair a corrupted database, unavailable disk, invalid permissions, host/power failure, a killed supervisor, or a persistently broken source/configuration. Repeated crashes restart indefinitely; inspect logs and correct the cause. Disk exhaustion/permissions can prevent logging itself. Data gaps cannot be recovered without another authoritative source. Tests do not establish OS boot behavior. The system binds to localhost and is a classroom application without authentication or remote monitoring.
