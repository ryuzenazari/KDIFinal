# KONFIGURASI MIKROTIK ROUTEROS 7.8 - OPTIMIZED WITH VPN
## PROJECT JARINGAN KANTOR - OSPF, VPN, DNS, NAT
---
## INFORMASI PROJECT
**Topology:**
- 3 Kantor: Pusat, Cabang 1, Cabang 2
- Router Core: R1, R2, R3 (OSPF Network)
- VPN: Office-to-Office dan Client-to-Office (PPTP Protocol)
- DNS Server: R3 dengan domain kantorpusat.co.id
- Internet Gateway: 192.168.204.2

**IP Address Schema:**
| Device | Interface | IP Address | Keterangan |
|--------|-----------|------------|------------|
| R1 | ETH0 | 12.12.12.1/24 | Link ke R2 |
| | ETH1 | 13.13.13.1/24 | Link ke R3 |
| | ETH2 | 1.1.1.1/24 | Ke R-Cabang-1 |
| R2 | ETH0 | 12.12.12.2/24 | Link ke R1 |
| | ETH1 | 23.23.23.2/24 | Link ke R3 |
| | ETH2 | 2.2.2.2/24 | Ke R-Cabang-2 |
| R3 | ETH0 | 13.13.13.3/24 | Link ke R1 |
| | ETH1 | 23.23.23.3/24 | Link ke R2 |
| | ETH2 | 3.3.12.3/24 | Ke R-Pusat |
| | ETH3 | 4.4.4.3/24 | DNS Network |
| | ETH4 | DHCP Client | Internet |
| **R-Pusat** | **ETH0** | **3.3.12.13/24** | **Public IP** |
| | **ETH1** | **192.168.12.13/24** | **LAN Pusat** |
| **R-Cabang-1** | **ETH0** | **1.1.1.11/24** | **Link ke R1** |
| | **ETH1** | **192.168.1.11/24** | **LAN Cabang-1** |
| **R-Cabang-2** | **ETH0** | **2.2.2.12/24** | **Link ke R2** |
| | **ETH1** | **192.168.2.12/24** | **LAN Cabang-2** |

**Client Devices IP Schema:**
| Device | IP Address | Gateway | DNS | Keterangan |
|--------|------------|---------|-----|------------|
| **PC1 (Cabang-1)** | **192.168.1.10/24** | **192.168.1.11** | **4.4.4.3** | **Client Office-to-Office VPN** |
| **PC2 (Cabang-2)** | **192.168.2.10/24** | **192.168.2.12** | **4.4.4.3** | **Client-to-Office PPTP VPN** |
| **Server (Web Server)** | **192.168.12.10/24** | **192.168.12.13** | **4.4.4.3** | **Apache/Nginx Port 80** |
| **PC Public** | **4.4.4.10/24** | **4.4.4.3** | **4.4.4.3** | **Client Eksternal** |

**VPN Configuration:**
- Office-to-Office VPN: R-Cabang-1 ↔ R-Pusat
- Client-to-Office VPN: PC2 (Cabang-2) → R-Pusat
- VPN Pool Office: 192.168.100.10-192.168.100.20
- VPN Pool Client: 192.168.101.10-192.168.101.20

