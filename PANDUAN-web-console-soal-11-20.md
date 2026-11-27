# Panduan konfigurasi via Web Console (soal 11-20)

Semua dikerjakan dari **web console masing-masing node** di GNS3. Tidak perlu `vi`: setiap script ditulis dengan `cat > file << 'EOF'`.

## Aturan tempel
1. **Cek prompt dulu** (`root@NAMA`) sebelum menempel. Salah node = salah konfigurasi.
2. Tempel **satu blok kode sekaligus**, dari baris `cat >` sampai `wc -l`. Jangan ikut menempel tanda ``` atau komentar `# harus ...`.
3. Setelah tempel, cek `wc -l`: angkanya harus sama dengan yang tertulis. Kalau beda, ada baris terpotong, ulangi (script yang sama aman ditimpa).
4. Kalau web console memotong tempelan panjang, tempel per separuh: ketik `cat > /root/x.sh << 'EOF'`, tempel separuh isi, lalu separuh lagi, lalu `EOF`.
5. Jalankan dengan `bash /root/<nama>.sh`, bukan `sh`.
6. Urutan umum: Gendou online dulu (`bash /root/gendou.sh`, lalu router lain), baru node yang butuh `apt`.
7. Kalau `apt-get` gagal: `echo "nameserver 192.168.122.1" > /etc/resolv.conf`.

---

## Soal 11: Magi-Casper

Prompt: `root@Magi-Casper`

```
cat > /root/casper.sh << 'EOF'
#!/bin/bash
# casper.sh - soal 7 (listener TCP 5000) + soal 11 (Casper jadi hantu). Aman dijalankan berulang.
export DEBIAN_FRONTEND=noninteractive
APT="-o Acquire::Check-Date=false -o Acquire::ForceIPv4=true"

# 1. paket (dilewati kalau sudah ada)
if ! command -v nc.traditional >/dev/null || ! command -v iptables >/dev/null; then
  apt-get $APT update
  apt-get $APT install -y iptables netcat-traditional
fi

# 2. listener TCP 5000 (dinyalakan hanya kalau belum mendengar)
listen() {
  port=$1
  if ss -ltn | grep -q ":$port "; then
    echo "port $port sudah mendengar, dilewati"
  else
    setsid nohup bash -c "while true; do nc.traditional -l -p $port >> /root/pesan-$port.log; done" >/dev/null 2>&1 &
    sleep 1
    echo "listener port $port dinyalakan"
  fi
}
listen 5000

# 3. soal 11: tolak ping MASUK saja (echo-request). Ping KELUAR tetap jalan,
#    karena balasannya (echo-reply) adalah jenis paket lain. Policy tidak diubah, TCP 5000 tidak tersentuh.
iptables -N HANTU 2>/dev/null; iptables -F HANTU
iptables -A HANTU -p icmp --icmp-type echo-request -j DROP
iptables -C INPUT -j HANTU 2>/dev/null || iptables -I INPUT 1 -j HANTU
EOF
wc -l /root/casper.sh    # harus 29
```
```
bash /root/casper.sh
iptables -S
```
Harus ada `-N HANTU`, `-A INPUT -j HANTU`, `-A HANTU -p icmp -m icmp --icmp-type 8 -j DROP`.

**Uji:**
```
# di Magi-Balthasar lalu Lilith
ping -c 3 -W 2 192.237.4.58      # harus 100% loss

# di Magi-Casper
ping -c 3 192.237.4.59            # berhasil
ping -c 3 192.237.4.74            # berhasil
ss -ltn | grep :5000
iptables -L HANTU -n -v          # counter naik
```

---

## Soal 12: Magi-Casper (break_fw.sh)

Prompt: `root@Magi-Casper`

