## Learning Objectives

- Understand how to investigate an unavailable application.
- Check service status after a server reboot.
- Investigate service failures using journalctl.
- Understand application and database dependencies.
- Understand database connectivity and network ports.
- Inspect Linux file permissions and ownership.
- Apply the Principle of Least Privilege.

---

## 1. Production Application Troubleshooting

### Scenario

A production application becomes unavailable after a server reboot.

As a DevOps engineer, the first step is to check whether the application service is running.

### Check service status

```bash
sudo systemctl status SERVICE_NAME
```

Example:

```bash
sudo systemctl status nginx.service
```

Important fields:

- `Loaded`: Shows whether the service unit is loaded and its enablement state.
- `Active`: Shows whether the service is currently running, inactive or failed.
- `Main PID`: Identifies the service's main process.
- Recent log messages: Provide initial information about service activity or failure.

### Active vs Enabled

These are two different properties.

| State | Meaning |
|---|---|
| Active | Service is currently running. |
| Inactive | Service is not currently running. |
| Enabled | Configured for automatic startup through systemd boot enablement. |
| Disabled | Not configured for automatic startup through systemd boot enablement. |

A service can be active but disabled.

If a service must start automatically after future reboots:

```bash
sudo systemctl enable SERVICE_NAME
```

To enable it and start it immediately:

```bash
sudo systemctl enable --now SERVICE_NAME
```

---

## 2. Investigating Service Failures Using journalctl

When a service fails, checking its logs helps identify the reason.

### View recent logs

```bash
sudo journalctl -u SERVICE_NAME -b -n 20 --no-pager
```

Command options:

| Option | Purpose |
|---|---|
| `-u` | Filter logs by service unit. |
| `-b` | Show logs from the current boot. |
| `-n 20` | Show the latest 20 matching entries. |
| `-p err` | Filter for error-priority messages and more severe priorities. |
| `-b -1` | Show logs from the previous boot, if retained. |
| `--no-pager` | Display output directly. |

### Example service failure

```text
app.service: Failed to start Application Service.
app: Error: Unable to connect to database.
app.service: Main process exited, status=1/FAILURE
```

Interpretation:

- The application could not connect to its database.
- The application process exited with failure code 1.
- The database connection problem is a likely root cause to investigate.

**Important engineering principle:**

Do not restart a failed application repeatedly without investigating the underlying problem.

Investigate the dependency, fix the cause, restart the application and verify its health.

---

## 3. Understanding Application and Database Dependencies

Applications often depend on databases to retrieve and store information.

Example:

Application → Database → Stored data

A database management system provides access to the stored data.

If the database service is inactive:

- The data does not normally disappear simply because the service stops.
- The application generally cannot retrieve data through that database service.
- The database service must be available and accepting connections before the application can use it normally.

### Important terminology

- SQL: A language used to query and manage relational databases.
- PostgreSQL: A relational database management system.
- PL/SQL: Oracle's procedural extension to SQL.

These are different things.

For an entry-level DevOps role, understand database dependencies and basic connectivity. Advanced database administration and PL/SQL programming are not current priorities.

### Checking PostgreSQL service status

```bash
sudo systemctl status postgresql
```

We tried this command on our Ubuntu machine.

Observed output:

```text
Unit postgresql.service could not be found.
```

Reason: PostgreSQL has not been installed or configured as a host-level service on our machine.

We did not install PostgreSQL because this was a hypothetical troubleshooting exercise.

---

## 4. Database Connectivity and Ports

A database service can be running while an application still cannot connect to it.

Possible causes include:

1. Incorrect database host.
2. Incorrect port configuration.
3. Network connectivity problems.
4. Firewall restrictions.
5. Incorrect username or password.
6. Application configuration problems.

### PostgreSQL default port

PostgreSQL commonly uses TCP port:

```text
5432
```

Example:

- PostgreSQL listens on port 5432.
- Application tries connecting to port 5433.
- The application may fail to establish a connection.

This is a connectivity or configuration issue, not necessarily a database service failure.

### Inspect listening TCP ports

A useful Linux command is:

```bash
sudo ss -ltnp
```

It displays listening TCP sockets and associated processes.

We discussed this command conceptually; we have not executed this practical check yet.

**Important:** An active database service does not automatically guarantee that application connectivity is working.

---

## 5. Linux File Permissions

Linux permissions control who can access files and directories and what operations they can perform.

Three permission categories:

1. Owner
2. Group
3. Others

Three basic permission types:

