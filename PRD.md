# Glasses: Product Requirements Document

- **Author**: Yashas
- **Date**: 9th Aug 2026
- **Status**: Approved
- **Reviewer**: Shreyas

### Summary

Glasses records your PC's resource usage in the background so you can look at what your machine was doing in the past, not only what it is doing right now. It is a command-line tool for anyone who has watched their PC slow down during a busy week and had no way to find out why once the problem passed. Existing monitors show you the present moment; Glasses keeps the history.

## Problem

Your PC was slowing down for a week. You had work to do, so you could not stop and investigate at the moment it was happening. On the weekend you finally have time, you turn the machine on, and it is fine! No issue whatsoever. The evidence is gone, and you still do not know what caused it.

Every tool available answers the wrong question. Task Manager, htop and top all show what is happening *right now*. None of them keep a record, so none of them can tell you what was happening at 5:45pm last Tuesday when the machine froze. The moment you are not watching is exactly the moment you needed the data.

So people guess. They blame the browser, close things at random, reinstall software, or buy more RAM they may not have needed — and they never find out whether any of it helped, because they had no measurement before and have none after.

## User

A PC user who wants to understand why and which process is eating system resources, either right now or a week ago. They are comfortable in a terminal, but they are not running server monitoring infrastructure for a single laptop.

## Goals

- A user can look back at any moment in the last month and see what their machine was doing
- A user can identify *which process* was responsible for a slowdown, not just that a slowdown happened
- A user gets this without having anticipated the problem or switched anything on beforehand
- Running Glasses does not itself make the machine slower
- A user can see whether it is running, turn it off, and remove it, without difficulty

## Success metrics

- The recorder uses under 1% CPU averaged over a minute, and under 50MB of memory
- After 30 days of continuous recording, stored data is under 150MB
- Over a 24-hour run, fewer than 1% of expected samples are missing
- Every goal above can be answered by a single command
- A read command over a 24-hour window returns in under 2 seconds

## Requirements

Priority: **M** = Must have for v1, **S** = Should have, **C** = Could have.

### Recording

| # | Pri | Requirement |
|---|---|---|
| R1 | M | Recording starts automatically after system boot, without the user launching anything. |
| R2 | M | Every 30 seconds, the recorder stores: timestamp (UTC), CPU %, memory used and %, disk used %, battery % and whether power is plugged in, network bytes sent and received. |
| R3 | M | At each sample, the recorder stores the 10 processes using the most CPU and the 10 using the most memory: process name, PID, CPU %, memory MB. |
| R4 | M | Recorded data survives a reboot. |
| R5 | M | Data older than 30 days is deleted automatically, with no user action. |
| R6 | S | If the recorder is killed or crashes, at most one sample is lost and previously recorded data stays readable. |
| R7 | M | Periods where nothing was recorded (PC off or asleep) are stored in a way that is distinguishable from a genuine reading of zero. |

### Reading

| # | Pri | Requirement |
|---|---|---|
| R8 | M | `glasses now` prints the current values of everything in R2, plus the current top processes. |
| R9 | M | `glasses last 24h` prints, for the window: peak CPU with the time it happened and the top process at that moment; peak memory with the same; averages; and how much of the window actually has data. |
| R10 | M | `glasses at "<timestamp>"` prints the sample nearest to that time, and states how far from the requested time it actually is. Timestamps are given in a fixed format; natural language input is out of scope. |
| R11 | S | `glasses top --last 24h` ranks processes by resource use across the whole window, not just at one moment. |
| R12 | M | Every command states plainly when data is missing for part of the requested range. It never presents a gap as a zero. |
| R13 | C | `--json` on any command prints machine-readable output. |
| R14 | M | Output is a plain text table, readable in an 80-column terminal. |

### Lifecycle

| # | Pri | Requirement |
|---|---|---|
| R15 | M | A user can install and enable background recording with a single documented command. |
| R16 | M | A user can stop recording without uninstalling. While disabled, recording does not restart at boot. |
| R17 | M | A user can check whether the recorder is running and when the last sample was taken. |
| R18 | S | A user can uninstall Glasses completely, including its stored data, with a single documented command. |

### Non-functional

| # | Pri | Requirement |
|---|---|---|
| R19 | M | The recorder uses under 1% CPU averaged over a minute, and under 50MB memory. |
| R20 | M | After 30 days of continuous recording, stored data stays under 150MB. |
| R21 | S | Any read command returns within 2 seconds for a 24-hour window. |
| R22 | M | Runs on Windows 10 and 11. |

## In-Scope

- Windows 10 and 11
- A background recorder sampling every 30 seconds, starting automatically at boot
- System metrics: CPU, memory, disk, battery, network I/O
- The top 10 processes by CPU and by memory at each sample
- 30 days of retention, pruned automatically
- Read commands: current state, summary over a window, state at a point in time, process ranking
- Lifecycle commands: install, enable, disable, status, uninstall
- Plain-text table output, fixed-format timestamp input

## Non-goals

Each of these is deliberately excluded from v1.

- **Starting or killing processes** — Glasses observes, it does not control
- **A graphical interface** — CLI only for v1 (see Future work)
- **Mobile or Android** — needs a network service and authentication, a project in itself
- **Cloud storage or sync** — all data stays on the local machine
- **Monitoring a different machine** — local only
- **Temperature and GPU metrics** — not available cross-platform through one interface
- **Alerting or notifications** — Glasses is queried, it does not interrupt
- **Linux and macOS** — Windows only for v1
- **Recording every process** — only the top consumers per sample, to bound cost and storage
- **Per-process lifecycle tracking** — creation-to-exit history is v2
- **Natural language time input** — fixed timestamp formats only
- **Recommending or performing remediation** — Glasses tells you what happened; deciding what to do is up to the user

## Open questions

- What should happen if stored data reaches the size budget before the retention period is up — prune early, or stop recording? (to be decided in the design doc)
- Should the recorder reduce its sampling rate on battery power to save energy?

## Future work

- v2: Configuration options for data retention and the number of processes recorded per sample
- v3: Tracking processes from creation to kill
- v4: Add GUI 
- v5: Add graphs and visual tools
- v6: Add ability to store data to S3 in real time
- v7: Monitoring one system from a different one
- v8: Add Android app
- v9: Add website