**a. Foto sebelum sabotase**
```
mkdir -p /root/baseline
cat > /root/snap.sh << 'EOF'
#!/bin/bash
{ date; ip -br addr; ip route; ip rule; sysctl net.ipv4.conf.all.rp_filter net.ipv4.ip_forward
  cat /etc/resolv.conf; iptables -S; iptables -t nat -S; iptables -t mangle -S; ip neigh; } 2>&1
EOF
bash /root/snap.sh > /root/baseline/sebelum.txt
```
**b. Unduh `break_fw.sh` (ganti `<ID_FILE>` dengan link dari asisten) dan jalankan tanpa membuka isinya**
```
curl -L -o /root/break_fw.sh "https://drive.usercontent.google.com/download?id=<ID_FILE>&export=download&confirm=t"
wc -c /root/break_fw.sh; sha256sum /root/break_fw.sh | cut -c1-16
sh /root/break_fw.sh
bash /root/snap.sh > /root/baseline/sesudah.txt
diff /root/baseline/sebelum.txt /root/baseline/sesudah.txt
```
Screenshot `diff` dan gejalanya: `ping -c 2 -W 2 8.8.8.8`, `apt-get -o Acquire::Check-Date=false update`, dan dari Unit-00 `nc -w 3 -z 192.237.4.58 5000; echo rc=$?`.

**c. Perbaikan**

```
cat > /root/casper-fix.sh << 'EOF'
#!/bin/bash
# casper-fix.sh - soal 12: perbaikan setelah break_fw.sh. Firewall Casper DIBANGUN ULANG dari nol
# supaya penyebab apa pun di tabel filter ikut bersih. Tujuan hardening tetap:
#   policy INPUT = DROP, ping masuk ditolak, semua masuk yang tidak diizinkan dibuang.
# Aman dijalankan berulang.

# 0. buka sementara supaya tidak terkunci saat flush
iptables -P INPUT ACCEPT; iptables -P FORWARD ACCEPT; iptables -P OUTPUT ACCEPT
iptables -F; iptables -X
iptables -t nat -F; iptables -t mangle -F

# 1. aturan izin (urutan penting)
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT   # balasan ping 8.8.8.8, DNS, apt
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP                # tetap hantu (soal 11)
iptables -A INPUT -p tcp --dport 5000 -j ACCEPT                           # layanan MAGI (Unit-00 dkk)

# 2. keluar bebas, masuk selain di atas dibuang
iptables -P OUTPUT ACCEPT
iptables -P FORWARD DROP
iptables -P INPUT DROP

# 3. pastikan jalur keluar dan DNS utuh
ip route replace default via 192.237.4.57
grep -q nameserver /etc/resolv.conf || echo "nameserver 192.168.122.1" > /etc/resolv.conf
EOF
wc -l /root/casper-fix.sh    # harus 25
```
```
bash /root/casper-fix.sh
bash /root/snap.sh > /root/baseline/pulih.txt
diff /root/baseline/sebelum.txt /root/baseline/pulih.txt
iptables -S | head -1      # -P INPUT DROP
```
**Uji:**
```
# di Unit-00
echo cek | nc -N -w 3 192.237.4.58 5000; echo rc=$?     # rc=0
# di Magi-Casper
tail -1 /root/pesan-5000.log
apt-get -o Acquire::Check-Date=false update           # berhasil
# di Magi-Balthasar
ping -c 2 -W 2 192.237.4.58                            # tetap gagal
```

---

## Soal 13: Magi-Balthasar

Prompt: `root@Magi-Balthasar`

