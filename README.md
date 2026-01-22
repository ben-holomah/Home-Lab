# Malware-Analysis-Lab




## Troubleshooting

**Issue**
- Pinging my win10 machine from my kali machine didn't work.
- I went into my win10 machine to try pinging my kali machine.
- I believe the firewall on the win10 machine blocked incoming traffic from the Kali machine because Kali was successfully pinged from there.

---

## Configuration

-I jumped into the VM settings and created a new LAN segment. I named it "test"
I found a list online of safe network options for specific use cases.
-NAT:  Access to Internet: Yes Use case: Test Tools
- NAT Network:  Access to Internet: Yes  Use case: Test Tools
- Bridged:  Access to Internet: Yes  Use case: Test Tools
- Host-Only: Access to Internet: No  Use case: Analyze Malware
- Internal Net:  Access to Internet: No  Use Case: Analyze Malware
- Not Attached:  Access to Internet: No  Use Case: Analyze Malware

-I then moved to statically assigning IP addresses to both machines so that they can communicate with eachother.

**Assigning IP addresses**

**Win10**
- I started with my win10 machine first.
  1. I opened up the internet & network settings
  2. Scrolled down to change adapter options
  3. Right clicked Ethernet0, chose properties, looked for IPv4, selected and clicked properties. It brought me to the general properties tab where I was then able to statically assign the IP address. I assigned the IP address of 192.168.20.10 and left the subnet mask at 255.255.255.0
  4. I opened up command prompt to check to if the IP address was successfully assigned using "ipconfig". (It was successfully assigned)

**Kali**
- I right clicked the ethernet symbol, and chose edit connections.
  1. I clicked the IPv4 settings tab
  2. I changed the method of automatic DCHP to Manual
  3. I assigned the IP address to 192.168.20.11
  4. I opened up the terminal and ran 'ifconfig' to again confirm that the IP address was successfully assigned (It was successfully assigned)
 
- After assigning both machines IP addresses successfully, I pinged the win10 machine from the Kali one.

**Issue**
- Pinging my win10 machine from my kali machine didn't work.
- I went into my win10 machine to try pinging my kali machine.
- I believe the firewall on the win10 machine blocked incoming traffic from the Kali machine because Kali was successfully pinged from there.

- I am now done with configuration with the intention of protecting my host machine while executing and sending malicious attacks from my Kali machine to my win10 machine.

**Logic:** I want to reduce the risk of compromising my host machine while executing and sending malicious attacks to my win10 machine.


**Installing Kali as the Attacker's Machine:**
- I noticed that the download file for Kali was a 7 zip file extension, so i had to download it.
