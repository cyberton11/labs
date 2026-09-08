---
share: true
title: Isolated Security Lab Deployment and Host Based Hardening Automation
---
This project demonstrates the design, deployment, and hardening of an isolated virtual home lab. The primary goal is to simulate a segmented corporate network to safely conduct security testing, ensuring that malicious traffic is contained and cannot leak into the production network or the internet.

### Technical objetives:

- **Virtualization:** Deploy virtual machines using a Type 2 hypervisor (VirtualBox).
- **Network Segmentation:** Configure an isolated internal network independent of external DHCP services.
- **Hardening:** Implement firewall rules via iptables to enforce the principle of least privilege on the target system.

---
## 2. Network Architecture 

The lab comprises two main hosts interconnected through a private, isolated network segment.
### Components: 

- **Attacker Host:** Kali Linux virtual machine configured as the penetration testing platform.
- **Target Host:** Metasploitable 3 virtual machine used as the intentionally vulnerable Linux system.
- **Network Segment:** Virtualized internal network (`192.168.100.0/24`).
### Network diagram:
![Netdiagram](undefineddocs/laboratories/lab-01/images/Netdiagram.png)

---
## 3. Hypervisor and Network Adapter Configuration

To ensure absolute containment, both virtual machines are deployed within **Oracle VM VirtualBox** using an **Internal Network** configuration. This setup isolates the traffic entirely within the hypervisor's virtual switch, preventing data leaks to the physical host or the external internet.

### Network Specifications: 

- **Network Type:** Internal Network 
- **Network Name:** 01-Defensive-Lab  
- **Subnet:** 192.168.100.0/24

### Step 1: Kali Linux Configuration in VirtualBox

Network Adapter 1 was manually configured with the following settings:

- **Name:** Deflab
- **Attached to:** Internal network
- **Promiscuous mode:** Allow VMs

### Step 2: Metasploitable 3 Configuration in VirtualBox

In this case, network adapter 1 vas configured as the Kali one:

- **Name:** Deflab
- **Attached to:** Internal network
- **Promiscuous mode:** Allow VMs

---

## 4. Operating System IP Configuration (Static Assignment)

Since an internal network does not include a DHCP server, IP addresses must be manually assigned within both virtual machines to enable communication across the `192.168.100.0/24` subnet.

### 4.1: Assigning a Session-Based Static IP in Kali Linux (Attacker)

To maintain flexibility for future projects that require external network or internet access, a non-persistent static IP address is assigned for the active session. This configuration will automatically reset upon the next reboot.

Network Adapter 1 (`eth0`) is manually assigned a static IP address and brought up using the following commands:

	sudo ip addr add 192.168.100.5/24 dev eth0
	sudo ip link set eth0 up

Connectivity and interface status are verified to confirm that the IP address was successfully assigned to the session:

	ip addr show eth0
   
![Pasted image 20260903172320](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903172320.png)

### 4.2. Assigning a Permanent Static IP in Metasploitable 3 (Target Host)

Since this image was pre-built via Vagrant, its default network configuration relies on a dual-interface setup. To ensure the network configuration remains persistent across reboots, the Debian network interfaces file must be modified directly.

After booting **Metasploitable 3**, log in using the default credentials (`vagrant`/`vagrant`).

Next, open the network interfaces configuration file using a text editor:

	sudo nano /etc/network/interfaces
   
Modify the file to bind the virtual interface (`eth0`) to the custom internal subnet, then save and exit the editor.

Finally, restart the networking service to apply the configuration changes permanently:

	sudo service networking restart
   
Finally, verify the persistent IP address assignment to confirm that the changes remain active:

	ip addr show eth0
  
![Pasted image 20260903175744](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903175744.png)

---
## 5. Connectivity and Isolation Verification

Before executing any security testing, network communication must be verified to ensure it functions correctly within the lab while remaining strictly isolated from external networks.

### Test 1: Internal Lab Connectivity (Ping Test)

From the **Kali Linux** terminal, execute a ping command to the Metasploitable 3 host to verify mutual connectivity:

	ping -c 4 192.168.100.10

