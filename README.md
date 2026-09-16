# Jarkom-Modul-1-2026-K-36

**Kelompok** : K-36

**Anggota** :

| Nama                    | NRP        |
| ----------------------- | ---------- |
| Nazwa Aulia Dwi Purnomo | 5027251118 |
| Putri Permata Sabila    | 5027251047 |

## Laporan Resmi

### Soal 1

Soal 1 minta router Lain bikin 3 Switch/Gateway: Switch 1 ke Alice & Mika, Switch 2 ke Chisa, Switch 3 ke Knights & Eiri, semua Entitas dikonfigurasi sebagai Client di GNS3 pakai prefix IP kelompok.

Router (Lain) — eth1, eth2, eth3 masing-masing jadi gateway satu switch:

```sh
auto eth1
iface eth1 inet static
address 192.229.1.1
netmask 255.255.255.0

auto eth2
iface eth2 inet static
address 192.229.2.1
netmask 255.255.255.0

auto eth3
iface eth3 inet static
address 192.229.3.1
netmask 255.255.255.0
```

Alice dan Mika (Switch 1):

```sh
# Alice
auto eth0
iface eth0 inet static
address 192.229.1.2
netmask 255.255.255.0
gateway 192.229.1.1

# Mika
auto eth0
iface eth0 inet static
address 192.229.1.3
netmask 255.255.255.0
gateway 192.229.1.1
```

Chisa (Switch 2)

```sh
# Chisa
auto eth0
iface eth0 inet static
address 192.229.2.2
netmask 255.255.255.0
gateway 192.229.2.1

```

Eiri dan Knights (Switch 3):

```sh
# Eiri
auto eth0
iface eth0 inet static
address 192.229.3.2
netmask 255.255.255.0
gateway 192.229.3.1

# Knights
auto eth0
iface eth0 inet static
address 192.229.3.3
netmask 255.255.255.0
gateway 192.229.3.1

```

#### Output

## ![Topologi Jaringan](images/topologi.png)

### Soal 2

Soal 2 minta router Lain konek ke internet publik lewat NAT/DHCP di interface eth0, soalnya The Wired awalnya masih terisolasi.

```sh
auto eth0
iface eth0 inet dhcp
```

eth0 dibiarin dapat IP otomatis dari DHCP jaringan luar, jadi router langsung punya akses internet tanpa perlu setting IP manual.

---

### Soal 3

Pada soal 3 seluruh Entitas di bawah Switch 1, 2, dan 3 harus bisa saling terhubung dan berkomunikasi lewat konfigurasi routing.

Karena router Lain terhubung langsung ke ketiga subnet lewat eth1, eth2, dan eth3, routing antar subnet sudah terbentuk otomatis dari routing table router tanpa perlu tambahan konfigurasi khusus. Pembuktiannya dilakukan dengan ping ke IP Address Entitas di subnet lain:

```sh
ping -c 2 <IP_tujuan>
```

#### Output

![](images/ping-antar-subnet1.png)
![](images/ping-antar-subnet2.png)
![](images/ping-antar-subnet3.png)
![](images/ping-antar-subnet4.png)

---

#### Soal 4

Pada soal 4 tiap Entitas diminta mandiri akses internet, bisa ping ke 8.8.8.8 dan buka google.com, lewat konfigurasi firewall/iptables NAT Masquerade dan DNS resolver.

Langkah pertama, buat file `router.sh` di router Lain yang isinya command NAT dan forwarding:

```sh
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
```

`MASQUERADE` dan `FORWARD` ini dua hal yang beda fungsi. `MASQUERADE` di tabel `nat` bertugas nge-rewrite source IP private client (192.229.x.x) jadi IP publik router pas paket keluar lewat eth0, supaya balasan dari internet bisa balik lagi ke router dan di-translate ulang ke client yang benar. Tanpa ini, client dengan IP private gak akan bisa langsung diajak komunikasi sama server di internet.

`FORWARD` di sisi lain ngatur izin, boleh atau enggak sebuah paket lewat (diteruskan) dari satu interface ke interface lain di router. Default policy `FORWARD` di banyak sistem itu `ACCEPT`, makanya soal 3 tadi bisa langsung jalan tanpa rule tambahan. Tapi kalau default policy-nya `DROP`, paket dari client yang mau keluar ke eth0 (internet) bakal ketahan di router meskipun NAT-nya udah bener. Jadi tiga baris `ACCEPT` itu ditambahin buat mastiin secara eksplisit trafik dari eth1, eth2, dan eth3 menuju eth0 diizinkan lewat, jaga-jaga kalau default policy-nya ketat.

Untuk DNS resolver, tiap client diset otomatis pas interface naik, contoh di Eiri:

