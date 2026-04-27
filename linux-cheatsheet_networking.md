# Linux Command Reference — Networking
> Generated from course notes | April 2026


---

## Quick Reference Table

| Command | One-liner description |
|---------|----------------------|
| `ip link` | Show/manage network interfaces |
| `ip addr` / `ip a` | Show IP addresses on all interfaces |
| `ip route` / `ip r` | Show/manage the routing table |
| `ip link set dev <iface> up/down` | Bring an interface up or down |
| `ip addr add` | Add an IPv4 or IPv6 address to an interface |
| `ip addr delete` | Remove an IP address from an interface |
| `ip link delete` | Delete a virtual interface (bridge, bond) |
| `netplan get` | Show current Netplan configuration tree |
| `netplan try` | Test Netplan config with auto-revert |
| `netplan apply` | Apply Netplan config immediately |
| `ss` | Socket statistics — who is listening and on which port |
| `netstat` | Legacy socket/port listing tool |
| `ps` | Show process info by PID |
| `lsof` | List open files / sockets for a process |
| `resolvectl` | Query and manage DNS resolution |
| `systemctl` | Start/stop/enable/disable/status of services |
| `ufw` | Uncomplicated Firewall — manage packet filter rules |
| `iptables` | Low-level NAT and packet filter rules |
| `nft` | Modern nftables front-end (replaces iptables) |
| `sysctl` | Read/write kernel parameters at runtime |
| `nginx` | Web server / reverse proxy / load balancer |
| `ssh` | SSH client — connect to a remote server |
| `ssh-keygen` | Generate SSH key pairs |
| `ssh-copy-id` | Copy public key to a remote server |
| `timedatectl` | View and set system time, timezone, NTP |
| `ping` | Test basic network reachability |
| `cut` | Extract fields from delimited text |
| `egrep` | Extended regex grep for pattern matching |
| `awk` | Field-based text processing |
| `chmod` | Change file or directory permissions |
| `ln -s` | Create a symbolic (soft) link |

---

## Networking — Interface Management (`ip`)

### `ip link`
**What it does:** Lists all network interfaces (NICs) and their state (UP/DOWN), without IP address details.

**Syntax:**
```bash
ip link
ip -c link          # colorized output
ip link set dev <IFACE> up
ip link set dev <IFACE> down
ip link delete <IFACE>
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-c` | Colorize output for easier reading |
| `set dev <iface> up` | Bring interface up |
| `set dev <iface> down` | Take interface down |
| `delete <iface>` | Delete a virtual interface (bridge, bond) |

**Examples:**
```bash
# List all interfaces with color
ip -c link

# Bring the secondary interface up
sudo ip link set dev enp0s8 up

# Take an interface down before reconfiguring
sudo ip link set dev enp0s8 down

# Remove a bridge interface (after deleting bridge.yaml)
sudo ip link delete br0
```

**Quick tip:** `ip link` shows layer-2 state (MAC, UP/DOWN). Use `ip addr` to see IP addresses. After `ip link delete br0`, do a reboot to ensure all related routes and configs are fully flushed.

---

### `ip addr`
**What it does:** Shows all IP addresses (IPv4 and IPv6) assigned to network interfaces.

**Syntax:**
```bash
ip addr
ip a                          # shorthand
ip -c addr                    # colorized
ip addr show dev <IFACE>      # single interface
ip addr add <IP/PREFIX> dev <IFACE>
ip addr delete <IP/PREFIX> dev <IFACE>
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-c` | Colorize output |
| `show dev <iface>` | Limit output to a single interface |
| `add` | Assign an IP address to an interface (temporary, lost on reboot) |
| `delete` | Remove an IP address from an interface |

**Examples:**
```bash
# Show all addresses with color
ip -c addr

# Show only eth0
ip addr show dev eth0

# Add a temporary IPv4 address
sudo ip addr add 10.0.0.40/24 dev enp0s8

# Add a temporary IPv6 address
sudo ip addr add fe80::5054:ff:fe1f:8050/64 dev enp0s8

# Remove an IPv6 address
sudo ip addr delete fe80::5054:ff:fe1f:8050/64 dev enp0s8

# Extract only the IP (without subnet mask) from eth0
ip addr show dev eth0 | grep inet | awk '{print $2}' | cut -d'/' -f1
```

**Quick tip:** Changes made with `ip addr add` are temporary and survive only until reboot. Use Netplan YAML files for persistent configuration.

---

### `ip route`
**What it does:** Displays the kernel routing table — how the system decides where to send packets.

**Syntax:**
```bash
ip route
ip r              # shorthand
ip route > /home/bob/route.txt   # save output to file
```

