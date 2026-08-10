# Splunk SIEM Deployment & Real-Time SSH Brute-Force Detection Lab

An end-to-end implementation guide and project documentation for deploying a **Splunk Enterprise SIEM** environment, configuring a **Splunk Universal Forwarder** on a Linux target, defining **SPL detection rules**, and validating real-time security alert triggers using **Hydra SSH brute-force attack simulations**.

---

## 📌 Table of Contents
1. [Architecture Overview](#-architecture-overview)
2. [Prerequisites & Resources](#-prerequisites--resources)
3. [Virtual Environment Setup](#-virtual-environment-setup)
4. [SIEM Server Deployment (Splunk Enterprise)](#-siem-server-deployment-splunk-enterprise)
5. [Log Source Deployment (Splunk Universal Forwarder)](#-log-source-deployment-splunk-universal-forwarder)
6. [Data Ingestion & Forwarding Verification](#-data-ingestion--forwarding-verification)
7. [Real-Time Incident Detection Rule Creation](#-real-time-incident-detection-rule-creation)
8. [Attack Simulation & Alert Verification](#-attack-simulation--alert-verification)
9. [Troubleshooting & Handy Commands](#-troubleshooting--handy-commands)

---

## 🏗 Architecture Overview

| Component | Role | OS / Platform | IP Address | Credentials |
| :--- | :--- | :--- | :--- | :--- |
| **SIEM Server** | Centralized Logging & Analysis | Ubuntu Linux | `<SIEM_SERVER_IP>` | `<SIEM_USER>` / `<SIEM_PASSWORD>` |
| **Victim Target** | Log Producer (Monitored Host) | Ubuntu Linux | `<TARGET_HOST_IP>` | `<TARGET_USER>` / `<TARGET_PASSWORD>` |
| **Attacker Host** | Red Team Attack Simulator | Kali Linux | Isolated Lab Network | N/A |
| **Splunk Web UI** | Security Operations Management | Web Dashboard | `<SIEM_SERVER_IP>:8000` | `admin` / `<SPLUNK_ADMIN_PASSWORD>` |

---

## 🧰 Prerequisites & Resources

* **Virtualization**: Oracle VirtualBox
* **Official Downloads**: [Splunk Official Downloads](https://www.splunk.com/en_us/download.html)
* **Packages**:
  * Splunk Enterprise `10.0.1` (`.deb`)
  * Splunk Universal Forwarder `10.0.1` (`.deb`)
  * Kali Linux ISO / Virtual Appliance

---

## ⚙️ Virtual Environment Setup

### VirtualBox Network Configuration
To establish isolated and predictable network communications between the SIEM, Victim, and Kali hosts:

1. **Motherboard & Processor**: Adjust RAM and CPU allocation according to host system limits.
2. **Adapter 1**: Set to **NAT Network** (e.g., `<LAB_NAT_NETWORK>`).
3. **Adapter 2**: Set to **Host-Only Adapter** with promiscuous mode set to `Deny` and cable connected.
4. Apply network settings to both the **SIEM Server** and **Victim Target** VMs.
5. Boot Kali Linux on the same **NAT Network** (`<LAB_NAT_NETWORK>`).

---

## 🚀 SIEM Server Deployment (Splunk Enterprise)

### 1. Remote SSH Access
```bash
ssh <SIEM_USER>@<SIEM_SERVER_IP>
```

### 2. Download Splunk Enterprise Package
```bash
sudo wget -O splunk-10.0.1-c486717c322b-linux-amd64.deb "[https://download.splunk.com/products/splunk/releases/10.0.1/linux/splunk-10.0.1-c486717c322b-linux-amd64.deb](https://download.splunk.com/products/splunk/releases/10.0.1/linux/splunk-10.0.1-c486717c322b-linux-amd64.deb)"
```

### 3. Execute Automated Setup Script
```bash
cd splunk-siem/
./setup_splunk_siem.sh
```
*When prompted, enter the absolute path to the `.deb` file:*
```text
/home/<SIEM_USER>/splunk-10.0.1-c486717c322b-linux-amd64.deb
```

### 4. Start Splunk Enterprise Daemon
```bash
sudo /opt/splunk/bin/splunk start
```

---

## 📡 Log Source Deployment (Splunk Universal Forwarder)

### 1. Remote SSH Access to Victim Host
```bash
ssh <TARGET_USER>@<TARGET_HOST_IP>
```

### 2. Download Universal Forwarder Package
```bash
sudo wget -O splunkforwarder-10.0.1-c486717c322b-linux-amd64.deb "[https://download.splunk.com/products/universalforwarder/releases/10.0.1/linux/splunkforwarder-10.0.1-c486717c322b-linux-amd64.deb](https://download.splunk.com/products/universalforwarder/releases/10.0.1/linux/splunkforwarder-10.0.1-c486717c322b-linux-amd64.deb)"
```

### 3. Execute Automated Forwarder Setup Script
```bash
cd splunk-forwarder/
./install_splunk_forwarder.sh
```
*When prompted, enter the path:*
```text
/home/<TARGET_USER>/splunk-forwarder/splunkforwarder-10.0.1-c486717c322b-linux-amd64.deb
```

---

## 🛠 Data Ingestion & Forwarding Verification

On the **Victim Machine** (`<TARGET_HOST_IP>`), configure log forwarding to the Splunk Enterprise SIEM Indexer on port `9997`.

### 1. Configure Receiver Target
```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server <SIEM_SERVER_IP>:9997
```

### 2. Add System Authentication Log Monitor
```bash
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log
```
*(Provide Splunk local admin credentials when prompted: Username: `admin` | Password: `<SPLUNK_ADMIN_PASSWORD>`)*

### 3. Verify Active Forwarders & Monitored Paths
```bash
# Check forwarding target status
sudo /opt/splunkforwarder/bin/splunk list forward-server

# Check actively monitored log files
sudo /opt/splunkforwarder/bin/splunk list monitor
```

### 4. Restart Services
```bash
# On SIEM Server (<SIEM_SERVER_IP>):
sudo /opt/splunk/bin/splunk restart

# On Victim Machine (<TARGET_HOST_IP>):
sudo /opt/splunkforwarder/bin/splunk restart
```

---

## 🔍 Real-Time Incident Detection Rule Creation

1. Access the Splunk Web Dashboard via browser: `http://<SIEM_SERVER_IP>:8000`
2. Login with `admin` / `<SPLUNK_ADMIN_PASSWORD>`.
3. Open **Search & Reporting** app and verify log ingestion using `index=*`.

### Real-Time Detection SPL Query
Navigate to **Settings** > **Searches, Reports, and Alerts** > **New Alert**:

```spl
index=main "Failed password" OR "Invalid user" OR "authentication failure" OR "authentication failure;" 
| table _time host sourcetype source _raw 
| sort - _time 
| head 50
```

### Alert Configuration Parameters

| Parameter | Configuration Setting |
| :--- | :--- |
| **Title** | `SSH Bruteforce` |
| **Permissions** | Shared in App |
| **Alert Type** | `Real-time` |
| **Expires** | `5 minute(s)` |
| **Trigger Conditions** | `Number of Results` is `greater than 0` in `1 minute(s)` |
| **Trigger Mode** | `For each result` |
| **Throttle** | Enabled (Suppress triggering for `60` seconds) |
| **Trigger Actions** | Add to Triggered Alerts (Severity: **High**) |

---

## ⚔️ Attack Simulation & Alert Verification

### 1. Launch SSH Brute-Force Attack (From Kali Linux)
Execute Hydra against the target SSH service to generate authentication failure logs:

```bash
hydra -l <TARGET_USER> -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_HOST_IP>
```

### 2. Verify Real-Time Detection in Splunk SIEM
1. In Splunk Web UI, navigate to **Activity** > **Triggered Alerts**.
2. Observe real-time alerts generated for `SSH Bruteforce` with **High Severity**.
3. Click **View Results** to inspect raw payloads, timestamps, source IPs, and detailed event metadata.

---

## 🔧 Troubleshooting & Handy Commands

| Action | Command | Target System |
| :--- | :--- | :--- |
| **Check SIEM Status** | `sudo /opt/splunk/bin/splunk status` | SIEM Server (`<SIEM_SERVER_IP>`) |
| **Restart SIEM Service** | `sudo /opt/splunk/bin/splunk restart` | SIEM Server (`<SIEM_SERVER_IP>`) |
| **Check Forwarder Status** | `sudo /opt/splunkforwarder/bin/splunk status` | Victim Host (`<TARGET_HOST_IP>`) |
| **Restart Forwarder** | `sudo /opt/splunkforwarder/bin/splunk restart` | Victim Host (`<TARGET_HOST_IP>`) |
| **Verify Forward Targets** | `sudo /opt/splunkforwarder/bin/splunk list forward-server` | Victim Host (`<TARGET_HOST_IP>`) |
| **Verify Log Inputs** | `sudo /opt/splunkforwarder/bin/splunk list monitor` | Victim Host (`<TARGET_HOST_IP>`) |

[📸 Hydra Attack Screenshot](https://github.com/Avinash-4693/Splunk-SIEM-Deployment-Incident-Detection/blob/main/Screenshot%202026-08-10%20171025.png)
[📸 Attack Triggered on splunk Screenshot](https://github.com/Avinash-4693/Splunk-SIEM-Deployment-Incident-Detection/blob/main/Screenshot%202026-08-10%20171041.png)
