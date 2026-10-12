# 06 — Practical Exercise: Configuring IP, Gateway, DNS and Security on Devices

> Scope: hands-on configuration. Every section gives the **command**, an **explanation of what it
> does**, and how to **verify** it. Vendor-agnostic principles first, then Cisco IOS, Linux,
> Windows, and a generic edge firewall.
>
> A full worked lab topology is at the end (§06.11) — an end-to-end brief you can complete as
> the exercise.

---

## 6.1 The Five Things Every Device Needs

| # | Setting | What it does | Why it breaks if wrong |
|---|---|---|---|
| 1 | **IP address** | Identifies the interface on the network | Device is invisible; no traffic |
| 2 | **Subnet mask / prefix** | Tells the device which addresses are local (on-link) | Sends local traffic to the gateway; traffic loops |
| 3 | **Default gateway** | Where to send traffic for destinations not on-link | Can reach local only; "destination host unreachable" |
| 4 | **DNS server** | Translates names to addresses | Everything by name fails (browsing, updates) while IPs work |
| 5 | **Security** | Auth, ACLs, encrypted management, hardened services | Full compromise; spoofing; DoS |

### The key concept: on-link vs off-link

The device uses **mask + address** to decide whether a destination is local:

```
  Destination IP      Subnet mask      Destination AND Mask   Local?      Action
  192.168.1.50        255.255.255.0    192.168.1.0           YES         ARP for 192.168.1.50
  192.168.2.50        255.255.255.0    192.168.2.0           NO          send to gateway
  10.0.0.5            255.255.255.0    10.0.0.0              NO          send to gateway
```

This is the single most-tested concept in device configuration. Get the mask wrong and the
device will try to reach local hosts through the gateway (or send local traffic out of the
interface), producing bizarre, hard-to-debug failures.

### Special-case: loopback and management interfaces

* **Loopback (Cisco) / `lo` (Linux)** — always up, never goes down when a cable is pulled, ideal
  for a management/routing ID address.
* **Out-of-band management** — a separate interface on a dedicated management VLAN/network,
  physically separate from user traffic.

---

## 6.2 Layer 2 First: VLANs

VLANs must exist **before** you can address a subnet.

```
  Switch (access ports)                          Switch (access ports)
  +----------------------+                       +----------------------+
  | Fa0/1  -> VLAN 10    |                       | Fa0/5  -> VLAN 10    |
  | Fa0/2  -> VLAN 10    |                       | Fa0/6  -> VLAN 20    |
  | Fa0/3  -> VLAN 20    |                       | Fa0/7  -> VLAN 20    |
  +----------|-----------+                       +-----------+----------+
             |                                            |
             +---------------- Trunk -----------------------+
                      802.1Q tagged
```

### Cisco IOS — create VLANs and a trunk

```
Switch# vlan database
Switch(vlan)#  vlan 10
Switch(vlan)#  vlan 20
Switch(vlan)#  exit

! (or the newer global form)
Switch# vlan 10
Switch# vlan 20

! --- configure the trunk ---
Switch# interface GigabitEthernet0/1
Switch(config-if)# description TRUNK-TO-SWITCH-B
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20
Switch(config-if)# switchport trunk native vlan 1     ! match on both ends
Switch(config-if)# no shutdown

! --- configure an access port ---
Switch# interface GigabitEthernet0/2
Switch(config-if)# description PC1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 1
Switch(config-if)# switchport port-security violation restrict
Switch(config-if)# switchport port-security mac-address sticky
Switch(config-if)# spanning-tree portfast
Switch(config-if)# spanning-tree bpduguard enable
Switch(config-if)# no shutdown
```

**Port security** (very important hardening):
* Limits the number of MAC addresses learned on the port (default 1).
* `violation restrict` — drops excess frames and increments a counter (recommended).
* `violation protect` — drops silently (dangerous: no log).
* `violation shutdown` — shuts the port (secure but disruptive).
* `mac-address sticky` — learns one MAC and remembers it across reboots.

**BPDU Guard** shuts the port if a switch receives a BPDU on an access port — blocking rogue
switch attacks and accidental loop creation.

**Verification:**
```
Switch# show vlan brief
Switch# show interfaces trunk
Switch# show interfaces gigabitEthernet0/2 switchport
Switch# show port-security interface gigabitEthernet0/2
Switch# show spanning-tree interface gigabitEthernet0/2
```

### VLAN 1 — delete it

`VLAN 1` is the **default VLAN** and **cannot be deleted** on most platforms. Best practice is
to move all user ports to other VLANs and leave VLAN 1 unused, or (on some platforms) rename it
and use it for management. A device on VLAN 1 is reachable by anyone who plugs in — a real risk.

---

## 6.3 IP, Mask and Default Gateway — Cisco IOS

### 6.3.1 Physical and logical interfaces

```
  Physical  = a real port on the box (Ethernet, serial)
  Logical   = the SVI (VLAN interface) or subinterface that carries the IP
```

