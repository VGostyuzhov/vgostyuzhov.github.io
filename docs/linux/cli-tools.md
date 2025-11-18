# Linux CLI Security Tools

Essential command-line tools for security analysis, monitoring, and incident response on Linux systems.

## Table of Contents

## System Information & Monitoring

Critical for understanding system state, identifying anomalies, and incident response.

### System Information

**Basic System Details**
```bash
# System and kernel information
uname -a                    # All system info
uname -r                    # Kernel version
hostnamectl                 # Hostname and system info
lscpu                       # CPU architecture details
lsmem                       # Memory information
```

**Reading uname output:**
```
Linux hostname 5.15.0-56-generic #62-Ubuntu SMP x86_64 GNU/Linux
 │       │            │                     │      │     │
 │       │            │                     │      │     └── OS type
 │       │            │                     │      └── Hardware platform
 │       │            │                     └── Processor type
 │       │            └── Kernel version
 │       └── Hostname
 └── Kernel name
```

**Reading lscpu key fields:**
- **Architecture**: x86_64, aarch64, etc.
- **CPU(s)**: Total number of logical CPUs
- **Core(s) per socket**: Physical cores per CPU
- **Socket(s)**: Number of physical CPU packages
- **Flags**: CPU features (security: smep, smap, nx, pae)

**Hardware Information**
```bash
# Detailed hardware info (requires root)
dmidecode -t system         # System manufacturer, model
dmidecode -t processor      # CPU details
dmidecode -t memory         # RAM information
lshw -short                 # Hardware summary
lspci                       # PCI devices
lsusb                       # USB devices
```

**User Activity Tracking**
```bash
# Current and recent user activity
who                         # Currently logged in users
w                          # Detailed user activity
last                       # Login history
lastlog                    # Last login for each user
uptime                     # System uptime and load
```

**Reading who/w output:**
```
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU  WHAT
john     pts/0    192.168.1.100    09:30    0.00s  0.04s  0.00s  w
```
- **TTY**: Terminal type (pts = pseudo-terminal, tty = physical)
- **FROM**: Source IP/hostname
- **IDLE**: Time since last activity
- **JCPU**: Time used by all processes attached to tty
- **PCPU**: Time used by current process

**Reading last output:**
```
john     pts/0        192.168.1.100    Mon Dec 11 09:30   still logged in
```
- Shows login/logout times, session duration
- Look for: unusual login times, unknown IPs, multiple concurrent sessions

**Reading uptime output:**
```
09:45:23 up 5 days, 3:15, 2 users, load average: 0.15, 0.10, 0.05
    │        │              │                │
    │        │              │                └── Load averages (1, 5, 15 min)
    │        │              └── Number of users
    │        └── System uptime
    └── Current time
```
- **Load average**: CPU usage (>1.0 means overloaded on single-core systems)

### Process Monitoring

**Process Listing and Analysis**
```bash
# Basic process information
ps aux                     # All processes with details
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu
pstree                     # Process tree view
pstree -p                  # Include PIDs

# Find specific processes
pidof process_name         # Get PID by name
pgrep -f pattern          # Find PIDs by pattern
pkill -f pattern          # Kill processes by pattern
```

**Reading ps aux output:**
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  22568  1064 ?        Ss   Dec06   0:01 /sbin/init
```
- **PID**: Process ID
- **%CPU**: CPU usage percentage
- **%MEM**: Memory usage percentage
- **VSZ**: Virtual memory size (KB)
- **RSS**: Resident memory size (KB)
- **TTY**: Controlling terminal (? = no terminal)
- **STAT**: Process state
  - **R**: Running
  - **S**: Sleeping (interruptible)
  - **D**: Sleeping (uninterruptible, usually I/O)
  - **Z**: Zombie process
  - **T**: Stopped
  - **s**: Session leader
  - **+**: Foreground process group

**Security-relevant STAT codes:**
- **Z**: Zombie processes (potential DoS or bug)
- **D**: Uninterruptible sleep (possible I/O issues)
- **<**: High priority (negative nice value)
- **N**: Low priority (positive nice value)

**Advanced Process Monitoring**
```bash
# Real-time process monitoring
top                        # Classic process monitor
htop                       # Enhanced interactive monitor
iotop                      # I/O monitoring (requires root)
```

**File and Network Usage**
```bash
# Open files and network connections
lsof                       # List all open files
lsof -i                    # Network connections only
lsof -p PID               # Files opened by specific PID
lsof /path/to/file        # Processes using specific file
fuser -v /path/to/file    # Processes accessing file
```

**Reading lsof output:**
```
COMMAND   PID   USER   FD   TYPE DEVICE  SIZE/OFF    NODE NAME
ssh      1234   john    3u  IPv4  12345      0t0     TCP *:22 (LISTEN)
```
- **COMMAND**: Process name
- **PID**: Process ID
- **USER**: Process owner
- **FD**: File descriptor
  - **0**: stdin, **1**: stdout, **2**: stderr
  - **u**: read/write, **r**: read-only, **w**: write-only
- **TYPE**: File type (REG=regular, DIR=directory, IPv4/IPv6=network)
- **NODE**: Inode number or protocol info

**Reading lsof -i (network) output:**
```
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd    1234 root    3u  IPv4  56789      0t0  TCP *:22 (LISTEN)
```
- **NAME format**: `protocol:port` or `local_ip:port->remote_ip:port`
- **States**: LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT

**System Call Tracing**
```bash
# Debug and trace system calls
strace -p PID             # Trace system calls of running process
strace -o trace.log cmd   # Save trace to file
ltrace -p PID             # Trace library calls
```

### Resource Monitoring

**Memory Analysis**
```bash
# Memory usage information
free -h                    # Human-readable memory info
cat /proc/meminfo         # Detailed memory statistics
vmstat 1                  # Virtual memory statistics (1-second interval)
smem -p                   # Per-process memory usage
```

**Reading free output:**
```
              total        used        free      shared  buff/cache   available
