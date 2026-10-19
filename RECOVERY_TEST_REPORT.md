# Recovery Test Report — Module 7

This report summarizes the recorded assistant-environment test, not a claim that the student personally performed it. Environment: Linux; synthetic fault inputs, delivered every 0.5 seconds. The Module 6 archive was imported unchanged from the saved five-sensor CSV.

## Method
The application was launched through its supervisor. The test accessed the running Flask server over HTTP, raised a persistent high-temperature alert, cleared it with a lower accepted temperature, paused input to produce STALE and FAILED states, and supplied an explicit MISSING record. It then resumed valid input, force-killed the application child, and waited for the supervisor to launch a new child. Finally it checked HTTP availability and SQLite integrity.

## Recorded results

| Check | Recorded result |
|---|---|
| Initial healthy dashboard | Health HEALTHY; data ACCEPTED; alert NORMAL |
| Threshold alert active | Health DEGRADED; data ACCEPTED; alert ALERT |
| Alert cleared | Health HEALTHY; data ACCEPTED; alert NORMAL |
| Stale delivery | Health DEGRADED; data STALE; alert NORMAL |
| Extended interruption | Health FAILED; data STALE; alert NORMAL |
| Explicit missing input | Health FAILED; data MISSING; alert NORMAL |
| After automatic restart | Health HEALTHY; data ACCEPTED; alert NORMAL |
| Child PID | 6 → 12 |
| Replay records before/after crash | 7 → 8 |
| Original archive preserved | 8,780 records |
| SQLite integrity_check | ok |

## Evidence files
`verification.json`, per-state JSON, HTML responses, `application.jsonl`, and `verified_monitor.db` in this directory contain the actual recorded output. No browser screenshots were captured in the assistant environment. HTML responses are not screenshots.

## Interpretation and remaining work
The test supports recovery from an application child crash while the supervisor remains running. It does not prove OS boot startup, host-failure recovery, sensor calibration, or operation on the student's computer. The student must run/review the project, create actual dashboard screenshots, configure the startup method used, and provide the real GitHub repository link. `GENERAR_EVIDENCIAS.cmd` repeats the test with screenshots when browser installation is available.
