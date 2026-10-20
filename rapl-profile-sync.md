# CachyOS: Otomatis Mengatur CPU PL1 Berdasarkan Power Profile

Tutorial untuk mengatur batas daya jangka panjang CPU (PL1) secara otomatis berdasarkan profil daya di CachyOS Linux menggunakan `power-profiles-daemon` dan `systemd`.

Konfigurasi ini ditujukan untuk laptop Intel yang menyediakan antarmuka RAPL MSR dan MMIO, misalnya Dell Latitude 7320 dengan Intel Core i7-1185G7.

## 1. Cara Kerja

Script memeriksa profil daya setiap 3 detik dan menyesuaikan batas PL1 pada dua antarmuka:

| Power profile | MSR PL1 | MMIO PL1 |
|---|---:|---:|
| `performance` | 17 W | 17 W |
| `balanced` | 13 W | 13 W |
| `power-saver` | 13 W | 13 W |
| Profil lain atau gagal dibaca | 13 W | 13 W |

Nilai ditulis dalam mikrowatt (`µW`):

- 13 W = `13000000`
- 17 W = `17000000`

**Penting:** nilai di atas merupakan target yang diminta oleh script. BIOS, firmware, kernel, atau pengelola daya lain mungkin mengubah, membatasi, atau menolak pengaturan tersebut. Nilai PL1 bukan konsumsi daya CPU yang sebenarnya.

## 2. Persyaratan

Pastikan sistem menggunakan systemd dan `power-profiles-daemon`.

Periksa profil daya:

```bash
powerprofilesctl get
```

Periksa status daemon:

```bash
systemctl is-active power-profiles-daemon.service
```

Jika perintah pertama menampilkan profil daya dan perintah kedua menampilkan `active`, lanjutkan ke langkah berikutnya.

## 3. Membuat Script Otomatis PL1

Buat script di `/usr/local/sbin/rapl-profile-sync`:

```bash
sudo tee /usr/local/sbin/rapl-profile-sync >/dev/null <<'EOF'
#!/bin/bash
set -u

while true; do
    profile=$(/usr/bin/powerprofilesctl get 2>/dev/null || echo unknown)

    if [[ "$profile" == "performance" ]]; then
        target=17000000
    else
        target=13000000
    fi

    for f in \
        /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw \
        /sys/class/powercap/intel-rapl-mmio:0/constraint_0_power_limit_uw
    do
        current=$(cat "$f" 2>/dev/null || echo unknown)

        if [[ "$current" != "$target" ]]; then
            if ! printf '%s\n' "$target" > "$f"; then
                /usr/bin/logger -t rapl-profile-sync \
                    "Gagal mengatur $f ke $target"
            fi
        fi
    done

    sleep 3
done
EOF

sudo chmod 755 /usr/local/sbin/rapl-profile-sync
```

### Penjelasan script

1. Membaca profil daya menggunakan `powerprofilesctl`.
2. Memilih target 17 W jika profil `performance`; selain itu memilih 13 W.
3. Memeriksa nilai PL1 MSR dan MMIO.
4. Menulis target hanya jika nilai yang terbaca berbeda.
5. Mengulangi pemeriksaan setiap 3 detik.

Jika salah satu file RAPL tidak tersedia atau penulisan gagal, script mencatat pesan ke systemd journal melalui `logger`.

## 4. Membuat Service systemd

Buat file service:

```bash
sudo tee /etc/systemd/system/rapl-profile-sync.service >/dev/null <<'EOF'
[Unit]
Description=Synchronize CPU PL1 with power profile

[Service]
Type=simple
ExecStart=/usr/local/sbin/rapl-profile-sync
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
```

Service ini sengaja tidak menggunakan `After=power-profiles-daemon.service` atau `Wants=power-profiles-daemon.service` karena konfigurasi tersebut sebelumnya menyebabkan *ordering cycle* saat boot pada sistem yang digunakan.

Script akan memeriksa profil daya secara berkala sehingga tidak memerlukan dependensi pengurutan tersebut.

## 5. Mengaktifkan Service

Muat ulang konfigurasi systemd, aktifkan service saat boot, kemudian jalankan:

```bash
sudo systemctl daemon-reload
sudo systemctl enable rapl-profile-sync.service
sudo systemctl restart rapl-profile-sync.service
```

Periksa status:

```bash
systemctl status rapl-profile-sync.service --no-pager
```

Status yang diharapkan:

```text
Active: active (running)
```

Periksa juga konfigurasi service yang benar-benar digunakan:

```bash
systemctl cat rapl-profile-sync.service
```

Pastikan tidak ada konfigurasi tambahan yang kembali menambahkan `After=power-profiles-daemon.service` atau `Wants=power-profiles-daemon.service`.

## 6. Menguji Perubahan PL1

Periksa profil daya dan nilai kedua antarmuka:

```bash
echo "Power profile: $(powerprofilesctl get)"

echo "MSR PL1:"
cat /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw

echo "MMIO PL1:"
cat /sys/class/powercap/intel-rapl-mmio:0/constraint_0_power_limit_uw
```