**Examples:**
```bash
# Show full routing table
ip route

# Check default gateway (first hop for all external traffic)
ip route | grep default

# Extract all IPv4 addresses from routing table with regex
ip route | egrep -o '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'

# Save routing table to a file
ip route > /home/bob/route.txt
```

**Quick tip:** `ip route` does NOT show DNS configuration. Use `resolvectl status` for DNS. The `default via` line is your gateway to the internet.

---

## 🧪 Lab: Interface & IP Management

**Objective:** Use `ip` commands to inspect and document network configuration.

**Tasks:**
1. List all interfaces and identify which are UP — `ip -c link`
2. Find the IP and subnet of interface `eth0` → `ip addr show dev eth0`
3. Save only the IP addresses (no subnet mask) to `/home/bob/ip` → combine `grep inet`, `awk`, `cut`
4. Find the default gateway IP and save it to `/home/bob/gateway.txt` → use `ip route | grep default`
5. **(Challenge)** Add a temporary static IP `10.0.0.99/24` to a secondary interface, verify with `ip -c addr`, then bring the interface down.

**Solution:**
```bash
# Task 1
ip -c link

# Task 2
ip addr show dev eth0

# Task 3 — save IPs without subnet mask
ip addr show dev eth0 | grep inet | awk '{print $2}' | cut -d'/' -f1 > /home/bob/ip

# Task 4 — save default gateway IP
ip route | grep default | awk '{print $3}' > /home/bob/gateway.txt

# Task 5 — challenge
sudo ip addr add 10.0.0.99/24 dev enp0s8
ip -c addr
sudo ip link set dev enp0s8 down
```

---

## Netplan — Persistent Network Configuration

### `netplan get`
**What it does:** Reads and displays the active Netplan configuration in a tree hierarchy.

**Syntax:**
```bash
sudo netplan get
```

**Examples:**
```bash
# View current configuration tree
sudo netplan get

# View the raw YAML file
sudo cat /etc/netplan/50-cloud-init.yaml

# List all netplan config files
ls /etc/netplan
```

**Quick tip:** `netplan get` shows the merged config from all YAML files. Always verify here after `netplan apply` to confirm settings took effect and will survive reboot.

---

### `netplan try`
**What it does:** Applies the configuration temporarily, then automatically reverts if you don't confirm within the timeout — a safety net for remote sessions.

**Syntax:**
```bash
sudo netplan try
sudo netplan try --timeout 30
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `--timeout <sec>` | Override the auto-revert countdown (default: 120s) |

**Examples:**
```bash
# Apply and wait for confirmation (120s auto-revert)
sudo netplan try
# Press Enter to confirm, Ctrl+C to cancel

# Short 30-second window — useful for remote sessions
sudo netplan try --timeout 30
```

**Quick tip:** Always use `netplan try` before `netplan apply` on a remote server. If your config breaks SSH, you'll be locked out — but `netplan try` gives you 2 minutes before auto-reverting.

⚠️ Note: Bond configurations (`bonds:`) do NOT work with `netplan try` — use `netplan apply` directly for bonding.

---

### `netplan apply`
**What it does:** Applies Netplan configuration immediately and permanently (no revert).

**Syntax:**
```bash
sudo netplan apply
```

**Examples:**
```bash
# Apply config after confirming with netplan try
sudo netplan apply

# Full workflow for a new interface config
sudo vim /etc/netplan/00-mysettings.yaml
sudo chmod 600 /etc/netplan/00-mysettings.yaml
sudo netplan try
sudo netplan apply
ip -c addr
```

**Quick tip:** Set `chmod 600` on all Netplan YAML files to suppress permission warnings. Netplan ignores world-readable configs on modern Ubuntu versions.

---

### Netplan YAML — Static IP with DNS and Routes

**What it does:** Defines persistent network settings that survive reboots.

**File location:** `/etc/netplan/00-mysettings.yaml` (custom files should NOT modify `50-cloud-init.yaml`)

**Full example:**
```yaml
network:
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
        - to: 192.168.0.0/24
          via: 10.0.0.100
        - to: default
          via: 10.0.0.1
  version: 2
```

**Quick tip:** `to: default` is your internet gateway (replaces the old `gateway4` key). `to: 192.168.0.0/24 via: 10.0.0.100` is a static route to a network behind another router.

---

## 🧪 Lab: Netplan Persistent Configuration

**Objective:** Configure a static IP, DNS, and default gateway for a secondary interface that persists across reboots.

**Tasks:**
1. Find your interface names → `ip link`
2. Create a new Netplan file `/etc/netplan/00-mysettings.yaml`
3. Set a static IPv4 of `10.0.0.50/24`, DNS `8.8.8.8`, default gateway `10.0.0.1`
4. Set correct permissions → `chmod 600`
5. Test safely → `netplan try`
6. Confirm and apply → press Enter, then `netplan apply`
7. **(Challenge)** Add a static route: traffic to `172.16.0.0/24` goes via `10.0.0.254`

**Solution:**
```bash
# Task 1
ip link