---
## KONFIGURASI R1 (ROUTER OSPF CORE)
### System Reset
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
```
### System Identity
```bash
/system identity set name="R1"
```
### IP Address Configuration
```bash
/ip address
add address=12.12.12.1/24 interface=ether1 comment="Link ke R2"
add address=13.13.13.1/24 interface=ether2 comment="Link ke R3"
add address=1.1.1.1/24 interface=ether3 comment="Link ke R-Cabang-1"
```
### OSPF Configuration
```bash
/routing ospf instance
add name=default version=2 router-id=1.1.1.1
/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default
/routing ospf interface-template
add area=backbone networks=12.12.12.0/24
add area=backbone networks=13.13.13.0/24
add area=backbone networks=1.1.1.0/24
```
### Ultra Minimal Firewall (Essential Only)
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input action=drop
```
---
## KONFIGURASI R2 (ROUTER OSPF CORE)
### System Reset
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
```
### System Identity
```bash
/system identity set name="R2"
```
### IP Address Configuration
```bash
/ip address
add address=12.12.12.2/24 interface=ether1 comment="Link ke R1"
add address=23.23.23.2/24 interface=ether2 comment="Link ke R3"
add address=2.2.2.2/24 interface=ether3 comment="Link ke R-Cabang-2"
```
### OSPF Configuration
```bash
/routing ospf instance
add name=default version=2 router-id=2.2.2.2
/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default
/routing ospf interface-template
add area=backbone networks=12.12.12.0/24
add area=backbone networks=23.23.23.0/24
add area=backbone networks=2.2.2.0/24
```
### Ultra Minimal Firewall (Essential Only)
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input action=drop
```
---
## KONFIGURASI R3 (INTERNET GATEWAY + DNS SERVER)
### System Reset
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
```
### System Identity
```bash
/system identity set name="R3"
```
### IP Address Configuration
```bash
/ip address
add address=13.13.13.3/24 interface=ether1 comment="Link ke R1"
add address=23.23.23.3/24 interface=ether2 comment="Link ke R2"
add address=3.3.12.3/24 interface=ether3 comment="Link ke R-Pusat"
add address=4.4.4.3/24 interface=ether4 comment="DNS Server Network"
```
### Internet Connection
```bash
/ip dhcp-client
add interface=ether5 disabled=no comment="Internet Connection"
```
### OSPF Configuration
```bash
/routing ospf instance
add name=default version=2 router-id=3.3.12.3
/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default
/routing ospf interface-template
add area=backbone networks=13.13.13.0/24
add area=backbone networks=23.23.23.0/24
add area=backbone networks=3.3.12.0/24
```
### Default Route Distribution
```bash
/routing ospf instance
set default redistribute=connected,static
set default originate-default=always
```
### DNS Server Configuration
```bash
/ip dns
set servers=8.8.8.8,8.8.4.4 allow-remote-requests=yes
/ip dns static
add name=kantorpusat.co.id address=3.3.12.13 comment="Domain untuk R-Pusat"
```
### NAT Configuration
```bash
/ip firewall nat
add chain=srcnat out-interface=ether5 action=masquerade comment="Internet NAT"
```
### Ultra Minimal Firewall (DNS Only)
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input dst-port=53 protocol=udp action=accept comment="DNS"
add chain=input action=drop
```
---
## KONFIGURASI R-CABANG-1 (KANTOR CABANG 1 + VPN CLIENT)
### System Reset
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
```
### System Identity
```bash
/system identity set name="R-Cabang-1"
```
### IP Address Configuration
```bash
/ip address
add address=1.1.1.11/24 interface=ether1 comment="Link ke R1"
add address=192.168.1.11/24 interface=ether2 comment="LAN Cabang-1"
```
### OSPF Configuration
```bash
/routing ospf instance
add name=default version=2 router-id=192.168.1.11
/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default
/routing ospf interface-template
add area=backbone networks=1.1.1.0/24
add area=backbone networks=192.168.1.0/24
```
### NAT Configuration
```bash
/ip firewall nat
add chain=srcnat src-address=192.168.1.0/24 action=masquerade comment="NAT Cabang-1"
```
### VPN Office-to-Office Configuration
```bash
/interface pptp-client
add name=vpn-to-pusat connect-to=3.3.12.13 user=cabang1 password=cabang1pass disabled=no comment="VPN ke Kantor Pusat"
```
### VPN Routing Configuration
```bash
/ip route
add dst-address=192.168.12.0/24 gateway=vpn-to-pusat distance=1 comment="Route ke LAN Pusat via VPN"
add dst-address=192.168.101.0/24 gateway=vpn-to-pusat distance=1 comment="Route ke Client VPN Pool"
```
### DNS Configuration
```bash
/ip dns
set servers=4.4.4.3
```
### Firewall with VPN Support
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input src-address=192.168.12.0/24 action=accept comment="Accept dari LAN Pusat via VPN"
add chain=input action=drop
```
---
## KONFIGURASI R-CABANG-2 (KANTOR CABANG 2)
### System Reset
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
```
### System Identity
```bash
/system identity set name="R-Cabang-2"
```
### IP Address Configuration
```bash
/ip address
add address=2.2.2.12/24 interface=ether1 comment="Link ke R2"
add address=192.168.2.12/24 interface=ether2 comment="LAN Cabang-2"
```
### OSPF Configuration
```bash
/routing ospf instance
add name=default version=2 router-id=192.168.2.12
/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default
/routing ospf interface-template
add area=backbone networks=2.2.2.0/24
add area=backbone networks=192.168.2.0/24
```
### NAT Configuration
```bash
/ip firewall nat
add chain=srcnat src-address=192.168.2.0/24 action=masquerade comment="NAT Cabang-2"
```
### DNS Configuration
```bash
/ip dns
set servers=4.4.4.3
```
### Ultra Minimal Firewall (Essential Only)
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input action=drop
```
---
## KONFIGURASI R-PUSAT (KANTOR PUSAT + VPN SERVER)
### System Reset
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
```
### System Identity
```bash
/system identity set name="R-Pusat"
```
### IP Address Configuration
```bash
/ip address
add address=3.3.12.13/24 interface=ether1 comment="Public IP"
add address=192.168.12.13/24 interface=ether2 comment="LAN Pusat"
```
### OSPF Configuration
```bash
/routing ospf instance
add name=default version=2 router-id=192.168.12.13
/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default
/routing ospf interface-template
add area=backbone networks=3.3.12.0/24
add area=backbone networks=192.168.12.0/24
```
### VPN Server Configuration
```bash
# IP Pool untuk VPN Client
/ip pool
add name=vpn-pool-office ranges=192.168.100.10-192.168.100.20 comment="Pool untuk Office-to-Office"
add name=vpn-pool-client ranges=192.168.101.10-192.168.101.20 comment="Pool untuk Client-to-Office"