### Menguji profil performance

```bash
powerprofilesctl set performance
sleep 5

cat /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw
cat /sys/class/powercap/intel-rapl-mmio:0/constraint_0_power_limit_uw
```

Target untuk kedua file adalah:

```text
17000000
17000000
```

### Menguji profil balanced

```bash
powerprofilesctl set balanced
sleep 5

cat /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw
cat /sys/class/powercap/intel-rapl-mmio:0/constraint_0_power_limit_uw
```

Target untuk kedua file adalah:

```text
13000000
13000000
```

Jika ingin menguji profil hemat daya:

```bash
powerprofilesctl set power-saver
sleep 5
```

Periksa kedua file RAPL sekali lagi.

Setelah pengujian, kembalikan profil ke preferensi yang diinginkan, misalnya:

```bash
powerprofilesctl set balanced
```

**Catatan:** mengganti profil daya dapat memengaruhi pengaturan daya lain yang dikelola `power-profiles-daemon`. Script ini tidak mengubah profil daya secara otomatis; script hanya membaca profil yang sedang aktif.

## 7. Melihat Log

Untuk melihat log service:

```bash
journalctl -u rapl-profile-sync.service -b --no-pager
```

Untuk melihat pesan terkait penulisan PL1:

```bash
journalctl -t rapl-profile-sync -b --no-pager
```

Pantau log ketika mengubah profil atau setelah reboot.

## 8. Troubleshooting

### Service berstatus inactive atau failed

Periksa:

```bash
systemctl status rapl-profile-sync.service --no-pager -l
journalctl -u rapl-profile-sync.service -b --no-pager -n 50
```

Pastikan script tersedia dan dapat dieksekusi:

```bash
ls -l /usr/local/sbin/rapl-profile-sync
```

Jika service tidak aktif, periksa pesan error sebelum mencoba restart.

### Terjadi ordering cycle

Pastikan file service tidak memiliki dependensi pengurutan yang menyebabkan siklus:

```bash
systemctl cat rapl-profile-sync.service
```

Gunakan konfigurasi service pada langkah 4, lalu jalankan:

```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed rapl-profile-sync.service
sudo systemctl restart rapl-profile-sync.service
```

### Nilai tetap 13 W saat performance

Periksa profil:

```bash
powerprofilesctl get
```

Periksa service:

```bash
systemctl is-active rapl-profile-sync.service
```

Periksa pesan log:

```bash
journalctl -t rapl-profile-sync -b --no-pager
```

Jika penulisan ditolak atau nilai kembali berubah, kemungkinan ada pembatasan firmware, antarmuka RAPL yang tidak didukung, atau konflik dengan komponen pengelola daya lain.

Jangan berasumsi bahwa kedua batas berhasil diterapkan hanya karena service berstatus `active`. Selalu verifikasi nilai aktual yang terbaca.

## 9. Menghentikan atau Menghapus Konfigurasi

Untuk menghentikan service sementara:

```bash
sudo systemctl stop rapl-profile-sync.service
```

Untuk mencegah service berjalan otomatis saat boot:

```bash
sudo systemctl disable rapl-profile-sync.service
```

Untuk menghapus konfigurasi service dan script:

```bash
sudo systemctl disable --now rapl-profile-sync.service
sudo rm /etc/systemd/system/rapl-profile-sync.service
sudo rm /usr/local/sbin/rapl-profile-sync
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

Menghapus script tidak otomatis mengembalikan batas PL1 ke nilai BIOS awal. Setelah service dihentikan, periksa kembali nilai RAPL dan gunakan mekanisme pengelolaan daya yang sesuai dengan perangkat jika ingin memulihkan pengaturan sebelumnya.

## 10. Catatan Keamanan dan Suhu

- Menaikkan PL1 dapat meningkatkan performa dalam beban CPU tertentu, tetapi hasilnya bergantung pada pendinginan, firmware, dan beban kerja.
- Batas 17 W bukan jaminan CPU akan menggunakan daya sebesar itu.
- Pantau suhu CPU saat bermain game atau menjalankan beban berat.
- Hentikan pengujian jika terjadi panas berlebihan, throttling berat, ketidakstabilan, atau shutdown akibat suhu.
- Jangan menganggap semua laptop Intel memiliki jalur RAPL atau perilaku firmware yang sama.

## Ringkasan

Konfigurasi ini menggunakan script Bash yang berjalan sebagai service systemd untuk menyinkronkan target PL1 MSR dan MMIO dengan profil daya.

- `performance` → target 17 W untuk MSR dan MMIO.
- Profil lainnya → target 13 W untuk MSR dan MMIO.
- Pemeriksaan ulang setiap 3 detik.
- Service otomatis berjalan saat boot setelah diaktifkan.

Hasil akhirnya tetap bergantung pada dukungan perangkat keras, kernel, firmware, dan pengelola daya sistem.