```
cat > /root/balthasar.sh << 'EOF'
#!/bin/bash
# balthasar.sh - soal 7 (listener 5000, 5001) + soal 13 (brankas). Aman dijalankan berulang.
export DEBIAN_FRONTEND=noninteractive
APT="-o Acquire::Check-Date=false -o Acquire::ForceIPv4=true"

if ! command -v nc.traditional >/dev/null || ! command -v iptables >/dev/null; then
  apt-get $APT update
  apt-get $APT install -y iptables netcat-traditional
fi

listen() {
  port=$1
  if ss -ltn | grep -q ":$port "; then
    echo "port $port sudah mendengar, dilewati"
  else
    setsid nohup bash -c "while true; do nc.traditional -l -p $port >> /root/pesan-$port.log; done" >/dev/null 2>&1 &
    sleep 1
    echo "listener port $port dinyalakan"
  fi
}
listen 5000
listen 5001

# --- soal 13: hanya Casper + semua Pilot yang boleh ke TCP 5000 dan 5001 ---
CASPER="192.237.4.58"
PILOT="192.237.4.66 192.237.4.67 192.237.4.68 192.237.3.130"   # Unit-01, Unit-02, Unit-00, Sachiel

iptables -N BRANKAS 2>/dev/null; iptables -F BRANKAS
for src in $CASPER $PILOT; do
  iptables -A BRANKAS -s $src -p tcp -m multiport --dports 5000,5001 -j ACCEPT
done
# selain itu: ditolak SEKETIKA dengan TCP RST (bukan DROP yang membuat klien menggantung)
iptables -A BRANKAS -p tcp -m multiport --dports 5000,5001 -j REJECT --reject-with tcp-reset
iptables -C INPUT -j BRANKAS 2>/dev/null || iptables -I INPUT 1 -j BRANKAS
# policy INPUT tidak diubah, jadi ping/balasan/lalu lintas lain tidak terganggu
EOF
wc -l /root/balthasar.sh    # harus 35
```
```
bash /root/balthasar.sh
iptables -S BRANKAS
```
**Uji:**
```
# di Unit-01
time sh -c 'echo cek | nc -N -w 3 192.237.4.59 5000'; echo rc=$?
time sh -c 'echo cek | nc -N -w 3 192.237.4.59 5001'; echo rc=$?

# di Lilith, lalu Magi-Melchior (ulangi 5000 dan 5001)
time nc -v -w 5 192.237.4.59 5000; echo rc=$?     # refused seketika

# di Magi-Balthasar
tail /root/pesan-5000.log /root/pesan-5001.log
ping -c 2 8.8.8.8
iptables -L BRANKAS -n -v
```

---

## Soal 14: Pen-Pen

Prompt: `root@Pen-Pen`

```
cat > /root/pen-pen.sh << 'EOF'
#!/bin/bash
# pen-pen.sh - soal 8 (port 80) + soal 14 (jam kunjungan) + listener 9000 (soal 17)
# ---------------- JADWAL (iptables membaca UTC = WIB - 7 jam) ----------------
# Bisa ditimpa dari luar untuk uji:  PILOT_DAYS=Tue bash /root/pen-pen.sh
PILOT_START="${PILOT_START:-01:00}"                       # 08.00 WIB
PILOT_STOP="${PILOT_STOP:-09:00}"                         # 16.00 WIB
PILOT_DAYS="${PILOT_DAYS:-Mon,Tue,Wed,Thu,Fri}"           # Senin-Jumat
STAF_START="${STAF_START:-10:00}"                         # 17.00 WIB
STAF_STOP="${STAF_STOP:-16:00}"                           # 23.00 WIB (setiap hari)
# -----------------------------------------------------------------------------
export DEBIAN_FRONTEND=noninteractive
APT="-o Acquire::Check-Date=false -o Acquire::ForceIPv4=true"

if ! command -v nc.traditional >/dev/null || ! command -v iptables >/dev/null; then
  apt-get $APT update
  apt-get $APT install -y iptables netcat-traditional
fi

serve() {
  port=$1; text=$2
  if ss -ltn | grep -q ":$port "; then echo "port $port sudah mendengar, dilewati"
  else
    setsid nohup bash -c "while true; do echo '$text' | nc.traditional -l -p $port -q 1; done" >/dev/null 2>&1 &
    sleep 1; echo "layanan port $port dinyalakan"
  fi
}
listen() {
  port=$1
  if ss -ltn | grep -q ":$port "; then echo "port $port sudah mendengar, dilewati"
  else
    setsid nohup bash -c "while true; do nc.traditional -l -p $port >> /root/pesan-$port.log; done" >/dev/null 2>&1 &
    sleep 1; echo "listener port $port dinyalakan"
  fi
}
serve 80 "Welcome to $(hostname)"
listen 9000                                               # soal 17

# --- soal 14: jam kunjungan. Berlaku untuk ping (echo-request) dan TCP 80 dari mana pun ---
PILOT="192.237.4.66,192.237.4.67,192.237.4.68,192.237.3.130"   # Unit-01, Unit-02, Unit-00, Sachiel
STAF="192.237.4.74,192.237.4.75"                               # Lilith, Adam
DUMMY="192.237.4.50,192.237.4.51,192.237.4.102"                # Entry-Plug, Dummy-Plug, Spy

iptables -N KUNJUNGAN 2>/dev/null; iptables -F KUNJUNGAN
for m in "-p icmp --icmp-type echo-request" "-p tcp --dport 80"; do
  iptables -A KUNJUNGAN $m -s $DUMMY -j DROP                                  # tidak pernah
  iptables -A KUNJUNGAN $m -s $PILOT -m time --utc --timestart $PILOT_START --timestop $PILOT_STOP --weekdays $PILOT_DAYS -j ACCEPT
  iptables -A KUNJUNGAN $m -s $PILOT -j DROP                                  # pilot di luar jadwal
  iptables -A KUNJUNGAN $m -s $STAF  -m time --utc --timestart $STAF_START --timestop $STAF_STOP -j ACCEPT
  iptables -A KUNJUNGAN $m -s $STAF  -j DROP                                  # staf di luar jadwal
done
# sumber lain ("Lainnya") tidak cocok aturan mana pun, kembali ke INPUT dan lolos (bebas)
iptables -C INPUT -j KUNJUNGAN 2>/dev/null || iptables -I INPUT 1 -j KUNJUNGAN
EOF
wc -l /root/pen-pen.sh    # harus 52
```
```
bash /root/pen-pen.sh
iptables -S KUNJUNGAN
```
Kalau muncul `Couldn't load match 'time'`, cek `iptables -m time --help`.

