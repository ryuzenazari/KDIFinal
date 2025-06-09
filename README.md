# KONFIGURASI MIKROTIK ROUTEROS 7.8 - PROJECT AKHIR (OPTIMIZED)
## PROJECT JARINGAN KANTOR - OSPF, VPN, DNS, NAT

---

## INFORMASI PROJECT BERDASARKAN REQUIREMENTS

**Topology:**
- 3 Kantor: Pusat, Cabang 1, Cabang 2
- Router Core OSPF: R1, R2, R3 (Area dalam lingkaran biru muda)
- Router Kantor: R-Pusat, R-Cabang-1, R-Cabang-2 (Terhubung ke router core)
- VPN: Office-to-Office (R-Cabang-1 ke R-Pusat) dan Client-to-Office (PC-2 ke R-Pusat)
- DNS Server: R3 dengan domain kantorpusat.co.id → 3.3.XY.13
- Internet Gateway: R3 (pintu penghubung ke Internet)
- NAT: Setiap kantor menggunakan NAT untuk akses keluar

**Contoh dengan XY = 12 (gunakan 2 digit terakhir NIM Anda):**
- X = 1, Y = 2
- Jika X+Y ≠ Y dan X+Y ≠ X, maka normal
- Jika X+Y = Y atau X+Y = X, maka LAN Server = XY+1

**IP Address Schema (Contoh XY = 12):**

| Device | Interface | IP Address | Keterangan |
|--------|-----------|------------|------------|
| R1 | ETH0 | 12.12.12.1/24 | Link ke R2 (OSPF) |
| | ETH1 | 13.13.13.1/24 | Link ke R3 (OSPF) |
| | ETH2 | 1.1.1.1/24 | Ke R-Cabang-1 |
| R2 | ETH0 | 12.12.12.2/24 | Link ke R1 (OSPF) |
| | ETH1 | 23.23.23.2/24 | Link ke R3 (OSPF) |
| | ETH2 | 2.2.2.2/24 | Ke R-Cabang-2 |
| R3 | ETH0 | 13.13.13.3/24 | Link ke R1 (OSPF) |
| | ETH1 | 23.23.23.3/24 | Link ke R2 (OSPF) |
| | ETH2 | 3.3.12.3/24 | Ke R-Pusat |
| | ETH3 | 4.4.4.3/24 | Network PC Public |
| | ETH4 | DHCP Client | Internet |
| R-Cabang-1 | ETH0 | 1.1.1.11/24 | Ke R1 |
| | ETH1 | 192.168.1.11/24 | LAN Cabang-1 (Gateway) |
| R-Cabang-2 | ETH0 | 2.2.2.12/24 | Ke R2 |
| | ETH1 | 192.168.2.12/24 | LAN Cabang-2 (Gateway) |
| R-Pusat | ETH0 | 3.3.12.13/24 | Ke R3 |
| | ETH1 | 192.168.12.13/24 | LAN Pusat (Gateway) |

**Static IP Devices:**
- PC1 (Cabang-1): 192.168.1.10/24
- PC2 (Cabang-2): 192.168.2.10/24  
- Server (Pusat): 192.168.12.10/24
- PC-Public: 4.4.4.10/24

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
add address=12.12.12.1/24 interface=ether1 comment="Link ke R2 - OSPF"
add address=13.13.13.1/24 interface=ether2 comment="Link ke R3 - OSPF"
add address=1.1.1.1/24 interface=ether3 comment="Link ke R-Cabang-1"
```

### OSPF Configuration (RouterOS 7.x) - Area OSPF Core
```bash
/routing ospf instance
add name=default version=2 router-id=1.1.1.1

/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default

# OSPF Networks (lingkaran biru muda)
/routing ospf interface-template
add area=backbone networks=12.12.12.0/24 comment="OSPF - Network ke R2"
add area=backbone networks=13.13.13.0/24 comment="OSPF - Network ke R3"
add area=backbone networks=1.1.1.0/24 comment="OSPF - Network ke R-Cabang-1"
```

### Static Route ke LAN Cabang-1
```bash
/ip route
add dst-address=192.168.1.0/24 gateway=1.1.1.11 comment="Route ke LAN Cabang-1"
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
add address=12.12.12.2/24 interface=ether1 comment="Link ke R1 - OSPF"
add address=23.23.23.2/24 interface=ether2 comment="Link ke R3 - OSPF"
add address=2.2.2.2/24 interface=ether3 comment="Link ke R-Cabang-2"
```

### OSPF Configuration - Area OSPF Core
```bash
/routing ospf instance
add name=default version=2 router-id=2.2.2.2

/routing ospf area
add name=backbone area-id=0.0.0.0 instance=default