# PPP Profile
/ppp profile
add name=vpn-office-profile local-address=192.168.100.1 remote-address=vpn-pool-office use-encryption=yes comment="Profile Office-to-Office"
add name=vpn-client-profile local-address=192.168.101.1 remote-address=vpn-pool-client use-encryption=yes comment="Profile Client-to-Office"

# Enable PPTP Server
/interface pptp-server server
set enabled=yes default-profile=vpn-office-profile authentication=mschap2

# VPN Users
/ppp secret
add name=cabang1 password=cabang1pass profile=vpn-office-profile service=pptp comment="Office-to-Office: Cabang-1"
add name=pc2user password=pc2userpass profile=vpn-client-profile service=pptp comment="Client-to-Office: PC2"
```
### VPN Routing Configuration
```bash
/ip route
add dst-address=192.168.2.0/24 gateway=192.168.101.10 distance=1 comment="Route ke Cabang-2 untuk VPN Client"
```
### Port Forwarding dan NAT
```bash
/ip firewall nat
add chain=dstnat dst-address=3.3.12.13 dst-port=80 protocol=tcp action=dst-nat to-addresses=192.168.12.10 comment="Web Server"
add chain=srcnat src-address=192.168.12.0/24 action=masquerade comment="NAT Pusat"
add chain=srcnat src-address=192.168.100.0/24 action=masquerade comment="NAT Office-to-Office VPN"
add chain=srcnat src-address=192.168.101.0/24 action=masquerade comment="NAT Client-to-Office VPN"
```
### Firewall dengan VPN Support
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input dst-port=1723 protocol=tcp action=accept comment="PPTP VPN"
add chain=input protocol=gre action=accept comment="GRE for PPTP"
add chain=input protocol=icmp src-address=192.168.0.0/16 action=accept comment="Ping dari VPN Clients"
add chain=input protocol=ospf action=accept
add chain=input dst-port=80 protocol=tcp action=accept comment="Web Server"
add chain=input action=drop comment="Drop all other"
```
### DNS Configuration
```bash
/ip dns
set servers=4.4.4.3
```
---
## KONFIGURASI CLIENT DEVICES