**Kondisi 1: jadwal cocok** (di Pen-Pen)
```
date -u; date
PILOT_START=$(date -u -d '-1 hour' +%H:%M) PILOT_STOP=$(date -u -d '+1 hour' +%H:%M) PILOT_DAYS=$(date -u +%a) \
STAF_START=$(date -u -d '-1 hour' +%H:%M) STAF_STOP=$(date -u -d '+1 hour' +%H:%M) bash /root/pen-pen.sh
iptables -S KUNJUNGAN
```
Lalu di **Unit-01, Lilith, Spy**: `ping -c 3 -W 2 192.237.3.195`. Harapan: Unit-01 OK, Lilith OK, Spy gagal.

**Kondisi 2: jadwal tidak cocok** (di Pen-Pen)
```
PILOT_START=$(date -u -d '+2 hour' +%H:%M) PILOT_STOP=$(date -u -d '+3 hour' +%H:%M) \
STAF_START=$(date -u -d '+2 hour' +%H:%M) STAF_STOP=$(date -u -d '+3 hour' +%H:%M) bash /root/pen-pen.sh
iptables -S KUNJUNGAN
```
Ulangi ping dari ketiganya. Harapan: semuanya gagal.

---

## Soal 15 dan 16: Tokyo3-Core

Prompt: `root@Tokyo3-Core`

```
cat > /root/tokyo3-core.sh << 'EOF'
#!/bin/bash
# tokyo3-core.sh - soal 8 (80, 8080) + soal 15 (batas ping) + soal 16 (jebakan pemindai port)
export DEBIAN_FRONTEND=noninteractive
APT="-o Acquire::Check-Date=false -o Acquire::ForceIPv4=true"

if ! command -v nc.traditional >/dev/null || ! command -v iptables >/dev/null; then
  apt-get $APT update
  apt-get $APT install -y iptables netcat-traditional
fi

serve() {
  port=$1; text=$2
  if ss -ltn | grep -q ":$port "; then echo "port $port sudah mendengar, dilewati"
  else
    setsid nohup bash -c "while true; do echo '$text' | nc.traditional -l -p $port -q 1; done" >/dev/null 2>&1 &
    sleep 1; echo "layanan port $port dinyalakan"
  fi
}
serve 80 "Welcome to $(hostname)"
serve 8080 "syifa imutz"

# --- firewall: satu chain NERV, dibangun ulang tiap script dijalankan ---
DUMMY="192.237.4.51"     # Dummy-Plug
iptables -N NERV 2>/dev/null; iptables -F NERV

# soal 16: catat SETIAP koneksi baru (SYN) ke port sensitif, lalu banting dengan RST
iptables -A NERV -s $DUMMY -p tcp --syn -m multiport --dports 22,23,3306 -j LOG --log-prefix "SCAN-NERV: " --log-level 4
iptables -A NERV -s $DUMMY -p tcp -m multiport --dports 22,23,3306 -j REJECT --reject-with tcp-reset

# soal 15: HANYA echo-request masuk, maks 5/detik. Echo-reply (balasan ping milik Tokyo3-Core sendiri) tidak ikut terhitung
iptables -A NERV -p icmp --icmp-type echo-request -m limit --limit 5/second --limit-burst 5 -j ACCEPT
iptables -A NERV -p icmp --icmp-type echo-request -j DROP

# chain dipasang di posisi 1 supaya tidak ada ACCEPT lain yang mendahului
iptables -C INPUT -j NERV 2>/dev/null || iptables -I INPUT 1 -j NERV
EOF
wc -l /root/tokyo3-core.sh    # harus 35
```
```
bash /root/tokyo3-core.sh
iptables -L NERV -n -v --line-numbers
```
**Uji soal 15:**
```
# di Lilith
ping -i 0.1 -c 50 192.237.4.106        # sebagian hilang
ping -i 3 -c 5 192.237.4.106           # 0% loss
# di Tokyo3-Core
iptables -L NERV -n -v --line-numbers  # counter DROP naik
ping -i 0.1 -c 50 192.237.4.74         # ping keluar, 0% loss
```
**Uji soal 16:**
```
# di Dummy-Plug
for p in 22 23 3306; do time nc -w 5 192.237.4.106 $p; echo "port $p rc=$?"; done
nc -w 3 192.237.4.106 80
nc -w 3 192.237.4.106 8080
# di Tokyo3-Core
iptables -L NERV -n -v --line-numbers  # counter LOG dan REJECT naik
```