# OSPF Networks (lingkaran biru muda)
/routing ospf interface-template
add area=backbone networks=12.12.12.0/24 comment="OSPF - Network ke R1"
add area=backbone networks=23.23.23.0/24 comment="OSPF - Network ke R3"
add area=backbone networks=2.2.2.0/24 comment="OSPF - Network ke R-Cabang-2"
```

### Static Route ke LAN Cabang-2
```bash
/ip route
add dst-address=192.168.2.0/24 gateway=2.2.2.12 comment="Route ke LAN Cabang-2"
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
add address=13.13.13.3/24 interface=ether1 comment="Link ke R1 - OSPF"
add address=23.23.23.3/24 interface=ether2 comment="Link ke R2 - OSPF"
add address=3.3.12.3/24 interface=ether3 comment="Link ke R-Pusat"
add address=4.4.4.3/24 interface=ether4 comment="Network PC Public"
```

### Internet Connection
```bash
/ip dhcp-client
add interface=ether5 disabled=no comment="Internet Connection"
```

### OSPF Configuration - Area OSPF Core
```bash
/routing ospf instance
add name=default version=2 router-id=3.3.12.3

/routing ospf area  
add name=backbone area-id=0.0.0.0 instance=default

# OSPF Networks (lingkaran biru muda)
/routing ospf interface-template
add area=backbone networks=13.13.13.0/24 comment="OSPF - Network ke R1"
add area=backbone networks=23.23.23.0/24 comment="OSPF - Network ke R2"
add area=backbone networks=3.3.12.0/24 comment="OSPF - Network ke R-Pusat"
```

### OSPF Default Route Distribution
```bash
/routing ospf instance
set default redistribute=connected,static originate-default=always
```

### DNS Server Configuration
```bash
/ip dns
set servers=8.8.8.8,1.1.1.1 allow-remote-requests=yes

# Domain registration: kantorpusat.co.id → 3.3.12.13 (R-Pusat Public IP)
/ip dns static
add name=kantorpusat.co.id address=3.3.12.13 comment="Domain untuk akses Web Server via R-Pusat"
```

### Static Routes
```bash
/ip route
add dst-address=192.168.12.0/24 gateway=3.3.12.13 comment="Route ke LAN Pusat"
```

### NAT Configuration untuk PC Public
```bash
/ip firewall nat
add chain=srcnat src-address=4.4.4.0/24 out-interface=ether5 action=masquerade comment="NAT PC Public ke Internet"
```

### Firewall untuk R3 - Consolidated Rules
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input protocol=icmp action=accept comment="Allow ICMP"
add chain=input protocol=ospf action=accept comment="OSPF Protocol" 
add chain=input dst-port=53 protocol=tcp action=accept comment="DNS Services"
add chain=input dst-port=53 protocol=udp action=accept comment="DNS Services"
add chain=input action=drop comment="Drop all other input"
```

---

## KONFIGURASI R-CABANG-1 (KANTOR CABANG 1)

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
/system identity set name="R-Cabang-1"
```

### Complete Network and VPN Configuration
```bash
# IP Configuration
/ip address
add address=1.1.1.11/24 interface=ether1 comment="Link ke R1"
add address=192.168.1.11/24 interface=ether2 comment="LAN Gateway"

# NAT untuk outbound traffic
/ip firewall nat add chain=srcnat src-address=192.168.1.0/24 out-interface=ether1 action=masquerade

# VPN dan Routes
/interface pptp-client add name=vpn-to-pusat connect-to=3.3.12.13 user=cabang1 password=cabang1pass disabled=no
/ip route
add dst-address=192.168.12.0/24 gateway=vpn-to-pusat comment="VPN Route ke LAN Pusat"
add dst-address=0.0.0.0/0 gateway=1.1.1.1 comment="Default via R1"

# DNS
/ip dns set servers=3.3.12.3
```

---

## KONFIGURASI R-CABANG-2 (KANTOR CABANG 2)

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
/system identity set name="R-Cabang-2"  
```

### Complete Network Configuration
```bash
# IP Configuration
/ip address
add address=2.2.2.12/24 interface=ether1 comment="Link ke R2"
add address=192.168.2.12/24 interface=ether2 comment="LAN Gateway"

# NAT dan Routes
/ip firewall nat add chain=srcnat src-address=192.168.2.0/24 out-interface=ether1 action=masquerade
/ip route add dst-address=0.0.0.0/0 gateway=2.2.2.2 comment="Default via R2"

# DNS
/ip dns set servers=3.3.12.3
```

---

## KONFIGURASI R-PUSAT (KANTOR PUSAT + VPN SERVER)

### Reset dan Identity
```bash
/system reset-configuration no-defaults=yes skip-backup=yes
/system identity set name="R-Pusat"
```

