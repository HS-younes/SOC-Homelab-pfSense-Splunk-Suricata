[readme_md.md](https://github.com/user-attachments/files/33614754/readme_md.md)
# Building a Cybersecurity Homelab for Detection & Monitoring

**Environment:** VMware Workstation Pro, pfSense, Splunk Enterprise, Suricata IDS, Active Directory (Windows Server 2019 & Windows 10), Kali Linux

## Project Objectives

- Virtualized SOC environment using VMware Workstation and pfSense as the core firewall for network segmentation across Windows and Linux systems.
- Centralized log collection and alert analysis using Splunk Enterprise.
- Integration of Suricata IDS via a SPAN port to monitor network traffic and detect suspicious activity.
- Deployment of an Active Directory domain (1 Domain Controller and 2 Windows 10 clients) to run controlled detection scenarios mapped to MITRE ATT&CK.

## Network Topology

<img width="1382" height="842" alt="image" src="https://github.com/user-attachments/assets/ad326407-5b3b-4ca9-8bc7-c8926c45a7c3" />


### Subnet & Interface Table

| pfSense Interface | Adapter | VMware Network | Subnet / Gateway | Connected Machines |
| :--- | :--- | :--- | :--- | :--- |
| WAN | `em0` | Bridged / NAT | DHCP | Internet access via host |
| LAN (Kali) | `em1` | `VMnet 2` | `192.168.1.1/24` | Kali Linux (`192.168.1.10`) |
| OPT1 (VictimNetwork) | `em2` | `VMnet 3` | `192.168.2.1/24` | AD Domain Controller (`192.168.2.10`) & 2 Win10 Clients (`192.168.2.21`, `192.168.2.22`) |
| OPT2 (IDS_Mgmt) | `em3` | `VMnet 4` | `192.168.3.1/24` | Suricata IDS Management (`192.168.3.10`) |
| OPT3 (SPANPORT) | `em4` | `VMnet 5` | None (Bridge) | Traffic mirror from `em2` (`VictimNetwork`) to Suricata IDS |
| OPT4 (Splunk) | `em5` | `VMnet 6` | `192.168.4.1/24` | Splunk Enterprise Server (`192.168.4.10`) |

---

## Lab Hardware & VM Requirements

This lab was built on VMware Workstation Pro. Below are the specifications allocated to each virtual machine:

- **pfSense Firewall:** 2 GB RAM, 20 GB Disk, 6 Network Adapters (`NAT/Bridged`, `VMnet 2` through `VMnet 6`)
- **Kali Linux:** 4 GB RAM, 80 GB Disk, 1 Network Adapter (`VMnet 2`)
- **Windows Server 2019 (DC):** 4 GB RAM, 60 GB Disk, 1 Network Adapter (`VMnet 3`)
- **Windows 10 Client 1 & Client 2:** 2 GB RAM each, 40 GB Disk each, 1 Network Adapter (`VMnet 3`)
- **Suricata IDS (Ubuntu Server):** 4 GB RAM, 50 GB Disk, 2 Network Adapters (`VMnet 4` for management/forwarding, `VMnet 5` for packet capture)
- **Splunk Server (Ubuntu Server):** 4 GB - 8 GB RAM, 100 GB Disk, 1 Network Adapter (`VMnet 6`)

---

## 1. pfSense Firewall Configuration

### VM Setup and Console Configuration
1. Download the pfSense Community Edition `.iso` and extract it.
2. In VMware Workstation, create a new VM using the installer image (FreeBSD 64-bit, 2 GB RAM, 20 GB disk split into multiple files).
3. Before booting, open the VM settings and add 5 custom network adapters assigned to `VMnet 2`, `VMnet 3`, `VMnet 4`, `VMnet 5`, and `VMnet 6`.
4. Boot the VM, accept the default installation settings (Guided Root-on-ZFS, Stripe, select virtual disk), and reboot after removing the installation ISO.
5. In the pfSense console menu, select **Option 1 (Assign Interfaces)**:
   - Configure VLANs: `n`
   - WAN: `em0`
   - LAN: `em1`
   - OPT1: `em2`
   - OPT2: `em3`
   - OPT3: `em4`
   - OPT4: `em5`
6. Select **Option 2 (Set interface(s) IP address)** to configure the subnets and DHCP ranges:
   - **LAN (`em1`):** `192.168.1.1/24`, DHCP enabled (`192.168.1.10` to `192.168.1.245`)
   - **OPT1 (`em2`):** `192.168.2.1/24`, DHCP enabled (`192.168.2.10` to `192.168.2.245`)
   - **OPT2 (`em3`):** `192.168.3.1/24`, DHCP enabled (`192.168.3.10` to `192.168.3.245`)
   - **OPT3 (`em4`):** Unnumbered interface (used as a dedicated SPAN port)
   - **OPT4 (`em5`):** `192.168.4.1/24`, DHCP enabled (`192.168.4.10` to `192.168.4.245`)

![pfSense Console Interfaces](images/pfsense-interfaces.png)

### WebGUI and SPAN Port Setup
1. Boot the Kali Linux VM on `VMnet 2` and open `https://192.168.1.1` in the web browser.
2. Log in with the default credentials (`admin` / `pfsense`) and go through the initial setup wizard:
   - Set Primary DNS to `8.8.8.8` and Secondary DNS to `4.4.4.4`.
   - Uncheck **Block RFC1918 Private Networks** and **Block bogon networks** on the WAN configuration page.
   - Update the admin password and reload the firewall.
3. Go to **Interfaces > Assignments** and rename the interfaces to match the topology:
   - `LAN` -> `Kali`
   - `OPT1` -> `VictimNetwork`
   - `OPT2` -> `IDS_Mgmt`
   - `OPT3` -> `SPANPORT` (Enable interface, leave IPv4 Configuration Type as `None`)
   - `OPT4` -> `Splunk`
4. To copy traffic from the victim subnet to Suricata, go to **Interfaces > Assignments > Bridges** and click **Add**:
   - Select `VictimNetwork` as the **Member Interface**.
   - Click **Display Advanced**, scroll to **Span Port**, and select `SPANPORT`.
   - Save and apply the configuration.

![pfSense SPAN Bridge](images/pfsense-bridge.png)

---

## 2. Active Directory Environment Setup

### Domain Controller (Windows Server 2019)
1. Create a VM using the Windows Server 2019 Evaluation ISO on `VMnet 3` (4 GB RAM, 60 GB disk).
2. In the VM settings, remove the Floppy drive so the OS installer does not look for an unattended install file, then boot the VM and select **Windows Server 2019 Standard Evaluation (Desktop Experience)**.
3. Install VMware Tools and assign a static IP address in Network Connections (`ncpa.cpl`):
   - IP Address: `192.168.2.10`
   - Subnet Mask: `255.255.255.0`
   - Default Gateway: `192.168.2.1`
   - Preferred DNS: `127.0.0.1`
4. Rename the server to `DC01` and restart.
5. In **Server Manager**, click **Manage > Add Roles and Features** and select **Active Directory Domain Services** and **Active Directory Certificate Services**.
6. After installation finishes, click the notification flag in Server Manager:
   - Click **Promote this server to a domain controller**.
   - Select **Add a new forest** and enter `CYBERWOX.local` as the root domain name.
   - Set the DSRM password and complete the promotion (the server will reboot automatically).
7. Log back in as `CYBERWOX\Administrator`, click the notification flag again, and select **Configure Active Directory Certificate Services**:
   - Check **Certification Authority**, keep the default Enterprise CA and Root CA options, set the validity period to `99 years`, and apply.
8. Open **Active Directory Users and Computers**, go to `CYBERWOX.local > Users`, and create two standard domain user accounts (`user1` and `user2`) for the client machines.
9. In pfSense (**Services > DHCP Server > VICTIMSNETWORK**), set the DNS Server to `192.168.2.10` and the Domain Name to `CYBERWOX.local`.

![Active Directory Setup](images/ad-domain.png)

### Joining the Two Windows 10 Clients
1. Create two Windows 10 Enterprise Evaluation VMs on `VMnet 3` (`Win10-Client1` and `Win10-Client2`).
2. During setup, select **Domain join instead** to create a local account first.
3. Configure static IPs on both clients:
   - **Client 1:** `192.168.2.21/24`, Gateway `192.168.2.1`, DNS `192.168.2.10`
   - **Client 2:** `192.168.2.22/24`, Gateway `192.168.2.1`, DNS `192.168.2.10`
4. Go to **Settings > Accounts > Access work or school > Connect**, click **Join this device to a local Active Directory domain**, type `CYBERWOX.local`, and authenticate with the Domain Administrator account.
5. Reboot both clients and log in using the domain user accounts created earlier.

---

## 3. Suricata IDS Setup

Instead of Security Onion, Suricata IDS is installed on an Ubuntu Server VM to monitor the mirrored traffic coming from `VMnet 5`.

1. Create an Ubuntu Server VM with two network adapters:
   - Adapter 1 (`ens33`): `VMnet 4` with static IP `192.168.3.10/24` (Gateway `192.168.3.1`)
   - Adapter 2 (`ens35`): `VMnet 5` (SPAN port interface, no IP assigned)
2. Set the monitoring interface (`ens35`) to promiscuous mode so it captures all mirrored packets from the Active Directory subnet:
   ```bash
   sudo ip link set ens35 up
   sudo ip link set ens35 promisc on
   ```
3. Install Suricata from the official repository:
   ```bash
   sudo add-apt-repository ppa:oisf/suricata-stable -y
   sudo apt update
   sudo apt install suricata jq -y
   ```
4. Edit `/etc/suricata/suricata.yaml`:
   - Set `HOME_NET: "[192.168.2.0/24, 192.168.4.0/24]"`
   - Under the `af-packet` section, change the listening interface to `ens35`.
   - Ensure `eve-log` is enabled and writes events to `/var/log/suricata/eve.json`.
5. Update the Emerging Threats ruleset and start the Suricata service:
   ```bash
   sudo suricata-update
   sudo systemctl enable suricata
   sudo systemctl restart suricata
   ```

![Suricata Logs](images/suricata-logs.png)

---

## 4. Splunk Enterprise & Log Forwarding

### Splunk Server Installation (`192.168.4.10`)
1. Deploy an Ubuntu Server VM on `VMnet 6` with static IP `192.168.4.10/24` (Gateway `192.168.4.1`).
2. Download the Splunk Enterprise `.tgz` package for Linux, extract it to `/opt`, and start Splunk:
   ```bash
   sudo tar -xvzf splunk-*.tgz -C /opt/
   sudo /opt/splunk/bin/splunk start --accept-license
   ```
3. Open `http://192.168.4.10:8000` and log in.
4. Go to **Settings > Forwarding and receiving > Configure receiving** and add port `9997`.
5. Go to **Settings > Indexes > New Index** and create two indexes:
   - `wineventlog` (for Active Directory and Windows 10 logs)
   - `suricata` (for Suricata IDS alerts and network logs)

### Installing Splunk Universal Forwarders
1. **On the Windows Domain Controller and both Windows 10 Clients:**
   - Download and run the Splunk Universal Forwarder `.msi` installer.
   - Check the option for **On-Premises Splunk Enterprise**.
   - Enter `192.168.4.10` and port `8089` for the Deployment Server.
   - Enter `192.168.4.10` and port `9997` for the Receiving Indexer.
   - In the Splunk Web interface (**Settings > Add Data > Forward**), select the Windows hosts, choose **Local Event Logs** (`Application`, `Security`, `System`), and assign them to the `wineventlog` index.
2. **On the Suricata IDS Server (`192.168.3.10`):**
   - Install the Linux Splunk Universal Forwarder in `/opt/splunkforwarder`.
   - Add the Receiving Indexer:
     ```bash
     sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.4.10:9997
     ```
   - Configure `/opt/splunkforwarder/etc/system/local/inputs.conf` to monitor the Suricata JSON log file:
     ```ini
     [monitor:///var/log/suricata/eve.json]
     disabled = false
     index = suricata
     sourcetype = suricata:eve
     ```
   - Restart the forwarder:
     ```bash
     sudo /opt/splunkforwarder/bin/splunk restart
     ```

![Splunk Search Results](images/splunk-events.png)

---

## 5. Detection Scenarios (MITRE ATT&CK)

With all logs reporting to Splunk, the following tests were run from the Kali Linux machine (`192.168.1.10`) and the domain clients to verify detection rules.

### Scenario 1: Network Port Scanning (MITRE ATT&CK: T1046)
- **Technique:** Network Service Discovery
- **Action:** Run an Nmap scan from Kali Linux against the Active Directory subnet:
  ```bash
  nmap -sS -sV 192.168.2.0/24
  ```
- **Splunk Query (Suricata IDS):**
  ```spl
  index="suricata" event_type="alert" src_ip="192.168.1.10"
  | stats count by alert.signature, src_ip, dest_ip, dest_port
  ```

### Scenario 2: SMB / Domain Brute Force (MITRE ATT&CK: T1110.001)
- **Technique:** Brute Force - Password Guessing
- **Action:** Attempt multiple failed logins against the Domain Controller (`192.168.2.10`) using Hydra or CrackMapExec from Kali Linux.
- **Splunk Query (Windows Security Logs):**
  Look for Event ID `4625` (Failed Logon) and Event ID `4740` (Account Lockout):
  ```spl
  index="wineventlog" source="WinEventLog:Security" (EventCode=4625 OR EventCode=4740)
  | stats count by TargetUserName, IpAddress, ComputerName, EventCode
  ```

### Scenario 3: Unauthorized Domain Account Creation (MITRE ATT&CK: T1136.002)
- **Technique:** Create Account - Domain Account & Account Manipulation (`T1098`)
- **Action:** Create a new user on the Domain Controller and add it to the `Domain Admins` group:
  ```cmd
  net user support_adm Pass1234! /add /domain
  net group "Domain Admins" support_adm /add /domain
  ```
- **Splunk Query (Windows Security Logs):**
  Look for Event ID `4720` (User Account Created) and Event ID `4728` (Member Added to Global Security Group):
  ```spl
  index="wineventlog" source="WinEventLog:Security" (EventCode=4720 OR EventCode=4728)
  | table _time, ComputerName, EventCode, SubjectUserName, TargetUserName
  ```

### Scenario 4: Suspicious PowerShell Execution (MITRE ATT&CK: T1059.001)
- **Technique:** Command and Scripting Interpreter - PowerShell
- **Action:** Execute an encoded PowerShell command on one of the Windows 10 clients (`192.168.2.21`):
  ```powershell
  powershell.exe -NoProfile -ExecutionPolicy Bypass -EncodedCommand JABjAGwAaQBlAG4AdAA=
  ```
- **Splunk Query (Windows Event Logs):**
  Look for Event ID `4688` (Process Creation) or `4104` (PowerShell Script Block Logging):
  ```spl
  index="wineventlog" (EventCode=4688 OR EventCode=4104) ("-EncodedCommand" OR "-enc" OR "Bypass")
  | table _time, host, User, New_Process_Name, Process_Command_Line
  ```