# Task 2-3
sudo vim /etc/netplan/00-mysettings.yaml
# Paste:
# network:
#   ethernets:
#     enp0s8:
#       dhcp4: false
#       addresses: [10.0.0.50/24]
#       nameservers:
#         addresses: [8.8.8.8]
#       routes:
#         - to: default
#           via: 10.0.0.1
#   version: 2

# Task 4
sudo chmod 600 /etc/netplan/00-mysettings.yaml

# Task 5-6
sudo netplan try
# press Enter
sudo netplan apply

# Verify
ip -c addr && ip route && resolvectl status
```

---

## DNS Resolution

### `resolvectl`
**What it does:** Queries the systemd-resolved DNS service — shows per-interface and global DNS servers, and resolution status.

**Syntax:**
```bash
resolvectl status
resolvectl dns
timedatectl show-timesync      # NTP-specific (different tool)
```

**Examples:**
```bash
# Show full DNS configuration for all interfaces
resolvectl status

# Show only DNS server assignments
resolvectl dns

# Add global DNS via systemd config
sudo vim /etc/systemd/resolved.conf
# Add: DNS=1.1.1.1 9.9.9.9

# Restart DNS daemon
sudo systemctl restart systemd-resolved.service

# Verify global DNS applied
resolvectl status
```

**Quick tip:** Interface-level DNS (from Netplan `nameservers:`) overrides the global DNS in `/etc/systemd/resolved.conf`. To set a fallback for ALL interfaces, edit `resolved.conf`.

---

### `/etc/hosts` — Static Hostname Resolution
**What it does:** Maps hostnames to IP addresses locally, before DNS is consulted.

**File:** `/etc/hosts`

**Examples:**
```bash
# View current hosts file
cat /etc/hosts

# Add a hostname mapping (edit the file)
sudo vim /etc/hosts
# Add line:
# 127.0.123.123 dbserver

# Test the mapping
ping dbserver

# Simulate a domain for local testing
# 1.2.3.4  example.com
ping example.com
```

**Quick tip:** `/etc/hosts` is checked before DNS. Use it in local/lab environments to avoid needing a real DNS server. The `/etc/hostname` file stores the machine's own hostname.

---

## 🧪 Lab: DNS & Hostname Resolution

**Objective:** Configure global DNS and test local hostname resolution.

**Tasks:**
1. Check current DNS servers → `resolvectl status`
2. Add global DNS `8.8.8.8` to `/etc/systemd/resolved.conf`
3. Restart the resolver → `systemctl restart systemd-resolved.service`
4. Verify the global DNS appears in `resolvectl status`
5. Add a local hostname `webserver` pointing to `192.168.1.100` in `/etc/hosts`
6. **(Challenge)** Ping `webserver` and trace which DNS step resolves it (hint: `strace -e trace=network ping webserver`)

**Solution:**
```bash
# Task 1
resolvectl status

# Task 2
sudo vim /etc/systemd/resolved.conf
# Add under [Resolve]: DNS=8.8.8.8

# Task 3
sudo systemctl restart systemd-resolved.service

# Task 4
resolvectl status | grep "DNS Servers"

# Task 5
echo "192.168.1.100 webserver" | sudo tee -a /etc/hosts

# Task 6
ping webserver
```

---

## Socket & Port Inspection (`ss`, `netstat`, `lsof`, `ps`)

### `ss`
**What it does:** Shows socket statistics — which processes are listening on which ports (modern replacement for `netstat`).

**Syntax:**
```bash
ss [OPTIONS]
sudo ss -ltunp
sudo ss -tunlp     # same flags, "tunnel" mnemonic
```

**Common flags:**
| Flag | Description |
|------|-------------|
| `-l` | Show only listening sockets |
| `-t` | TCP sockets |
| `-u` | UDP sockets |
| `-n` | Numeric ports (no service name lookup) |
| `-p` | Show process name and PID |

**Examples:**
```bash
# Show all listening TCP/UDP sockets with process names
sudo ss -ltunp

# Memory aid: "tunnel" — same flags, easier to remember
sudo ss -tunlp

# Find which process is listening on port 22
sudo ss -ltunp | grep :22

# Extract PID of process on port 22
sudo ss -ltunp | egrep ':22 ' | egrep -o 'pid=[0-9]+' | cut -d= -f2 | head -1

# Show only active (established) TCP connections
ss -tn

