# LAPORAN RESMI PRAKTIKUM JARINGAN KOMPUTER

## Modul: The Wired — Konfigurasi Jaringan, Layanan, dan Forensik Paket

**Kelompok** : K-36 <br>
**Anggota** :

- Nazwa Aulia Dwi Purnomo — 5027251118
- Putri Permata Sabila — 5027251047

## Topologi Jaringan

Router Lain memiliki 4 interface: eth0 terhubung ke cloud NAT1 (internet publik), eth1 ke Switch1 (Alice dan Mika), eth2 ke Switch2 (Chisa), dan eth3 ke Switch3 (Knights dan Eiri). Skema IP yang dipakai kelompok K-36:

| Node    | Interface | IP Address       | Gateway     |
| ------- | --------- | ---------------- | ----------- |
| Lain    | eth0      | DHCP (dari NAT1) | -           |
| Lain    | eth1      | 172.16.36.1/24   | -           |
| Lain    | eth2      | 172.16.37.1/24   | -           |
| Lain    | eth3      | 172.16.38.1/24   | -           |
| Alice   | eth0      | 172.16.36.10/24  | 172.16.36.1 |
| Mika    | eth0      | 172.16.36.20/24  | 172.16.36.1 |
| Chisa   | eth0      | 172.16.37.10/24  | 172.16.37.1 |
| Knights | eth0      | 172.16.38.10/24  | 172.16.38.1 |
| Eiri    | eth0      | 172.16.38.20/24  | 172.16.38.1 |

---

## Laporan Resmi

### Soal 1

Pada soal 1 kita diminta membangun topologi The Wired di GNS3: router Lain dengan tiga switch/gateway (Switch1 ke Alice & Mika, Switch2 ke Chisa, Switch3 ke Knights & Eiri), lalu menyambungkan router Lain ke internet publik lewat NAT/DHCP di eth0, dan memastikan semua entitas bisa saling komunikasi.

#### A) Konfigurasi IP tiap interface di Router Lain

```sh
ip addr add 172.16.36.1/24 dev eth1
ip addr add 172.16.37.1/24 dev eth2
ip addr add 172.16.38.1/24 dev eth3
dhclient eth0
```

`dhclient eth0` dipakai supaya eth0 dapat IP otomatis dari cloud NAT1 di GNS3, sedangkan eth1-eth3 di-set statis karena jadi gateway tiap subnet client.

#### B) Mengaktifkan IP forwarding

```sh
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Baris ini yang bikin Lain bisa meneruskan paket antar interface (jadi router beneran), bukan cuma endpoint. Tanpa ini, client di subnet berbeda gak akan bisa saling ping walau satu router yang sama.

#### C) Konfigurasi IP dan gateway di tiap client

```sh
# Contoh di node Alice
ip addr add 172.16.36.10/24 dev eth0
ip route add default via 172.16.36.1
```

Perintah yang sama disesuaikan IP-nya untuk Mika, Chisa, Knights, dan Eiri. Karena semua subnet langsung terhubung ke Lain (connected route), gak perlu static route tambahan, cukup pastikan tiap client set default gateway ke interface Lain yang sesuai.

#### Output

<!-- masukkan screenshot ip -br a dari router Lain dan hasil ping antar entitas -->

---

### Soal 2

Soal ini minta Lain dikonfigurasi NAT Masquerade dan DNS resolver supaya tiap client bisa internetan sendiri (ping 8.8.8.8 dan buka google.com), bukan cuma saling terhubung ke sesama client.

#### A) NAT Masquerade

```sh
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth3 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
```

MASQUERADE nge-translate source IP privat client jadi IP publik eth0 pas keluar ke internet, dan aturan FORWARD di atas cuma ngizinin trafik yang memang berasal dari inisiasi client (state RELATED,ESTABLISHED) buat masuk balik.

#### B) DNS Resolver

```sh
apt install dnsmasq -y
```

```conf
# /etc/dnsmasq.conf
interface=eth1
interface=eth2
interface=eth3
server=8.8.8.8
```

```sh
systemctl restart dnsmasq
```

Client cukup arahkan `/etc/resolv.conf` ke IP Lain di subnet masing-masing (misal `nameserver 172.16.36.1` di Alice/Mika), dnsmasq di Lain yang meneruskan query ke 8.8.8.8.

#### Output

<!-- masukkan screenshot ping 8.8.8.8 dan curl/browser buka google.com dari salah satu client -->

---

### Soal 3

Eiri berusaha bikin kekacauan lewat restart mendadak, jadi soal ini minta konfigurasi jaringan tetap ada walau node di-restart, plus script verifikasi `/root/cek_status.sh`.

#### A) Menyimpan konfigurasi IP secara permanen

```conf
# /etc/network/interfaces
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 172.16.36.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 172.16.37.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 172.16.38.1
    netmask 255.255.255.0