---

## Soal 17: Fuyutsuki, Pen-Pen, Zeruel, Casper

**Fuyutsuki** (prompt `root@Fuyutsuki`). Script ini menimpa `/root/fuyutsuki.sh` lama dengan versi yang sudah memuat soal 17.

```
cat > /root/fuyutsuki.sh << 'EOF'
#!/bin/bash
# fuyutsuki.sh - konfigurasi Fuyutsuki (aman dijalankan berulang). Soal 2-3 + soal 17 (DNAT jebakan)

# IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

ip addr replace 192.237.4.134/30   dev eth0   # uplink ke Gendou
ip addr replace 192.237.4.161/30   dev eth1   # link ke Maya
ip addr replace 192.237.4.165/30   dev eth2   # link ke Toji
ip addr replace 192.237.4.57/29    dev eth3   # gateway LAN Switch-Terminal
for i in 0 1 2 3; do ip link set eth$i up; done
# --- static routes ---
ip route replace 192.237.3.192/27  via 192.237.4.162   # Maya
ip route replace 192.237.4.72/29   via 192.237.4.166   # Toji
ip route replace default via 192.237.4.133   # Gendou

# pasang iptables kalau belum ada (butuh Gendou online)
if ! command -v iptables >/dev/null; then
  grep -q nameserver /etc/resolv.conf || echo "nameserver 192.168.122.1" > /etc/resolv.conf
  DEBIAN_FRONTEND=noninteractive apt-get -o Acquire::Check-Date=false -o Acquire::ForceIPv4=true update
  DEBIAN_FRONTEND=noninteractive apt-get -o Acquire::Check-Date=false -o Acquire::ForceIPv4=true install -y iptables
fi

# --- soal 17: TCP dari Casper ke Zeruel dialihkan ke Pen-Pen. Ping tidak (-p tcp saja) ---
# DNAT di PREROUTING. Balasan Pen-Pen lewat Fuyutsuki lagi (Maya -> Fuyutsuki), jadi conntrack
# otomatis mengubah sumber balasan kembali menjadi IP Zeruel. Tidak perlu MASQUERADE.
iptables -t nat -C PREROUTING -s 192.237.4.58 -d 192.237.4.82 -p tcp -j DNAT --to-destination 192.237.3.195 2>/dev/null || \
iptables -t nat -A PREROUTING -s 192.237.4.58 -d 192.237.4.82 -p tcp -j DNAT --to-destination 192.237.3.195
EOF
wc -l /root/fuyutsuki.sh    # harus 28
```
```
bash /root/fuyutsuki.sh
iptables -t nat -S PREROUTING
```
**Zeruel** (prompt `root@Zeruel`). `zeruel.sh` juga memuat isolasi soal 19, jadi **jangan dijalankan dulu**. Untuk soal 17 cukup listener:
```
export DEBIAN_FRONTEND=noninteractive
command -v nc.traditional >/dev/null || { apt-get -o Acquire::Check-Date=false -o Acquire::ForceIPv4=true update; apt-get -o Acquire::Check-Date=false -o Acquire::ForceIPv4=true install -y netcat-traditional; }
ss -ltn | grep -q ':9000 ' || { setsid nohup bash -c 'while true; do nc.traditional -l -p 9000 >> /root/pesan-9000.log; done' >/dev/null 2>&1 & sleep 1; }
ss -ltn | grep :9000
```
**Pen-Pen:** `bash /root/pen-pen.sh` (sudah memuat listener 9000 dari soal 14).