# List all open ports in sorted order
sudo ss -ltnup | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort -n | uniq
```

**Quick tip:** Run with `sudo` to see process names — without it, most process columns are blank. The mnemonic **"tunnel"** (`-tunlp`) helps remember the flags.

---

### `netstat`
**What it does:** Legacy tool showing open ports and listening processes — not available on all systems.

**Syntax:**
```bash
sudo netstat -ltunp
sudo netstat -tulpn | grep LISTEN
```

**Examples:**
```bash
# List all listening ports with process names
sudo netstat -ltunp

# Filter to show only LISTEN state (cleaner output)
sudo netstat -tulpn | grep LISTEN

# Save listening ports to a file
sudo netstat -tulpn | grep LISTEN > /home/bob/incoming.txt
```

**Quick tip:** Prefer `ss` on modern systems. `netstat` requires the `net-tools` package which may not be installed by default.

---

### `lsof`
**What it does:** Lists all open files — including network sockets — for a given process.

**Syntax:**
```bash
sudo lsof -p <PID>
```

**Examples:**
```bash
# Find which files/sockets PID 697 has open
sudo lsof -p 697

# Workflow: find PID via ss, then inspect with lsof
sudo ss -ltunp | grep :22
sudo lsof -p 1114
```

**Quick tip:** On Linux, everything is a file — network connections appear in `lsof` output as socket file descriptors. Great for debugging what a suspicious process is actually doing.

---

### `ps`
**What it does:** Shows running process info given a PID.

**Syntax:**
```bash
ps <PID>
```

**Examples:**
```bash
# Get details about a process by PID
ps 697

# Find PID from ss output then inspect it
sudo ss -ltunp | egrep 53 | egrep -o 'pid=[0-9]+' | head -1 | cut -d= -f2
ps 640
```

---

## 🧪 Lab: Socket & Port Inspection

**Objective:** Identify listening services and investigate processes by port.

**Tasks:**
1. List all listening TCP/UDP ports → `sudo ss -ltunp`
2. Find the process name listening on port 22 → `grep` and `awk`
3. Extract just the PID of the process on port 53 → save to `/home/bob/process_pid`
4. Use `lsof` to see all open files for that PID
5. **(Challenge)** Write a one-liner to find the process NAME on port 8080 and save it to `/home/bob/process`

**Solution:**
```bash
# Task 1
sudo ss -ltunp

# Task 2
sudo ss -ltunp | grep ':22 '

# Task 3
sudo ss -ltunp | egrep ':53 ' | egrep -o 'pid=[0-9]+' | head -1 | cut -d= -f2 > /home/bob/process_pid

# Task 4
sudo lsof -p $(cat /home/bob/process_pid)

# Task 5 — extract process name on port 8080
sudo ss -ltunp | grep ':8080' | grep -oP '"\K[^"]+' | head -1 > /home/bob/process
```

---

## systemctl — Service Management

### `systemctl`
**What it does:** Controls systemd services (start, stop, enable, disable, status).

**Syntax:**
```bash
systemctl status <service>
sudo systemctl start <service>
sudo systemctl stop <service>
sudo systemctl restart <service>
sudo systemctl reload <service>
sudo systemctl enable <service>
sudo systemctl disable <service>
```

**Examples:**
```bash
# Check SSH service status (Ubuntu calls it ssh.service)
systemctl status ssh
systemctl status sshd    # Red Hat / CentOS name

# Stop a database service
sudo systemctl stop mariadb.service

# Disable a service from starting at boot
sudo systemctl disable mariadb.service

# Enable and start in one shot
sudo systemctl enable --now mariadb.service

# Reload config without full restart (e.g. after editing sshd_config)
sudo systemctl reload ssh.service

# Restart DNS resolver after config change
sudo systemctl restart systemd-resolved.service
```

**Quick tip:** On Ubuntu the SSH daemon service is `ssh.service`, but the daemon binary is `sshd` and the config file is `sshd_config`. On Red Hat/CentOS the service is named `sshd.service`.

---

## Bridge & Bonding

### Bridge — Connecting Multiple Interfaces into One LAN

**What it does:** A software switch that connects two or more physical interfaces so all devices appear on the same Layer-2 network.

**Netplan YAML (`/etc/netplan/99-bridge.yaml`):**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      dhcp4: no
    enp0s9:
      dhcp4: no
  bridges:
    br0:
      dhcp4: yes
      interfaces:
        - enp0s8
        - enp0s9
```

