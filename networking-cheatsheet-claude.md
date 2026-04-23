# Linux Networking — Cheat Sheet
> Generated from course notes | April 2026

---

## Quick Reference Table

| Command | Description |
|---------|-------------|
| `ip link` | Show network interfaces |
| `ip addr` / `ip a` | Show IP addresses on interfaces |
| `ip -c addr` | Show IPs with color output |
| `ip addr add` | Add IP address to interface (temporary) |
| `ip addr delete` | Remove IP address from interface |
| `ip addr show dev <if>` | Show specific interface details |
| `ip link set dev <if> up/down` | Bring interface UP or DOWN |
| `ip route` | Show routing table |
| `ss -ltunp` | Show listening ports + processes |
| `netstat -tulpn` | Show listening ports (alternative) |
| `lsof -p <PID>` | Show files opened by a process |
| `ps <PID>` | Show process info by PID |
| `ping` | Test connectivity to host |
| `resolvectl status` | Show DNS resolver status |
| `resolvectl dns` | Show only DNS servers |
| `netplan get` | Show current Netplan config tree |
| `netplan try` | Test config with auto-revert |
| `netplan apply` | Apply Netplan config immediately |
| `networkctl reload` | Reload network config |
| `networkctl reconfigure <if>` | Reconfigure specific interface |
| `systemctl status <service>` | Check service status |
| `systemctl start/stop/enable/disable` | Manage services |

---

## 🌐 Interface Management — `ip` command

---

### `ip link`
**What it does:** Shows all network interfaces (cards) and their state (UP/DOWN).

**Syntax:**
```bash
ip link [COMMAND] [OPTIONS]
```

**Examples:**
```bash
# List all network interfaces
ip link

# Bring interface UP
sudo ip link set dev enp0s8 up

# Bring interface DOWN
sudo ip link set dev enp0s8 down
```

**Quick tip:** Always run `ip link` first to get the exact interface names — they vary by system (eth0, enp0s3, enp6s0, etc.).

---

### `ip addr` / `ip address`
**What it does:** Shows IP addresses (IPv4 and IPv6) configured on all interfaces.

**Syntax:**
```bash
ip addr [show] [dev INTERFACE]
ip a          # short alias
ip -c addr    # colored output
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-c` | Colorize output (easier to read) |
| `show dev <if>` | Show only specific interface |

**Examples:**
```bash
# Show all IPs on all interfaces
ip addr

# Short alias — same result
ip a

# Colored output (recommended)
ip -c addr

# Show only eth0 interface
ip addr show dev eth0

# Extract just the IP address of eth0 (pipe trick)
ip addr show dev eth0 | grep inet | awk '{print $2}'

# Extract IP without subnet mask
ip addr show dev eth0 | grep inet | awk '{print $2}' | cut -d'/' -f1
```

**Quick tip:** `ip -c addr` is much easier to read at a glance — the color highlights UP/DOWN state and IP addresses.

---

### `ip addr add` / `ip addr delete`
**What it does:** Adds or removes an IP address from an interface. **Temporary — lost after reboot.**

**Syntax:**
```bash
sudo ip addr add <IP/PREFIX> dev <INTERFACE>
sudo ip addr delete <IP/PREFIX> dev <INTERFACE>
```

**Examples:**
```bash
# Add IPv4 address to interface
sudo ip addr add 10.0.0.40/24 dev enp0s8

# Add IPv6 address
sudo ip addr add fe80::5054:ff:fe1f:8050/64 dev enp0s8

# Add IP to eth1 (even if DOWN)
sudo ip addr add 192.168.9.3/24 dev eth1

# Remove IPv4 address
sudo ip addr delete 10.0.0.5/24 dev enp0s8

# Remove IPv6 address
sudo ip addr delete fe80::5054:ff:fe1f:8050/64 dev enp0s8
```

**Quick tip:** ⚠️ These changes are **temporary** — they disappear on reboot. Use Netplan for permanent configuration.