**Uji (di Magi-Casper):**
```
echo "pesan rahasia untuk Zeruel" | nc.traditional -w 3 192.237.4.82 9000
```
**Cek hasil:**
```
# di Pen-Pen
cat /root/pesan-9000.log        # pesan muncul di sini
# di Zeruel
cat /root/pesan-9000.log        # tidak ada pesan itu
# di Fuyutsuki
apt-get -o Acquire::Check-Date=false install -y conntrack
conntrack -L -p tcp | grep 9000      # atau: grep 9000 /proc/net/nf_conntrack
iptables -t nat -L PREROUTING -n -v
```

---

## Soal 18: Shigeru

Prompt: `root@Shigeru`

```
cat > /root/shigeru-fw.sh << 'EOF'
#!/bin/bash
# shigeru-fw.sh - soal 18 (zona sensitif) dan soal 19 (isolasi Armisael). Aman dijalankan berulang.
#   bash /root/shigeru-fw.sh 18   -> hanya soal 18
#   bash /root/shigeru-fw.sh 19   -> soal 18 + 19 (default)
LEVEL="${1:-19}"
# eth0 = sisi menghadap Gendou (luar). eth1 = Makoto/Radar, eth2 = Monolith (dalam zona).
OUT_IF=eth0
TERMINAL="192.237.4.56/29"
ARMISAEL="192.237.4.83"

# pasang iptables kalau belum ada (butuh Gendou online)
if ! command -v iptables >/dev/null; then
  grep -q nameserver /etc/resolv.conf || echo "nameserver 192.168.122.1" > /etc/resolv.conf
  DEBIAN_FRONTEND=noninteractive apt-get -o Acquire::Check-Date=false -o Acquire::ForceIPv4=true update
  DEBIAN_FRONTEND=noninteractive apt-get -o Acquire::Check-Date=false -o Acquire::ForceIPv4=true install -y iptables
fi

# lepas dulu kedua chain dari FORWARD supaya urutannya selalu benar
for c in ZONA ISOLASI; do while iptables -D FORWARD -j $c 2>/dev/null; do :; done; done

# --- soal 18 ---
iptables -N ZONA 2>/dev/null; iptables -F ZONA
iptables -A ZONA -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT                      # balasan koneksi dari dalam
iptables -A ZONA -i $OUT_IF -s $TERMINAL -p icmp --icmp-type echo-request -j ACCEPT        # satu-satunya pengecualian
iptables -A ZONA -i $OUT_IF -j DROP                                                        # semua yang datang dari arah Gendou
iptables -I FORWARD 1 -j ZONA
# traffic dari dalam (eth1/eth2) tidak cocok aturan DROP, jadi bebas ke mana saja

# --- soal 19: isolasi Armisael dari luar Armisael ---
iptables -N ISOLASI 2>/dev/null; iptables -F ISOLASI
if [ "$LEVEL" -ge 19 ]; then
  iptables -A ISOLASI -s $ARMISAEL -j DROP
  iptables -A ISOLASI -d $ARMISAEL -j DROP
  iptables -I FORWARD 1 -j ISOLASI          # di atas ZONA
  # ke/dari Shigeru sendiri (gateway Armisael) lewat INPUT dan OUTPUT, bukan FORWARD
  iptables -C INPUT  -s $ARMISAEL -j DROP 2>/dev/null || iptables -I INPUT  1 -s $ARMISAEL -j DROP
  iptables -C OUTPUT -d $ARMISAEL -j DROP 2>/dev/null || iptables -I OUTPUT 1 -d $ARMISAEL -j DROP
fi
EOF
wc -l /root/shigeru-fw.sh    # harus 38
```
```
bash /root/shigeru-fw.sh 18
iptables -S
```
**Uji:**
```
# di Magi-Casper
ping -c 3 192.237.4.82            # berhasil
# di Lilith
ping -c 3 -W 2 192.237.4.82       # gagal
# di Shigeru
iptables -L ZONA -n -v --line-numbers
```