```

#### B) Menyimpan aturan iptables

```sh
apt install iptables-persistent -y
netfilter-persistent save
```

`iptables-persistent` bakal otomatis restore semua rule NAT/FORWARD dari `/etc/iptables/rules.v4` setiap boot, jadi gak perlu ngetik ulang manual.

#### C) Script verifikasi `/root/cek_status.sh`

```sh
#!/bin/bash
echo "=== Ringkasan Interface ==="
ip -br a
echo ""
echo "=== Status NAT Table ==="
iptables -t nat -L -v -n
```

```sh
chmod +x /root/cek_status.sh
```

Script ini tinggal dijalankan manual (`./cek_status.sh`) tiap habis reboot buat memastikan IP dan NAT masih sesuai konfigurasi awal.

#### Output

<!-- masukkan screenshot hasil cek_status.sh setelah node di-restart -->

---

### Soal 4

Mika curiga ada anomali traffic, jadi kita jalankan traffic generator di node Mika lalu sniffing pakai Wireshark dengan filter khusus DNS dan ICMP.

```sh
# jalankan traffic generator yang disediakan soal
python3 traffic_generator.py
```

Sambil traffic generator jalan, buka Wireshark di interface eth0 node Mika, lalu terapkan display filter:

dns || icmp

Filter ini nyaring dua jenis paket: query/response DNS (biasanya nunjukin domain apa aja yang diakses) dan paket ICMP (echo request/reply, bisa nunjukin ping mencurigakan/scanning).

#### Output

<!-- masukkan screenshot hasil filter Wireshark dan ringkasan jumlah paket DNS/ICMP yang lolos -->

---

### Soal 5

Chisa mendirikan FTP Server dengan shared folder `/var/wired/data`, dengan kebijakan alice full akses, mika read-only, dan eiri diblacklist total.

#### A) Instalasi dan setup folder

```sh
apt install vsftpd -y
mkdir -p /var/wired/data
chmod 755 /var/wired/data
```

#### B) Konfigurasi dasar vsftpd

```conf
# /etc/vsftpd.conf
local_enable=YES
write_enable=YES
chroot_local_user=YES
local_root=/var/wired/data
user_config_dir=/etc/vsftpd/user_conf
userlist_enable=YES
userlist_deny=YES
userlist_file=/etc/vsftpd/blacklist
```

#### C) Kebijakan per user

```sh
useradd -M -d /var/wired/data alice
useradd -M -d /var/wired/data mika
useradd -M -d /var/wired/data eiri
echo "alice:passalice" | chpasswd
echo "mika:passmika" | chpasswd
```

```conf
# /etc/vsftpd/user_conf/mika
write_enable=NO
```

```sh
# /etc/vsftpd/blacklist
eiri
```

`user_config_dir` memungkinkan override konfigurasi global per user. Mika di-override `write_enable=NO` supaya cuma bisa read, sedangkan eiri langsung dimasukkan ke `userlist_file` dengan `userlist_deny=YES` sehingga login-nya ditolak dari awal sebelum sempat autentikasi.

```sh
systemctl restart vsftpd
```

#### D) Pembuktian

```sh
# dari node Alice
ftp 172.16.37.10
# login alice, lalu:
put signal_alice.txt
```

```sh
# dari node Eiri
ftp 172.16.37.10
# login eiri -> ditolak
```

#### Output

<!-- masukkan screenshot signal_alice.txt berhasil terupload dan login eiri ditolak -->

---

### Soal 6

Knights mengirim dokumen ke FTP Server Chisa pakai akun alice, lalu kita analisis sesi FTP-nya di Wireshark.

```sh
# dari node Knights
ftp 172.16.37.10
# login alice
put laporan_intelijen.pdf
```

Capture Wireshark di interface Knights dengan filter:

ftp || ftp-data

Dari capture ini yang perlu diidentifikasi:

- Perintah `STOR laporan_intelijen.pdf` sebagai perintah upload
- Response `226 Transfer complete` sebagai kode sukses
- Port data TCP yang dinegosiasikan lewat perintah `PASV`, biasa muncul di response `227 Entering Passive Mode (h1,h2,h3,h4,p1,p2)` dengan port = `p1*256 + p2`

#### Output

<!-- masukkan screenshot detail paket STOR, 226, dan PASV dari Wireshark -->

---

### Soal 7

Mika mengunduh dokumen Protokol Tujuh dari FTP Server Chisa, lalu membuktikan pembatasan read-only dengan mencoba upload dan menangkap error 550.

```sh
# dari node Mika
ftp 172.16.37.10
# login mika
get protokol_tujuh.pdf
put file_baru.txt
```

Karena akun mika sudah di-override `write_enable=NO`, percobaan `put` bakal ditolak server dengan response `550 Permission denied`.

#### Output

<!-- masukkan screenshot proses get berhasil dan pesan error 550 saat put -->

---

### Soal 8

Knights menguji ketahanan koneksi ke Chisa dengan ping payload 128 byte, interval 0.3 detik, sebanyak 77 paket.

```sh
ping -c 77 -s 128 -i 0.3 172.16.37.10
```

Capture Wireshark dengan filter `icmp`, lalu identifikasi:

- Echo Request: ICMP Type 8, Code 0
- Echo Reply: ICMP Type 0, Code 0

Packet loss dan RTT (min/avg/max) langsung terbaca dari ringkasan output `ping` di terminal, bagian `--- 172.16.37.10 ping statistics ---`.

#### Output

<!-- masukkan screenshot output ping dan detail Type/Code ICMP di Wireshark -->

---

### Soal 9

Soal ini membuktikan kelemahan Telnet lewat akun phantom_user di Chisa yang diakses dari Eiri, lalu credential-nya ditangkap plaintext di Wireshark.

```sh
# di node Chisa
apt install telnetd xinetd -y
useradd phantom_user
echo "phantom_user:wired_ghost" | chpasswd
systemctl restart xinetd
```

```sh
# dari node Eiri
telnet 172.16.37.10
```

Capture Wireshark di interface Eiri lalu klik kanan salah satu paket TCP sesi telnet, pilih **Follow > TCP Stream**. Username dan password akan terlihat dalam bentuk teks biasa karena Telnet tidak melakukan enkripsi apapun pada payload-nya.

Alasan tiap karakter terkirim dalam paket TCP terpisah adalah karena Telnet secara default berjalan dalam mode karakter-per-karakter (bukan line-buffered), setiap tombol yang ditekan langsung dikirim sebagai satu paket TCP kecil ke server untuk mendukung echo interaktif secara real-time.

#### Output

<!-- masukkan screenshot Follow TCP Stream yang menampilkan kredensial plaintext -->

---

### Soal 10

Alice curiga Knights menjalankan layanan rahasia, jadi dilakukan port scanning pakai Netcat ke port 22, 80, dan 7777.

```sh
# dari node Alice
nc -zv 172.16.38.10 22
nc -zv 172.16.38.10 80
nc -zv 172.16.38.10 7777
```

Capture Wireshark filter `tcp.flags.syn==1` di interface Alice. Untuk port terbuka (22, 80), server membalas dengan flag **SYN-ACK**. Untuk port tertutup (7777), server membalas dengan flag **RST-ACK** karena tidak ada service yang listen di port tersebut sehingga koneksi langsung ditolak oleh kernel.

#### Output

<!-- masukkan screenshot hasil nc dan perbandingan SYN-ACK vs RST-ACK di Wireshark -->

---