---

### `ip route`
**What it does:** Shows the routing table — how traffic is directed to different networks.

**Syntax:**
```bash
ip route
ip route | grep default   # show default gateway only
```

**Examples:**
```bash
# Show full routing table
ip route

# Save routing table to file
ip route > /home/bob/route.txt

# Extract only the default gateway IP
ip route | grep default | awk '{print $3}'

# Extract all IPs using regex
ip route | egrep -o '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'
```

**Reading the output:**
```
default via 10.0.0.1 dev enp0s8 proto static      ← default gateway (internet)
192.168.0.0/24 via 10.0.0.100 dev enp0s8 proto static  ← static route
```

| Destination | Via (gateway) | Meaning |
|-------------|--------------|---------|
| `default` | `10.0.0.1` | All internet traffic → 10.0.0.1 |
| `192.168.0.0/24` | `10.0.0.100` | That subnet → 10.0.0.100 |

**Quick tip:** `ip route` does NOT show DNS config — use `resolvectl status` for that.

---

## 🧪 Lab: Interface Management

**Objective:** Manage network interfaces and IP addresses using the `ip` command.

**Tasks:**
1. List all network interfaces and identify which are UP
2. Add a temporary IP `192.168.50.10/24` to `eth1`
3. Verify the IP was added with `ip -c addr`
4. Show only the `eth0` interface and save its IP to `/home/bob/myip.txt`
5. Remove the IP you added from `eth1`
6. *(Challenge)* Extract only the IPv4 address of eth0 (no subnet, no IPv6) and save to a file

**Solution:**
```bash
# Task 1
ip link

# Task 2
sudo ip addr add 192.168.50.10/24 dev eth1

# Task 3
ip -c addr

# Task 4
ip addr show dev eth0 > /home/bob/myip.txt

# Task 5
sudo ip addr delete 192.168.50.10/24 dev eth1

# Challenge
ip addr show dev eth0 | grep 'inet ' | awk '{print $2}' | cut -d'/' -f1 > /home/bob/myip.txt
```

---

## 📋 Netplan — Permanent Network Configuration

Netplan is Ubuntu's network configuration manager. Config files live in `/etc/netplan/` and are written in YAML format.

---

### `netplan get`
**What it does:** Shows the current Netplan configuration as a tree.

```bash
sudo netplan get
```

---

### `netplan try`
**What it does:** Tests new configuration with **automatic revert** if not confirmed. Safety net for remote servers.

**Syntax:**
```bash
sudo netplan try
sudo netplan try --timeout 30   # revert after 30 seconds instead of default 120
```

**How it works:**
- Applies new config immediately
- Waits for you to press **Enter** to confirm
- If you don't confirm (or get disconnected), it **reverts automatically** after timeout
- Press **Ctrl+C** to cancel and revert immediately

**Quick tip:** Always use `netplan try` on remote servers — if you make a mistake you won't lock yourself out permanently.

---

### `netplan apply`
**What it does:** Applies Netplan configuration **immediately and permanently** without auto-revert.

```bash
sudo netplan apply
```

**Quick tip:** Use `netplan try` first to validate, then `netplan apply` when confident.

---

### `networkctl reload` / `networkctl reconfigure`
**What it does:** Reloads network config or reconfigures a specific interface without full apply.

```bash
sudo networkctl reload                  # reload all
sudo networkctl reconfigure enp6s0      # reconfigure one interface
```

---

### Netplan Config File Structure

Config files go in `/etc/netplan/` — naming convention: `NN-name.yaml` (lower number = higher priority).

```bash
# List config files
ls /etc/netplan/

# View existing config
sudo cat /etc/netplan/50-cloud-init.yaml

# Create new config file
sudo vim /etc/netplan/00-mysettings.yaml
```

**⚠️ File permissions must be 600:**
```bash
sudo chmod 600 /etc/netplan/00-mysettings.yaml
```
If permissions are too open (644 or 755), Netplan will show warnings.