**Commands:**
```bash
# Copy example from system docs
sudo cp /usr/share/doc/netplan/examples/bridge.yaml /etc/netplan/99-bridge.yaml
sudo chmod 600 /etc/netplan/99-bridge.yaml
sudo vim /etc/netplan/99-bridge.yaml

# Apply
sudo netplan apply

# Verify bridge interface and its members
ip -c link    # look for "master br0" next to enp0s8 and enp0s9
ip -c addr    # br0 should have an IP from DHCP

# Remove bridge (before configuring bond)
sudo rm /etc/netplan/99-bridge.yaml
sudo ip link delete br0
# Then reboot for clean state
```

**Quick tip:** Use bridging when you want multiple NICs or VMs to behave as if they are on the same physical LAN. Use bonding when you want redundancy or higher throughput.

---

### Bonding — Combining NICs for Redundancy or Throughput

**What it does:** Merges multiple physical NICs into one logical interface.

**Bonding Modes:**

| Mode | Name | Description |
|------|------|-------------|
| 0 | Round Robin | Packets sent across all interfaces in order |
| 1 | Active-Backup | One active NIC; failover to standby if it fails |
| 2 | XOR | Same interface per destination (based on MAC hash) |
| 3 | Broadcast | All data sent out all interfaces |
| 4 | 802.3ad (LACP) | Link aggregation — increases throughput (switch must support) |
| 5 | Adaptive TLB | Transmit load balancing — sends via least busy interface |
| 6 | Adaptive ALB | Load balances both TX and RX traffic |

**Netplan YAML (`/etc/netplan/99-bond.yaml`):**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      dhcp4: no
    enp0s9:
      dhcp4: no
  bonds:
    bond0:
      dhcp4: yes
      interfaces:
        - enp0s8
        - enp0s9
      parameters:
        mode: active-backup
        primary: enp0s8
        mii-monitor-interval: 100
```

**Commands:**
```bash
# Copy example config
sudo cp /usr/share/doc/netplan/examples/bonding.yaml /etc/netplan/99-bond.yaml
sudo chmod 600 /etc/netplan/99-bond.yaml
sudo vim /etc/netplan/99-bond.yaml

# Bond MUST use apply (not try)
sudo netplan apply

# Verify bond is up and members are assigned
ip -c addr        # bond0 should show UP and an IP
ip -c link        # look for "master bond0" next to enp0s8/enp0s9

# Inspect bond state (active slave, mode, etc.)
cat /proc/net/bonding/bond0

# Temporary interface management (lost on reboot)
sudo ip link set dev bond0 down
sudo ip link set dev bond0 up
```

**Quick tip:** `netplan try` does NOT work for bonds — always use `netplan apply`. Verify your backup (`active-backup`, mode 1) by unplugging the primary NIC and checking `cat /proc/net/bonding/bond0` to confirm failover occurred.

---

## UFW — Uncomplicated Firewall

### `ufw`
**What it does:** Manages the Linux packet filter (iptables/nftables) via a simplified rule interface.

**Syntax:**
```bash
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered
sudo ufw enable
sudo ufw disable
sudo ufw allow <PORT>
sudo ufw allow <PORT>/<PROTO>
sudo ufw deny from <IP>
sudo ufw delete <RULE_NUMBER>
sudo ufw insert <POSITION> <RULE>
```

**Examples:**
```bash
# ALWAYS allow SSH before enabling UFW
sudo ufw allow 22
sudo ufw allow 22/tcp      # TCP only
sudo ufw enable

# Verbose status
sudo ufw status verbose

# View rules with their processing order
sudo ufw status numbered

# Allow from specific IP to any port
sudo ufw allow from 10.0.0.2 to any port

# Allow from an IP range to port 22
sudo ufw allow from 10.0.0.0/24 to any port 22

# Deny a specific IP
sudo ufw deny from 10.0.0.37

# Delete rule by ID (from numbered list)
sudo ufw delete 2

# Delete a rule by spec (removes both IPv4 and IPv6)
sudo ufw delete allow 22

# INSERT a deny rule at position 1 (before any allow rules)
sudo ufw insert 1 deny from 10.0.0.37

# Allow traffic for NAT forwarding
sudo ufw route allow from 10.0.0.0/24 to 192.168.0.5

# Allow a port range
sudo ufw allow 8000:8080/tcp
```

**Quick tip:** UFW processes rules **top to bottom** — the first match wins. If you add a `deny from 10.0.0.37` but there's already an `allow from 10.0.0.0/24` above it, the deny is never reached. Use `ufw insert 1 deny ...` to move denies to the top.

---

## 🧪 Lab: UFW Firewall Rules

**Objective:** Build and manage a working UFW ruleset.

**Setup:**
```bash
sudo ufw allow 22       # Allow SSH first!
sudo ufw enable
```

**Tasks:**
1. Allow HTTP (port 80/tcp) from anywhere
2. Allow MySQL (port 3306) only from `10.0.0.0/24`
3. Deny all traffic from `10.0.0.50`
4. Check the rule order — ensure deny appears BEFORE the subnet allow
5. **(Challenge)** Fix the rule order if deny is after allow: delete and re-insert at position 1

**Solution:**
```bash
# Task 1
sudo ufw allow 80/tcp

