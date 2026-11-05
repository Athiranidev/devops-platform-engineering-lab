# Day 3 — systemd Service Management: Transient and Permanent Services

## 1. Learning Objectives

Understand how Linux manages background services using systemd.

Topics covered:

- What is systemd?
- What is a systemd service?
- Transient vs permanent services.
- Creating transient services using `systemd-run`.
- Inspecting service status.
- Starting, stopping and restarting services.
- Understanding Main PID and process changes.
- Understanding system-level and user-level systemd managers.
- Investigating service failures.

---

## 2. What is systemd?

systemd is the system and service manager used by Ubuntu.

It is responsible for managing system services and their lifecycle.

Important responsibilities include:

- Starting services during system boot.
- Stopping services.
- Restarting services.
- Managing service dependencies.
- Monitoring service states.
- Managing service startup configuration.
- Collecting service logs through the system journal.

### Example

Docker is managed by systemd.

The Docker daemon (`dockerd`) is the main process that performs Docker's container-management operations.

Earlier inspection:

```bash
systemctl status docker
```

Observed:

```text
Active: active (running)
Main PID: 5273 (dockerd)
```

This confirmed that Docker was running as a system service.

---

## 3. Transient Services

A transient service is a temporary systemd unit created dynamically rather than from a permanently installed service unit file.

It can be created using:

```bash
systemd-run
```

### Practical example

```bash
systemd-run --user --unit=devops-lab-sleep-02 /usr/bin/sleep 600
```

Explanation:

- `systemd-run`: Requests systemd to create and start a unit.
- `--user`: Uses the current user's systemd manager.
- `--unit`: Specifies the service unit name.
- `/usr/bin/sleep`: Program to execute.
- `600`: Duration in seconds.

The command created:

```text
devops-lab-sleep-02.service
```

### Inspecting a transient service

```bash
systemctl --user status devops-lab-sleep-02.service
```

Observed:

```text
Loaded: loaded
Transient: yes
Active: active (running)
Main PID: 21437 (sleep)
```

This confirmed that:

- systemd successfully created the unit.
- The unit was transient.
- The service was running.
- The sleep process was its main process.

### Important characteristics

- Created dynamically.
- Does not require a permanent service unit file.
- Managed by systemd while loaded.
- Can be started and stopped through systemd.
- Is not automatically recreated after a reboot.

A stopped transient unit may be unloaded by systemd, so it may no longer be available for another `systemctl start` operation.

---

## 4. Permanent Services

A permanent service is defined through a persistent systemd unit file.

Example system-level location:

```text
/etc/systemd/system/my-python-app.service
```

Example user-level location:

```text
~/.config/systemd/user/my-python-app.service
```

A permanent unit file contains instructions describing how systemd should manage an application.

It can define:

- Application executable.
- Application arguments.
- Working directory.
- User and group.
- Dependencies.
- Restart policy.
- Startup behaviour.

### Important characteristics

- Defined in a persistent unit file.
- Remains available after the service stops.
- Can be started and stopped using systemctl.
- Can be configured to start automatically at boot.
- Supports service dependencies and restart policies.

### Example commands

```bash
sudo systemctl start my-python-app.service
sudo systemctl stop my-python-app.service
sudo systemctl restart my-python-app.service
sudo systemctl enable my-python-app.service
sudo systemctl disable my-python-app.service
```

For a user-level service, use `systemctl --user` instead of `sudo systemctl`.

**Important:** User-level services generally start with the user's systemd session. Starting them at boot independently of login may require enabling lingering.

---

## 5. Transient vs Permanent Services

| Feature | Transient Service | Permanent Service |
|---|---|---|
| Creation | Dynamically using `systemd-run` | Using a unit file |
| Persistent unit file | Not required | Required |
| Unit definition after stopping | May be unloaded | Remains available |
| Start again | Usually recreate the transient unit | Use `systemctl start` |
| Automatic startup at boot | Not normally configured as a persistent service | Supported through `enable` |
| Dependencies | Limited by how the transient unit is configured | Can be defined in the unit file |
| Typical use | Temporary tasks and experiments | Long-running applications |

### Important distinction

Transient and permanent describe how the service definition is created and retained.

They do not necessarily describe how long the application runs.

A transient service can run for a long time, while a permanent service can stop shortly after starting.

---

## 6. Practical Exercise — Creating a Transient Service

### First attempt: Failed service

Command:

```bash
systemd-run --user --unit=devops-lab-sleep /usr/bin/sleep 600
```

The service creation attempt returned:

```text
Failed to start transient service unit:
Unit devops-lab-sleep.service was already loaded
or has a fragment file.
```

We inspected it:

```bash
systemctl --user status devops-lab-sleep.service
```

Observed:

```text
Active: failed (Result: exit-code)
ExecStart=/usr/bin/sleep 600time
```

The service logs showed:

```text
sleep: invalid time interval '600time'
```