### 6.3.2 Router-on-a-stick (the classic lab)

```
                       R1
              Gi0/1.10 | Gi0/1.20
           10.1.1.1/24  | 10.1.2.1/24
                       |
     PC1 ---------- SW1 --------- PC2
   10.1.1.10        trunk         10.1.2.10
```

```
Router(config)# interface GigabitEthernet0/1
Router(config-if)# description ROUTER-ON-A-STICK
Router(config-if)# no ip address
Router(config-if)# no shutdown

! --- create the subinterfaces (encapsulation is REQUIRED before an IP) ---
Router(config)# interface GigabitEthernet0/1.10
Router(config-subif)# encapsulation dot1q 10
Router(config-subif)# ip address 10.1.1.1 255.255.255.0

Router(config)# interface GigabitEthernet0/1.20
Router(config-subif)# encapsulation dot1q 20
Router(config-subif)# ip address 10.1.2.1 255.255.255.0

! --- or use SVIs on a Layer-3 switch instead (preferred) ---
Switch(config)# interface vlan 10
Switch(config-if)# ip address 10.1.1.1 255.255.255.0
Switch(config-if)# no shutdown
```

> **Common error:** assigning `encapsulation dot1q 10` to VLAN **1**. Cisco will refuse it —
> VLAN 1 is untagged/native and needs no encapsulation.

### 6.3.3 The default gateway on a router

A **router does not need a default gateway for its own connected networks** — those are
directly connected. It needs routes for everything else.

```
Router(config)# ip route 0.0.0.0 0.0.0.0 192.168.1.1
                        ^dest  ^mask  ^next-hop
```

### 6.3.4 Complete configuration block

```
enable
configure terminal

hostname R1

! ---------- AAA / local user ----------
username admin privilege 15 secret Str0ngP@ssw0rd!
username backup privilege 5 secret B@ckupPass!
enable secret Cl0ck$tr0ng
enable algorithm-type scrypt
service password-encryption
no service password-recovery        ! disables the "enable" backdoor via physical access
ip domain-name lab.example.com
crypto key generate rsa modulus 2048

! ---------- logging ----------
logging buffered 16384 informational
logging monitor informational
no logging console
exec-timeout 10 0
line console 0
 logging synchronous
 exec-timeout 10 0
line vty 0 15
 exec-timeout 10 0
 transport input ssh
 login local

! ---------- interfaces ----------
interface Loopback0
 description MANAGEMENT-AND-OSPF-RID
 ip address 10.255.0.1 255.255.255.255
 ip ospf 1 area 0

interface GigabitEthernet0/0
 description LAN-10
 ip address 10.1.1.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown

interface GigabitEthernet0/1
 description LAN-20
 ip address 10.1.2.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown

interface GigabitEthernet0/2
 description UPLINK-TO-ISP
 ip address 192.168.1.2 255.255.255.0
 ip nat outside
 no shutdown

! ---------- default route ----------
ip route 0.0.0.0 0.0.0.0 192.168.1.1

! ---------- DNS resolution on the device ----------
ip domain lookup
ip domain name lab.example.com
ip name-server 8.8.8.8
ip name-server 1.1.1.1

! ---------- hardening ----------
ip cef
no ip source-route
no ip finger
no ip http server
no ip http secure-server
no service tcp-small-servers
no service udp-small-servers
service timestamps log datetime msec
banner login ^C
  ====================================================================
   AUTHORISED ACCESS ONLY. All activity is logged and monitored.
  ====================================================================
^C

end
write memory
```

### 6.3.5 Verification — the essential command set

```
R1# show ip interface brief        ! IS EVERY INTERFACE UP AND CORRECT?
     Interface              IP-Address      OK? Method Status    Protocol
     GigabitEthernet0/0     10.1.1.1        YES manual up        up
     GigabitEthernet0/1     10.1.2.1        YES manual up        up
     GigabitEthernet0/2     192.168.1.2     YES manual up        up
     Loopback0              10.255.0.1      YES manual up        up

R1# show ip interface GigabitEthernet0/0
     ... line protocol is up, Internet address is 10.1.1.1, ...
     Outgoing access list is NOT set
     ARP type: ARPA, ARP Timeout 04:00:00

R1# show ip route
     Codes: L - local, C - connected, S - static, R - RIP, O - OSPF
     Gateway of last resort is 192.168.1.1 to network 0.0.0.0
     S*    0.0.0.0/0 [1/0] via 192.168.1.1
     C     10.1.1.0/24 is directly connected, GigabitEthernet0/0
     L     10.1.1.1/32 is directly connected, GigabitEthernet0/0
     O     10.1.3.0/24 [110/2] via 10.1.2.2, 00:31:22, GigabitEthernet0/1

R1# ping 10.1.2.10
     64 bytes from 10.1.2.10: icmp_seq=1 ttl=64 time=0.5 ms
R1# ping 8.8.8.8 source Loopback0        ! test a specific source
R1# ping 8.8.8.8 repeat 5 size 1400 df    ! Path MTU Discovery test
R1# arp 10.1.2.10                        ! is the MAC resolving?
R1# traceroute 8.8.8.8                    ! path
R1# telnet 10.1.2.10 / show cdp neighbors / show ip ospf neighbor
```