# Task 2
sudo ufw allow from 10.0.0.0/24 to any port 3306

# Task 3
sudo ufw deny from 10.0.0.50

# Task 4
sudo ufw status numbered

# Task 5 — if deny is in wrong position
sudo ufw delete <deny_rule_id>
sudo ufw insert 1 deny from 10.0.0.50
sudo ufw status numbered
```

---

## NAT & Port Forwarding (`iptables`, `nft`, `sysctl`)

### `sysctl` — Enable IP Forwarding
**What it does:** Sets kernel networking parameters at runtime.

**Syntax:**
```bash
sudo sysctl -a | grep forward
sudo sysctl -p /etc/sysctl.d/99-sysctl.conf
```

**Examples:**
```bash
# Edit persistent config (survives updates)
sudo vim /etc/sysctl.d/99-sysctl.conf
# Uncomment:
# net.ipv4.ip_forward=1
# net.ipv6.conf.all.forwarding=1

# Load the changes
sudo sysctl --system

# Verify
sudo sysctl -a | grep forward
```

**Quick tip:** Never edit `/etc/sysctl.conf` directly — it gets overwritten by OS updates. Use `/etc/sysctl.d/99-sysctl.conf` instead.

---

### `iptables` — NAT Port Forwarding (DNAT + MASQUERADE)
**What it does:** Low-level packet filter and NAT rules. The `nat` table handles DNAT (redirect incoming) and MASQUERADE (rewrite source IP for return traffic).

**Syntax:**
```bash
sudo iptables -t nat -A PREROUTING <MATCH> -j DNAT --to-destination <IP:PORT>
sudo iptables -t nat -A POSTROUTING <MATCH> -j MASQUERADE
sudo iptables --list-rules --table nat
sudo iptables --flush --table nat
```

**Examples:**
```bash
# DNAT: forward TCP port 8080 arriving on enp1s0 from 10.0.0.0/24 → 192.168.0.5:80
sudo iptables -t nat -A PREROUTING \
    -i enp1s0 \
    -s 10.0.0.0/24 \
    -p tcp --dport 8080 \
    -j DNAT --to-destination 192.168.0.5:80

# MASQUERADE: rewrite source IP for traffic leaving enp6s0
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o enp6s0 -j MASQUERADE

# View current NAT rules
sudo iptables --list-rules --table nat

# Flush (clear) all NAT rules to start over
sudo iptables --flush --table nat

# Make rules persistent across reboots
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

**Quick tip:** MASQUERADE is required when doing DNAT — without it, return packets bypass the router and the connection breaks. DNAT alone is not enough.

---

### `nft` — Modern nftables (Replaces iptables)
**What it does:** The modern Netfilter front-end, with unified IPv4/IPv6 support and better performance.

**Examples:**
```bash
# View the full ruleset
sudo nft list ruleset

# Equivalent nftables rules for the iptables DNAT example above:
# table ip nat {
#     chain PREROUTING {
#         type nat hook prerouting priority dstnat; policy accept;
#         iifname "enp1s0" ip saddr 10.0.0.0/24 tcp dport 8080 dnat to 192.168.0.5:80
#     }
#     chain POSTROUTING {
#         type nat hook postrouting priority srcnat; policy accept;
#         oifname "enp6s0" ip saddr 10.0.0.0/24 masquerade
#     }
# }
```

**Quick tip:** nftables supports sets and maps for O(1) lookups — much faster than iptables when you have thousands of rules. On modern Ubuntu, UFW still uses iptables under the hood but nftables is the recommended replacement.

---

## Reverse Proxy & Load Balancer (NGINX)

### `nginx` — Install and Configure

**What it does:** Web server, reverse proxy, and load balancer.

**Examples:**
```bash
# Install
sudo apt install nginx

# Test config for syntax errors
sudo nginx -t

# Reload after config changes (no downtime)
sudo systemctl reload nginx.service

# Enable a site config
sudo ln -s /etc/nginx/sites-available/proxy.conf /etc/nginx/sites-enabled/

# Disable a site
sudo rm /etc/nginx/sites-enabled/proxy.conf
```

**Reverse Proxy Config (`/etc/nginx/sites-available/proxy.conf`):**
```nginx
server {
    listen 80;
    location / {
        proxy_pass http://1.1.1.1;
        include proxy_params;    # passes Host, X-Real-IP headers to backend
    }
}
```