### Complete Network Configuration
```bash
# IP Configuration  
/ip address
add address=3.3.12.13/24 interface=ether1 comment="Link ke R3"
add address=192.168.12.13/24 interface=ether2 comment="LAN Gateway"

# Static Route dan DNS
/ip route add dst-address=0.0.0.0/0 gateway=3.3.12.3 comment="Default via R3"
/ip dns set servers=3.3.12.3
```

### VPN Server Configuration - Consolidated
```bash
# VPN Pool dan Profile
/ip pool add name=vpn-pool ranges=192.168.100.10-192.168.100.20
/ppp profile add name=vpn-profile local-address=192.168.100.1 remote-address=vpn-pool use-encryption=yes

# PPTP Server dan Users
/interface pptp-server server set enabled=yes default-profile=vpn-profile authentication=mschap2
/ppp secret
add name=cabang1 password=cabang1pass profile=vpn-profile service=pptp comment="Office-to-Office VPN"
add name=pc2user password=pc2userpass profile=vpn-profile service=pptp comment="Client-to-Office VPN"
```

### All-in-One NAT Configuration
```bash
/ip firewall nat
# Web Server Port Forward
add chain=dstnat dst-address=3.3.12.13 dst-port=80 protocol=tcp action=dst-nat to-addresses=192.168.12.10 to-ports=80 comment="Web Server Port Forward"

# Source NAT untuk semua outbound traffic
add chain=srcnat src-address=192.168.12.0/24,192.168.100.0/24 out-interface=ether1 action=masquerade comment="NAT untuk LAN Pusat dan VPN Pool"
```

### Essential Firewall Rules - Streamlined
```bash
/ip firewall filter
add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input dst-port=1723 protocol=tcp action=accept comment="PPTP VPN"
add chain=input protocol=gre action=accept comment="GRE for VPN"
add chain=input protocol=icmp src-address=192.168.100.0/24 action=accept comment="VPN Ping Only"
add chain=input protocol=icmp action=drop comment="Block Other Ping"
add chain=input dst-port=80 protocol=tcp action=accept comment="Web Server"
add chain=input protocol=ospf action=accept comment="OSPF"
add chain=input action=drop comment="Drop All Other"
```

### Static Routes
```bash
/ip route add dst-address=0.0.0.0/0 gateway=3.3.12.3 comment="Default via R3"
```

### DNS Configuration
```bash
/ip dns set servers=3.3.12.3
```

---

## KONFIGURASI STATIC IP DEVICES

### PC1 (Client Cabang-1)
- **IP Address:** 192.168.1.10/24
- **Gateway:** 192.168.1.11 (R-Cabang-1)
- **DNS:** 3.3.12.3 (R3)

### PC2 (Client Cabang-2) - PPTP VPN Client  
- **IP Address:** 192.168.2.10/24
- **Gateway:** 192.168.2.12 (R-Cabang-2)
- **DNS:** 3.3.12.3 (R3)
- **PPTP VPN Client:** Connect to 3.3.12.13 (username: pc2user, password: pc2userpass)

### Server (Web Server Pusat)
- **IP Address:** 192.168.12.10/24
- **Gateway:** 192.168.12.13 (R-Pusat)
- **DNS:** 3.3.12.3 (R3)
- **Web Server:** Apache/Nginx running on port 80

### PC Public (Client Luar)
- **IP Address:** 4.4.4.10/24
- **Gateway:** 4.4.4.3 (R3)  
- **DNS:** 4.4.4.3 (R3)

---

## PENGUJIAN PROJECT SESUAI REQUIREMENTS

### 1. Pengujian Jaringan OSPF → PING Test
**Dari R-Cabang-1:**
```bash
/ping 3.3.12.13 count=5
```
**Dari R-Cabang-2:**
```bash
/ping 3.3.12.13 count=5
```
**Expected Result:** ✅ PING berhasil melalui jalur OSPF (R-Cabang → R1/R2 → R3 → R-Pusat)

### 2. Pengujian Koneksi Internet → Distributed Default Route OSPF
**Test dari PC1 (192.168.1.10):**
- Buka browser → http://google.co.id

**Test dari PC2 (192.168.2.10):**
- Buka browser → http://google.co.id

**Expected Result:** ✅ Website Google terbuka normal (NAT dari kantor → OSPF → R3 → Internet)

### 3. Pengujian Firewall DST-NAT → Akses Web Server via Domain
**Test dari PC-Public (4.4.4.10):**
- Buka browser → http://kantorpusat.co.id

**Test dari PC-1 (192.168.1.10):**
- Buka browser → http://kantorpusat.co.id

**Expected Result:** ✅ Web server dapat diakses dari kedua lokasi tanpa VPN melalui port forwarding (DST-NAT)

