# Sysmon + Splunk Endpoint Detection Lab

## Overview
This project documents building an endpoint detection pipeline: deploying 
Sysmon on a Windows 11 VM to generate detailed system telemetry, then 
ingesting and searching that data in Splunk Enterprise to identify suspicious 
activity.

## Environment
- **Target machine:** Windows 11 Enterprise Evaluation (VMware)
- **Tools used:** Sysmon (with SwiftOnSecurity config), Splunk Enterprise (free tier)

---

## Step 1: Deploy Sysmon

Installed Sysmon using the industry-standard SwiftOnSecurity configuration 
file, which provides sensible default logging rules for process creation, 
network connections, and file activity — far more detailed than Windows' 
default logging.

Sysmon64.exe -i sysmonconfig-export.xml


To validate detection, an encoded PowerShell command was executed — a common 
technique attackers use to obscure malicious commands from casual observation: powershell.exe -enc SQBFAFgAIAAoAE4AZQB3AC0AT

<img width="2880" height="1800" alt="Screenshot (17)" src="https://github.com/user-attachments/assets/03c88efd-3df3-42a3-9bf0-52c1b54426d6" />

<img width="2880" height="1800" alt="Screenshot (18)" src="https://github.com/user-attachments/assets/964636ee-4e29-44b8-8417-4757e7cc577b" />


Sysmon captured the full process detail: command line, parent process, user 
context, integrity level, and file hashes — exactly the kind of telemetry a 
SOC analyst uses to determine whether activity is malicious.

## Step 2: Ingest Logs into Splunk

Installed Splunk Enterprise locally and configured it to ingest the Sysmon 
event log channel. This channel isn't available through Splunk's default 
Event Log picker (it only lists standard Windows channels), so it required 
a manual input definition instead:[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = main


This file was added to:C:\Program Files\Splunk\etc\system\local\inputs.conf

## Step 3: Search for Suspicious Activity

With Sysmon data flowing into Splunk, a targeted search isolated the 
encoded PowerShell execution from all other endpoint activity:index=main source="WinEventLog:Microsoft-Windows-Sysmon/Operational" CommandLine="-enc"

<img width="2880" height="1800" alt="Screenshot (19)" src="https://github.com/user-attachments/assets/32c01040-03fc-4aef-99c9-aa73793c807f" />



This single query filtered out normal system noise and surfaced exactly the 
event of interest — demonstrating how a SOC analyst would hunt for specific 
suspicious behavior across a large volume of endpoint logs.

## What I Learned
- How to deploy Sysmon with a real-world configuration (SwiftOnSecurity) 
  rather than default settings, and why default Windows logging isn't 
  sufficient for security monitoring
- How Sysmon Event ID 1 (Process Creation) captures the details needed to 
  evaluate whether a command is suspicious — full command line, parent 
  process, and hashes
- How to configure Splunk to ingest a custom Windows Event Log channel that 
  isn't exposed in the standard GUI wizard
- How to write a targeted SPL (Search Processing Language) query to filter 
  large volumes of log data down to a specific indicator
- The practical difference between a Universal Forwarder architecture 
  (multiple machines forwarding to a central Splunk instance over port 9997) 
  and local monitoring (Splunk reading logs directly on the same machine) — 
  and why the latter was the right fit for a single-VM lab

## Disclaimer
All activity in this lab was performed on a personal, isolated virtual 
machine for educational purposes. The encoded PowerShell command used for 
testing did not execute any harmful payload.