**Reading `show ip route`:**

```
  S*    0.0.0.0/0 [1/0] via 192.168.1.1
  ^    ^         ^  ^^   ^
  |    |         |  ||   next hop
  |    |         |  |+--- metric (to next hop)
  |    |         |  +---- administrative distance
  |    |         +------- destination prefix
  |    +----------------- S = static, * = default route
  +---------------------- route source
```

**Common interface states to diagnose:**

| State | Meaning | Cause |
|---|---|---|
| `down/down` | No link, no protocol | Cable unplugged, port disabled, wrong port |
| `up/down` | Link is up but no IP/protocol | Missing `no shutdown`, or the IP is not configured |
| `administratively down` | Port disabled | Missing `no shutdown` |
| `up/up` | Working | — |

---

## 6.4 DHCP — Cisco IOS

### 6.4.1 DHCP server on a router

```
ip dhcp excluded-address 10.1.1.1 10.1.1.20      ! protect static/gateway addresses

ip dhcp pool LAN10
 network 10.1.1.0 255.255.255.0
 default-router 10.1.1.1                        ! <-- the DEFAULT GATEWAY handed to clients
 domain-name lab.example.com
 dns-server 8.8.8.8 1.1.1.1                      ! <-- DNS handed to clients
 lease 7 0 0                                     ! 7 days
 server-name DHCP-R1
!
ip dhcp pool LAN20
 network 10.1.2.0 255.255.255.0
 default-router 10.1.2.1
 dns-server 8.8.8.8 1.1.1.1
 lease 7 0 0
!
! optional global options
ip dhcp domain-name lab.example.com
```

### 6.4.2 DHCP client and relay

```
! --- client ---
interface GigabitEthernet0/3
 ip address dhcp

! --- relay agent (router is between the clients and the server) ---
interface GigabitEthernet0/4
 ip address 10.1.1.2 255.255.255.0
 ip helper-address 10.255.0.10      ! the DHCP server's address
```

```
R1# show ip dhcp binding             ! who has what
R1# show ip dhcp pool LAN10          ! usage statistics
R1# debug ip dhcp server events      ! troubleshooting (remove afterwards!)
```

### 6.4.3 Option 43 (PXE / phone provisioning)

```
ip dhcp pool VOICE
 network 10.1.3.0 255.255.255.0
 default-router 10.1.3.1
 option 43 ascii 00:1a:2b:3c:4d:5e:00:11:22:33:44:55     ! TFTP server MAC
```

---

## 6.5 DNS — Configuring Name Resolution

### 6.5.1 Cisco IOS

```
ip domain lookup                ! enable DNS lookup (ON by default)
ip domain name lab.example.com  ! append this when a name has no dot
ip name-server 8.8.8.8          ! up to 6 servers
ip name-server 1.1.1.1

! send queries over UDP 53 to specific servers
ip domain name-server 8.8.8.8

! per-interface resolver
interface GigabitEthernet0/0
 ip domain resolver 8.8.8.8 1.1.1.1
```

```
R1# ping www.google.com
R1# nslookup www.google.com
R1# show ip domain lookup
```

> **Exam point:** the difference between "pinging a name fails" and "pinging the IP works" is
> almost always a **missing `ip name-server`**. A router cannot resolve names unless you
> configure a DNS server explicitly.

### 6.5.2 Linux — `systemd-resolved` (modern) vs `/etc/resolv.conf` (classic)

```bash
# --- classic: /etc/resolv.conf ---
sudo tee /etc/resolv.conf <<'EOF'
search lab.example.com
nameserver 8.8.8.8
nameserver 1.1.1.1
options timeout:2 attempts:2 rotate
EOF

# --- modern: systemd-resolved ---
sudo resolvectl dns eth0 8.8.8.8 1.1.1.1
sudo resolvectl domain eth0 lab.example.com
sudo resolvectl status eth0
```

Netplan (`/etc/netplan/01-netcfg.yaml`):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.1.1.10/24
      routes:
        - to: default
          via: 10.1.1.1            # <-- DEFAULT GATEWAY
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
        search: [lab.example.com]
  renderer: networkd
```
```
sudo netplan apply
```

---

## 6.6 Linux — Full IP, Gateway, DNS and Security Configuration

### 6.6.1 Address and route

```bash
# --- inspect current state ---
ip -br addr
ip route
cat /etc/resolv.conf

# --- set the address ---
sudo ip addr add 10.1.1.10/24 dev eth0
sudo ip addr flush dev eth0
sudo ip addr add 10.1.1.10/24 dev eth0