### Root cause

The sleep command received an invalid duration argument: `600time`.

The application exited with status 1, and systemd marked the service as failed.

The service unit was already loaded, so attempting to create another transient unit with the same name caused a name conflict.

### Resolution

We used a new service name and corrected the duration argument.

```bash
systemd-run --user --unit=devops-lab-sleep-02 /usr/bin/sleep 600
```

The service started successfully.

**Learning:** Read service failure details before attempting a fix. A failed process and an existing loaded unit are two different things.

---

## 7. Practical Exercise — Checking Service Status

Command:

```bash
systemctl --user status devops-lab-sleep-02.service
```

Important fields:

| Field | Meaning |
|---|---|
| Loaded | Whether systemd has loaded the unit |
| Transient | Whether it is a dynamically created transient unit |
| Active | Current service state |
| Main PID | Main process associated with the service |
| Tasks | Number of tasks associated with the unit |
| Memory | Memory usage reported for the unit |
| CGroup | Control group associated with the service |

The service was confirmed active and running.

---

## 8. Practical Exercise — Starting, Restarting and Stopping

Created another temporary service:

```bash
systemd-run --user --unit=devops-lab-sleep-03 /usr/bin/sleep 600
```

Inspected it:

```bash
systemctl --user status devops-lab-sleep-03.service
```

Observed:

```text
Active: active (running)
Main PID: 25704 (sleep)
Transient: yes
```

### Restarting the service

Command:

```bash
systemctl --user restart devops-lab-sleep-03.service
```

After restarting, inspected its status again.

Observed:

```text
Main PID: 26366 (sleep)
Active: active (running)
```

The journal showed:

```text
Stopped devops-lab-sleep-03.service
Started devops-lab-sleep-03.service
```

### Verifying the PID change

```bash
ps -p 25704,26366 -o pid,ppid,stat,cmd
```

Observed:

```text
PID     PPID    STAT   CMD
26366   3764    Ss     /usr/bin/sleep 600
```

This confirmed:

- The old PID 25704 no longer existed.
- The new process had PID 26366.
- The service was running again.

### Stopping the service

Command:

```bash
systemctl --user stop devops-lab-sleep-03.service
```

We verified the process IDs:

```bash
ps -p 25704,26366 -o pid,ppid,stat,cmd
```

The output showed no process entries.

A subsequent status check returned:

```text
Unit devops-lab-sleep-03.service could not be found.
```

This was expected for our transient unit after it was stopped and unloaded.

---

## 9. system-level vs user-level systemd

There are two important systemd management contexts.

### System-level systemd

Used for managing system services.

Example:

```bash
systemctl status docker
```

The Docker daemon was observed with:

```text
PID: 5273
PPID: 1
```

Its parent was the system-level systemd process.

### User-level systemd

Used for managing services belonging to an individual user.

Example:

```bash
systemctl --user status devops-lab-sleep-03.service
```

Our temporary sleep service was managed through the user-level systemd manager.

Its process was observed with:

```text
PPID: 3764
```

This was the PID of the user-level systemd manager.

### Key difference

| Command | Manager |
|---|---|
| `systemctl status` | System-level manager |
| `systemctl --user status` | Current user's service manager |

Use the appropriate manager when inspecting and controlling a service.

---

## 10. Important systemd Operations

| Command | Purpose |
|---|---|
| `systemctl status service` | Inspect system-level service |
| `systemctl start service` | Start service |
| `systemctl stop service` | Stop service |
| `systemctl restart service` | Restart service |
| `systemctl enable service` | Configure automatic startup at boot |
| `systemctl disable service` | Remove automatic startup configuration |
| `systemctl is-active service` | Check current runtime state |
| `systemctl is-enabled service` | Check boot enablement configuration |

For user-level services, use the corresponding `systemctl --user` commands.

### Active vs Enabled

Active and enabled describe different things.

- Active: Is the service running now?
- Enabled: Is the service configured to start automatically at boot?

A service can be active but disabled, or inactive but enabled.

Example:

```bash
systemctl is-enabled docker
systemctl is-active docker
```

Our Docker service was confirmed as both enabled and active.

---

## 11. Real-world DevOps Applications

Imagine a Python application deployed on an Ubuntu server.

It needs to:

1. Run continuously in the background.
2. Start automatically after reboot.
3. Be managed using systemctl.
4. Produce logs for troubleshooting.
5. Recover from certain unexpected failures.

A permanent systemd service unit is suitable for this requirement.

A transient service is better suited to temporary tasks, experiments and short-lived operations.

### Engineering Principles Learned

1. Understand whether a service is transient or permanent.
2. Check whether a unit already exists before creating another with the same name.
3. Inspect service status before attempting a restart.
4. Read error messages to identify root causes.
5. Distinguish the system-level and user-level service managers.
6. Understand the relationship between service names and process IDs.
7. Verify service state after performing an operation.

---


