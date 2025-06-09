# KONFIGURASI MIKROTIK ROUTEROS 7.8
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
| R-Cabang-1 | ETH0 | 1.1.1.11/24 | Ke R1 |
| | ETH1 | 192.168.1.11/24 | LAN Cabang-1 |
| R-Cabang-2 | ETH0 | 2.2.2.12/24 | Ke R2 |
| | ETH1 | 192.168.2.12/24 | LAN Cabang-2 |
| R-Pusat | ETH0 | 3.3.12.13/24 | Public IP |
| | ETH1 | 192.168.12.13/24 | LAN Pusat |

---

## KONFIGURASI R1 (ROUTER OSPF CORE)

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
/system identity set name="R1"
```

### IP Address Configuration
```bash
/ip address
add address=12.12.12.1/24 interface=ether1 comment="Link ke R2"
add address=13.13.13.1/24 interface=ether2 comment="Link ke R3"
add address=1.1.1.1/24 interface=ether3 comment="Link ke R-Cabang-1"
```

### OSPF Configuration (RouterOS 7.x)
```bash
/routing ospf instance
add name=default version=2 router-id=1.1.1.1

/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default

/routing ospf interface-template
add area=backbone networks=12.12.12.0/24 comment="Network ke R2"
add area=backbone networks=13.13.13.0/24 comment="Network ke R3"
add area=backbone networks=1.1.1.0/24 comment="Network ke R-Cabang-1"
```

### Firewall Configuration
```bash
/ip firewall filter
add chain=forward dst-address=192.168.12.0/24 src-address=192.168.1.0/24 action=accept comment="Allow Cabang-1 to LAN Pusat (for VPN)"
add chain=forward dst-address=192.168.12.0/24 src-address=192.168.2.0/24 action=drop comment="Block Cabang-2 direct access to LAN Pusat"
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop
add chain=forward action=accept
```

---

## KONFIGURASI R2 (ROUTER OSPF CORE)

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
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
add area=backbone networks=12.12.12.0/24 comment="Network ke R1"
add area=backbone networks=23.23.23.0/24 comment="Network ke R3"
add area=backbone networks=2.2.2.0/24 comment="Network ke R-Cabang-2"
```

### Firewall Configuration
```bash
/ip firewall filter
add chain=forward dst-address=192.168.12.0/24 src-address=192.168.2.0/24 action=drop comment="Block direct access to LAN Pusat"
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop
add chain=forward action=accept
```

### Firewall Filter Rules
```bash
/ip firewall filter
add chain=forward dst-address=192.168.12.0/24 action=drop comment="Block direct access to LAN Pusat without VPN"
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop
add chain=forward action=accept comment="Allow all other forwarded traffic"
```

---

## KONFIGURASI R3 (INTERNET GATEWAY + DNS SERVER)

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
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
add interface=ether5 disabled=no use-peer-dns=yes comment="Internet Connection"

# Hapus default route statis, biarkan DHCP client menambahkannya otomatis
```

### OSPF Configuration
```bash
/routing ospf instance
add name=default version=2 router-id=3.3.12.3

/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default

/routing ospf interface-template
add area=backbone networks=13.13.13.0/24 comment="Network ke R1"
add area=backbone networks=23.23.23.0/24 comment="Network ke R2"
add area=backbone networks=3.3.12.0/24 comment="Network ke R-Pusat"
```

### Default Route Distribution
```bash
/routing ospf instance
set default redistribute=connected,static,ospf,rip
set default originate-default=always
```

### DNS Server Configuration
```bash
/ip dns
set servers=8.8.8.8,8.8.4.4 allow-remote-requests=yes

/ip dns static
add name=kantorpusat.co.id address=3.3.12.13 comment="Domain untuk R-Pusat"
add name=kantorpusat.co.id address=4.4.4.3 comment="Domain untuk akses dari PC-Public"
```

### NAT Configuration
```bash
/ip firewall nat
add chain=srcnat out-interface=ether5 action=masquerade comment="Masquerade semua traffic keluar ke internet"
```

### Firewall Configuration
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input protocol=icmp action=accept
add chain=input protocol=ospf action=accept
add chain=forward protocol=ospf action=accept comment="Allow OSPF Forward"
add chain=input dst-port=53 protocol=udp action=accept comment="DNS Access"
add chain=input dst-port=53 protocol=tcp action=accept comment="DNS Access"
add chain=input dst-port=80 protocol=tcp action=accept comment="Allow Web Access"
add chain=forward dst-port=80 protocol=tcp action=accept comment="Allow Web Traffic Forward"
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop 
add chain=forward action=accept comment="Allow all forwarded traffic during testing"
add chain=input action=drop
```