**Load Balancer Config (`/etc/nginx/sites-available/lb.conf`):**
```nginx
upstream mywebservers {
    least_conn;                       # route to least-busy server
    server 1.2.3.4 weight=3;          # 3 out of 4 requests go here
    server 5.6.7.8:8081;              # custom port
    server 10.20.30.40 backup;        # only used if primaries fail
    # server 9.9.9.9 down;            # mark as under maintenance
}

server {
    listen 80;
    location / {
        proxy_pass http://mywebservers;
    }
}
```

```bash
# Enable the load balancer config
sudo ln -s /etc/nginx/sites-available/lb.conf /etc/nginx/sites-enabled/lb.conf
sudo nginx -t
sudo systemctl reload nginx.service
```

**Quick tip:** The `sites-available` / `sites-enabled` pattern is standard Debian/Ubuntu. Think of it as: `sites-available` = storage room, `sites-enabled` = what's actually plugged in. NGINX only reads `sites-enabled`. Reference examples in `/usr/share/doc/netplan/examples/` and general docs in `/usr/share/doc/`.

---

## Time Synchronization (`timedatectl`)

### `timedatectl`
**What it does:** Views and manages system time, timezone, and NTP synchronization.

**Syntax:**
```bash
timedatectl
timedatectl list-timezones
sudo timedatectl set-timezone <ZONE>
sudo timedatectl set-ntp true
timedatectl show-timesync
timedatectl timesync-status
```

**Examples:**
```bash
# View current time and NTP status
timedatectl

# List available timezones
timedatectl list-timezones | grep America

# Set timezone
sudo timedatectl set-timezone America/Los_Angeles

# Enable NTP sync
sudo timedatectl set-ntp true

# Install NTP sync daemon if missing
sudo apt install systemd-timesyncd

# Configure NTP servers
sudo vim /etc/systemd/timesyncd.conf
# [Time]
# NTP=0.us.pool.ntp.org 1.us.pool.ntp.org

# Restart NTP daemon
sudo systemctl restart systemd-timesyncd

# Check NTP daemon status
sudo systemctl status systemd-timesyncd.service

# Show detailed NTP sync info
timedatectl show-timesync
timedatectl timesync-status
```

**Quick tip:** If `NTP service: inactive` appears in `timedatectl` output, install and enable `systemd-timesyncd`. The config file is `/etc/systemd/timesyncd.conf` — add NTP server IPs there and restart the daemon.

---

## SSH Server (`sshd`, `sshd_config`)

### SSH Server Configuration
**Config file:** `/etc/ssh/sshd_config` (server) | `/etc/ssh/ssh_config` (client)

**Key directives:**

| Directive | Value | Meaning |
|-----------|-------|---------|
| `Port` | `22` | Listening port (change to harden) |
| `AddressFamily` | `inet` / `inet6` / `any` | IPv4, IPv6, or both |
| `PermitRootLogin` | `no` | Block direct root SSH login (best practice) |
| `PermitRootLogin` | `prohibit-password` | Root via key only (no password) |
| `PasswordAuthentication` | `no` | Keys only — disables brute-force attacks |
| `KbdInteractiveAuthentication` | `no` | Disables OTP/PAM challenge prompts |
| `X11Forwarding` | `yes` | Allow graphical apps over SSH tunnel |

**Examples:**
```bash
# View SSH server config
sudo vim /etc/ssh/sshd_config

# Check manual for all options
man sshd_config

# Reload SSH after config changes (no active connections dropped)
sudo systemctl reload ssh.service

# Check service name (Ubuntu vs Red Hat)
systemctl status ssh       # Ubuntu
systemctl status sshd      # Red Hat / CentOS

# Create a drop-in config file (survives OS updates)
sudo vim /etc/ssh/sshd_config.d/99-our-settings.conf
# Example: Port 2222
```

**Quick tip:** The daemon is `sshd`, the Ubuntu service is `ssh.service`, the config is `sshd_config`. On RHEL/CentOS the service is `sshd.service`. Always use `reload` not `restart` when SSH users are connected.

---

## SSH Client — Keys & Config

### `ssh-keygen`
**What it does:** Generates an SSH private/public key pair on the client.

**Syntax:**
```bash
ssh-keygen
ssh-keygen -R <HOST_IP>
```

**Examples:**
```bash
# Generate a new key pair (saved to ~/.ssh/id_ed25519 and id_ed25519.pub)
ssh-keygen

# List SSH client files
ls -la ~/.ssh

# Remove a stored server fingerprint (after server rebuild)
ssh-keygen -R 10.0.0.251

# Remove ALL stored fingerprints (lab/test environment reset)
rm ~/.ssh/known_hosts
```

---

### `ssh-copy-id`
**What it does:** Copies your public key to a remote server's `~/.ssh/authorized_keys`.