# --- set the DEFAULT GATEWAY ---
sudo ip route add default via 10.1.1.1 dev eth0
sudo ip route replace default via 10.1.1.1 dev eth0

# --- or, persistently ---
sudo ip addr add 192.168.50.10/24 dev eth0 label eth0:1
sudo ip route add default via 192.168.50.1 dev eth0:1

# --- static ARP entry ---
sudo ip neigh add 192.168.50.1 lladdr 00:1a:2b:3c:4d:5e dev eth0 nud permanent

# --- useful extras ---
sudo ip link set eth0 mtu 9000
sudo ip link set eth0 up
```

### 6.6.2 The permanent way — `nmcli` (NetworkManager)

```bash
nmcli con show
sudo nmcli con add type ethernet ifname eth0 con-name static-eth0 \
     ipv4.method manual \
     ipv4.addresses 10.1.1.10/24 \
     ipv4.gateway 10.1.1.1 \
     ipv4.dns "8.8.8.8,1.1.1.1" \
     ipv4.dns-search "lab.example.com" \
     ipv6.method auto
sudo nmcli con up static-eth0
sudo nmcli con modify static-eth0 ipv4.dns "10.1.1.53" ipv4.ignore-auto-dns yes
sudo nmcli dev show eth0 | grep -E 'IP4|GATEWAY|DNS'
```

### 6.6.3 Linux firewall — `nftables` (modern) and `ufw` (simple)

```bash
# ---------- nftables ----------
sudo tee /etc/nftables.conf <<'EOF'
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        ct state established,related accept     # let return traffic in
        ct state invalid drop
        iif "lo" accept                         # loopback is always trusted

        ip protocol icmp icmp type { echo-request, destination-unreachable,
                                     time-exceeded, parameter-problem } accept
        ip6 nexthdr ipv6-icmp icmpv6-type { echo-request, destination-unreachable,
                     packet-too-big, time-exceeded, nd-neighbor-solicit,
                     nd-neighbor-advert, nd-router-advert, router-solicit } accept
                                     # NOTE: do NOT blanket-block ICMP/ICMPv6 - it breaks
                                     # Path MTU Discovery and IPv6 entirely

        tcp dport 22 accept                      # SSH
        tcp dport 80 accept                      # HTTP
        tcp dport { 443 5432 } accept            # HTTPS, PostgreSQL

        # rate limit SSH brute force
        tcp dport 22 ct state new limit rate 10/minute burst 5 packets accept

        ip saddr { 10.1.1.0/24 } accept           # trusted LAN
        log prefix "nft-input-drop " level info
    }

    chain forward { type filter hook forward priority 0; policy drop;
        ct state established,related accept
        ip saddr 10.1.1.0/24 ip daddr 10.2.2.0/24 accept
    }

    chain output { type filter hook output priority 0; policy accept; }
}
EOF
sudo nft -c -f /etc/nftables.conf     # CHECK SYNTAX FIRST
sudo nft -f /etc/nftables.conf
sudo systemctl enable nftables
sudo nft list ruleset

# ---------- ufw (simple wrapper) ----------
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.1.1.0/24 to any
sudo ufw allow 22/tcp
sudo ufw limit 22/tcp
sudo ufw enable
sudo ufw status verbose
```

### 6.6.4 Linux hardening

```bash
# --- SSH ---
sudo install -d -m 755 /etc/ssh
sudo ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N ""
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf <<'EOF'
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers deploy
AllowGroups sre
Banner /etc/ssh/banner.txt
EOF
sudo systemctl restart sshd

# --- firewall on SSH itself ---
sudo ufw limit 22/tcp

# --- kernel/network hardening in sysctl ---
sudo tee /etc/sysctl.d/99-net-hardening.conf <<'EOF'
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 1          # reverse-path filtering: drops spoofed source
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1 # ignore broadcast ping (smurf)
net.ipv4.tcp_syncookies = 1
net.ipv6.conf.all.accept_ra = 0
EOF
sudo sysctl --system

# --- disable unused services ---
sudo systemctl disable --now telnet
sudo systemctl disable --now rsh-server

# --- updates ---
sudo apt update && sudo apt upgrade
sudo apt install unattended-upgrades && sudo dpkg-reconfigure -plow unattended-upgrades
```

---

## 6.7 Windows

### 6.7.1 PowerShell (modern)

```powershell
# --- inspect ---
Get-NetIPConfiguration
Get-NetIPAddress -AddressFamily IPv4
Get-DnsClientServerAddress
Get-NetRoute -DestinationPrefix 0.0.0.0/0

# --- set a static IPv4 address, gateway, and DNS in one shot ---
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 10.1.1.10 -PrefixLength 24 -DefaultGateway 10.1.1.1

# --- set DNS ---
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 8.8.8.8,1.1.1.1
Set-DnsClient -InterfaceAlias "Ethernet" -Domain "lab.example.com"
Set-DnsClientGlobalSetting -SuffixSearchList @("lab.example.com","corp.local")