### Port Forwarding di R3
```bash
/ip firewall nat
add chain=dstnat dst-address=4.4.4.3 dst-port=80 protocol=tcp action=dst-nat to-addresses=192.168.12.10 to-ports=80 comment="Web Server via DNS Server"

# Pastikan traffic dari PC-Public ke web server diizinkan
/ip firewall filter
add chain=forward src-address=4.4.4.0/24 dst-address=192.168.12.10 dst-port=80 protocol=tcp action=accept comment="Allow PC-Public to Web Server"
```

### Default Route
```bash
# Jangan tambahkan default route manual, gunakan yang ditambahkan otomatis oleh DHCP client
# Untuk melihat route yang ditambahkan:
# /ip route print
```

### Pengujian DNS dari PC-Public
```bash
# Di PC-Public, jalankan:
nslookup kantorpusat.co.id 4.4.4.3
# Seharusnya menampilkan IP 4.4.4.3 dan/atau 3.3.12.13

# Atau coba akses langsung via IP:
curl http://4.4.4.3
# Seharusnya menampilkan halaman web yang sama dengan http://kantorpusat.co.id
```

---

## KONFIGURASI R-CABANG-1 (KANTOR CABANG 1)

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
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
add area=backbone networks=1.1.1.0/24 comment="Network ke R1"
add area=backbone networks=192.168.1.0/24 comment="LAN Cabang-1"
```

### DHCP Server Configuration
```bash
/ip pool
add name=pool-cabang1 ranges=192.168.1.20-192.168.1.100

/ip dhcp-server
add name=dhcp-cabang1 interface=ether2 address-pool=pool-cabang1 disabled=no

/ip dhcp-server network
add address=192.168.1.0/24 gateway=192.168.1.11 dns-server=4.4.4.3 comment="DHCP Network Cabang-1"
```

### NAT Configuration
```bash
/ip firewall nat
add chain=srcnat src-address=192.168.1.0/24 out-interface=ether1 action=masquerade comment="NAT Cabang-1 to OSPF"
add chain=srcnat src-address=192.168.1.0/24 action=masquerade comment="Masquerade all traffic from Cabang-1"
```

### VPN Office-to-Office Configuration (PPTP)
```bash
/interface pptp-client
add name=vpn-to-pusat connect-to=3.3.12.13 user=cabang1 password=cabang1pass disabled=no comment="PPTP VPN Office to Office"

/ip route
add dst-address=192.168.12.0/24 gateway=vpn-to-pusat comment="Route ke Server via PPTP VPN Office to Office"
```

### DNS Configuration
```bash
/ip dns
set servers=4.4.4.3
```

---

## KONFIGURASI R-CABANG-2 (KANTOR CABANG 2)

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
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
add area=backbone networks=2.2.2.0/24 comment="Network ke R2"
add area=backbone networks=192.168.2.0/24 comment="LAN Cabang-2"
```

### Firewall Filter Rules
```bash
/ip firewall filter
add chain=forward dst-address=192.168.12.0/24 action=drop comment="Block direct access to LAN Pusat without VPN"
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop
add chain=forward action=accept comment="Allow all other forwarded traffic"
```

### DHCP Server Configuration
```bash
/ip pool
add name=pool-cabang2 ranges=192.168.2.20-192.168.2.100

/ip dhcp-server
add name=dhcp-cabang2 interface=ether2 address-pool=pool-cabang2 disabled=no

/ip dhcp-server network
add address=192.168.2.0/24 gateway=192.168.2.12 dns-server=4.4.4.3 comment="DHCP Network Cabang-2"
```

### NAT Configuration
```bash
/ip firewall nat
add chain=srcnat src-address=192.168.2.0/24 out-interface=ether1 action=masquerade comment="NAT Cabang-2 to OSPF"
add chain=srcnat src-address=192.168.2.0/24 action=masquerade comment="Masquerade all traffic from Cabang-2"
```

### DNS Configuration
```bash
/ip dns
set servers=4.4.4.3
```

---

## KONFIGURASI R-PUSAT (KANTOR PUSAT + VPN SERVER) - PPTP VERSION

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
/system identity set name="R-Pusat"
```

### IP Address Configuration
```bash
/ip address
add address=3.3.12.13/24 interface=ether1 comment="Public IP/NAT"
add address=192.168.12.13/24 interface=ether2 comment="LAN Pusat"
```

### OSPF Configuration
```bash
/routing ospf instance
add name=default version=2 router-id=192.168.12.13

/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default

/routing ospf interface-template
add area=backbone networks=3.3.12.0/24 comment="Network ke R3"
add area=backbone networks=192.168.12.0/24 comment="LAN Pusat"
```

### DHCP Server Configuration
```bash
/ip pool
add name=pool-pusat ranges=192.168.12.20-192.168.12.100