| Symbol | Permission | Numeric value |
|---|---|---:|
| r | Read | 4 |
| w | Write | 2 |
| x | Execute | 1 |
| - | No permission | 0 |

### Inspect file permissions

```bash
ls -l FILE_NAME
```

Example output:

```text
-rw-r----- 1 root devops 1024 Nov 7 10:00 config.yaml
```

Interpretation:

| Category | Permission |
|---|---|
| Owner (root) | Read and write |
| Group (devops) | Read only |
| Others | No access |

The first character indicates the file type. The remaining nine characters describe the permissions for owner, group and others.

### Important commands

```bash
ls -l config.yaml
```

Displays current file permissions and ownership.

```bash
chmod 400 config.yaml
```

Changes permissions so only the owner has read access.

```bash
chmod 444 config.yaml
```

Gives read-only access to owner, group and others.

```bash
chown USER_NAME config.yaml
```

Changes file ownership.

Remember:

- `ls -l` inspects permissions.
- `chmod` changes permissions.
- `chown` changes ownership.

### Numeric permissions

| Permission | Meaning |
|---|---|
| 400 | Owner read only |
| 444 | Everyone read only |
| 600 | Owner read and write |
| 644 | Owner read/write; group and others read |
| 755 | Owner full permissions; group and others read/execute |
| 777 | Everyone read, write and execute |

### Principle of Least Privilege

Give users and applications only the permissions they actually need.

Example:

If an application only needs to read a configuration file, it does not need write or execute permissions on that file.

Avoid blindly using:

```bash
chmod 777 config.yaml
```

It grants unnecessary permissions and can introduce security risks.

### Important ownership consideration

Suppose a file belongs to `root`, but an application runs as `athirani`.

Even if the file is configured with:

```bash
chmod 400 config.yaml
```

The application cannot read it unless it has access through the applicable ownership, group or other access mechanism.

Permissions must be selected based on the actual application user.

---

## 6. Interview Troubleshooting Workflow

### Scenario

An application is unavailable after a server reboot.

A systematic investigation:

1. Check service status.

```bash
sudo systemctl status SERVICE_NAME
```

2. Check whether it is configured for automatic startup.

```bash
systemctl is-enabled SERVICE_NAME
```

3. Inspect current-boot service logs.

```bash
sudo journalctl -u SERVICE_NAME -b -n 20 --no-pager
```

4. Check error-priority logs if needed.

```bash
sudo journalctl -u SERVICE_NAME -b -p err --no-pager
```

5. Identify failed dependencies such as a database or network service.

6. Investigate configuration, permissions, ports and connectivity as indicated by the logs.

7. Fix the underlying issue.

8. Restart the service if appropriate.

9. Verify service status and application availability.

### Interview-ready explanation

"If an application becomes unavailable after a reboot, I would first check its systemd service status. Then I would inspect the journal logs to identify the failure reason. I would investigate dependencies such as databases, networking, configuration and file permissions. After resolving the root cause, I would restart the service and verify that it is functioning correctly. I would also check whether the service is enabled for automatic startup."

---

## 7. Practical Learning Progress

### Concepts covered

- Application service troubleshooting.
- Active versus enabled service states.
- Automatic startup after reboot.
- Journal log investigation.
- Application and database dependencies.
- SQL vs PostgreSQL vs PL/SQL.
- Database ports and connection failures.
- Linux file permissions and ownership.
- chmod and chown.
- Principle of Least Privilege.

### Practical commands discussed

| Command | Purpose |
|---|---|
| `systemctl status` | Inspect service status. |
| `systemctl is-enabled` | Check boot enablement. |
| `journalctl -u` | Inspect service logs. |
| `journalctl -p err` | Filter error-priority messages. |
| `ss -ltnp` | Inspect listening TCP ports. |
| `ls -l` | Inspect permissions and ownership. |
| `chmod` | Change permissions. |
| `chown` | Change ownership. |

### Actual practical work versus concepts

- We inspected systemd service status and journal logs in our earlier permanent-service lab.
- We attempted to inspect PostgreSQL and confirmed that its service unit was unavailable on this machine.
- Database connectivity, port inspection and file-permission changes were discussed as concepts and commands. We did not configure or modify a real database or application.
- No real production failure was simulated.

---

## 8. Topics to Study Later

- Advanced Linux ACLs.
- Advanced database administration.
- PL/SQL programming.
- Advanced network troubleshooting.
- Kernel-level debugging.
- Advanced systemd unit configuration.

---