### PC1 (Client Cabang-1) - Office-to-Office VPN Access
**Network Configuration:**
```bash
IP Address: 192.168.1.10/24
Subnet Mask: 255.255.255.0
Gateway: 192.168.1.11 (R-Cabang-1)
DNS Server: 4.4.4.3 (R3 DNS Server)
```

**Windows Network Settings:**
1. Open Control Panel → Network and Internet → Network Connections
2. Right-click Local Area Connection → Properties
3. Select Internet Protocol Version 4 (TCP/IPv4) → Properties
4. Use the following IP address:
   - IP address: 192.168.1.10
   - Subnet mask: 255.255.255.0
   - Default gateway: 192.168.1.11
   - Preferred DNS server: 4.4.4.3

**Testing Commands:**
```cmd
# Test local gateway
ping 192.168.1.11

# Test connectivity to Server Pusat (via Office-to-Office VPN)
ping 192.168.12.10

# Test web access to Server Pusat
# Open browser: http://192.168.12.10
```

### PC2 (Client Cabang-2) - PPTP VPN Client
**Network Configuration:**
```bash
IP Address: 192.168.2.10/24
Subnet Mask: 255.255.255.0
Gateway: 192.168.2.12 (R-Cabang-2)
DNS Server: 4.4.4.3 (R3 DNS Server)
```

**Windows Network Settings:**
1. Configure basic network settings (same as PC1)
2. **PPTP VPN Client Setup:**
   - Open Settings → Network & Internet → VPN
   - Add VPN Connection:
     - VPN Provider: Windows (built-in)
     - Connection Name: VPN-to-Kantor-Pusat
     - Server name or address: 3.3.12.13
     - VPN type: Point to Point Tunneling Protocol (PPTP)
     - Username: pc2user
     - Password: pc2userpass

**Testing Scenarios:**
```cmd
# Before VPN Connection (should FAIL)
ping 192.168.12.10

# After VPN Connection (should SUCCESS)
ping 192.168.12.10

# Check VPN IP assignment
ipconfig

# Test web access via VPN
# Open browser: http://192.168.12.10
```

### Server (Web Server Pusat)
**Network Configuration:**
```bash
IP Address: 192.168.12.10/24
Subnet Mask: 255.255.255.0
Gateway: 192.168.12.13 (R-Pusat)
DNS Server: 4.4.4.3 (R3 DNS Server)
```

**Linux/Ubuntu Server Setup:**
```bash
# Configure network interface
sudo nano /etc/netplan/00-netcfg.yaml

network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.12.10/24]
      gateway4: 192.168.12.13
      nameservers:
        addresses: [4.4.4.3, 8.8.8.8]

# Apply configuration
sudo netplan apply

# Install Apache Web Server
sudo apt update
sudo apt install apache2 -y

# Enable and start Apache
sudo systemctl enable apache2
sudo systemctl start apache2

# Create test web page
echo "<h1>Welcome to Kantor Pusat Web Server</h1>" | sudo tee /var/www/html/index.html
```

**Windows Server Setup:**
```cmd
# Configure network via GUI or netsh
netsh interface ip set address "Local Area Connection" static 192.168.12.10 255.255.255.0 192.168.12.13

# Set DNS
netsh interface ip set dns "Local Area Connection" static 4.4.4.3

# Install IIS (Windows Server)
# Use Server Manager → Add Roles and Features → Web Server (IIS)
```