/ip dhcp-server
add name=dhcp-pusat interface=ether2 address-pool=pool-pusat disabled=no

/ip dhcp-server network
add address=192.168.12.0/24 gateway=192.168.12.13 dns-server=4.4.4.3 comment="DHCP Network Pusat"
```

### VPN PPTP Server Configuration
```bash
# Buat IP Pool untuk VPN terlebih dahulu
/ip pool
add name=vpn-pool ranges=192.168.100.10-192.168.100.20

# Buat PPP Profile untuk PPTP
/ppp profile
add name=vpn-profile local-address=192.168.100.1 remote-address=vpn-pool use-encryption=yes

# Aktifkan PPTP Server
/interface pptp-server server
set enabled=yes default-profile=vpn-profile authentication=mschap2

# Tambahkan PPP Secrets untuk PPTP
/ppp secret
add name=cabang1 password=cabang1pass profile=vpn-profile service=pptp comment="PPTP VPN Office to Office - R-Cabang-1"
add name=pc2user password=pc2userpass profile=vpn-profile service=pptp comment="PPTP VPN Client to Office - PC-2"

# Route untuk VPN Pool
/ip route
add dst-address=192.168.100.0/24 gateway=192.168.12.13 comment="Route VPN Pool"
```

### Port Forwarding dan NAT
```bash
/ip firewall nat
add chain=dstnat dst-address=3.3.12.13 dst-port=80 protocol=tcp action=dst-nat to-addresses=192.168.12.10 to-ports=80 comment="Web Server Port Forward"
add chain=dstnat dst-address=4.4.4.3 dst-port=80 protocol=tcp action=dst-nat to-addresses=192.168.12.10 to-ports=80 comment="DNS Server Port Forward"
add chain=srcnat src-address=192.168.12.0/24 out-interface=ether1 action=masquerade comment="NAT Pusat to OSPF"
add chain=srcnat src-address=192.168.12.0/24 action=masquerade comment="Masquerade all traffic from Pusat"
add chain=srcnat src-address=192.168.100.0/24 action=masquerade comment="NAT VPN Pool"
```

### Firewall Filter Rules - PPTP Version
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input protocol=icmp src-address=192.168.1.0/24 action=accept comment="Allow ping from Cabang-1"
add chain=input protocol=icmp src-address=192.168.2.0/24 action=accept comment="Allow ping from Cabang-2"
add chain=input protocol=icmp src-address=4.4.4.0/24 action=drop comment="Block ping from Public"
add chain=input protocol=icmp action=drop comment="Block other ping"
add chain=forward protocol=icmp action=accept comment="Allow ICMP in forward chain"
add chain=input protocol=ospf action=accept comment="Allow OSPF"
add chain=forward protocol=ospf action=accept comment="Allow OSPF in forward chain"
add chain=input dst-port=1723 protocol=tcp action=accept comment="PPTP VPN"
add chain=input protocol=gre action=accept comment="GRE for PPTP"
add chain=input dst-port=80 protocol=tcp action=accept comment="Web Server Access"
add chain=input dst-port=53 protocol=udp action=accept comment="Allow DNS access"
add chain=input dst-port=53 protocol=tcp action=accept comment="Allow DNS access"
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop
add chain=forward dst-address=192.168.12.10 dst-port=80 protocol=tcp action=accept comment="Allow access to Web Server for all clients"
add chain=forward dst-address=192.168.12.0/24 src-address=192.168.100.0/24 action=accept comment="Allow access to LAN Pusat from VPN clients"
add chain=forward dst-address=192.168.12.0/24 src-address=!192.168.12.0/24 action=drop comment="Block direct access to LAN Pusat from non-VPN (except web server)"
add chain=forward action=accept comment="Allow all other forwarded traffic during testing"
add chain=input action=drop comment="Drop all other input"
```

### DNS Configuration
```bash
/ip dns
set servers=4.4.4.3
```

---

## KONFIGURASI CLIENT DEVICES

### PC1 (Client Cabang-1)
- **IP Address:** 192.168.1.10/24
- **Gateway:** 192.168.1.11
- **DNS:** 4.4.4.3

### PC2 (Client Cabang-2) - PPTP VPN Client
- **IP Address:** 192.168.2.10/24
- **Gateway:** 192.168.2.12
- **DNS:** 4.4.4.3
- **PPTP VPN Client:** Connect to 3.3.12.13 (username: pc2user, password: pc2userpass)
- **VPN Route Configuration:**
  ```
  # Konfigurasi di Windows setelah terhubung VPN
  # Buka command prompt sebagai administrator
  route add 192.168.12.0 mask 255.255.255.0 192.168.100.1 metric 1
  # Atau aktifkan "Use default gateway on remote network" di Properties VPN Connection
  ```

