

## 1. systemd, systemctl and service processes

- **systemd:** System and service manager. The system-level manager runs as PID 1.
- **systemctl:** Command-line tool used to communicate with systemd and manage units.
- **Service process:** The actual program started by systemd.

In our lab, the service process is `/usr/bin/sleep`.

Use:
- `sudo systemctl` for system-level services.
- `systemctl --user` for services managed by the current user's systemd manager.

---

## 2. Our permanent service lab

We created this system-level unit file:

`/etc/systemd/system/devops-lab-permanent.service`

```ini
[Unit]
Description=DevOps Permanent Service Learning Lab
After=network.target

[Service]
Type=simple
User=athirani
ExecStart=/usr/bin/sleep infinity

[Install]
WantedBy=multi-user.target
```

### Understanding the directives

- `Description=` provides a human-readable description.
- `After=network.target` defines startup ordering. It does not start the other unit or guarantee that networking is fully ready.
- `Type=simple` tells systemd to consider the service started when its process is launched.
- `User=athirani` runs the service process as this user.
- `ExecStart=` specifies the command systemd runs.
- `WantedBy=multi-user.target` defines the target relationship used when enabling the service.

### Verify the unit file

```bash
systemd-analyze verify /etc/systemd/system/devops-lab-permanent.service
```

No output was produced, indicating no reported unit-file problems.

After creating or changing a unit file, reload systemd:

```bash
sudo systemctl daemon-reload
```

This makes systemd reread unit configuration. It does not automatically restart the service.

---

## 3. Runtime state versus boot enablement

These are two independent properties.

- **Active / inactive:** Is the service running right now?
- **Enabled / disabled:** Is the service configured for automatic startup through its boot target relationships?

| Runtime state | Boot enablement | Meaning |
|---|---|---|
| Active | Enabled | Running now and configured for automatic startup. |
| Active | Disabled | Running now but not configured for automatic startup. |
| Inactive | Enabled | Not running now but configured for automatic startup. |
| Inactive | Disabled | Not running and not configured for automatic startup. |

### Important systemctl commands

```bash
sudo systemctl start SERVICE
sudo systemctl stop SERVICE
sudo systemctl restart SERVICE
sudo systemctl enable SERVICE
sudo systemctl disable SERVICE
sudo systemctl enable --now SERVICE
```

- `start` starts the service now.
- `stop` stops the service now.
- `restart` stops and starts the service.
- `enable` configures automatic startup.
- `disable` removes automatic startup configuration.
- `enable --now` enables automatic startup and starts the service immediately.

**Important:** Disabling a service does not stop a service that is already running. Starting a disabled service manually is allowed.

### Check service state

```bash
systemctl is-active SERVICE
systemctl is-enabled SERVICE
sudo systemctl status SERVICE
```

- `is-active` checks current runtime activity.
- `is-enabled` checks boot enablement.
- `status` displays both, along with process information and recent logs.

---

## 4. What we observed in our lab

We enabled the service and later disabled it.

Disabling removed the enablement symlink, but the service continued running. We subsequently stopped it and manually started it again while it remained disabled.

Our latest status showed:

```text
Loaded: loaded (/etc/systemd/system/devops-lab-permanent.service; disabled; preset: enabled)
Active: active (running)
Main PID: 8719 (sleep)
└─8719 /usr/bin/sleep infinity
```

Our observed state:

- **Active:** Yes.
- **Enabled:** No.
- **Main process:** PID 8719.
- **Command:** `/usr/bin/sleep infinity`.

The `preset: enabled` value represents the distribution's preset policy. It does not mean our service is currently enabled. Its actual enablement state is `disabled`.

We intentionally left this service active but disabled for the learning exercise.

Since `sleep infinity` is only a placeholder process, it does not need to start automatically at every boot.

---

## 5. Understanding dependencies and startup ordering

We inspected the service properties:

```bash
systemctl show devops-lab-permanent.service -p After -p Wants -p Requires
```

Output included:

```text
Requires=sysinit.target system.slice
Wants=
After=basic.target sysinit.target system.slice network.target systemd-journald.socket
```

Systemd can add implicit or default dependencies in addition to the directives written in a unit file.

- `Requires=` expresses a stronger dependency. The required unit is pulled into the start transaction; failure or stopping behaviour can affect the dependent unit.
- `Wants=` expresses a weaker dependency. Systemd tries to start the wanted unit, but its failure does not normally prevent the requesting unit from starting.
- `After=` and `Before=` define ordering only. They do not start another unit by themselves.

**Remember:** Dependencies and ordering are separate concepts.

A unit can require another unit and still need an ordering directive to specify which starts first.

---

## 6. Reading service logs with journalctl

The systemd journal records service events and application messages.

We inspected our permanent service's logs:

```bash
sudo journalctl -u devops-lab-permanent.service -n 10 --no-pager
```