### PC Public (Client Eksternal)
**Network Configuration:**
```bash
IP Address: 4.4.4.10/24
Subnet Mask: 255.255.255.0
Gateway: 4.4.4.3 (R3)
DNS Server: 4.4.4.3 (R3 DNS Server)
```

**Testing External Access:**
```cmd
# Test DNS resolution
nslookup kantorpusat.co.id

# Test web access via public IP
# Open browser: http://3.3.12.13

# Test ping to public IP
ping 3.3.12.13
```

---
## TESTING DAN VERIFIKASI VPN
### A. Testing Office-to-Office VPN (R-Cabang-1 ↔ R-Pusat)
**Dari R-Cabang-1:**
```bash
# Test koneksi ke LAN Pusat
/ping 192.168.12.10 count=5
# Test koneksi ke Server di Pusat
/ping 192.168.12.13 count=5
# Cek status VPN interface
/interface print where name=vpn-to-pusat
# Cek routing table
/ip route print where dst-address=192.168.12.0/24
```
**Dari PC1 (Cabang-1):**
```bash
# Test akses ke Server di Pusat
ping 192.168.12.10
# Test akses ke web server
# Buka browser: http://192.168.12.10
```
### B. Testing Client-to-Office VPN (PC2 → R-Pusat)
**Setting VPN Connection di Windows (PC2):**
1. **Buka Network & Internet Settings**
2. **Add VPN Connection**:
   - VPN Provider: Windows (built-in)
   - Connection Name: VPN-to-Kantor-Pusat
   - Server: 3.3.12.13
   - VPN Type: Point to Point Tunneling Protocol (PPTP)
   - Username: pc2user
   - Password: pc2userpass

**Testing dari PC2:**
```bash
# Sebelum VPN Connect (harus GAGAL)
ping 192.168.12.10

# Setelah VPN Connect (harus BERHASIL)
ping 192.168.12.10
# Cek IP address baru dari VPN
ipconfig
# Test akses web server
# Buka browser: http://192.168.12.10
```
### C. Monitoring VPN di R-Pusat
```bash
# Cek active VPN sessions
/ppp active print
# Cek VPN interface yang terbentuk
/interface print where type=pptp
# Monitor traffic VPN
/interface monitor-traffic interface=<pptp-interface-name>
```
---
## TROUBLESHOOTING VPN
### A. Jika VPN Tidak Bisa Connect
**Check di R-Pusat:**
```bash
# Cek apakah PPTP server aktif
/interface pptp-server server print
# Cek firewall rules
/ip firewall filter print where chain=input
# Cek user credentials
/ppp secret print
```
**Check di R-Cabang-1:**
```bash
# Cek status PPTP client
/interface pptp-client print
# Cek log errors
/log print where topics~"pptp"
```
### B. Jika VPN Connect tapi Tidak Bisa Akses LAN
**Check routing:**
```bash
# Di R-Pusat
/ip route print where gateway~"pptp"
# Di R-Cabang-1
/ip route print where gateway~"vpn"
```
**Check NAT rules:**
```bash
# Di R-Pusat
/ip firewall nat print where src-address~"192.168.100" or src-address~"192.168.101"
```
---
## SECURITY CONSIDERATIONS
### A. Firewall Optimization untuk VPN
```bash
# Di R-Pusat - Specific VPN rules
/ip firewall filter
add chain=input src-address=192.168.1.0/24 dst-port=1723 protocol=tcp action=accept comment="PPTP dari Cabang-1"
add chain=input src-address=192.168.2.0/24 dst-port=1723 protocol=tcp action=accept comment="PPTP dari Cabang-2"
add chain=input dst-port=1723 protocol=tcp action=drop comment="Block PPTP dari IP lain"
```
### B. Monitoring VPN Activity
```bash
# Log VPN connections
/system logging
add topics=ppp,pptp action=memory
# Check logs
/log print where topics~"ppp"
```
---
## ADVANCED CONFIGURATION
### A. Backup VPN Configuration
```bash
# Export semua konfigurasi VPN
/export file=vpn-config-backup
# Export specific sections
/ppp secret export file=vpn-users-backup
/ip firewall nat export file=vpn-nat-backup
```
### B. Automatic VPN Reconnection (R-Cabang-1)
```bash
# Enable auto-reconnect
/interface pptp-client
set vpn-to-pusat add-default-route=no keepalive-timeout=60 max-mtu=1460 max-mru=1460
```
---
## SUMMARY KONFIGURASI LENGKAP
### ✅ **Yang Sudah Dikonfigurasi:**
1. **OSPF Network** untuk semua router (R1, R2, R3)
2. **Internet Gateway** dan DNS Server di R3
3. **PPTP VPN Server** di R-Pusat dengan 2 profile berbeda
4. **Office-to-Office VPN**: R-Cabang-1 ↔ R-Pusat
5. **Client-to-Office VPN**: PC2 → R-Pusat  
6. **Routing** untuk VPN traffic
7. **Firewall rules** minimal untuk VPN
8. **NAT** untuk semua network termasuk VPN pools
9. **DNS Server** dengan domain kantorpusat.co.id
10. **Client Device Configuration** untuk semua PC dan Server