* **Expected Result:** 4 packets transmitted, 4 received, 0% packet loss. This confirms that the virtual switch is operating correctly and internal network communication is fully functional.

![Pasted image 20260903132235](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903132235.png)

From the **Metasploitable 3** host, execute a ping command to the Kali Linux host to verify two-way communication:

	ping -c 4 192.168.100.5
	
- **Expected Result:** 4 packets transmitted, 4 received, 0% packet loss. This confirms that bidirectional traffic is successfully established within the isolated network segment.

![Pasted image 20260903133211](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903133211.png)

### Test 2: Absolute Isolation (Egress Traffic Test)

From either host, attempt to ping an external server (such as Google Public DNS at `8.8.8.8`) to verify complete outbound containment:

	ping -c 4 8.8.8.8

#### From Metasploitable3 Host

![Pasted image 20260903133349](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903133349.png)

- **Expected Result:** `Network is unreachable` or `100% packet loss`. This confirms absolute network containment, verifying that the target host cannot communicate with the external internet.
#### From Kali Linux Host
 
![Pasted image 20260903133422](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903133422.png)

* **Expected Result:** `Network is unreachable` or `100% packet loss.` This confirms that the laboratory is strictly isolated and secure for offensive operations.

---
## 6. Defensive Hardening: Firewall Configuration (iptables)

To demonstrate a defensive posture, the built-in Linux firewall (`iptables`) is configured on the **Metasploitable 3** (Target) host.

By default, Metasploitable exposes multiple vulnerable ports to network traffic. This implementation enforces a **least privilege** defensive policy: **drop all inbound traffic by default, except for HTTP requests (Port 80)**, restricting network access exclusively to the authorized attacker subnet.

***Step 1: Scanning Active Ports Before Hardening

From the **Kali Linux** terminal, a network scan is executed against the target host to identify the exposed attack surface and active services:

	nmap -p- 192.168.100.10

![Pasted image 20260903134136](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903134136.png)

*Observation: The target host exposes multiple legacy and high-risk services—such as SMB and FTP—directly to the internal network segment, significantly expanding its attack surface.*

### 6.1. Troubleshooting Interlude: Disabling the Default Ubuntu Firewall (UFW)

During the initial deployment of the native `iptables` rules, a conflict occurred where network ports remained open despite manual configurations. In Ubuntu-based distributions, **UFW (Uncomplicated Firewall)** operates as the default frontend wrapper over netfilter/iptables. If left enabled, UFW rules can conflict with or override manual, low-level `iptables` chains.

To resolve this conflict and ensure that custom rules dictate network access with absolute certainty, the default firewall management service was audited and deactivated on the **Metasploitable 3** (Target) host.

First, the status of the default firewall was verified:
  
	sudo ufw status
   
Next, disable the UFW service permanently to prevent further rule conflicts:

	sudo ufw disable
  
Finally, flush all pre-existing rules and user-defined chains to ensure a clean slate for the custom firewall deployment:
   
	sudo iptables -F

*Result: Once UFW was deactivated and the active tables were flushed, the native `iptables` rules took effect as intended, successfully modifying the port states according to the defensive policy.

***Step 2: Applying Defensive Rules on Metasploitable 3

After flushing the existing chains using the `-F` flag, a strict default security policy was established, and specific traffic exceptions were defined on the target host:

- 2.1. **Establishing the Default Policy (DROP)

The firewall was configured to implement a "deny-by-default" security posture. Under this configuration, the default policy for the `INPUT` chain is set to `DROP`, meaning any inbound packet that is not explicitly allowed by a subsequent rule will be automatically discarded:

	-P INPUT DROP

- 2.2. Opening Specific Ports for Services (FTP and SMB)

Selective rules were inserted (`-I`) at the top of the chain to enable legitimate TCP traffic to essential infrastructure services:

- **Port 21 (FTP - Control):** Allows incoming connection requests and command management.
- **Port 445 (SMB):** Allows network file sharing and access to shared system resources.

	`sudo iptables -I INPUT -p tcp --dport 20 -j ACCEPT`
	`sudo iptables -I INPUT -p tcp --dport 445 -j ACCEPT`