# --- remove an address ---
Remove-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.1.1.10 -Confirm:$false

# --- DHCP ---
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled

# --- routes ---
New-NetRoute -DestinationPrefix "10.2.2.0/24" -InterfaceAlias "Ethernet" `
  -NextHop 10.1.1.254 -RouteMetric 1
Remove-NetRoute -DestinationPrefix "10.2.2.0/24" -Confirm:$false

# --- test ---
Test-NetConnection -ComputerName 8.8.8.8 -Port 443
Test-NetConnection -ComputerName www.google.com -Port 443
Resolve-DnsName www.google.com
Test-NetConnection 8.8.8.8 -DiagnoseRouting
```

### 6.7.2 Legacy `netsh` and `ipconfig`

```cmd
netsh interface ip set address "Ethernet" static 10.1.1.10 255.255.255.0 10.1.1.1
netsh interface ip set dns "Ethernet" static 8.8.8.8
netsh interface ip add dns "Ethernet" 1.1.1.1 index=2
netsh interface ip set address "Ethernet" source=dhcp
netsh interface ip set dnsservers "Ethernet" source=dhcp

ipconfig /all
ipconfig /release
ipconfig /renew
ipconfig /flushdns
route print
tracert 8.8.8.8
netsh advfirewall show allprofiles
netsh advfirewall set allprofiles state on
netsh advfirewall firewall add rule name="SSH" dir=in action=allow protocol=TCP localport=22
```

### 6.7.3 Windows Defender Firewall profiles

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
Set-NetFirewallProfile -Profile Domain -DefaultInboundAction Block -DefaultOutboundAction Allow
Set-NetFirewallProfile -Profile Public  -DefaultInboundAction Block -DefaultOutboundAction Block
Get-NetFirewallProfile
```

> **Tip:** set the network **profile** correctly. A server on a "Public" profile with a strict
> policy can behave very differently from one on a "Private" profile.

---

## 6.8 NAT and Port Forwarding

### 6.8.1 Cisco IOS

```
ip nat inside
ip nat outside

! --- one-to-many (PAT) ---
ip nat inside source static 10.1.1.100 192.168.1.100
access-list 1 permit 10.1.1.0 0.0.0.7
ip nat inside source list 1 interface GigabitEthernet0/2 overload

! --- overload uses a pool ---
ip nat pool OVERLOAD 192.168.1.100 192.168.1.110 netmask 255.255.255.0
ip nat inside source list 1 pool OVERLOAD overload

! --- port forwarding (static NAT + a redirect) ---
ip nat inside source static tcp 10.1.1.100 80 192.168.1.100 8080
ip nat inside source static tcp 10.1.1.100 443 192.168.1.100 8443
ip nat inside source static udp 10.1.1.110 53 192.168.1.100 53

ip route 0.0.0.0 0.0.0.0 192.168.1.1
```
```
R1# show ip nat translations
R1# show ip nat statistics
R1# debug ip nat
```

### 6.8.2 Linux

```bash
# --- PAT (masquerade) ---
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# --- explicit SNAT with a pool ---
sudo iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 203.0.113.7

# --- port forwarding (DNAT) ---
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
     -j DNAT --to-destination 10.1.1.100:80
sudo iptables -A FORWARD -p tcp -d 10.1.1.100 --dport 80 -j ACCEPT

# --- hairpinning (internal clients hitting the public IP) ---
sudo iptables -t nat -A OUTPUT -d 192.168.1.100 -p tcp --dport 80 \
     -j DNAT --to-destination 10.1.1.100:80
sudo iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -d 10.1.1.100 -p tcp --dport 80 \
     -j MASQUERADE

sudo ufw route allow proto tcp from any to 10.1.1.100 port 80 comment 'reverse proxy'
```

### 6.8.3 Home routers (generic)

```
Port Forwarding:  external 8080/TCP  ->  10.1.1.100:80
Static (reserved) leases: 10.1.1.100 -> MAC 00:1a:2b:3c:4d:5e   (so NAT rules never break)
DMZ host: 10.1.1.50  (everything to this host - dangerous)
UPnP / NAT-PMP: allow the client to create its own mappings
```

---

## 6.9 Security Hardening Checklist

### 6.9.1 Devices

| Area | Control | Cisco | Linux | Windows |
|---|---|---|---|---|
| **Admin access** | SSH only, no Telnet | `transport input ssh` | `PasswordAuthentication no` | RDP restricted by firewall |
| **Credentials** | Encrypted secrets | `enable secret`, `service password-encryption` | `chpass -e`, `/etc/shadow` | LAPS |
| **Privilege** | No shared accounts; named users | `username <name> privilege 15` | `sudo`, named users, `sudoers` | named admins |
| **Management** | Out-of-band / management VLAN | separate SVI, ACL | `ansible`/`mgmt` VLAN | jump host |
| **Unused services** | Disable | `no ip http server`, `no service finger` | `systemctl disable` | — |
| **Logging** | Centralised syslog | `logging host 10.0.0.5` | `rsyslog`/`journald` → SIEM | Event Log → SIEM |
| **Access control** | ACLs restricting who can reach the mgmt plane | `access-class 10 in vty 0 15` | `ufw allow from 10.0.0.0/8` | firewall rules |
| **Control plane** | Protect against DoS | `control-plane`, CoPP | `sysctl`, `nft` | — |
| **Banner** | Legal warning | `banner login` | `/etc/ssh/banner.txt` | — |
| **Timeout** | Idle lockout | `exec-timeout 10 0` | `ClientAliveCountMax 2` | RDP idle timeout |

### 6.9.2 ACLs — the concept

An **ACL is an ordered list of match conditions**. On a Cisco device:

* **Numbered ACLs**: 1–99 (IP), 100–199 (protocol), 1000–1999 (extended).
* **Named ACLs**: recommended (easier to read, and you can edit specific lines later).
* A rule that **permits** traffic and then **denies** everything else is a good practice —
  an empty ACL on an interface permits nothing.

```
ip access-list extended MGMT-IN
 permit tcp 10.1.1.0 0.0.0.255 any eq 22
 permit tcp 10.1.1.0 0.0.0.255 any eq 443
 permit icmp any any echo-reply
 deny   ip any any log