### 🔧 **Testing Scenarios:**
- PC1 (192.168.1.10) → Server Pusat (192.168.12.10) via Office-to-Office VPN
- PC2 (192.168.2.10) → Server Pusat (192.168.12.10) via Client-to-Office VPN
- PC Public (4.4.4.10) → Web Server via Port Forwarding (3.3.12.13:80)
- Monitoring dan troubleshooting VPN connections
- Web server access melalui domain kantorpusat.co.id

### 🛡️ **Security Features:**
- Encrypted PPTP connections (MSCHAP2)
- Minimal firewall rules untuk optimal performance
- Separate VPN pools untuk Office dan Client connections
- OSPF authentication ready (jika diperlukan)
- Monitoring dan logging VPN activity

### 📋 **Complete Network Summary:**
- **Core Network**: OSPF Area 0 (Backbone)
- **Office Networks**: 
  - Cabang-1: 192.168.1.0/24 (PC1: 192.168.1.10, Gateway: 192.168.1.11)
  - Cabang-2: 192.168.2.0/24 (PC2: 192.168.2.10, Gateway: 192.168.2.12)  
  - Pusat: 192.168.12.0/24 (Server: 192.168.12.10, Gateway: 192.168.12.13)
- **VPN Networks**: 
  - Office-to-Office: 192.168.100.0/24
  - Client-to-Office: 192.168.101.0/24
- **DNS Network**: 4.4.4.0/24 (PC Public: 4.4.4.10, DNS Server: 4.4.4.3)
- **Internet Gateway**: R3 dengan DHCP Client

### 🖥️ **Device Summary:**
- **R-Pusat**: 3.3.12.13/24, 192.168.12.13/24 (VPN Server + Web Server Gateway)
- **R-Cabang-1**: 1.1.1.11/24, 192.168.1.11/24 (Office-to-Office VPN Client)
- **R-Cabang-2**: 2.2.2.12/24, 192.168.2.12/24 (Standard Gateway)
- **PC1**: 192.168.1.10/24 (Akses via Office-to-Office VPN)
- **PC2**: 192.168.2.10/24 (Client-to-Office PPTP VPN)
- **Server**: 192.168.12.10/24 (Apache/IIS Web Server)
- **PC Public**: 4.4.4.10/24 (External Client)

Konfigurasi ini memberikan solusi jaringan kantor yang lengkap dengan VPN, DNS, dan routing optimal menggunakan OSPF, termasuk konfigurasi detail untuk semua client devices sesuai dengan requirement project jaringan kantor Anda.
