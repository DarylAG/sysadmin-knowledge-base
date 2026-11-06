Technical documentation, essential commands, and procedures for the administration of Linux-based operating systems.

## Directory and File Management
```bash
# Print the current working directory
pwd

# List all files, including hidden ones, in a detailed long format
ls -la

# Create nested directories, ensuring parent directores are created without errors if they already exist
mkdir -p /path/to/parent/child/directories

# Copy files and directories recursively while displaying detailed operation output (verbose mode)
cp -rv /source/file_or_directory /destination/

# Move or rename files and directories within the file system
mv old_file.txt new_file.txt

# Remove files or directories recursively and forcefully
rm -rf /path/to/remove
```

## User and Permission Management

```bash
# Create a new user, assigning a home directory and setting Bash as the default shell
sudo useradd -m -s /bin/bash user_name

# Set or change the password for a specific user
sudo passwd user_name

# Add an existing user to a supplementary group, such as the sudo group for administrative privileges
sudo usermod -aG sudo user_name

# Recursively modify the access permissions of a file or directory
chmod -R 755 /path/to/directory

# Recursively change the ownership(user and group) of a resource
sudo chown -R user_name:group_name /path/to/resource
```

## SSH and Key-based Authentication Management

### Key Pair Generation
- **Option A (No password / No passphrase):** Ideal for automation or scripts where manual intervention is not required. When prompted for the password, simply press Enter twice to leave it empty.
```Bash
ssh-keygen -t rsa -b 4096
```

- **Option B (Encrypted with password / With passphrase):** Safer for production environments. It protects the private key on the disk with an additional password that will be requested when using it (or when loading it into the SSH agent)
```Bash
ssh-keygen -t rsa -b 4096 -N "your_secure_password" -f ~/.ssh/id_rsa_encrypted
```

### Copy the Key to the Remote Server
```Bash
ssh-copy-id user@server_ip_or_host
```

## Processes and Services (Daemons)
```Bash
# List all active processes with detailed information
ps aux

# Monitor processes and system resource consumption in real-time
top
# (A more modern and visual alternative: htop)

# Gracefully terminates a process by its ID (PID), allowing it to close down cleanly
kill 1234

# Forcefully terminate a process by its ID (PID) if it becomes unresponsive
kill -9 1234

# Manage system daemons (systemd)
sudo systemctl start service_name
sudo systemctl stop service_name
sudo systemctl restart service_name
sudo systemctl enable --now service_name
sudo systemctl status service_name
```

## System Information and Hardware Auditing

```bash
# Display system architecture, kernel version, and hostname
uname -a

# Show detailed information about the Linux distribution and OS release
cat /etc/os-release

# Display CPU architecture, core count, and detailed processor information
lscpu

# Show total, used, and free memory (RAM and Swap) in human-readable format
free -h

# Display file system disk space usage across all mounted partitions in human-readable format
df -h

# List all PCI buses and connected devices (e.g., VGA, Network controllers)
lspci

# List USB buses and attached devices
lsusb

# Display a compact and detailed summary of the entire hardware configuration
sudo lshw -short
```
## Networking and Connectivity

### NetworkManager (`nmcli`)
```Bash
# List available network interfaces and their status
nmcli device status

# List configured network connections
nmcli connection show

# Bring up or take down a specific network interface/connection
nmcli connection up "Connection_name"
nmcli connection down "Connection_name"

# Configure a static IP on a connection
nmcli connection modify "Connection_Name" ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns 8.8.8.8 ipv4.method manual
```

### systemd-networkd
```Bash
# Verify the network service status
sudo systemctl status systemd-networkd

# Network configuracion files are located in /etc/systemd/network/
# Example of reloading the configuration
sudo networkctl reload
```

### Quick Bluetooth Configuration via Terminal
```Bash
# Verify the bluetooth service status
systemctl status bluetooth

# Enter the interactive bluetooth control interface
bluetoothctl

# (Once inside the bluetoothctl console, execute the following commands):
power on
agent on
default-agent
scan on
pair XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
exit
```

## Drive Mounting and Unmounting
```Bash
# List disks and partitions connected to the system
lsblk
sudo fdisk -l

# Mount a partition or drive to a target directory
sudo mount /dev/sdb1 /mnt/mount_point

# Mount a drive explicitly indicating the file system type
sudo mount -t ext4 /dev/sdb1 /mnt/mount_point

# Safely unmount a drive
sudo umount /mnt/punto_montaje

# Forceful unmount (in case the resource is busy)
sudo umount -f /mnt/mount_point
```

## Package Compression and Decompression
```Bash
# Compress a directory into a tar.gz
tar -czvf compressed_file.tar.gz /path/to/directory/

# Decompress a tar.gz file in the current directory
tar -xzvf compress_file.tar.gz

# Compress in zip format
zip -r file.zip /path/to/directory/

# Decompress a zip file
unzip file.zip
```

## Package Management (APT - Debian/Ubuntu/Mint)
```Bash
# Update the local repository list
sudo apt update

# Upgrade the local repository list
sudo apt upgrade -y

# Install a new application or tool
sudo apt install package_name -y

#Uninstall a package, either keeping or purging its configuration files
sudo apt remove package_name
sudo apt purge package_name

# Clean orphaned packages and unnecessary cache
sudo apt autoremove -y && sudo apt clean
```

## Replace the existing global mirrors 
Change the default Debian mirror repositories to the Universidad de Costa Rica (UCR) mirrors. This is required to workaround when operating within the INA network, where global mirrors are often blocked by institutional firewalls. The UCR mirrors are whitelisted in INA's systems, preventing connection timeouts or blocks when trying to update the OS or download packages.

```bash
nano /etc/apt/sources.list
```

```text
deb https://espejos.ucr.ac.cr/debian trixie main contrib non-free non-free-firmware
deb https://espejos.ucr.ac.cr/debian trixie-updates main contrib non-free non-free-firmware
```

## Text Filtering and Manipulation
```Bash
# grep: Search for patterns or keyworks within files or command outputs
cat /etc/passwd | grep "root"

# sed: Replace text on the fly (e.g., change "hello" to "goodbye" in a file)
sed -i 's/hello/goodbye/g' file.txt

# awk: Process columns or delimited fields (e.g., print the first and ninth columns)
ls -l | awk '{print $1, $9}'

# nl: Number the lines of a file or standard output
cat file.txt | nl

# wc: Count lines, words, and characters (Word Count)
wc -l file.txt

# Extract specific sections, columns, or fields from each line of a file using a delimiter
cut -d':' -f1 /etc/passwd

# Translate, squeeze, or delete characters (e.g., convert lowercase to uppercase)
cat file.txt | tr 'a-z' 'A-Z'

```

## Diagnostic
Determine the file type and architecture of the executable binary (in this case, rm).
```bash
file /bin/rm
```



