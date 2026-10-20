# IT Support Lab

Hands-on IT support practice, with each fault written up in ticket format: symptom, checks, root cause, fix and lesson.

## Labs

| Lab | Environment | What it covers |
|---|---|---|
| [Microsoft Defender Antivirus checks](defender-windows11/) | Windows 11, PowerShell, Event Viewer | Case 8: tracing antivirus detections (EICAR test file) through PowerShell and the event log, including a permissions issue and a timing gap between detection and removal. Case 9: confirming that Tamper Protection blocked an attempt to disable real-time protection. |

Cases 1 to 7 (Linux and Windows Server troubleshooting, monitoring and CI/CD on AWS) are in my [AWS Operations Lab](https://github.com/Yuka-30v/aws-ops-lab).

## How I work through a fault

1. Confirm the symptom and what the user expected to happen.
2. Check from the bottom up: network, then service, then configuration.
3. Confirm every finding through a second source, such as the event log.
4. Fix, test, and record what happened so the next person does not start from zero.