---

## Soal 19: Armisael, Lilith, Zeruel, Shigeru

**Urutan:** listener dulu (butuh internet), isolasi terakhir.

**1. Armisael dan Lilith** (masing-masing di console sendiri): tulis dan jalankan listener.

```
cat > /root/listener9000.sh << 'EOF'
#!/bin/bash
# listener9000.sh - dipakai di Armisael dan Lilith (soal 19). Salin sebagai /root/armisael.sh dan /root/lilith.sh.
export DEBIAN_FRONTEND=noninteractive
APT="-o Acquire::Check-Date=false -o Acquire::ForceIPv4=true"
command -v nc.traditional >/dev/null || { apt-get $APT update; apt-get $APT install -y netcat-traditional; }
ss -ltn | grep -q ":9000 " && { echo "9000 sudah mendengar"; exit 0; }
setsid nohup bash -c "while true; do nc.traditional -l -p 9000 >> /root/pesan-9000.log; done" >/dev/null 2>&1 &
sleep 1; echo "listener 9000 dinyalakan"
EOF
wc -l /root/listener9000.sh    # harus 8
```
```
bash /root/listener9000.sh
ss -ltn | grep :9000
```
**2. Zeruel** (prompt `root@Zeruel`): script lengkap dengan isolasi.

```
cat > /root/zeruel.sh << 'EOF'
#!/bin/bash
# zeruel.sh - listener TCP 9000 (soal 17, 19) + isolasi Armisael dari sisi Zeruel (soal 19)
# Zeruel dan Armisael satu switch, jadi lalu lintas mereka tidak lewat Shigeru. Satu-satunya cara
# memutusnya tanpa menyentuh Armisael adalah menolak alamat Armisael di Zeruel sendiri.
# Urutan: jalankan SEBELUM isolasi (butuh internet untuk apt) atau pasang netcat lebih dulu.
export DEBIAN_FRONTEND=noninteractive
APT="-o Acquire::Check-Date=false -o Acquire::ForceIPv4=true"
if ! command -v nc.traditional >/dev/null || ! command -v iptables >/dev/null; then
  apt-get $APT update
  apt-get $APT install -y iptables netcat-traditional
fi

ss -ltn | grep -q ":9000 " || { setsid nohup bash -c "while true; do nc.traditional -l -p 9000 >> /root/pesan-9000.log; done" >/dev/null 2>&1 & sleep 1; }

ARMISAEL="192.237.4.83"
iptables -N ISOLASI 2>/dev/null; iptables -F ISOLASI
iptables -A ISOLASI -s $ARMISAEL -j DROP
iptables -A ISOLASI -d $ARMISAEL -j DROP
iptables -C INPUT  -j ISOLASI 2>/dev/null || iptables -I INPUT  1 -j ISOLASI
iptables -C OUTPUT -j ISOLASI 2>/dev/null || iptables -I OUTPUT 1 -j ISOLASI
EOF
wc -l /root/zeruel.sh    # harus 20
```
```
bash /root/zeruel.sh
iptables -S ISOLASI
```
**3. Shigeru** (prompt `root@Shigeru`):
```
bash /root/shigeru-fw.sh 19
iptables -S
```
**Uji dari Armisael (semuanya harus gagal):**
```
ping -c 2 -W 2 8.8.8.8; echo rc=$?
ping -c 2 -W 2 192.237.4.82; echo rc=$?
ping -c 2 -W 2 192.237.4.74; echo rc=$?
time nc -w 3 -z 8.8.8.8 53; echo rc=$?
time nc -w 3 -z 192.237.4.82 9000; echo rc=$?
time nc -w 3 -z 192.237.4.74 9000; echo rc=$?
```
**Uji dari Zeruel (semuanya harus berhasil):**
```
ping -c 2 8.8.8.8
ping -c 2 192.237.4.74
nc -w 3 -z 8.8.8.8 53; echo rc=$?
nc -w 3 -z 192.237.4.74 9000; echo rc=$?
```
Counter: `iptables -L ISOLASI -n -v` di Shigeru dan Zeruel.

