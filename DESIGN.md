# **Glasses**: Design Document

- **Author**: Yashas
- **Date**: 
- **Status**: Draft
- **Reviewer**: Shrihari
- **PRD**: [PRD.md](PRD.md)

##  Context

Read the [PRD.md](./PRD.md) file, it will tell you want we are building. In summary we are building a tool for understanding system resource usage by applications/process.

## Technical goals & non-goals

| **Goals** | **Description**                                              |
| --------- | ------------------------------------------------------------ |
| G1        | The recorder will be a background system process which will consume less than 1% of CPU and consume less than 50MBs of memory |
| G2        | 30 day data must be less than 150mb on the disk              |
| G3        | Data older than 30days gets removed automatically            |
| G4        | All display of data or query must happen under 2 seconds from the call |
| G5        | If the recorder crashes, previously recorded data stays readable and at most one sample is lost |
| G6        | In 24 hour time span, no more than 1% of samples can be lost |
| G7        | Read from display must not block writes on the storage       |
| G8        | A sample will only be read when its complete                 |
| G9        | A corrupt sample will have "-"s                              |
| G10       | When there are no samples the display will have no values i,e the display is empty |
| G11       | Once enabled, recording resumes automatically on every subsequent system boot without further user action, until explicitly disabled |
| G12       | After enabling, recording of first sample happens after 30s  |
| G13       | Once disabled, recording does not start on reboot            |
| G14       | A gap where the system was off is shown as an explicit marker line, not as individual timestamped rows for that period |
| G15       | If data is unreadable/corrupt/missing there will be timestamps. If system was off then timestamps will not be available |
| G16       | Tool will be compatible with Windows 11 and 10               |
| G17       | Output must fit within an 80-column terminal width, with no line wrapping or truncation |
| G18       | Data stored on the disk is persistent across reboots         |
| G19       | The tool detects and flags manual system clock changes, so a clock jump is never mistaken for elapsed real time |
| G20       | There will be only one recording instance on one machine. If a second instance is started it will not start and user will see a message saying recorder is running |
| G21       | The recorder is system level not user level. i.e if another user logs into windows and the recorder was previously enabled recorder will be collecting data |
| G22       | If there are multiple disks they will all be kept track of as long as they are inside the device. |
| G23       | If the battery is not connected here will be no data in that cell |
| G24       | If the battery has no power but its connected it will show up as 0% |
| G25       | When the tool is uninstalled all the data related to it will be removed off the system |
| G26       | Network IO will track of all and each physical input output channels |

| **Non-Goals** | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| NG1           | Tool not compatible with Linux, MacOS, Open/FreeBSD          |
| NG2           | Tool will not inform why corrupt data is bad                 |
| NG3           | Tool will not work when memory is at 100%                    |
| NG4           | Tool will not record when disk is at 100%                    |
| NG5           | For v1, sampling interval and data retention period are constant |
| NG6           | If a clock change occurs past timestamps are not updated     |
| NG7           | The storage of collected data will occur on a single disk    |
| NG8           | Externally connected disks (USB, external HDD/SSD) are not tracked. |
| NG9           | Non-physical network IO will not be tracked                  |

## Overview

There are three parts-Recorder, Storage and Display

![](./Glasses-overview.drawio.png)
