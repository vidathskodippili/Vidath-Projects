# Wazuh Home Lab: File Integrity Monitoring

A small detection lab that uses Wazuh to monitor a Windows 11 machine for file changes in real time. A Wazuh manager runs on Kali Linux, a Wazuh agent runs on Windows 11, and File Integrity Monitoring (FIM) alerts appear in the Wazuh dashboard whenever a file is created, modified, or deleted in a monitored folder.

## What I built

I used UTM to set up two virtual machines on the same network:

| Machine | IP address | Role |
|---|---|---|
| Kali Linux VM | 192.168.64.9 | Wazuh manager |
| Windows 11 VM | 192.168.64.8 | Wazuh agent |

With the agent connected to the manager, I enabled File Integrity Monitoring by adding the following line to the agent's `ossec.conf` file:

```xml
<directories realtime="yes">C:\Users\Windows\Desktop\Wazuh-Test</directories>
```

The `realtime="yes"` option makes the agent report changes to the `Wazuh-Test` folder as they happen instead of waiting for the next scheduled scan, which makes it much easier to see the effect of creating, editing, or deleting files.

## Purpose

Unexpected file changes are often the first visible sign of a compromise. A malicious file downloaded from the internet or dropped by an attacker can quickly lead to data theft, ransomware, or system outages. This lab shows how FIM gives SOC analysts early visibility of these events. When a file is created, modified, or deleted in a monitored location, Wazuh raises an alert with details such as the file path and hash. Analysts can then investigate the file and respond before the damage spreads.

This matters because a single malicious file can affect all three parts of the CIA triad:

- **Confidentiality:** spyware or infostealers can expose sensitive company data.
- **Integrity:** tampered files or configurations can no longer be trusted.
- **Availability:** ransomware can encrypt files and halt business operations.

## Architecture

![Architecture diagram showing a Windows 11 VM with a Wazuh agent sending events to a Wazuh manager on a Kali Linux VM, both hosted in UTM](images/architecture.png)

The agent on the Windows VM watches the `Wazuh-Test` folder. When something changes, it sends an event to the manager on Kali, which analyses it and raises a FIM alert.

## Setup

1. Created a Kali Linux VM and a Windows 11 VM in UTM on the same network.
2. Set up the Wazuh manager on the Kali VM (192.168.64.9).
3. Installed the Wazuh agent on the Windows VM (192.168.64.8) and connected it to the manager.
4. Edited the agent's `ossec.conf` (default location: `C:\Program Files (x86)\ossec-agent\ossec.conf`) and added the real-time monitoring line shown above inside the `<syscheck>` section.
5. Created the `Wazuh-Test` folder on the Windows desktop and restarted the agent so the new configuration was loaded.
6. Opened the Wazuh dashboard from the Kali VM and went to the File Integrity Monitoring module to view events.

## Testing and results

To test the setup, I created a new file, then created, edited, and deleted a text file called `a.txt` inside the monitored folder. Wazuh picked up every action and reported it in the dashboard:

![Wazuh dashboard File Integrity Monitoring events showing added, modified, and deleted events for files in the Wazuh-Test folder](images/fim-events.png)

| Time (3 Oct 2026) | Action I performed | `syscheck.event` | Rule ID | Rule description | Level |
|---|---|---|---|---|---|
| 23:34:39 | Created a new bitmap image file | added | 554 | File added to the system | 5 |
| 23:35:50 | Created `a.txt` | added | 554 | File added to the system | 5 |
| 23:35:55 | Edited `a.txt` | modified | 550 | Integrity checksum changed | 7 |
| 23:36:00 | Deleted `a.txt` | deleted | 553 | File deleted | 7 |

Each action produced its own alert within seconds, including the full file path in `syscheck.path`, which confirmed that real-time monitoring was working.

## What I learnt

- How the Wazuh agent and manager split the work: the agent collects data and watches the files, and the manager analyses events and raises alerts.
- The difference between real-time and scheduled FIM scans, and how a single `realtime="yes"` setting in `ossec.conf` changes how quickly changes are reported.
- How to read FIM events in the Wazuh dashboard, including the `syscheck.path` and `syscheck.event` fields and what rules 550, 553, and 554 mean.
- That FIM tells you a file changed, not that it is malicious. An analyst still needs to investigate, for example by checking the file hash against a threat intelligence source.
- How file-level events connect to the CIA triad and why early detection matters to a SOC.
- How to build and network a multi-VM lab in UTM so the manager and agent can communicate.

## Problems I faced

| Problem | Cause | How I fixed it |
|---|---|---|
| The Wazuh agent would not connect to the manager | The agent version did not match the manager version, and an agent cannot be newer than the manager it reports to | Installed an agent version compatible with the manager |
| FIM events for added, modified, and deleted files were not showing in the dashboard | The agent was not applying the monitoring rule | Checked that the `<directories>` line sat inside the `<syscheck>` block with the correct folder path, then restarted the agent so the configuration was reloaded |

## Next steps

- Integrate Wazuh with VirusTotal so file hashes from FIM events are checked automatically.
- Monitor higher-risk locations such as the Downloads folder.
- Test with a harmless simulated malware file (such as the EICAR test string) and write up the analyst response.
- Explore Wazuh Active Response to automatically act on suspicious files.

## Tools used

Wazuh (manager and agent), UTM, Kali Linux, Windows 11
