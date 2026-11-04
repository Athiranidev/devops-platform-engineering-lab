## 1. Learning Objectives

Understand how Linux communicates with running processes and how users manage foreground and background jobs.

Practical objectives:
- Understand Linux signals.
- Differentiate graceful and forceful process termination.
- Understand the default and application-specific behaviour of SIGHUP.
- Manage foreground and background processes.
- Understand shell job numbers versus Linux PIDs.
- Practise process suspension and resumption.

---

## 2. Linux Signals

A signal is a notification sent to a process to inform it that an event or action has occurred.

Linux uses signals for process control, termination and other communication.

### Important Signals

| Signal | Number | Purpose |
|---|---:|---|
| SIGHUP | 1 | Hangup signal; some applications use it to reload configuration |
| SIGINT | 2 | Interrupts a process, commonly through Ctrl+C |
| SIGKILL | 9 | Forces immediate process termination |
| SIGTERM | 15 | Requests graceful process termination |
| SIGTSTP | 20 | Suspends a foreground process, commonly through Ctrl+Z |
| SIGCONT | 18 | Resumes a stopped process |

### Command to list available signals

```bash
kill -l
```

This displays the signal names and their associated numbers.

---

## 3. SIGTERM — Graceful Termination

SIGTERM requests a process to terminate.

The application can handle this signal and perform cleanup operations before exiting.

Examples of cleanup:
- Finish pending operations.
- Close connections.
- Save necessary data.
- Release resources.

### Practical Exercise

Created a temporary process:

```bash
sleep 600 &
```

Identified the process:

```bash
ps -p 23478 -o pid,ppid,stat,etime,cmd
```

Sent SIGTERM:

```bash
kill -TERM 23478
```

Verified termination:

```bash
ps -p 23478 -o pid,ppid,stat,etime,cmd
```

**Observation:** The process terminated and no longer appeared in the process listing.

Note: A first experiment using `sleep 300` completed naturally before SIGTERM was sent. This demonstrated why process existence should be checked before sending a signal.

---

## 4. SIGKILL — Forceful Termination

SIGKILL forces the kernel to terminate a process.

Important characteristics:
- The process cannot catch SIGKILL.
- The process cannot ignore or block SIGKILL.
- The application cannot perform its own cleanup after receiving it.

### Practical Exercise

Created a temporary process:

```bash
sleep 600 &
```

Inspected the process:

```bash
ps -p 24342 -o pid,ppid,stat,etime,cmd
```

Sent SIGKILL:

```bash
kill -KILL 24342
```

Verified termination:

```bash
ps -p 24342 -o pid,ppid,stat,etime,cmd
```

**Observation:** The process was forcibly terminated.

### Engineering Principle

Prefer SIGTERM when an application should shut down gracefully.

Use SIGKILL only when necessary, because it does not allow application-level cleanup.

---

## 5. SIGHUP — Hangup and Configuration Reload

SIGHUP is traditionally associated with a terminal or controlling-session disconnect.

Some background applications are programmed to interpret SIGHUP as a configuration reload request.

### Practical Exercise

Created a temporary process:

```bash
sleep 600 &
```

Inspected it:

```bash
ps -p 28708 -o pid,ppid,stat,etime,cmd
```

Sent SIGHUP:

```bash
kill -HUP 28708
```

Verified:

```bash
ps -p 28708 -o pid,ppid,stat,etime,cmd
```

**Observation:** The sleep process terminated because it did not implement a special SIGHUP handler.

### Real-world Example: NGINX

NGINX can interpret SIGHUP as a configuration reload request.

When the configuration is valid, its master process can apply the updated configuration while continuing to run.

If the new configuration is invalid, NGINX rejects it and continues using the previous working configuration.

**Important:** Signal behaviour depends on how the application handles the signal. SIGHUP does not automatically reload every application.

---

## 6. Foreground and Background Processes

### Foreground Process

A foreground process occupies the terminal while it runs.

The shell normally waits for the foreground command to finish or change state before accepting another command.

### Background Process

A background process runs without occupying the terminal, allowing the user to execute other commands.

The `&` operator starts a command in the background.

Example:

```bash
sleep 1200 &
```

### Shell Job Number vs Linux PID

Example output:

```text
[1] 12447
```

- `[1]` represents the shell job number.
- `12447` represents the Linux Process ID (PID).

The shell assigns job numbers within the current shell session. Linux assigns PIDs to processes.

### Important Job Management Commands

| Command | Purpose |
|---|---|
| `jobs` | Lists background and suspended jobs |
| `jobs -l` | Lists jobs with their PIDs |
| `fg %1` | Brings job 1 into the foreground |
| `bg %1` | Resumes suspended job 1 in the background |

---

## 7. Practical Job Control Exercise

### Step 1: Create a background process

```bash
sleep 1200 &
```

Observed output:

```text
[1] 10861
```

### Step 2: List background jobs

```bash
jobs
```

### Step 3: Display job numbers and PIDs

```bash
jobs -l
```

### Step 4: Inspect the process

```bash
ps -p 10861 -o pid,ppid,cmd
```

A syntax mistake was encountered initially by separating the output columns incorrectly.

Correct syntax:

```bash
ps -p 10861 -o pid,ppid,cmd
```

### Step 5: Bring the job into the foreground

```bash
fg %1
```

The terminal became occupied by the running sleep command.

### Step 6: Suspend the foreground process

Pressed:

```text
Ctrl + Z
```

Observed:

```text
[1]+ Stopped sleep 1200
```

The process was suspended, not terminated.

### Step 7: Resume the job in the background

```bash
bg %1
```

The suspended process resumed in the background.

### Step 8: Bring it into the foreground again

```bash
fg %1
```

### Step 9: Interrupt the foreground process

Pressed:

```text
Ctrl + C
```

The process terminated after receiving SIGINT.

### Step 10: Verify the job list

```bash
jobs -l
```

Observed an empty output.

**Conclusion:** No active or suspended jobs remained in the current shell.

---

## 8. Important Keyboard Controls

| Keyboard shortcut | Signal | Behaviour |
|---|---|---|
| Ctrl + C | SIGINT | Interrupts the foreground process |
| Ctrl + Z | SIGTSTP | Suspends the foreground process |

The `bg` command resumes a stopped job using SIGCONT.

### Important Distinction

- Ctrl + C interrupts the foreground process.
- Ctrl + Z suspends the foreground process.
- `bg` resumes a suspended process in the background.
- `fg` brings a job into the foreground.

---

## 9. Real-world DevOps Applications

These concepts are useful when:

1. Managing long-running scripts and background tasks.
2. Gracefully shutting down applications.
3. Troubleshooting unresponsive processes.
4. Reloading application configurations.
5. Managing processes on Linux servers.
6. Investigating application shutdown and restart behaviour.

### Important Engineering Practices

- Identify the correct process before sending a signal.
- Verify that a process is still running before attempting termination.
- Prefer graceful termination over forceful termination.
- Understand application-specific signal handling.
- Avoid terminating production processes without understanding their impact.
- Remember that running a command with `&` does not guarantee that it survives terminal closure or logout.

---

## 10. Learning Status

**Completed:**
- Linux signal fundamentals.
- SIGTERM practical exercise.
- SIGKILL practical exercise.
- SIGHUP practical exercise.
- Foreground and background execution.
- Shell job numbers and process IDs.
- `jobs`, `jobs -l`, `fg` and `bg`.
- Process suspension and resumption.
- Ctrl+C and Ctrl+Z behaviour.

---


