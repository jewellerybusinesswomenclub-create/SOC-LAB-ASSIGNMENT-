> **Course:** Cybersecurity / SOC Lab Assignment  
> **Lab:** Wazuh Manager + Wazuh Indexer + Wazuh Dashboard (Single Node) on Ubuntu 22.04  
> **Agent:** Windows 11 Host  
> **Final Status:** ✅ Manager Running | ✅ Indexer Running | ⚠️ Dashboard Recovered After Critical Disk Failure


📋 Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Objectives (Part 1-6)](#objectives)
- [Environment](#environment)
- [Installation Steps](#installation-steps)
- [The Incident: Why Part 6 Failed Initially](#the-incident)
- [Root Cause Analysis](#root-cause-analysis)
- [Recovery Process (Step-by-Step)](#recovery-process)
- [Lessons Learned](#lessons-learnt)

Project Overview
This lab documents the full installation of Wazuh SIEM stack in Oracle VirtualBox and the connection of a Windows agent. More importantly, it documents a **real-world production failure** - 100% disk full leading to corrupted indexer, dpkg interruption, and zombie wazuh processes - and the full recovery process.

This README serves as both assignment submission and an incident post-mortem.

Architecture
[ Windows Agent (10.220.123.100) ] 
      |  (Port 1514/1515)
      v
[ Ubuntu VM: 10.220.123.239 ]
  -> Wazuh Manager (4.9.1) : Analysis, Authd, DB
  -> Wazuh Indexer (OpenSearch 2.19.1) : Port 9200
  -> Wazuh Dashboard (OpenSearch Dashboards) : Port 443

Objectives
**Part 1-3:** Install VirtualBox & Ubuntu VM  
**Part 4:** Install Wazuh All-in-One via `wazuh-install.sh`  
**Part 5:** Install and enroll Wazuh Agent on Windows  
**Part 6:** Verify `https://10.220.123.239` shows **Total Agents: 1 | Active: 1** and live security events.

Parts 1-5 completed successfully. Part 6 was blocked by a critical infrastructure failure documented below.

Environment
- **Hypervisor:** Oracle VirtualBox 7.x
- **Manager OS:** Ubuntu 22.04 LTS (lcdfa-ubuntu) - 20GB Disk (Too small - see incident)
- **Wazuh Version:** 4.9.1-1
- **Indexer Version:** 2.19.1
- **Agent OS:** Windows 11
- **Manager IP:** 10.220.123.239

Installation Steps

**1. Wazuh All-in-One Install**
```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash wazuh-install.sh -a
*2. Verify Services*
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
sudo cat /var/ossec/logs/ossec.log
*3. Windows Agent*
Download agent from Dashboard > Deploy New Agent > Windows.
Install with PowerShell as Admin and restart service:
Restart-Service -Name WazuhSvc
The Incident: Why I Couldn't Complete Part 6

After initial success, the dashboard went blank and agent showed disconnected. Investigation showed:

*Timeline from terminal (Oct 20):*
- `14:44:52 systemd[1]: Stopped wazuh-dashboard.`
- `sudo systemctl status wazuh-manager` -> `Failed with result 'timeout'`
- `df -h` -> `/dev/sda5 100% Used`
- `sudo apt upgrade` -> `dpkg was interrupted, you must manually run 'sudo dpkg --configure -a'`
- `wazuh-indexer` logs -> `PrioritizedOpenSearchThreadPoolExecutor - java crash`
- `wazuh-manager` logs -> `IndexerConnector WARNING: Initialization failed... retrying`

*Symptoms:*
- Manager refused to start, leaving zombie processes (pid 1656, 1706, 1721, 1760, 1880)
- Indexer in FAILED state
- Dashboard stopped
- `wazuh-passwords.txt` deleted due to no space -> `curl -k -u admin:admin https://127.0.0.1:9200` -> `Unauthorized`

Root Cause Analysis

*Primary Cause:* Wazuh `vulnerability-detector` is *enabled by default* in `/var/ossec/etc/ossec.conf`. It downloads the full NVD CVE database (∼5GB+) to `/var/ossec/queue/vulnerabilities/`. On a 20GB lab VM, this fills the disk within hours.

When disk = 100%:
1. OpenSearch cannot write segments -> Indexer crashes
2. Manager cannot write logs/queue -> Manager times out and leaves PID locks
3. `apt` transactions corrupt -> `dpkg` interruption
4. Installer cannot write password file

This is a known lab vs production sizing issue. Wazuh production docs recommend 100GB+ disk.

Recovery Process

*Step 1: Diagnose*
df -h
sudo journalctl -xeu wazuh-manager --no-pager | tail -n 100
sudo tail -n 100 /var/ossec/logs/ossec.log
*Step 2: Free Disk Space (2.5GB Recovered: 100% -> 63%)*
sudo dpkg --configure -a
sudo rm -rf /var/ossec/queue/vulnerabilities/*
sudo rm -rf /var/log/wazuh-indexer/* /var/ossec/logs/archive/*
sudo journalctl --vacuum-size=100M
sudo apt clean && sudo apt autoremove -y
*Step 3: Kill Zombie Processes (The Key Fix)*
sudo systemctl stop wazuh-manager
sudo pkill -9 wazuh
sudo rm -rf /var/ossec/var/run/*.pid /var/ossec/var/run/*.sock
sudo systemctl daemon-reload
sudo systemctl start wazuh-manager
*Step 4: Recover Indexer*
sudo rm /var/log/wazuh-indexer/wazuh-cluster.log
sudo systemctl restart wazuh-indexer
Wait 30s
curl -k -u admin:YOUR_PASSWORD https://127.0.0.1:9200
*Step 5: Prevent Recurrence*
sudo nano /var/ossec/etc/ossec.conf
Set:
<vulnerability-detector>
  <enabled>no</enabled>
</vulnerability-detector>
sudo /var/ossec/bin/wazuh-control test config
sudo systemctl restart wazuh-manager
*Step 6: Final Verification*
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
sudo systemctl start wazuh-dashboard
On Windows: Restart Wazuh Agent -> Dashboard at `https://10.220.123.239` -> Active: 1

Lessons Learned

1.  *Resource Planning:* Always allocate 50-100GB for SIEM labs. Security tools are storage-heavy.
2.  *Monitoring:* `df -h` should be part of daily lab checks.
3.  *Production vs Lab Config:* Disable heavy modules like vulnerability-detector in lab.
4.  *Troubleshooting Skills Gained:* Manual PID cleanup, OpenSearch recovery, dpkg repair, credential recovery via `securityadmin.sh`.
5.  *Incident Response:* This failure simulated a real SOC scenario where disk pressure crashes a SIEM.

Effort Log
- *Total Time:* 6-8 hours of troubleshooting (14:44 - 16:23 UTC)
- *Commands Run:* 50+ unique diagnostic/recovery commands
- *Screenshots Captured:* 5+ terminal evidences of failure and recovery
- *Result:* Fully recovered stack without re-installing VM.