### Server (Web Server Pusat)
- **IP Address:** 192.168.12.10/24
- **Gateway:** 192.168.12.13
- **DNS:** 4.4.4.3
- **Web Server:** Apache/Nginx on port 80

### PC Public (Client Luar)
- **IP Address:** 4.4.4.10/24
- **Gateway:** 4.4.4.3
- **DNS:** 4.4.4.3

---

## PENGUJIAN PROJECT (TESTING SCENARIOS)

### 1. Pengujian Jaringan OSPF → PING Test
**Command dari R-Cabang-1:**
```bash
/ping 3.3.12.13 count=5
```
**Command dari R-Cabang-2:**
```bash
/ping 3.3.12.13 count=5
```
**Expected Result:** PING berhasil (Reply dari R-Pusat)

### 2. Pengujian Koneksi Internet → Distributed Default Route OSPF
**Test dari PC1:**
- Buka browser → http://google.co.id

**Test dari PC2:**
- Buka browser → http://google.co.id

**Expected Result:** Website Google terbuka dengan normal

### 3. Pengujian Firewall DST-NAT → Akses Web Server via Domain
**Test dari PC-Publik:**
- Buka browser → http://kantorpusat.co.id
- Alternatif: Buka browser → http://4.4.4.3

**Test dari PC-1:**
- Buka browser → http://kantorpusat.co.id

**Expected Result:** Web server dapat diakses dari kedua lokasi

**Catatan pengecekan akses dari PC-Public:**
```bash
# Verifikasi resolusi DNS di PC-Public
nslookup kantorpusat.co.id 4.4.4.3

# Verifikasi akses HTTP langsung ke IP
curl -v http://4.4.4.3

# Verifikasi akses HTTP ke domain
curl -v http://kantorpusat.co.id

# Jika masih bermasalah, cek port forwarding di R3
/ip firewall nat print
```

### 4. Pengujian PPTP VPN → Akses Web Langsung ke IP Server
**Test dari PC-1 (via PPTP VPN Office-to-Office):**
- Buka browser → http://192.168.12.10

**Test dari PC-2 (via PPTP VPN Client-to-Office):**
- Connect PPTP VPN terlebih dahulu
- Buka browser → http://192.168.12.10

**Test tanpa VPN (harus gagal):**
- Pada PC-2, disconnect VPN jika terhubung
- Buka browser → http://192.168.12.10
- Expected: Koneksi timeout/gagal karena akses langsung ke jaringan Pusat diblokir firewall

**Expected Result untuk akses melalui VPN:** Web server dapat diakses langsung via IP internal

### 5. Pengujian Firewall Filter R-Pusat → Selective Ping Access
**Test dari PC-1:**
```bash
ping 3.3.12.13
```
**Expected:** SUCCESS ✅

**Test dari PC-2:**
```bash
ping 3.3.12.13
```
**Expected:** SUCCESS ✅

**Test dari PC-Publik:**
```bash
ping 3.3.12.13
```
**Expected:** FAILED ❌ (Blocked by firewall)

---

## MONITORING DAN TROUBLESHOOTING COMMANDS

### Cek OSPF Status
```bash
/routing ospf neighbor print
/routing ospf lsa print
/routing ospf state print
```

### Cek Routing Table
```bash
/ip route print
/ip route print where gateway=0.0.0.0
```

### Cek NAT Rules
```bash
/ip firewall nat print
```

### Cek Firewall Status
```bash
/ip firewall filter print
/ip firewall connection print
```

### Cek PPTP VPN Status
```bash
/ppp active print
/interface pptp-client print
/interface pptp-server print
/ppp profile print
/ppp secret print
/ip pool print
```

### Network Testing
```bash
/ping [destination_ip] count=5
/tool traceroute [destination_ip]
/tool bandwidth-test [destination_ip]
```

### Monitor Traffic
```bash
/interface monitor-traffic ether1
/tool torch interface=ether1
```

### Debug PPTP VPN Issues
```bash
/log print where topics~"pptp"
/log print where topics~"ppp"
/system logging add topics=pptp,ppp action=memory
```

---

**SELESAI - KONFIGURASI LENGKAP MIKROTIK ROUTEROS 7.8**

*Project ini mencakup OSPF routing, PPTP VPN (Office-to-Office & Client-to-Office), DNS server, port forwarding, dan firewall security. PPTP dipilih untuk kemudahan setup dan kompatibilitas, namun untuk environment production disarankan menggunakan protokol VPN yang lebih aman.*