Mem:          7.7Gi       2.1Gi       1.2Gi       654Mi       4.4Gi       4.7Gi
Swap:         2.0Gi          0B       2.0Gi
```
- **total**: Total installed memory
- **used**: Memory currently in use
- **free**: Completely unused memory
- **shared**: Memory shared between processes
- **buff/cache**: Buffers + cached data (can be freed if needed)
- **available**: Memory available for new processes

**Reading vmstat output:**
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 1234567  87654 345678    0    0    12    34  567  890 15  5 78  2  0
```
- **r**: Runnable processes
- **b**: Blocked processes (I/O wait)
- **swpd**: Swap memory used
- **si/so**: Swap in/out per second
- **bi/bo**: Blocks in/out per second (disk I/O)
- **in**: Interrupts per second
- **cs**: Context switches per second
- **us/sy/id/wa**: CPU time percentages (user/system/idle/wait)

**Storage and Filesystem**
```bash
# Disk usage and filesystem info
df -h                      # Filesystem usage
du -sh /path/*            # Directory sizes
lsblk                     # Block device tree
fdisk -l                  # Partition information (requires root)
mount | column -t         # Mounted filesystems
findmnt                   # Filesystem tree
```

**Reading df output:**
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        20G  15G  4.2G  79% /
/dev/sda2       100G  45G   50G  48% /home
```
- **Size**: Total filesystem size
- **Used**: Space currently used
- **Avail**: Available space for non-root users
- **Use%**: Percentage used (>90% is concerning)
- **Mounted on**: Mount point

**Reading lsblk output:**
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   20G  0 disk
├─sda1   8:1    0   19G  0 part /
└─sda2   8:2    0    1G  0 part [SWAP]
```
- **NAME**: Device name
- **MAJ:MIN**: Major:Minor device numbers
- **RM**: Removable device (1=yes, 0=no)
- **SIZE**: Device size
- **RO**: Read-only (1=yes, 0=no)
- **TYPE**: Device type (disk, part, lvm, etc.)

**Security considerations:**
- Watch for unexpected mounted filesystems
- Monitor disk usage spikes (potential log flooding)
- Check for unusual device types or sizes

**System Performance**
```bash
# Performance monitoring
iostat -x 1               # I/O statistics
sar -u 1 10              # CPU utilization (10 samples)
sar -r 1 10              # Memory utilization
sar -d 1 10              # Disk activity
```

**Kernel and Module Information**
```bash
# Kernel modules and parameters
lsmod                     # Loaded kernel modules
modinfo module_name       # Module information
sysctl -a                 # Kernel parameters
cat /proc/version         # Kernel version details
dmesg | tail             # Recent kernel messages
```

### Security-Focused Monitoring

**Suspicious Process Detection**
```bash
# Find processes running as different users
ps -eo pid,ppid,user,cmd | grep -v "^$USER"

# Processes without controlling terminal
ps -eo pid,ppid,tty,cmd | grep "?"

# Network connections by process
netstat -tulpn | grep LISTEN
ss -tulpn | grep LISTEN
```

**Resource Exhaustion Detection**
```bash
# Top CPU consumers
ps -eo pid,ppid,cmd,%cpu --sort=-%cpu | head -10

# Top memory consumers
ps -eo pid,ppid,cmd,%mem --sort=-%mem | head -10

# Processes with many open files
lsof | awk '{print $2}' | sort | uniq -c | sort -rn | head -10
```

### Cheat Sheet

| Command | Purpose | Key Flags |
|---------|---------|-----------|
| `ps aux` | List all processes | `-e` (all), `-f` (full), `-o` (format) |
| `lsof -i` | Network connections | `-i` (internet), `-p PID`, `-u user` |
| `strace -p PID` | Trace system calls | `-p` (attach to PID), `-o` (output file) |
| `sar -u 1 10` | CPU monitoring | `-u` (CPU), `-r` (memory), `-d` (disk) |
| `vmstat 1` | Virtual memory stats | interval in seconds |
| `iotop` | I/O monitoring | `-o` (only active), `-a` (accumulated) |