- 2.3. Allowing Local Traffic (Loopback Interface)

A rule was appended (`-A`) to authorize all inbound traffic through the local loopback interface (`lo`). This configuration is critical to ensure uninterrupted internal communication among the operating system's native services:

	 sudo iptables -A INPUT -i lo -j ACCEPT

- 2.4. Enforcing Default Drop Policies

To restrict network access and enforce a secure environment, the default policies were explicitly updated to automatically drop all inbound and transit traffic that does not match any established rule:

	sudo iptables -P INPUT DROP 
	sudo iptables -P FORWARD DROP 

- 2.5 Managing Stateful Connections

To ensure that active network sessions are not interrupted, a rule was appended (`-A`) to allow inbound traffic from already established or related connections:

	sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

- 2.6 Restricting HTTP Traffic to the Lab Subnet

To minimize the attack surface, HTTP traffic (Port 80) was strictly limited. The firewall is configured to allow inbound HTTP traffic originating exclusively from the authorized lab subnet range:

	sudo iptables -A INPUT -p tcp --dport 80 -s 192.168.100.0/24 -j ACCEPT

![Pasted image 20260903140542](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903140542.png)

***Step 3: Post-Hardening Verification Scan

After applying the custom `iptables` configuration, an identical verification scan was executed from the **Kali Linux** terminal to audit the new network posture:

	nmap -p 20,80,445 192.168.100.10

**Final Hardened Network State:**

* **Port 21 (FTP):** `filtered` _(Blocked by default drop policy)_
- **Port 80 (HTTP):** `open` _(Authorized corporate web service)_
- **Port 445 (SMB):** `filtered` _(Blocked by default drop policy)_

![Pasted image 20260903141102](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903141102.png)


*Observation: The host-based firewall successfully reduced the attack surface of these audited vectors by **66%**, isolating critical management protocols while keeping the core business application (Web Server) operational.*

### 6.2 Attack Simulation & Verification (Kali Linux)

Auditing the filtered states using `nmap` and `netcat`: 

	nmap -p 20,80,445 192.168.100.10 
	nc -nv 192.168.100.10 445  

* **Port 80 (HTTP):** Remains `open` for functional testing.
* **Ports 20 (FTP) & 445 (SMB):** Changed to `filtered`. 

**Note:** The `netcat` command outputs `Connection timed out`. Since the firewall drops packets silently rather than actively rejecting them, the attacker is forced to drain connection resources during reconnaissance.

![Pasted image 20260903142757](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903142757.png)

### 6.3 Forensic Log Analysis (Metasploitable 3) 

To establish a comprehensive security audit trail, a high-priority logging rule was positioned at the top of the `INPUT` chain. Inspecting the system kernel logs after the attack simulation reveals the captured defensive telemetry:

	 sudo grep "iptables:" /var/log/kern.log | tail -n 1
	 
**Captured Syslog Metadata:** 

	`Sep 3 19:28:57 metasploitable3-ub1404 kernel: iptables: IN=eth0 OUT= SRC=192.168.100.5 DST=192.168.100.10 PROTO=TCP SPT=43210 DPT=445 SYN`

![Pasted image 20260903142910](undefineddocs/laboratories/lab-01/images/Pasted%20image%2020260903142910.png)
#### Triage Analysis:

* - **`Timestamp & Hostname`:** Provides timeline correlation for incident responders.
- **`SRC=192.168.100.5`:** Pinpoints the exact origin address of the adversary (Kali Linux).
- **`DPT=445 (SYN)`:** Confirms that an unauthorized attempt to initiate a connection over the sensitive SMB management framework was successfully detected and mitigated before execution.

---

## 7. Key Takeaways & Cybersecurity Concepts Applied

To conclude this project, the following core infrastructure security principles were validated and analyzed throughout the deployment lifecycle:

1.  **Absolute Network Containment:** By selecting a VirtualBox Internal Network instead of a Bridged or NAT configuration, an air-gapped sandbox was successfully established. This architecture ensures that malicious payloads, scanning traffic, or automated exploits cannot escape to impact the physical host operating system or production network infrastructure.
2. **Host-Based Microsegmentation (Zero Trust):** Relying solely on perimeter defenses is an insufficient security strategy. By configuring granular access control rules directly on the target host via `iptables`, an internal defense-in-depth model was implemented, treating the local subnet under strict **Zero Trust** protocols.
3. **Attack Surface Reduction:** Legacy operating systems (such as this Metasploitable 3 Ubuntu image) often come bundled with insecure default configurations. Enforcing a default-deny (`DROP`) inbound policy safely reduced the target's network attack surface by **66%**, ensuring that the exploitation window is strictly restricted to authorized services without disrupting core business capabilities (HTTP web service).
4. **Defense-in-Depth Layering:** Even when a network asset runs legacy or unpatched software, implementing proper transport-layer access controls acts as a critical secondary defense line. This multi-tiered approach successfully stops unauthorized exploitation chains before they can reach vulnerable application layers.
---

## 8. Automation: Firewall Deployment Script (`hardener.sh`)

Since Netfilter/iptables rules reside within the kernel's volatile memory, they are non-persistent and reset automatically upon a system reboot. To ensure administrative efficiency and facilitate rapid, reproducible deployments, a custom **Bash automation script** was engineered.

The script automates the entire hardening workflow in a single execution: it verifies root privileges, disables conflicting frontend daemons (UFW), flushes active tables, injects granular audit logging chains, and secures the network interface according to the defined least-privilege policy.

### 8.1. Script Source Code (`/scripts/hardener.sh`):

Below is the complete Bash script engineered to automate the defensive hardening workflow on the target host:

	echo "[*] Starting Defensive Hardening Script..." # 

1. Check for root privileges 

	`if [ "\$EUID" -ne 0 ; then` 
	`echo "[-] Error: Please run this script as root (sudo)." >&2` 
	`exit 1` 
	`fi` 

2. Disable default UFW firewall to prevent conflicts 

	`echo "[*] Deactivating UFW default management wrapper..." ufw disable > /dev/null 2>&1` 

3. Flush existing rules and set Default Policies to DROP 

	`echo "[*] Flushing active tables and enforcing DEFAULT DROP policy..."` 
	`iptables -F` 
	`iptables -X` 
	`iptables -P INPUT DROP` 
	`iptables -P FORWARD DROP` 
	`iptables -P OUTPUT ACCEPT`

4. Allow essential local loopback traffic 

	`iptables -A INPUT -i lo -j ACCEPT` 

5. Allow established state-tracking sessions 

	`iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT` 

6. Inject Security Auditing Logging rule for Port 445 (SMB) 

	`echo "[*] Injecting Forensic Logging Chain for unauthorized SMB access..."` 
	`iptables -A INPUT -p tcp --dport 445 -j LOG --log-prefix "iptables: "` 

7. Strictly allow incoming Web traffic (Port 80) from the lab subnet 

	`echo "[*] Allowing HTTP traffic from authorized lab subnet (192.168.100.0/24)..."` 
	`iptables -A INPUT -p tcp --dport 80 -s 192.168.100.0/24 -j ACCEPT` 

	`echo "[+] Network Endorsement Complete! The system is now hardened."` 
	`echo "[+] Legitimate Web Traffic is OPEN. Sensitive services are FILTERED."` 

### 8.2. Deployment and Execution

To deploy and execute the automation script on the target host, follow these deployment steps: 

1. Create the script file using a text editor:  

	`nano hardener.sh` 

2. Grant absolute execution privileges to the owner: 

	`bash chmod u+x hardener.sh

3. Run the automation suite with administrative permissions:  

	`sudo ./hardener.sh` 


#### Deployment and Execution Instructions:

The deployment and execution of the automation script on the **Metasploitable 3** (Target) host are performed through the following operational steps:

1. Create the script file directly inside the target instance:
   
	`chmod +x hardener.sh`
   
2. Grant absolute execution privileges to the script file: 

	`sudo ./hardener.sh`

![[/Pasted image 20260903144708.png|/Pasted image 20260903144708.png]]