---

### Netplan Config Examples

**Basic DHCP config:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

**Static IPv4 + IPv6 config:**
```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      dhcp6: false
      addresses:
        - 10.0.0.9/24
        - fe80::5054:ff:fe1f:abcd/64
```

**Full config with DNS + static routes:**
```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      dhcp6: false
      addresses:
        - 10.0.0.9/24
        - fe80::5054:ff:fe1f:abcd/64
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        - to: 192.168.0.0/24     # traffic to this network...
          via: 10.0.0.100         # ...goes through this gateway
        - to: default             # all other traffic...
          via: 10.0.0.1           # ...goes through default gateway
```

**Route logic explained:**

| Destination | Via | Meaning |
|------------|-----|---------|
| `192.168.0.0/24` | `10.0.0.100` | Specific subnet behind another router |
| `default` | `10.0.0.1` | Everything else → internet gateway |

---

### Netplan Documentation

```bash
# Built-in manual
man netplan

# Official examples directory
ls /usr/share/doc/netplan/examples
# Contains: dhcp.yaml, static.yaml, bonding.yaml, bridge.yaml etc.

# All software documentation
ls /usr/share/doc
```

---

## 🧪 Lab: Netplan Configuration

**Objective:** Create a permanent static IP configuration using Netplan.

**Tasks:**
1. Check existing Netplan files: `ls /etc/netplan/`
2. Create `/etc/netplan/99-custom.yaml` with static IP `10.0.10.5/24` on `enp6s0`
3. Set correct permissions (600) on the file
4. Test with `netplan try` — confirm or cancel
5. Apply with `netplan apply`
6. Verify with `ip -c addr` and `sudo netplan get`
7. *(Challenge)* Add DNS servers `8.8.8.8` and `1.1.1.1` to the config and a default route via `10.0.10.1`

**Solution:**
```bash
# Task 1
ls /etc/netplan/

# Task 2
sudo vim /etc/netplan/99-custom.yaml
# Paste:
# network:
#   version: 2
#   ethernets:
#     enp6s0:
#       dhcp4: false
#       dhcp6: false
#       addresses:
#         - 10.0.10.5/24

# Task 3
sudo chmod 600 /etc/netplan/99-custom.yaml

# Task 4
sudo netplan try
# Press Enter to confirm

# Task 5
sudo netplan apply

# Task 6
ip -c addr
sudo netplan get

# Challenge — add to yaml:
#       nameservers:
#         addresses: [8.8.8.8, 1.1.1.1]
#       routes:
#         - to: default
#           via: 10.0.10.1
```

---

## 🔌 Socket Statistics — `ss` and `netstat`

---

### `ss`
**What it does:** Shows socket statistics — which ports are open and which processes are using them. Modern replacement for `netstat`.