---

## Soal 20: Gendou, Fuyutsuki, Shigeru

Tiap node butuh `/root/<nama>.sh` (rute) dan `/root/kebakaran.sh`. File `gendou.sh`, `fuyutsuki.sh`, dan `shigeru.sh` sudah ada dari soal 2-3 (Fuyutsuki versi soal 17). Tulis `kebakaran.sh` di **ketiga node**:

```
cat > /root/kebakaran.sh << 'EOF'
#!/bin/bash
# kebakaran.sh <nama> - soal 20: simpan, hapus, pulihkan 2x, bandingkan jumlah baris.
# Contoh: bash /root/kebakaran.sh gendou   (butuh /root/gendou.sh = script rute node itu)
N="$1"; [ -n "$N" ] || { echo "pakai: $0 <nama>"; exit 1; }
[ -f /root/$N.sh ] || { echo "GAGAL: /root/$N.sh tidak ada"; exit 1; }
cnt() { echo "  iptables -S      : $(iptables -S | wc -l) baris"
        echo "  iptables -t nat -S: $(iptables -t nat -S | wc -l) baris"
        echo "  ip route         : $(ip route | wc -l) baris"; }

echo "== 1. SEBELUM =="; cnt
iptables-save > /root/$N.rules
echo "tersimpan: /root/$N.rules ($(wc -l < /root/$N.rules) baris)"

echo "== 2. HAPUS semua rule (termasuk nat) dan rute buatan sendiri =="
for t in filter nat mangle raw; do
  iptables -t $t -F 2>/dev/null; iptables -t $t -X 2>/dev/null
done
iptables -P INPUT ACCEPT; iptables -P FORWARD ACCEPT; iptables -P OUTPUT ACCEPT
ip route flush proto boot        # rute yang ditambah lewat 'ip route add/replace'. Rute DHCP dan connected tidak ikut
cnt

restore() {
  iptables-restore < /root/$N.rules     # tanpa --noflush: tabel diganti utuh, jadi tidak dobel
  bash /root/$N.sh >/dev/null 2>&1
}
echo "== 3. PULIH ke-1 =="; restore; cnt
S1=$(iptables -S | wc -l); N1=$(iptables -t nat -S | wc -l); R1=$(ip route | wc -l)
echo "== 4. PULIH ke-2 =="; restore; cnt
S2=$(iptables -S | wc -l); N2=$(iptables -t nat -S | wc -l); R2=$(ip route | wc -l)

if [ "$S1" = "$S2" ] && [ "$N1" = "$N2" ] && [ "$R1" = "$R2" ]; then
  echo "HASIL: SAMA (filter $S1, nat $N1, rute $R1). Tidak ada rule/rute dobel."
else
  echo "HASIL: BEDA! filter $S1->$S2, nat $N1->$N2, rute $R1->$R2"
fi
EOF
wc -l /root/kebakaran.sh    # harus 35
```
Lalu jalankan di tiap node, **Gendou lebih dulu**:
```
# Gendou
bash /root/kebakaran.sh gendou
# Fuyutsuki
bash /root/kebakaran.sh fuyutsuki
# Shigeru
bash /root/kebakaran.sh shigeru
```
Di akhir tiap node tertulis `HASIL: SAMA`. Pastikan Shigeru sudah menjalankan `shigeru-fw.sh 19` dulu supaya aturan firewall ikut tersimpan di `shigeru.rules`.

**Bukti berfungsi lagi:**
```
# di Gendou
ping -c 2 8.8.8.8; iptables -t nat -L POSTROUTING -n -v
# di Armisael atau Lilith
ping -c 2 8.8.8.8
# di Magi-Casper (soal 17)
echo cek | nc.traditional -w 3 192.237.4.82 9000
# di Lilith dan Magi-Casper (soal 18)
ping -c 2 -W 2 192.237.4.82
```
Untuk bukti manual: `iptables -S | wc -l` dan `iptables -t nat -S | wc -l` setelah pulih ke-1 dan ke-2 harus sama.