line vty 0 15
 access-class MGMT-IN in
```

### 6.9.3 ACLs for infrastructure devices

```
! Protect the router's CPU from unnecessary traffic
ip access-list extended BLOCK-DANGEROUS
 deny   ip any 224.0.0.0/4        ! no routing to multicast via this box
 deny   ip any 127.0.0.0/8        ! loopback is local only
 deny   ip any 192.168.0.0/24     ! don't leak RFC1918 into the internet
 deny   ip any 172.16.0.0/12
 deny   ip any 10.0.0.0/8
 permit tcp any any established
 deny   ip any any

interface GigabitEthernet0/2
 ip access-group BLOCK-DANGEROUS in
```

---

## 6.10 Verification and Troubleshooting — a systematic method

### 6.10.1 The OSI bottom-up method

Always work bottom-up. Most "network problems" are Layer 1 or 2.

```
Layer 1  Is the link up?           show interfaces | ethtool eth0 | ip link
Layer 2  Is the right VLAN?        show interfaces trunk | show vlan | arp -a
Layer 3  Is the IP/mask right?     show ip interface brief | ip addr
         Is the gateway reachable?  ping <gateway> (on-link first)
         Is routing right?         show ip route | ip route
Layer 4  Is the service listening?  netstat -tlnp | show tcp brief
         Is the port open?          nc -zv host port | Test-NetConnection
Name    Does DNS resolve?           dig | nslookup | Resolve-DnsName
```

### 6.10.2 Layer 1 checks (Cisco)

```
show interfaces GigabitEthernet0/0
!  --- look for ---
   GigabitEthernet0/0 is up, line protocol is up
   Full-duplex, 1000Mb/s, media type is 10/100/1000BaseTX
   0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
   5 minute output rate 1000 bits/sec
```

Red flags: `runts`, `giants`, `CRC`, `input errors`, `output errors`, `collision`,
`duplex` mismatch (`(async)` in the output).

### 6.10.3 The `ping` decision tree

```
ping 8.8.8.8 fails, but ping 10.1.1.1 (gateway) works:
   -> the device is fine; the problem is upstream (ISP, route, ACL, or the far side).

ping to the gateway fails:
   -> check: link up? correct IP/mask? right VLAN? correct gateway address?

ping <own subnet host> fails but the gateway works:
   -> Layer 2 problem: VLAN mismatch, wrong trunk, port security, duplicate IP.

ping by name fails but by IP works:
   -> DNS misconfiguration: no nameserver, wrong search domain, or the DNS server is unreachable.

ping works but TCP/443 fails:
   -> Layer 3+4: ACL blocking, no NAT/static route, service not listening, wrong port,
      or an asymmetric return path.
```

### 6.10.4 Extended pings

```
R1# ping
  Protocol [ip]:
  Target [0.0.0.0]:
  Repeat count [5]:
  Datagram size [100]:
  Timeout in seconds [2]:
  Extended commands [n]: y
  Source address or interface:
  Type of service [0]:
  Set DF bit in IP header? [no]: y
  Number of timestamps [3]:
  Packet filter [none]:

! --- test a specific source and MTU ---
R1# ping 10.1.2.100 source Loopback0 size 1400 df repeat 10
```

### 6.10.5 Useful extra diagnostics

```
R1# traceroute 8.8.8.8
R1# traceroute 8.8.8.8 source Loopback0 timeout 1 probe 3
R1# show cdp neighbors detail          ! who is on the other end of this cable
R1# show lldp neighbors
R1# show spanning-tree
R1# show etherchannel summary
R1# show arp
R1# show ip ospf database
R1# show logging
```

```bash
# Linux
ip route get 8.8.8.8          # how would a packet to this address be forwarded?
ss -tulpn                     # what is listening, on which port
tcpdump -ni eth0 'icmp or port 53 or tcp port 22'
dig +trace www.example.com    # trace DNS resolution
traceroute -n 8.8.8.8
mtr 8.8.8.8                   # continuous traceroute with loss statistics
nmap -sn 10.1.1.0/24          # ping sweep
```

> `mtr` is the single most useful diagnostic tool: it shows **per-hop loss and latency over time**,
> which immediately distinguishes a real route problem from a single dropped ICMP packet.

---

## 6.11 Lab Brief — Full Configuration Exercise

### 6.11.1 Topology

```
                          Internet (simulated)
                                 |
                             [ ISP ]
                                 |  203.0.113.1/30
  ===============================|=================================
                                 |  203.0.113.2/30
                            +-----------+
                            |    R1     |  edge router: NAT + firewall + DHCP
                            | 10.1.1.1 |
                            +----+-+----+
                                 | |   (Gi0/2)
                Gi0/0 10.1.1.0/24| |10.1.2.0/24 Gi0/1
                                 | |
                         +-------+-------+
                         |      SW1      |  trunk between SW1 and SW2
                         +---+-------+---+
                             |       |
                        Fa0/1|       |Fa0/3
                    VLAN 10  |       |  VLAN 20
                             |       |
                          [PC1]   +-----------+  Gi0/3
                          10.1.1.10 10.1.2.1  +-----------+
                                    VLAN 20  |   SW2    |
                                             +-----+-----+
                                                   |    |
                                              Fa0/1|    |Fa0/2
                                          VLAN 20|    |VLAN 30
                                                   |    |
                                               [PC2]  [SRV]
                                               10.1.2.10 10.1.3.10