!!! tip "Security Monitoring Tips"
    - Monitor for processes running as unexpected users
    - Check for processes without controlling terminals (potential backdoors)
    - Watch for unusual network connections and high resource usage
    - Use `watch` command to continuously monitor: `watch -n 1 'ps aux --sort=-%cpu | head -10'`

### Network Security Tools
- **Network Information**
  - `ip`, `ifconfig`, `route`, `netstat`, `ss`
  - `arp`, `iwconfig`, `ethtool`
- **Network Analysis**
  - `tcpdump`, `wireshark`, `tshark`
  - `nmap`, `masscan`, `zmap`
  - `netcat` (`nc`), `socat`, `ncat`
- **Network Testing**
  - `ping`, `traceroute`, `mtr`
  - `telnet`, `curl`, `wget`
  - `dig`, `nslookup`, `host`

### File System & Permissions
- **File Operations**
  - `find`, `locate`, `which`, `whereis`
  - `ls`, `stat`, `file`, `strings`
  - `chmod`, `chown`, `chgrp`, `umask`
- **File Analysis**
  - `md5sum`, `sha256sum`, `xxd`, `hexdump`
  - `diff`, `cmp`, `grep`, `awk`, `sed`
  - `sort`, `uniq`, `cut`, `tr`
- **Archive & Compression**
  - `tar`, `gzip`, `zip`, `7z`
  - `rsync`, `scp`, `sftp`

### Security-Specific Tools
- **Access Control**
  - `sudo`, `su`, `runuser`
  - `passwd`, `chage`, `usermod`
  - `visudo`, `gpasswd`
- **Process Isolation**
  - `chroot`, `unshare`, `nsenter`
  - `systemd-nspawn`, `firejail`
- **Cryptography**
  - `openssl`, `gpg`, `ssh-keygen`
  - `base64`, `uuencode`, `uudecode`

### Logging & Auditing
- **System Logs**
  - `journalctl`, `dmesg`, `logger`
  - `logrotate`, `rsyslog` configuration
- **Audit Framework**
  - `auditctl`, `ausearch`, `aureport`
  - `aulast`, `aulastlog`, `auvirt`
- **Log Analysis**
  - `tail`, `head`, `less`, `more`
  - `zcat`, `zgrep`, `zless`

### Forensics & Incident Response
- **Memory Analysis**
  - `pmap`, `smem`, `/proc` filesystem
  - `gcore`, `crash`, `volatility`
- **Disk Forensics**
  - `dd`, `dcfldd`, `ddrescue`
  - `fsck`, `debugfs`, `testdisk`
- **Data Recovery**
  - `extundelete`, `photorec`, `scalpel`
  - `sleuthkit` tools (`fls`, `icat`, `mmls`)

### Malware Detection & Analysis
- **File Scanning**
  - `clamscan`, `rkhunter`, `chkrootkit`
  - `lynis`, `tiger`, `aide`
- **Behavioral Analysis**
  - `perf`, `ftrace`, `bpftrace`
  - Dynamic analysis with `ltrace`/`strace`

### Network Security & Firewalls
- **Firewall Management**
  - `iptables`, `ip6tables`, `ebtables`
  - `ufw`, `firewalld`, `nftables`
- **VPN & Tunneling**
  - `openvpn`, `strongswan`, `wireguard`
  - `ssh` tunneling, `stunnel`

### Text Processing & Analysis
- **Pattern Matching**
  - `grep`, `egrep`
  - Regular expressions, `regex` patterns
- **Data Manipulation**
  - `jq` (JSON processing)
  - `xmlstarlet` (XML processing)
  - `csvkit` (CSV processing)

### Package Management Security
- **Package Verification**
  - `rpm`, `dpkg`, `apt`, `yum`
  - Package signing and verification methods
- **Software Installation**
  - `snap`, `flatpak`, `appimage`
  - Containerized applications

### Performance & Debugging
- **Performance Profiling**
  - `time`, `timeout`, `nice`, `ionice`
  - `cpulimit`, `ulimit`, `prlimit`
- **Debugging Tools**
  - `valgrind`, `gprof`, `objdump`
  - `readelf`, `nm`, `ldd`

### Automation & Scripting
- **Shell Scripting Security**
  - Input validation, sanitization
  - Secure coding practices
- **Task Automation**
  - `cron`, `at`, `systemd` timers
  - `watch`, `timeout` utilities

!!! info "Prerequisites"
    For comprehensive Linux CLI reference, see:
    - [Linux Command Line Basics](https://linuxcommand.org/)
    - [SANS Linux CLI Cheat Sheet](https://www.sans.org/posters/linux-shell-survival-guide/)
    - [ExplainShell](https://explainshell.com/) for command breakdowns