**Syntax:**
```bash
ss [OPTIONS]
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-l` | Listening sockets only |
| `-t` | TCP connections |
| `-u` | UDP connections |
| `-n` | Numeric (don't resolve names) |
| `-p` | Show process name and PID |
| `-4` | IPv4 only |
| `-6` | IPv6 only |

**Memory trick:** `-ltunp` = **l**isten **t**cp **u**dp **n**umeric **p**rocess → "**TUNNEL** + p"

**Examples:**
```bash
# Show all listening TCP+UDP with process info (most useful)
sudo ss -ltunp

# Same flags, different order — works identically
sudo ss -tunlp

# Show all established connections
ss -t

# Filter for specific port
sudo ss -ltunp | grep :22

# Get PID of process listening on port 22
sudo ss -ltunp | egrep 22 | egrep -o 'pid=[0-9]+' | cut -d= -f2

# Get PID of process on port 53 (first result only)
sudo ss -ltunp | egrep 53 | egrep -o 'pid=[0-9]+' | head -1 | cut -d= -f2

# Get process NAME on port 8080
sudo ss -ltpun | egrep 8080 | egrep -o '\("([^"]+)"' | cut -d'"' -f2

# List all open port numbers only
sudo ss -ltnup | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort -n | uniq

# Get more options
ss --help
```

**Reading ss output:**
```
Netid  State  Recv-Q Send-Q  Local Address:Port  Peer Address:Port  Process
tcp    LISTEN  0     128     0.0.0.0:22           0.0.0.0:*         users:(("sshd",pid=1114,fd=3))
```
- `0.0.0.0:22` → listening on ALL interfaces, port 22
- `127.0.0.53:53` → listening only on localhost
- `users:(("sshd",pid=1114))` → process name and PID

---

### `netstat`
**What it does:** Shows network connections, routing table, and interface stats. Older tool, but still widely used.

**Syntax:**
```bash
netstat [OPTIONS]
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-t` | TCP |
| `-u` | UDP |
| `-l` | Listening only |
| `-p` | Show PID/program name |
| `-n` | Numeric addresses |

**Examples:**
```bash
# Show all listening ports with process info (equivalent to ss -ltunp)
sudo netstat -tulpn

# Show only LISTEN state
sudo netstat -tulpn | grep LISTEN

# Save listening ports to file
sudo netstat -tulpn | grep LISTEN > /home/bob/incoming.txt
```

**Quick tip:** `ss` is preferred on modern systems — `netstat` may not be installed by default. Install with `sudo apt install net-tools`.

---

### `lsof`
**What it does:** Lists open files — including network sockets opened by a process.

**Syntax:**
```bash
sudo lsof -p <PID>       # files opened by process
sudo lsof -i :<PORT>     # process using specific port
```

**Examples:**
```bash
# See all files/connections of process with PID 697
sudo lsof -p 697

# Find what's using port 80
sudo lsof -i :80

# Combined with ss workflow:
# 1. Find PID from ss
sudo ss -ltunp | grep :22
# 2. Inspect that process
sudo lsof -p 1114
```

**Quick tip:** Useful for debugging "port already in use" errors — `lsof -i :PORT` immediately shows you what's holding the port.

---

## 🧪 Lab: Ports & Processes

**Objective:** Identify what is listening on which ports and extract process info.

**Tasks:**
1. Show all listening TCP and UDP ports with process names
2. Find the PID of the process listening on port 22, save to `/home/bob/pid`
3. Find the process NAME listening on port 8080, save to `/home/bob/process`
4. Save all listening ports to `/home/bob/incoming.txt`
5. *(Challenge)* List only the port numbers (no duplicates, sorted) of all listening services

**Solution:**
```bash
# Task 1
sudo ss -ltunp

# Task 2
sudo ss -ltunp | egrep ':22 ' | egrep -o 'pid=[0-9]+' | head -1 | cut -d= -f2 > /home/bob/pid

# Task 3
sudo ss -ltpun | egrep 8080 | egrep -o '\("([^"]+)"' | cut -d'"' -f2 > /home/bob/process

# Task 4
sudo netstat -tulpn | grep LISTEN > /home/bob/incoming.txt

# Challenge
sudo ss -ltunp | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort -n | uniq
```

---

## 🔠 DNS Resolution

---

### `resolvectl`
**What it does:** Controls and queries the systemd-resolved DNS resolver.

**Syntax:**
```bash
resolvectl status          # full DNS status
resolvectl dns             # show DNS servers only
```

**Examples:**
```bash
# Full DNS status — shows global + per-interface DNS
resolvectl status

# Show only DNS servers (compact)
resolvectl dns

# Check after config change
sudo systemctl restart systemd-resolved.service
resolvectl status
```

**Reading the output:**
```
Global:
   DNS Servers: 1.1.1.1 9.9.9.9    ← global DNS (from /etc/systemd/resolved.conf)

Link 2 (enp0s8):
Current DNS Server: 8.8.4.4
       DNS Servers: 8.8.4.4 8.8.8.8  ← per-interface DNS (from Netplan)
```

---

### `/etc/hosts` — Static hostname resolution
**What it does:** Maps hostnames to IP addresses locally, before DNS is consulted.

```bash
# Edit hosts file
sudo vim /etc/hosts
```

**Format:** `IP_ADDRESS  hostname  [alias]`

**Examples:**
```bash
# Map internal server name
127.0.123.123  dbserver

# Map domain for local testing
8.8.8.8  example.com
1.2.3.4  myapp.local

# Test it works
ping dbserver
ping example.com
```

**Quick tip:** `/etc/hosts` takes priority over DNS. Great for local testing and overriding domains.

---

### Global DNS — `/etc/systemd/resolved.conf`
**What it does:** Sets DNS servers that apply to **all** network interfaces system-wide.

```bash
# Edit the file
sudo vim /etc/systemd/resolved.conf

# Add/modify this line:
DNS=1.1.1.1 9.9.9.9

# Restart to apply
sudo systemctl restart systemd-resolved.service

# Verify
resolvectl status
```

**DNS resolution priority:**
```
/etc/hosts → per-interface DNS (Netplan) → global DNS (resolved.conf)
```

---

### `/etc/resolv.conf`
```bash
# Verify DNS config (alternative to resolvectl)
cat /etc/resolv.conf
```

**Quick tip:** On Ubuntu with systemd, `/etc/resolv.conf` is usually a symlink managed automatically — don't edit it directly. Use `resolved.conf` instead.

---

## 🧪 Lab: DNS & Hostname Resolution

**Objective:** Configure static hostname resolution and global DNS.

**Tasks:**
1. View current `/etc/hosts` content
2. Add entry: `8.8.8.8 example.com` to `/etc/hosts`
3. Test with `ping example.com` — should resolve to 8.8.8.8
4. Set global DNS to `8.8.8.8` in `/etc/systemd/resolved.conf`
5. Restart the DNS service and verify with `resolvectl status`
6. *(Challenge)* Add a second DNS `1.1.1.1` as fallback and verify both appear in `resolvectl status`

**Solution:**
```bash
# Task 1
cat /etc/hosts

# Task 2
sudo vim /etc/hosts
# Add line: 8.8.8.8 example.com

# Task 3
ping example.com

# Task 4
sudo vim /etc/systemd/resolved.conf
# Add/set: DNS=8.8.8.8

# Task 5
sudo systemctl restart systemd-resolved.service
resolvectl status

# Challenge
# In resolved.conf set: DNS=8.8.8.8 1.1.1.1
sudo systemctl restart systemd-resolved.service
resolvectl status | grep "DNS Servers"
```

---

## ⚙️ Service Management — `systemctl`

---

### `systemctl`
**What it does:** Controls system services (start, stop, enable, disable, check status).

**Syntax:**
```bash
sudo systemctl <action> <service-name>
```

**Common actions:**
| Action | Description |
|--------|-------------|
| `status` | Show if service is running + recent logs |
| `start` | Start the service now |
| `stop` | Stop the service now |
| `restart` | Stop then start |
| `reload` | Reload config without restart (if supported) |
| `enable` | Start automatically on boot |
| `disable` | Don't start on boot |
| `is-active` | Quick check if running (returns active/inactive) |

**Examples:**
```bash
# Check status of MariaDB (shows PID, uptime, logs)
systemctl status mariadb.service

# Check SSH service (note: Ubuntu=ssh, RedHat=sshd)
systemctl status ssh.service      # Ubuntu
systemctl status sshd.service     # RedHat/CentOS

# Stop a service
sudo systemctl stop mariadb.service

# Start a service
sudo systemctl start mariadb.service

# Restart after config change
sudo systemctl restart nginx.service

# Enable to start on boot
sudo systemctl enable mariadb.service

# Disable from starting on boot
sudo systemctl disable mariadb.service

# Restart DNS resolver (after resolv.conf changes)
sudo systemctl restart systemd-resolved.service
```

**Quick tip:** ⚠️ Ubuntu uses `ssh` service name, RedHat/CentOS uses `sshd`. Always check with `systemctl status ssh` first.

---

### `ps`
**What it does:** Shows running process details by PID.

**Examples:**
```bash
# Get info about specific PID (found from ss output)
ps 1114

# See process with full details
ps aux | grep sshd
```

---

## 🔗 Network Bonding & Bridging

### Bonding Concepts

**What it is:** Combining 2+ network interfaces into one logical interface for **redundancy** or **throughput**.

**When to use:** When you need high availability (one card fails, other takes over) or increased bandwidth.

### Bonding Modes

| Mode | Name | Description |
|------|------|-------------|
| `0` | Round Robin (default) | Alternates between interfaces for each packet |
| `1` | Active-Backup | Only one active; second is standby for failover |
| `2` | XOR | Traffic to same destination always uses same interface |
| `3` | Broadcast | Sends all data to ALL interfaces simultaneously |
| `4` | IEEE 802.3ad (LACP) | Aggregates bandwidth — can increase transfer rates |
| `5` | Adaptive TX Load Balance | Sends on least-busy interface (TX only) |
| `6` | Adaptive Load Balance | Balances BOTH incoming AND outgoing traffic |

**Most common in production:**
- **Mode 1** (Active-Backup) — simple HA, no switch config needed
- **Mode 4** (LACP) — max bandwidth, requires switch support

### Netplan Bonding Config Example
```yaml
network:
  version: 2
  bonds:
    bond0:
      interfaces: [enp0s8, enp0s9]
      parameters:
        mode: active-backup    # mode 1
      dhcp4: true
```

### Bridging Concepts

**What it is:** A software bridge connects multiple network segments at Layer 2 — acts like a virtual switch.

**Common use case:** Virtual machines — bridge connects VMs to the physical network.

```yaml
network:
  version: 2
  bridges:
    br0:
      interfaces: [enp0s3]
      dhcp4: true
```

---

## 📎 Appendix: Essential One-Liners

```bash
# Get IP of specific interface (clean, no subnet)
ip addr show dev eth0 | grep 'inet ' | awk '{print $2}' | cut -d'/' -f1

# Save routing table to file
ip route > /home/bob/route.txt

# Get default gateway IP only
ip route | grep default | awk '{print $3}'

# Save listening ports to file
sudo netstat -tulpn | grep LISTEN > /home/bob/incoming.txt

# Get PID of process on specific port
sudo ss -ltunp | grep ':22 ' | grep -o 'pid=[0-9]*' | head -1 | cut -d= -f2

# Get process name on specific port
sudo ss -ltpun | grep 8080 | grep -o '"[^"]*"' | head -1 | tr -d '"'

# List all listening port numbers (clean, sorted)
sudo ss -ltunp | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort -n | uniq

# Test DNS resolution
ping -c 3 example.com

# Check what's using port 80
sudo lsof -i :80

# Full network debug workflow:
ip link                  # 1. Check interfaces
ip -c addr               # 2. Check IPs
ip route                 # 3. Check routes
resolvectl status        # 4. Check DNS
sudo ss -ltunp           # 5. Check open ports
```

---

## 📁 Key Config Files Reference

| File | Purpose |
|------|---------|
| `/etc/netplan/*.yaml` | Permanent network interface config |
| `/etc/hosts` | Static hostname → IP mapping |
| `/etc/hostname` | System hostname |
| `/etc/systemd/resolved.conf` | Global DNS resolver config |
| `/etc/resolv.conf` | DNS resolver (auto-managed, don't edit directly) |
| `/usr/share/doc/netplan/examples/` | Netplan config examples |

---

> ⚠️ **Disclaimer:** Always test changes with `netplan try` before `netplan apply` on remote servers. Use `man <command>` for the full reference.