```sh
auto eth0
iface eth0 inet static
address 192.229.3.3
netmask 255.255.255.0
gateway 192.229.3.1
up echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

`8.8.8.8` itu alamat Google Public DNS, dipakai supaya client bisa nerjemahin nama domain (google.com) jadi IP address. Tanpa ini client cuma bisa akses internet pakai IP langsung, gak bisa buka website lewat nama domainnya.

Baris `up echo "nameserver 8.8.8.8" > /etc/resolv.conf` itu bagian dari `/etc/network/interfaces`, bukan file terpisah. `up` di sini artinya command yang dijalankan otomatis tiap kali interface eth0 naik. Isi command-nya sendiri nulis ke `/etc/resolv.conf`, file khusus yang nyimpen daftar DNS server yang dipakai sistem buat resolve domain (terpisah dari `/etc/network/interfaces` yang isinya konfigurasi IP address dan routing interface). Jadi gak perlu edit manual `/etc/resolv.conf` di terminal, cukup taruh baris `up echo ...` itu di dalam `/etc/network/interfaces` (edit pakai `nano /etc/network/interfaces` di terminal), dan tiap kali eth0 up, `/etc/resolv.conf` otomatis ke-generate ulang. Cara yang sama diterapin ke Alice, Mika, Chisa, dan Knights.

#### Output

![](images/ping-8.8.8.8.png)
![](images/ping-google-alice.png.png)

---

### Soal 5

Pada soal 5 diminta membuat script verifikasi `/root/cek_status.sh` di router Lain yang menampilkan ringkasan interface (`ip -br a`) dan status tabel NAT (`iptables -t nat -L -v -n`), buat antisipasi kalau node di-restart mendadak.

```sh
nano /root/cek_status.sh
```

Isi `cek_status.sh`:

```sh
#!/bin/bash
ip -br a
iptables -t nat -L -v -n
```

```sh
chmod +x /root/cek_status.sh
```

Script ini ditaruh di `init.sh` supaya otomatis jalan tiap kali router boot atau di-restart:

```sh
/root/router.sh
/root/cek_status.sh
```

Jadi begitu router Lain nyala ulang, `router.sh` langsung nerapin ulang aturan NAT dan forwarding, sedangkan `cek_status.sh` langsung nampilin ringkasan interface dan status tabel NAT buat mastiin konfigurasinya masih sesuai, tanpa perlu ngecek manual satu-satu.

#### Output

![](images/no-5.png.png)

---

### Soal 6

Pada soal 6 diminta menjalankan traffic generator di node Mika, lalu sniffing pakai Wireshark dengan display filter khusus untuk paket DNS atau ICMP.

File generator diunduh dari link yang dikasih, isinya di-copas ke node Mika jadi `traffic_protocol7.sh`:

```sh
nano traffic_protocol7.sh
```

```sh
#!/bin/bash
# ============================================
# Traffic Generator — Protocol 7 Network
# Serial Experiments Lain — Modul 1 Jarkom 2026
# Jalankan di node MIKA untuk generate traffic DNS & ICMP
# ============================================

echo "============================================"
echo "  Protocol 7 Traffic Generator v2026"
echo "  Node: Mika Iwakura"
echo "============================================"
echo "[*] Generating DNS & ICMP traffic..."

# ICMP Traffic
ping -c 5 8.8.8.8 &
ping -c 5 1.1.1.1 &
ping -c 3 its.ac.id &

# DNS Queries
nslookup google.com 8.8.8.8 &
nslookup its.ac.id 8.8.8.8 &
nslookup github.com 1.1.1.1 &
dig @8.8.8.8 example.com A &
dig @1.1.1.1 cloudflare.com AAAA &