### 4. Pengujian VPN → Akses Web Server Langsung via IP Internal
**Test dari PC-1 (via VPN Office-to-Office):**
- Akses otomatis via VPN tunnel R-Cabang-1 ↔ R-Pusat
- Buka browser → http://192.168.12.10

**Test dari PC-2 (via VPN Client-to-Office):**  
- Connect PPTP VPN client terlebih dahulu ke 3.3.12.13
- Setelah VPN connected, buka browser → http://192.168.12.10

**Expected Result:** ✅ Web server dapat diakses langsung via IP internal melalui VPN tunnel

### 5. Pengujian Firewall Filter R-Pusat → Selective Ping Access (Updated Test)
**Test dari PC-2 dengan VPN Connected (192.168.100.x):**
```bash
ping 3.3.12.13
```
**Expected:** ✅ SUCCESS (Allowed - VPN client dalam range 192.168.100.0/24)

**Test dari PC-1 (192.168.1.10) tanpa VPN:**
```bash
ping 3.3.12.13  
```
**Expected:** ❌ TIMEOUT/FAILED (Blocked - bukan VPN client)

**Test dari PC-Public (4.4.4.10):**
```bash
ping 3.3.12.13
```
**Expected:** ❌ TIMEOUT/FAILED (Blocked - bukan VPN client)

---

## MONITORING DAN TROUBLESHOOTING COMMANDS

### Cek OSPF Status
```bash
/routing ospf neighbor print
/routing ospf lsa print  
/routing ospf interface-template print
/ip route print where ospf
```

### Cek VPN Status
```bash
/ppp active print
/interface pptp-client print
/interface pptp-server print
/ppp secret print
```

### Cek NAT dan Firewall
```bash
/ip firewall nat print
/ip firewall filter print
/ip firewall connection print
```

### Network Testing
```bash
/ping [destination] count=5
/tool traceroute [destination]
/tool bandwidth-test [destination]
```

### DNS Testing
```bash
/ip dns cache print
/tool nslookup name=kantorpusat.co.id server=3.3.12.3
```

---

## OPTIMISASI YANG DILAKUKAN

### **Firewall Rules Optimization:**
- **Sebelum:** 5 aturan ping yang redundan (allow cabang-1, allow cabang-2, allow VPN, block public, block others)
- **Sesudah:** 2 aturan saja (allow VPN clients, block all others)
- **Benefit:** Lebih efisien dan mudah maintenance

### **Logic Perubahan:**
1. **PC Cabang (192.168.1.x & 192.168.2.x)** → Harus menggunakan VPN untuk ping ke R-Pusat
2. **VPN Clients (192.168.100.x)** → Diizinkan ping
3. **Semua network lain** → Diblok secara default

### **Keuntungan Optimisasi:**
- Mengurangi kompleksitas firewall rules
- Memaksa penggunaan VPN untuk akses dari kantor cabang (lebih secure)
- Lebih mudah untuk troubleshooting dan maintenance
- Performance router lebih optimal

---

## CATATAN PENTING

### **OSPF Area Configuration:**
- OSPF **HANYA** berjalan di area "lingkaran biru muda": R1 ↔ R2 ↔ R3
- Router kantor (R-Pusat, R-Cabang-1, R-Cabang-2) terhubung ke router core tapi **TIDAK** menjalankan OSPF
- Menggunakan static routing untuk komunikasi antara router core dan router kantor

### **NAT Implementation:**
- Setiap kantor menggunakan NAT untuk traffic keluar dari jaringan LAN mereka
- NAT memungkinkan akses internet dari setiap kantor melalui jalur OSPF ke R3

### **VPN Configuration:**
- **Office-to-Office:** R-Cabang-1 otomatis terkoneksi ke R-Pusat via PPTP
- **Client-to-Office:** PC-2 manual connect ke R-Pusat via PPTP client

### **DNS and Domain:**
- R3 sebagai DNS Server untuk seluruh jaringan
- Domain kantorpusat.co.id → 3.3.12.13 (Public IP R-Pusat)
- Semua device menggunakan R3 sebagai DNS server

### **Security Implementation (Updated):**
- Port forwarding (DST-NAT) memungkinkan akses web server dari luar tanpa VPN
- Firewall filter di R-Pusat **HANYA** mengizinkan ping dari VPN clients (lebih restrictive)
- VPN memberikan akses langsung ke jaringan internal dan privilege untuk ping

---

**SELESAI - KONFIGURASI FINAL YANG SUDAH DIOPTIMALKAN**

*Konfigurasi ini sudah dioptimalkan dengan menghilangkan redundansi firewall rules, membuat keamanan lebih ketat dengan memaksa penggunaan VPN untuk akses dari kantor cabang, dan meningkatkan efisiensi konfigurasi secara keseluruhan.*
