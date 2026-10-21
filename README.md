# Linux Systemd Service Management, Monitoring & Troubleshooting

## Project Overview

This hands-on Linux administration lab demonstrates how to create a custom background service, manage it with **systemd**, inspect its logs with **journalctl**, simulate a service failure, and restore service operation manually.

The lab was completed on an **offline Ubuntu virtual machine hosted on Proxmox VE**, using built-in Linux tools and no additional software downloads.

> **Actual test result:** The service did **not** restart automatically after the simulated failure. I restarted it **manually**, then confirmed that it was running normally and producing heartbeat logs. Automatic recovery was configured but was **not verified as working** in this test.

## Lab Objectives

- Write a simple Bash script that generates periodic heartbeat messages.
- Configure the script as a systemd service.
- Enable and start the service.
- Check its status and monitor logs.
- Simulate an unexpected service termination.
- Troubleshoot the stopped service and perform a **manual restart**.
- Verify that the service is healthy after recovery.

## Lab Environment

| Component | Details |
| --- | --- |
| Hypervisor | Proxmox VE |
| Virtual machine | Ubuntu Linux |
| Network | Offline / isolated lab environment |
| Service manager | systemd |
| Log inspection | journalctl |
| Scripting | Bash |
| Custom service | `zen-heartbeat.service` |

## How the Lab Works

```text
Ubuntu VM (Proxmox VE)
        |
        v
systemd: zen-heartbeat.service
        |
        v
/usr/local/bin/zen-heartbeat.sh
        |
        +--> Prints a timestamped heartbeat every 10 seconds
        |
        +--> Output captured in the systemd journal
                      |
                      v
                journalctl
```

## Implementation

### 1. Create the heartbeat script

Created the script:

```bash
sudo nano /usr/local/bin/zen-heartbeat.sh
```

Script contents:

```bash
#!/bin/bash

while true; do
  echo "Zen Heartbeat OK at $(date)"
  sleep 10
done
```

Made the script executable:

```bash
sudo chmod +x /usr/local/bin/zen-heartbeat.sh
```

**Purpose:** The script simulates a long-running background application and produces a new log message approximately every 10 seconds.

### 2. Create the systemd unit

Created the service definition:

```bash
sudo nano /etc/systemd/system/zen-heartbeat.service
```

Unit contents:

```ini
[Unit]
Description=Zen Linux Heartbeat Service

[Service]
Type=simple
ExecStart=/usr/local/bin/zen-heartbeat.sh
User=nobody
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

**Configuration notes:**

| Directive | Purpose |
| --- | --- |
| `Type=simple` | systemd treats the launched process as the main service process. |
| `ExecStart=` | Specifies the script to execute. |
| `User=nobody` | Runs the script as an unprivileged user. |
| `Restart=on-failure` | Configures systemd to attempt recovery when the service fails under qualifying conditions. |
| `RestartSec=3` | Configures a three-second delay before a restart attempt. |
| `WantedBy=multi-user.target` | Supports enabling the service at boot. |

Although an automatic-restart policy was configured, **the observed service did not restart automatically during this test**. This project documents the actual result rather than claiming automatic recovery succeeded.

### 3. Load, enable, and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now zen-heartbeat.service
```

Checked its status:

```bash
systemctl status zen-heartbeat.service
```

**Observed result:** The service started successfully and reported `active (running)`.

### 4. Monitor heartbeat logs

Followed logs in real time:

```bash
sudo journalctl -u zen-heartbeat.service -f
```

The journal showed repeating heartbeat messages with timestamps. Pressed `Ctrl+C` to exit the log view.

To inspect recent entries without continuously following them:

```bash
sudo journalctl -u zen-heartbeat.service -n 30 --no-pager
```

**Why this matters:** Administrators use service logs to understand process activity, investigate failures, and verify recovery.

### 5. Simulate a service failure

Sent a termination signal to the main service process:

```bash
sudo systemctl kill -s SIGKILL --kill-whom=main zen-heartbeat.service
```

Checked service status and recent logs:

```bash
systemctl status zen-heartbeat.service
sudo journalctl -u zen-heartbeat.service -n 30 --no-pager
```

**Observed result:** The service did **not** restart automatically as expected. The underlying reason was not established during this lab.

### 6. Recover the service manually

Restarted the service manually:

```bash
sudo systemctl start zen-heartbeat.service
```

Validated its status:

```bash
systemctl status zen-heartbeat.service
systemctl is-active zen-heartbeat.service
```

Rechecked the latest logs:

```bash
sudo journalctl -u zen-heartbeat.service -n 30 --no-pager
```

**Observed result:** After the manual restart, the service was `active (running)` and heartbeat messages were being produced normally.

## Results

| Test | Outcome |
| --- | --- |
| Create and execute Bash script | Successful |
| Configure custom systemd unit | Successful |
| Enable and start service | Successful |
| View status using `systemctl` | Successful |
| Read logs using `journalctl` | Successful |
| Simulate service failure | Performed |
| Automatic restart after failure | **Not observed** |
| Manually restart service | **Successful** |
| Verify service status and logs after recovery | **Successful** |

## Troubleshooting Notes

The unit included `Restart=on-failure`, but the service did not automatically recover after the failure simulation. Rather than assuming a cause, I performed a manual recovery and verified normal operation.

A possible follow-up investigation would be to inspect the full journal and service properties:

```bash
sudo journalctl -u zen-heartbeat.service -b --no-pager
systemctl show zen-heartbeat.service -p Restart -p NRestarts -p Result
```

These commands are provided as **future troubleshooting steps**; this README does not claim that they identified the reason for the missing automatic restart.

## Key Skills Practiced

- Linux service administration with `systemctl`
- Creating and configuring systemd unit files
- Managing script permissions using `chmod`
- Monitoring Linux service logs with `journalctl`
- Simulating and responding to service interruptions
- Validating service recovery after manual remediation
- Working in an offline Ubuntu VM

## Evidence

Screenshots and a short demonstration video are uploaded separately in this repository. They show the service configuration, monitoring, failure simulation, and manually restored service. They are intentionally not embedded in this README.

## Conclusion

This lab provided practical experience with the Linux service lifecycle: **configuration, startup, log monitoring, failure testing, manual recovery, and post-recovery verification**. The service ran normally after manual intervention. The unexpected absence of automatic recovery is recorded as an open troubleshooting observation, not a successful auto-restart test.