**Syntax:**
```bash
ssh-copy-id <USER>@<HOST>
```

**Examples:**
```bash
# Copy your public key to a remote server
ssh-copy-id dani@10.0.0.0

# Log in using the key (no password needed)
ssh dani@10.0.0.0

# View authorized keys on the remote server
cat ~/.ssh/authorized_keys

# Manually add additional team members' keys
vim ~/.ssh/authorized_keys
# Paste their id_ed25519.pub content on a new line

# Fix permissions (SSH refuses keys with wrong permissions)
chmod 600 ~/.ssh/authorized_keys
```

---

### SSH Client Config (`~/.ssh/config`)
**What it does:** Creates named shortcuts for SSH connections.

**File:** `~/.ssh/config`

**Example:**
```
Host ubuntu-vm
    HostName 10.0.0.186
    Port 22
    User jeremy
```

```bash
# Create the config
vim ~/.ssh/config

# Set correct permissions
chmod 600 ~/.ssh/config

# Now connect with alias instead of full command
ssh ubuntu-vm         # equivalent to: ssh jeremy@10.0.0.186

# System-wide client config (use drop-in file, not the main file)
sudo vim /etc/ssh/ssh_config.d/99-our-settings.conf
# Example: Port 2222
```

**Quick tip:** If you store custom settings directly in `/etc/ssh/ssh_config`, a system update may overwrite them. Always use a numbered drop-in file in `ssh_config.d/`.

---

## 🧪 Lab: SSH Keys & Secure Access

**Objective:** Set up passwordless SSH key authentication between two systems.

**Tasks:**
1. Generate an SSH key pair → `ssh-keygen`
2. Copy public key to the target server → `ssh-copy-id user@10.0.0.x`
3. Verify login works without password → `ssh user@10.0.0.x`
4. Create a `~/.ssh/config` alias for the connection
5. On the server: disable password auth in `sshd_config`, reload SSH
6. **(Challenge)** Manually add a second team member's key to `authorized_keys` without using `ssh-copy-id`

**Solution:**
```bash
# Task 1
ssh-keygen

# Task 2
ssh-copy-id jeremy@10.0.0.186

# Task 3
ssh jeremy@10.0.0.186

# Task 4
cat >> ~/.ssh/config <<EOF
Host prod-server
    HostName 10.0.0.186
    Port 22
    User jeremy
EOF
chmod 600 ~/.ssh/config
ssh prod-server

# Task 5
sudo vim /etc/ssh/sshd_config
# Set: PasswordAuthentication no
sudo systemctl reload ssh.service

# Task 6
cat /home/colleague/.ssh/id_ed25519.pub
vim ~/.ssh/authorized_keys    # paste the key on a new line
chmod 600 ~/.ssh/authorized_keys
```

---

## Appendix: Common Patterns & One-liners

```bash
# Extract IP address of eth0 (no subnet mask)
ip addr show dev eth0 | grep 'inet ' | awk '{print $2}' | cut -d'/' -f1

# Find default gateway IP only
ip route | grep default | awk '{print $3}'

# Find PID of service on a specific port
sudo ss -ltunp | grep ':<PORT>' | grep -oP 'pid=\K[0-9]+' | head -1

# Find process NAME on a specific port
sudo ss -ltunp | grep ':8080' | grep -oP '"\K[^"]+'

# Save all listening ports to a file
sudo ss -ltnup | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort -n | uniq

# Full NAT forwarding setup (one-shot)
sudo sysctl --system                             # apply forwarding kernel config
sudo iptables -t nat -A PREROUTING -i enp1s0 -s 10.0.0.0/24 -p tcp --dport 8080 -j DNAT --to-destination 192.168.0.5:80
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o enp6s0 -j MASQUERADE
sudo ufw allow 22 && sudo ufw enable
sudo ufw route allow from 10.0.0.0/24 to 192.168.0.5

# Detect which network management system is active
ls /etc/netplan              # Netplan present?
networkctl status            # systemd-networkd active?
ls /etc/sysconfig/network-scripts/  # Red Hat ifcfg style?

# Check NGINX config and reload (zero downtime)
sudo nginx -t && sudo systemctl reload nginx.service

# Enable and activate a Netplan site in one workflow
sudo vim /etc/netplan/00-mysettings.yaml
sudo chmod 600 /etc/netplan/00-mysettings.yaml
sudo netplan try && sudo netplan apply
```

---

> **Disclaimer:** Generated from course notes. Always consult `man <command>` for the authoritative reference.
> Key man pages: `man ip`, `man netplan`, `man ufw`, `man iptables`, `man sshd_config`, `man ssh_config`, `man nft`