```

### 6.11.2 Addressing plan

| Device / Interface | Address | Mask | Notes |
|---|---|---|---|
| ISP | 203.0.113.1 | /30 | ISP side of the WAN |
| R1 Gi0/2 (WAN) | 203.0.113.2 | /30 | `ip nat outside` |
| R1 Gi0/0 (LAN 10) | 10.1.1.1 | /24 | gateway for VLAN 10, `ip nat inside` |
| R1 Gi0/1 (LAN 20) | 10.1.2.1 | /24 | gateway for VLAN 20, `ip nat inside` |
| R1 Loopback0 | 10.255.0.1 | /32 | management |
| SW1 Loopback0 | 10.255.0.11 | /32 | management |
| SW2 Loopback0 | 10.255.0.12 | /32 | management |
| SW1 Fa0/1 | access VLAN 10 | | PC1 |
| SW1 Fa0/3 | access VLAN 20 | | trunk to SW2 |
| SW2 Fa0/1 | access VLAN 20 | | PC2 |
| SW2 Fa0/2 | access VLAN 30 | | SRV |
| DHCP pool | 10.1.3.50 – 10.1.3.200 | /24 | gateway 10.1.3.1 |
| DNS | 10.255.0.1 (R1) → 8.8.8.8 | | |

### 6.11.3 Tasks

**Part A — VLANs and trunks**
1. Create VLANs 10, 20 and 30 on SW1 and SW2.
2. Configure SW1–SW2 as an 802.1Q trunk allowing VLANs 20 and 30.
3. Assign access ports; enable `portfast` and `bpduguard` on all access ports.
4. Verify: `show vlan brief`, `show interfaces trunk`.

**Part B — First-hop addressing**
5. Assign static IPs to all switch management addresses (Loopback0).
6. Configure the default gateway on each switch (`ip default-gateway 10.1.1.1`).
7. Verify management reachability: `ping 10.255.0.11` from R1.

**Part C — Routing**
8. Configure the three VLAN gateway SVIs on R1 with the addresses above.
9. Add a default route on R1 to 203.0.113.1.
10. On R1, configure a static route for 203.0.113.0/30 — or rely on connected status.
11. Verify: `show ip interface brief`, `show ip route`.

**Part D — DHCP and DNS**
12. Configure a DHCP pool for VLAN 30 with the correct default router and DNS server.
13. Configure PC2 and SRV to use DHCP.
14. On R1, configure `ip domain lookup`, `ip domain name` and `ip name-server`.
15. Verify: `show ip dhcp binding`, `ping www.google.com` from R1.

**Part E — NAT and internet access**
16. Configure PAT overload for 10.1.1.0/24, 10.1.2.0/24 and 10.1.3.0/24 out Gi0/2.
17. Add a port forward for SRV: external 8080/TCP → 10.1.3.10:80.
18. Verify: `show ip nat translations`; from PC1 `ping 8.8.8.8`.

**Part F — OSPF**
19. Enable OSPF area 0 on R1 using Loopback0 as the router ID.
20. Add `network` statements for the loopback and all LAN interfaces.
21. Verify: `show ip ospf neighbor`, `show ip route | include O`.

**Part G — Security hardening**
22. Set `enable secret`, create a local admin user, configure SSH and disable Telnet.
23. Disable HTTP, finger and small servers.
24. Write an extended ACL that permits SSH and HTTPS from VLAN 10 only, and denies everything
    else into R1.
25. Apply `access-class` to `line vty 0 15`.
26. On SW1/SW2, set a console password and `service password-encryption`.
27. Enable port security and BPDU guard on every access port.

**Part H — Verification (checklist)**

```
[ ] show ip interface brief      -> all interfaces up/up with the correct addresses
[ ] show vlan brief              -> VLANs 10, 20, 30 exist
[ ] show interfaces trunk        -> SW1<->SW2 carrying 20 and 30
[ ] ping from each VLAN          -> cross-VLAN traffic works
[ ] show ip dhcp binding         -> PC2 and SRV received correct leases
[ ] ping 8.8.8.8 from each VLAN  -> internet works via NAT
[ ] ping www.google.com           -> DNS works
[ ] show ip nat translations     -> dynamic translations appear
[ ] show ip route                 -> O routes present
[ ] telnet 10.255.0.11            -> FAILS (SSH only)
[ ] ssh admin@10.255.0.11         -> SUCCEEDS
[ ] telnet 10.1.1.5 from VLAN 20  -> BLOCKED by the ACL
```

### 6.11.4 Grading rubric

| Criteria | Marks |
|---|---|
| VLANs and trunking correct | 10 |
| IP addressing correct on all devices | 15 |
| DHCP handing out the correct gateway, mask and DNS | 10 |
| DNS resolution working from all VLANs | 5 |
| Routing correct, cross-VLAN traffic works | 10 |
| NAT / PAT working; port forward functional | 10 |
| OSPF adjacency up and routes learned | 10 |
| SSH-only management; Telnet and HTTP disabled | 10 |
| ACLs implemented and verified | 10 |
| Logs/verification output provided as evidence | 10 |
| **Total** | **100** |

### 6.11.5 Reflection questions

1. You set PC1's mask to `/25` instead of `/24`. PC1 can ping its own gateway but not other
   VLANs. Explain exactly why, using the on-link/off-link decision from §6.1.
2. `ping www.google.com` fails but `ping 8.8.8.8` works. List all four possible causes in the
   order you would test them.
3. You applied a `deny ip any any` extended ACL inbound on R1's LAN interface and now nothing
   works. Explain why, and how to design a safe version.
4. PC2 gets a DHCP address of `169.254.x.x`. What went wrong, and how do you find the DHCP server
   from the client side?
5. Why does blocking all ICMP break IPv6 completely but only partially break IPv4?
6. You add `ip nat inside` on Gi0/0 but forget `ip nat outside` on Gi0/2. Explain what breaks.
7. PC1 can reach the internet, but the web server on SRV cannot be reached from outside. List the
   three most likely causes.

---

## 6.12 Quick reference card

```
SHOW (Cisco)                       LINUX                         WINDOWS
--------------------------------   ---------------------------   --------------------------
show ip interface brief            ip -br addr                   Get-NetIPConfiguration
show ip interface                  ip addr show eth0             Get-NetIPAddress
show ip route                      ip route                      Get-NetRoute
show ip arp                        ip neigh                      Get-NetNeighbor
show interfaces                    ethtool eth0 | ip -s link      Get-NetAdapterStatistics
show vlan brief                    bridge vlan show              -
show interfaces trunk              cat /proc/net/bonding          -
show spanning-tree                 bridge link show              -
show cdp neighbors                 arp-scan 10.1.1.0/24           arp -a
ping X                             ping -c3 X                    ping X
traceroute X                       traceroute -n X               tracert X
nslookup X                         dig X                         Resolve-DnsName X
telnet host port                   nc -zv host port              Test-NetConnection -Port
show ip nat translations           sudo iptables -t nat -L -n     netsh interface ipv4 show
show ip dhcp binding               journalctl -u NetworkManager  ipconfig /all
show version                       uname -a                      systeminfo
show running-config                cat /etc/network/interfaces   Get-NetFirewallProfile
debug ip icmp / debug ip nat        tcpdump -ni eth0             pktmon start
```

```
CIRCUIT / TERMINOLOGY GLOSSARY
------------------------------
Loopback        logical interface, always up, used for management / OSPF RID
SVI             Switched Virtual Interface - the gateway for a VLAN
Trunk           link carrying multiple VLANs, tagged 802.1Q
Access port     single-VLAN untagged port
Default route    0.0.0.0/0, used when nothing else matches
On-link          destination determined to be on the local subnet by mask
NAT/PAT         translation of addresses and ports at the edge
Hardening       reducing the attack surface of a device
```

---

**Back to [01 — Network Layer Design Issues](01-network-layer-design-issues.md)** ·
Completed: 01 ✓ 02 ✓ 03 ✓ 04 ✓ 05 ✓ 06 ✓