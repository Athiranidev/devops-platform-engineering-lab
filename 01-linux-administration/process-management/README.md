# Linux Process and Service Management

## Learning Objective

Understand how Linux creates, manages, monitors and supervises processes and background services. Practise inspecting processes and investigating service behaviour using real system output.

## 1. Processes and Services

A **process** is a running instance of a program. Every process has a Process ID (PID).

A **service** is a background system component managed by a service manager such as systemd.

Examples:
- Docker daemon (`dockerd`)
- Grafana
- Prometheus
- NetworkManager

## 2. PID and PPID

- PID: Unique identifier assigned to a running process.
- PPID: Process ID of the parent process.

A process can start other processes, creating a parent-child relationship.

### Commands

```bash
ps -C dockerd -o pid,ppid,stat,etime,%cpu,%mem,cmd
ps -p 1 -o pid,ppid,stat,comm,args
```

### Observations

- Docker daemon was initially observed with PID 3683.
- Later, Docker was observed with PID 5273.
- The original PID was no longer present.
- PID 1 was identified as systemd.

**Learning:** PIDs identify process instances, not applications permanently. Always verify the current PID before investigating a process.

## 3. systemd and Service Management

systemd is the system and service manager used by Ubuntu.

### Command

```bash
systemctl status docker
```

Important output fields:

- Loaded: Service unit configuration status.
- Enabled: Whether the service is configured to start automatically at boot.
- Active: Whether the service is currently running.
- Main PID: Main process associated with the service.
- Tasks: Number of tasks associated with the service.
- Memory: Memory usage reported for the service.

### Practical observation

Docker was reported as active (running), with `dockerd` as its main process.

## 4. Process Hierarchy

Linux processes form parent-child relationships.

### Command

```bash
pstree -ps <PID>
```

### Practical observation

The Docker process tree showed:

- systemd (PID 1) as the ancestor.
- dockerd as the Docker daemon.
- docker-proxy processes handling published container ports.
- Multiple dockerd threads.

**Learning:** A process can create child processes and threads to perform different tasks.

## 5. Process States

| State | Meaning |
|---|---|
| R | Running or runnable |
| S | Interruptible sleep |
| D | Uninterruptible sleep |
| T | Stopped or traced |
| Z | Zombie |
| I | Idle kernel thread |

Additional process status indicators can include:

- s: Session leader
- l: Multithreaded process
- +: Foreground process group

A sleeping process is not necessarily unhealthy. Background services commonly sleep while waiting for events or requests.

## 6. Process Monitoring

### Command: Process snapshot

```bash
ps -eo pid,ppid,stat,%cpu,%mem,comm --sort=-%cpu | head -n 11
```

This displays the ten highest CPU-consuming processes in a snapshot.

### Command: Real-time monitoring

```bash
top
```

Important metrics:

- CPU usage and idle percentage.
- Load average.
- Running, sleeping, stopped and zombie tasks.
- Total, used and available memory.
- Swap usage.
- Individual process resource consumption.

### Practical observations

The system reported:

- 395 total tasks.
- 1 running task.
- 394 sleeping tasks.
- 0 stopped tasks.
- 0 zombie tasks.
- 98% CPU idle.
- Approximately 10.5 GiB available memory.
- 0 GiB swap used.

**Learning:** `ps` provides a process snapshot, while `top` continuously refreshes system and process information.

## 7. Docker Service and Container Inspection

### Commands

```bash
systemctl status docker
docker ps
docker ps -a
```

### Practical observations

The Docker service was running.

The monitoring containers included:

- Grafana
- Prometheus
- Node Exporter

An Alpine container named `restart-always` repeatedly exited with code 1 and restarted.

**Important:** This is an intentional restart-policy learning experiment created earlier, not an unexpected failure. Preserve it for a dedicated Docker troubleshooting exercise.

## 8. Engineering Principles Learned

1. A running service does not guarantee that every application it manages is healthy.
2. Always verify a process's current PID.
3. Understand process state before declaring a process unhealthy.
4. Investigate resource utilisation before taking corrective action.
5. Inspect logs and collect evidence before attempting a fix.
6. Distinguish intentional experimental behaviour from unexpected failures.

## 9. Topics Still to Complete

- Linux signals: SIGTERM, SIGKILL and SIGHUP.
- Safe process termination.
- systemd service start, stop, restart and enable operations.
- Journal logs and service troubleshooting.
- Practical troubleshooting exercise.
- Interview questions.

## 10. Learning Status

This document records hands-on observations and concepts studied on a local Ubuntu environment.

These exercises represent practical learning and do not claim production engineering experience.