wait
echo "[*] Traffic generation complete."
echo "[*] Check Wireshark for captured packets."
```

```sh
chmod +x traffic_protocol7.sh
```

Script ini generate dua jenis traffic sekaligus secara paralel (pakai `&`), ICMP lewat `ping` ke beberapa target (8.8.8.8, 1.1.1.1, its.ac.id) dan DNS query lewat `nslookup` dan `dig` ke beberapa domain. Baris `wait` di akhir mastiin script nunggu semua proses background itu selesai dulu sebelum nampilin pesan selesai.

Sebelum script dijalankan, capture Wireshark di interface node Mika distart dulu, baru script dieksekusi:

```sh
./traffic_protocol7.sh
```

Setelah trafficnya kerekam, di Wireshark diterapin display filter:

```
dns or icmp
```

#### Output

![](images/no-6.jpeg)

### Soal 7

Pada soal 7 Chisa mendirikan FTP Server dengan shared folder `/var/wired/data`. Kebijakan aksesnya, alice dapat read dan write, mika dibatasi read-only, eiri dibatasi tanpa izin akses sama sekali.

Bikin folder shared dulu:

```sh
mkdir -p /var/wired/data
chown root:root /var/wired/data
```

Bikin 3 user OS yang bakal jadi akun FTP:

```sh
adduser -D alice
adduser -D mika
adduser -D eiri
```

Install tools yang dibutuhin, termasuk `shadow` supaya `usermod` bisa dipakai:

```sh
apk update
apk add shadow
apk add vsftpd
```

Bikin grup akses khusus, alice dan mika dimasukin ke grup ini:

```sh
addgroup ftpaccess
adduser alice ftpaccess
adduser mika ftpaccess
```

Arahin home directory ketiga user ke folder shared:

```sh
usermod -d /var/wired/data alice
usermod -d /var/wired/data mika
usermod -d /var/wired/data eiri
```

```sh
chown root:ftpaccess /var/wired/data
chmod 770 /var/wired/data
```

Whitelist alice dan mika di userlist, otomatis eiri keblacklist karena gak masuk daftar:

```sh
echo -e "alice\nmika" > /etc/vsftpd.userlist
```

Setting read-only khusus buat mika:

```sh
mkdir -p /etc/vsftpd/user_conf
echo "write_enable=NO" > /etc/vsftpd/user_conf/mika
```

Isi `/etc/vsftpd.conf`:

```sh
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_root=/var/wired/data
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
user_config_dir=/etc/vsftpd/user_conf
seccomp_sandbox=NO
pam_service_name=vsftpd
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=30100
file_open_mode=0666
local_umask=002
```

Jalankan servernya:

```sh
vsftpd /etc/vsftpd.conf &
```

Hasil testing pakai lftp menunjukkan user alice bisa melakukan put, get, dan ls karena punya akses penuh, user mika bisa get tapi gagal put karena `write_enable=NO`, sedangkan user eiri gagal login sama sekali sehingga semua command tidak bisa dijalankan.

#### Output

![](images/alice-check.png)
![](images/eiri-check.png)
![](images/mika-check.png)

### Soal 8

Soal 8 minta Knights connect FTP ke server Chisa pakai akun alice buat upload dokumen, terus dianalisis di Wireshark: perintah STOR, status 226, dan port data PASV.

```sh
nano knights_report.txt
# isi file, copas dari drive
lftp alice@192.229.2.2
put knights_report.txt
```

Analisis dari Follow TCP Stream di Wireshark:

- Login: `USER alice` → `331 Please specify the password.` → `PASS ...` → `230 Login successful.`
- Mode binary: `TYPE I`
- Negosiasi PASV: request `PASV` → response `227 Entering Passive Mode (192.229.2.2,117,57)`, artinya port data = 117×256+57 = **30009** (masuk range `pasv_min_port`-`pasv_max_port` 30000-30100 yang udah diset di vsftpd.conf)
- Upload: `STOR knights_report.txt` → `226 Transfer complete.`

#### Output

![](images/no-9.png)

---

### Soal 9

Soal 9 minta Mika download `protocol7_manifesto.txt` pakai akun mika, terus buktiin read-only-nya jalan pas nyoba upload.

```
lftp mika@192.229.2.2:~> ls
-rw-rw-r--   1 1000     1000        1111 Sep 15 20:49 knights_report.txt
-rw-r--r--   1 0        0           1738 Sep 15 21:18 protocol7_manifesto.txt
-rw-rw-r--   1 1000     1003          16 Sep 15 20:12 signal_alice.txt
lftp mika@192.229.2.2:~> get protocol7_manifesto.txt
1738 bytes transferred
lftp mika@192.229.2.2:~> put protocol7_manifesto.txt
put: Access failed: 550 Permission denied. (protocol7_manifesto.txt)
```

Terbukti mika bisa `get` tapi kena `550 Permission denied` pas `put`, sesuai `write_enable=NO` yang khusus diset buat user mika.

---

### Soal 10

Soal 10 minta Knights ping ke Chisa buat uji latensi, payload 128 byte, interval 0.3 detik, sebanyak 77 paket.

```sh
ping -c 77 -s 128 -i 0.3 192.229.2.2
```

Di Wireshark, tiap Echo Request (ICMP Type 8, Code 0) dibales Echo Reply (ICMP Type 0, Code 0) dengan id dan seq yang sama, TTL request 63 dan reply 64 (beda karena lewat hop yang beda).

Hasil statistik:

![](<images/no-10(2).png>)

Gak ada packet loss (0%), RTT stabil di kisaran 0.4-1.06 ms, artinya koneksi ke server Chisa lancar.

#### Output

![](<images/no-10(1).png>)

---

### Kendala saat mengerjakan

- kesusahan dalam menemukan config yang tepat pada saat nomor 7
- kurang familiar untuk bagaimana setup gns yang bisa nyambung dengan wireshark terkait
