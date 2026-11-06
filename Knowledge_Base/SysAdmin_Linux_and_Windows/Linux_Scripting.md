# Bash Scripting Best Practices and Modular Functions
## Reusable Functions and Privilege Validation

Standardized logging functions for script output. `log_info` prints informative messages in green to track script progress. `log_error` prints error messages in red to the standard error stream and immediately exits the script with an error code.
```bash
log_info() { 
    echo -e "\n\e[32m[INFO]\e[0m $1" 
} 

log_error() { 
    echo -e "\n\e[31m[ERROR]\e[0m $1" >&2 
    exit 1 
}
```


A modular function to check for root privileges. It checks if the Effective User ID (EUID) is 0 and utilizes the custom `log_error` function to standardize the error output if the user lacks permissions.
```bash
verify_root() {
	if [[ "${EUID}" -ne 0]]; then
		log_error "This script must be run as root (use sudo)."
	fi
}
```


A modular function that creates a timestamped backup of a specified configuration file before it gets modified. It uses `cp -a` to preserve original file attributes (permissions, ownership) and logs the successful creation using the `log_info` function.
```bash
backup_file() {
    # PURPOSE: Ensures configuration resilience. Before injecting 
    # new parameters, the previous state is preserved to facilitate a rollback.
    local FILE=$1
    if [ -f "$FILE" ]; then
        local BACKUP_NAME
        BACKUP_NAME="${FILE}.bak_$(date +%F)"
        cp -a "$FILE" "$BACKUP_NAME"
        log_info "[Backup] Created: $BACKUP_NAME"
    fi
}
```


## Secure User Input (`read`)

To request data from the user in the terminal and store it in variables safely, it is best practice to use the `read` command with the `-r` and `-p` flags.

```bash
read -r -p "Domain Name (e.g., lab.local): " DOMAIN
```
(Note: The `-p` flag must be immediately followed by the text inside quotes).

**Parameter Breakdown:**

- **`read`**: A Bash internal command that reads a line from standard input (user keystrokes) and assigns that value to the specified variable (e.g., `DOMAIN`).

- **`-r` (Raw mode)**: Instructs Bash to treat the user's input entirely literally, preventing backslashes (`\`) from being interpreted as special escape characters.

- **`-p` (Prompt)**: Displays an explanatory message on the screen waiting for the response on the same line, saving the need to write an `echo` command before `read`.

## Using `EOF` (Here Documents) in Scripting

The `EOF` operator (End of File delimiter, commonly used with `cat`) allows sending multiple lines of text directly to a file or to the standard input of a command without needing to open an interactive editor like `nano` or `vim`. It is essential in automation scripts for generating configuration files, JSONs, or temporary scripts.
### Example: Creating a structured file (JSON)
```bash
cat <<EOF > request.json
{
  "user": "administrator",
  "permissions": "total",
  "status": "active"
}
EOF
```

## Script Diagnostics and Debugging

Read a Bash script to check for syntax errors without actually executing the code.
```bash
bash -n script.sh
```

Execute a Bash script in debug mode, printing each command to the terminal before execution to facilitate troubleshooting.
```bash
bash -x script.sh
```

Display the `$PATH` environment variable, replacing colon delimiters with newline characters to list each directory on an individual line.
```bash
echo $PATH | tr : '\n'
```