Observed output:

```text
Nov 06 10:41:02 ... Started devops-lab-permanent.service ...
Nov 06 12:18:18 ... Stopping devops-lab-permanent.service ...
Nov 06 12:18:18 ... Deactivated successfully.
Nov 06 12:18:18 ... Stopped devops-lab-permanent.service ...
-- Boot fe3e9da9ec10453dbc9c7917e6f00ae5 --
Nov 06 13:43:22 ... Started devops-lab-permanent.service ...
```

### Interpretation

1. The service started at 10:41.
2. It was stopped successfully at 12:18.
3. The boot marker separates entries from different boot sessions.
4. It started again at 13:43. This was consistent with our manual `systemctl start` command.

The log entry alone does not prove automatic startup.

Our service runs `sleep infinity`, which does not print regular application messages. Therefore, seeing only systemd lifecycle messages is normal.

### Useful journalctl options

| Option | Purpose |
|---|---|
| `-u SERVICE` | Filter logs for a particular service. |
| `-p err` | Show error-priority messages and more severe priorities. |
| `-n 20` | Show the latest 20 matching entries. |
| `-b` | Show logs from the current boot. |
| `-b -1` | Show logs from the previous boot, if retained. |
| `-f` | Follow new journal entries. |
| `--no-pager` | Print output directly in the terminal. |
| `-o short-iso` | Use ISO-style timestamps. |

### Examples

Latest 20 entries:

```bash
sudo journalctl -u nginx.service -n 20 --no-pager
```

Error-priority entries:

```bash
sudo journalctl -u nginx.service -p err --no-pager
```

Previous boot's logs:

```bash
sudo journalctl -u nginx.service -b -1 --no-pager
```

Previous boot's error logs:

```bash
sudo journalctl -u nginx.service -b -1 -p err -n 20 --no-pager
```

Follow logs in real time:

```bash
sudo journalctl -u nginx.service -f
```

### Our error-filtering exercise

We ran:

```bash
sudo journalctl -u devops-lab-permanent.service -p err --no-pager
```

Result:

```text
-- No entries --
```

This means there were no matching error-priority entries for this unit in the available journal data.

It does not mean the journal has no entries. Our unfiltered query showed normal start and stop events.

---

## 7. Unexpected reboot: what should an engineer check?

Suppose a production Nginx service is active but disabled.

It is running now, but it is not configured for automatic startup through its boot enablement mechanism.

After a reboot, it may remain stopped unless another dependency, trigger or administrator starts it.

### Enable automatic startup

```bash
sudo systemctl enable nginx.service
```

### Enable and start immediately

```bash
sudo systemctl enable --now nginx.service
```

### Investigate previous-boot logs

```bash
sudo journalctl -u nginx.service -b -1 --no-pager
```

To narrow the investigation to errors:

```bash
sudo journalctl -u nginx.service -b -1 -p err -n 20 --no-pager
```

An abrupt power loss or hard reset may not allow the service to log graceful `Stopping` or `Stopped` messages.

Previous-boot logs can help establish what happened before the interruption, provided the journal retained those records.

A missing stop message alone does not prove the cause of the reboot.

---

## 8. Interview-ready explanations

### Q1. A service is active but disabled. Is that a problem?

Not necessarily.

`active` means the service is running now. `disabled` means it is not configured for automatic startup through the systemd boot enablement mechanism.

If the service must return automatically after a reboot, I would use:

```bash
sudo systemctl enable SERVICE
```

If it must also start immediately:

```bash
sudo systemctl enable --now SERVICE
```

### Q2. How would you investigate a service that did not return after reboot?

I would:

1. Check its current status using `systemctl status SERVICE`.
2. Inspect its previous-boot logs using `journalctl -u SERVICE -b -1`.
3. Narrow the investigation using `-p err` and `-n` if required.
4. Check automatic startup configuration using `systemctl is-enabled SERVICE`.
5. Check current activity using `systemctl is-active SERVICE`.

### Q3. What is the difference between After= and Requires=?

`After=` defines startup ordering. It does not pull in the other unit.

`Requires=` expresses a dependency and pulls the required unit into the start transaction.

They solve different problems and may be used together.

---


**Lab state at the end of this session:** The permanent service was active but disabled. It should be stopped when the lab is no longer needed. Do not enable this placeholder `sleep infinity` service for boot.

---

## 10. Quick revision

1. Does `systemctl disable` stop a running service?  
   **No.**

2. Does `systemctl start` enable a service at boot?  
   **No.**

3. Which command checks current activity?  
   **`systemctl is-active SERVICE`**

4. Which command checks boot enablement?  
   **`systemctl is-enabled SERVICE`**

5. Which command shows previous-boot service logs?  
   **`journalctl -u SERVICE -b -1`**

6. Does `After=` start the named unit?  
   **No. It only specifies ordering.**
