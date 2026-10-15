# Auto-apply VPP config via systemd (vppcfg + vpptc)

Dua unit systemd yang menerapkan konfigurasi VPP **otomatis setiap kali VPP start**
(reboot, `systemctl restart vpp`, atau VPP crash-restart karena `Restart=always`),
**tanpa mengubah file `vpp.service` maupun `startup.conf`**.

| Unit | Tugas | Sumber kebenaran |
|---|---|---|
| `vpp-vppcfg.service` | Apply konfigurasi deklaratif (interface, bond, loopback, IP) | `/etc/vpp/vppcfg.yaml` |
| `vpp-vpptc.service` | Apply policer dari file startup CLI | `/etc/vpp/vpptc.vpp` (dihasilkan `vpptc`) |

---

## 1. Prinsip desain

- **Deklaratif, bukan imperative.** Yang diedit adalah YAML, lalu `vppcfg` yang
  menghitung bedanya sendiri.
- **Terikat ke lifecycle VPP.** `WantedBy=vpp.service` membuat apply ikut terjadi
  setiap VPP start; `PartOf=vpp.service` membuatnya ikut stop. File `vpp.service`
  tidak disentuh sama sekali.
- **Rantai berjenjang, bukan dua unit terpisah.**
  ```
  vpp.service
    └─ vpp-vppcfg.service          (satu-satunya titik kontrol; disable di sini mematikan semuanya)
         └─ vpp-vpptc.service      (anak: ikut mati otomatis kalau induknya dimatikan)
  ```
  `vpp-vpptc` memakai `After=` — **bukan** `Requires=` — sehingga tidak pernah
  menarik `vpp-vppcfg` untuk hidup. Urutannya tetap dijamin oleh `After=`.
- **Hanya ADD, tidak DELETE.** Unit `vpptc` dijamin jalan pada VPP yang baru
  start, jadi dataplane selalu kosong. Tidak ada policer yang perlu dihapus lebih
  dulu, dan karena itu tidak pernah ada delete yang gagal.
- **Tanpa retry pada unit `vpptc`.** File startup statis, jadi retry tidak
  memperbaiki kegagalan; retry hanya menghasilkan `policer add already exists`
  yang menyesatkan. Retry **dipertahankan** di `vppcfg` karena `vppcfg apply`
  idempoten (diff-based).
- **Tidak ada output yang disembunyikan.** Tidak ada `2>/dev/null` pada langkah
  yang bisa gagal, tidak ada `|| true` yang menelan error, tidak ada prefix `-`
  pada `ExecStartPre`. Kalau gagal, unit jadi `failed` dan alasannya di journal.
- **Tidak menambah file konfigurasi.** Tanpa script, lock, stamp, atau file log.
  Hanya satu file runtime di `/run` (lihat penanda freshness di bawah).

## 2. Kenapa bukan `exec` di `startup.conf`?

`startup.conf` punya `unix { exec <file> }` sehingga file `vpptc.vpp` secara teori
dimuat VPP sendiri tanpa systemd. **Tidak bisa dipakai:** file itu berisi
`policer input name ... BondEthernet0.100`, sedangkan sub-interface tersebut baru
ada **setelah** `vppcfg apply`. Saat fase `exec`, VPP menjawab `unknown input` dan
membatalkan sisa file. Karena itu apply `vppcfg` harus di luar VPP, dan urutan dua
tahap itu dipaksakan oleh systemd.

## 3. Pemasangan

```bash
cp vpp-vppcfg.service vpp-vpptc.service /etc/systemd/system/
systemd-analyze verify /etc/systemd/system/vpp-vppcfg.service /etc/systemd/system/vpp-vpptc.service
systemctl daemon-reload
systemctl enable vpp-vppcfg.service vpp-vpptc.service
systemctl restart vpp
systemctl status vpp-vppcfg.service vpp-vpptc.service
```

`systemctl enable` menghasilkan dua symlink (hierarkis):

```
/etc/systemd/system/vpp.service.wants/vpp-vppcfg.service
/etc/systemd/system/vpp-vppcfg.service.wants/vpp-vpptc.service
```

Path yang perlu disesuaikan di template bila environment berbeda: socket CLI
(`/run/vpp/cli.sock`), socket API (`/run/vpp/api.sock`), path config
(`/etc/vpp/vppcfg.yaml`, `/etc/vpp/vpptc.vpp`), binary `vppcfg`
(`/usr/local/bin/vppcfg`), dan penanda runtime `/run/vpp-vppcfg.applied`.

## 4. KontROL: enable / disable

| Perintah | Akibat |
|---|---|
| `systemctl enable vpp-vppcfg vpp-vpptc` | apply penuh: konfigurasi interface **dan** policer |
| `systemctl disable vpp-vppcfg vpp-vpptc` | jeda total: tidak ada yang di-apply |
| `systemctl disable vpp-vppcfg` | jeda total juga — `vpp-vpptc` ikut mati karena induknya tidak start |
| `systemctl disable vpp-vpptc` | konfigurasi interface saja, tanpa policer |

Perintah yang tersedia per unit:

| Aksi manual | `vpp-vppcfg` | `vpp-vpptc` |
|---|---|---|
| `start` / `stop` / `restart` | ✅ diizinkan (idempoten) | ❌ ditolak (`RefuseManualStart/Stop=yes`) |
| `disable` / `enable` | ✅ | ✅ |

Kenapa `vpp-vpptc` menolak manual: unit ini hanya aman pada VPP yang baru start.
Menjalankan manual berisiko apply ganda, dan `stop` manual **tidak** mencabut
policer dari dataplane sehingga hanya memberi ilusi. Untuk menjeda, pakai
`disable`.

> Catatan: `systemctl is-enabled vpp-vpptc` tetap melaporkan `enabled` walau
> `vpp-vppcfg` sedang disabled, karena symlink-nya masih ada (hanya tidak
> terjangkau). Status visual ini menyesatkan — funcinya unit memang tidak jalan.

## 5. Urutan eksekusi

```
vpp.service  (Type=simple → "started" begitu proses di-fork, CLI belum tentu siap)
 └─ vpp-vppcfg.service   After=vpp.service
      ExecStartPre : poll `vppctl show version` (bounded 60s, dengan diagnosis)
      ExecStart    : vppcfg apply  (validate → dump → plan → apply via API socket)
      ExecStartPost: tulis /run/vpp-vppcfg.applied = MainPID vpp.service
    └─ vpp-vpptc.service  After=vpp.service vpp-vppcfg.service
         ExecStartPre : penanda freshness (harus cocok dengan MainPID VPP sekarang)
         ExecStart    : vppctl exec /etc/vpp/vpptc.vpp   (hanya ADD)
         ExecStartPost: file ada? + jumlah policer file == jumlah policer VPP
```

**Penanda freshness** `/run/vpp-vppcfg.applied` berisi satu angka: `MainPID`
`vpp.service` saat apply **berhasil**. `vpp-vpptc` membandingkannya dengan PID VPP
saat ini. Tujuannya: policer hanya boleh di-apply kalau konfigurasi interface
memang sudah terpasang **pada instance VPP yang sama**. Bedanya dengan
`Requires=` (yang hanya ordering + auto-activate): file ini membedakan
"vppcfg sudah apply di VPP ini" dari "vppcfg tidak ikut jalan".

Marker ini **dihapus di awal** setiap eksekusi `vpp-vppcfg`, lalu ditulis lagi
hanya kalau apply sukses. Tanpa penghapusan itu, apply yang gagal setelah apply
berhasil sebelumnya pada instance VPP yang sama akan meninggalkan marker yang
kebetulan cocok, dan `vpp-vpptc` akan salah menganggap config sudah terpasang.
Isi file hilang otomatis tiap reboot karena berada di `/run`.

**Cegah runtime power management pada kedua fungsi X710** (penting — lihat
troubleshooting). Port yang menganggur di-suspend kernel ke `D3hot`, lalu reset
PCI (FLR) oleh DPDK ditolak `EBUSY` sehingga port tidak ter-probe. Jalankan sekali
setiap boot:
```bash
for d in /sys/bus/pci/devices/0000:01:00.*; do echo on > $d/power/control; done
```
Verifikasi: `cat /sys/bus/pci/devices/0000:01:00.1/power_state` harus `D0`, bukan
`D3hot`. Nilai ini **tidak persisten** lintas reboot, sehingga harus dipasang
ulang tiap boot (misalnya sebagai `ExecStartPre=` di unit ini atau file udev).

## 6. Perilaku pada berbagai kejadian

| Kejadian | Hasil |
|---|---|
| Boot / `systemctl restart vpp` | Kedua unit apply sekali, berurutan |
| VPP crash (`Restart=always`) | VPP naik lagi → apply ulang dari kondisi kosong |
| `systemctl stop vpp` | Kedua unit ikut stop (`PartOf=`) |
| `vpp-vppcfg` disabled | `vpp-vpptc` **tidak** menariknya; keduanya tidak apply |
| `vpp-vppcfg` enabled tapi gagal | Penanda **dihapus** di awal eksekusi lalu tidak ditulis ulang → `vpp-vpptc` ditolak penjaga dengan pesan yang menyebut kondisinya. `vpp-vppcfg` mencoba lagi sampai **5 kali dalam 60 detik** (`StartLimitBurst=5`), lalu dikunci `failed` |
| `vpp-vppcfg` gagal, lalu `systemctl start vpp-vppcfg` berhasil | Penanda ditulis lagi → `systemctl start vpp-vpptc` baru bisa berjalan |
| `vpp-vpptc` gagal | Tidak di-retry. Unit `failed` sampai sumber masalah diperbaiki lalu `systemctl restart vpp`. VPP tetap serving |
| `systemctl restart vpp-vppcfg` manual | Apply ulang idempoten, policer tidak disturbed |

## 7. Troubleshooting

```bash
systemctl status vpp-vppcfg.service vpp-vpptc.service   # tiap ExecStart + exit status
journalctl -u vpp-vppcfg.service -b -l                  # log vppcfg
journalctl -u vpp-vpptc.service -b -l                   # log vppctl + verifikasi
systemctl --failed                                      # unit mana yang gagal
vppctl show interface ; vppctl show policer              # hasil nyata di dataplane
cat /run/vpp-vppcfg.applied ; systemctl show vpp.service -p MainPID --value
```

> `systemctl show -p Result` menampilkan hasil **eksekusi terakhir**, bukan
> "baru saja jalan atau tidak". Karena unit ini `oneshot` tanpa `RemainAfterExit`,
> statusnya selalu `inactive (dead)` setelah sukses — `systemctl is-active`
> **tidak bisa** dipakai sebagai health check. Verifikasi yang benar adalah
> `vppctl show policer` / `vppctl show interface`.

**`GAGAL: /run/vpp-vppcfg.applied belum ada ...`** = apply `vpp-vppcfg` terakhir
tidak sukses pada instance VPP ini (belum pernah jalan, atau kegagalan
sebelumnya). Jalankan `systemctl start vpp-vppcfg`.

**`GAGAL: vpp-vppcfg belum apply pada instance VPP ini (marker PID X != MainPID VPP Y)`**
= VPP sudah restart sejak apply terakhir, jadi konfigurasi interface belum
terpasang pada instance sekarang. Jalankan `systemctl start vpp-vppcfg` atau
`systemctl restart vpp` (pastikan `power/control=on` sudah dipasang).

**`FATAL: /etc/vpp/vpptc.vpp tidak ada atau tidak terbaca`** = file startup hilang.
Tanpa pengecekan ini verifikasi akan membandingkan `0` policer di file dengan `0`
policer di VPP dan salah deems sukses. Sekarang unit **gagal** dengan pesan jelas.

**`FATAL: VPP CLI tidak siap dalam 60 detik`** = socket CLI/API belum siap.
Diagnosis otomatis (`vppctl` tanpa redirect + `ls -l /run/vpp/`) ikut tercetak
di journal sehingga penyebabnya terlihat tanpa perlu menebak.

**`exec: CLI line error:`** tanpa teks lanjutannya = VPP menolak salah satu baris
di file startup dan **membatalkan sisa file**. Penyebab paling sering karakter
`//` di komentar (parser CLI VPP hanya mengenal `#`), atau interface target belum ada.

**`Not all PHYs in the config exist in VPP` (dari `vppcfg`, status 252)** = salah
satu port fisik tidak muncul di `vppctl show hardware-interfaces` saat apply.
Konsekuensinya benar dan disengaja: `vppcfg` menolak apply (konfigurasi tidak
boleh dijalankan setengah), lalu `vpp-vpptc` ikut tertahan oleh penjaga freshness,
sehingga **tidak ada** policer yang attach ke interface yang belum ada. Periksa:
```bash
vppctl show hardware-interfaces | grep -E "sfpplus|address 0000"
dmesg -T | grep -iE "vfio|01:00|AER|pcie"
systemctl status dpdk.service
```
**Akar penyebabnya** (terbukti dari log DPDK di journal, `_COMM=vpp`):

```
dpdk: PCI_BUS: Unable to reset device! Error: 11 (Resource temporarily unavailable)
dpdk: PCI_BUS: 0000:01:00.1 setup device failed
dpdk: PCI_BUS: Requested device 0000:01:00.1 cannot be used
```

X710 2-port memakai **dua PCI function** (`0000:01:00.0` dan `0000:01:00.1`)
di bawah satu bridge. DPDK selalu meminta keduanya
(`-a 0000:01:00.0 -a 0000:01:00.1`), tapi reset PCI (FLR) pada fungsi kedua
ditolak `EBUSY`.

**Penyebab sebenarnya adalah runtime power management kernel, bukan kecepatan
restart.** Bukti pengukuran power state saat insiden terjadi:

| Device | power_state | runtime_status | Hasil probe |
|---|---|---|---|
| `0000:01:00.0` | `D0` | `active` | ter-probe |
| `0000:01:00.1` | **`D3hot`** | **suspended** | FLR gagal |

Kernel (`power/control=auto`) men-suspend port yang idle ke `D3hot`. Saat VPP
start, DPDK menjalankan FLR pada device yang masih suspended, dan kernel
mengembalikan `EBUSY`. Port pertama lolos karena kebetulan masih `active`.

Data 69 restart pada hari pengujian: insiden terjadi pada gap 6,7s hingga 455s —
jadi **tidak berkorelasi dengan kecepatan restart**. Setelah `power/control=on`
di kedua fungsi, 5 restart berturut-turut (jeda 20 detik) semuanya berhasil,
sedangkan sebelumnya jeda yang sama sering gagal.

Konsekuensi: **tidak merusak apa pun** — `vppcfg` menolak apply
(`Not all PHYs in the config exist in VPP`), jadi tidak pernah ada konfigurasi
setengah; `vpp-vpptc` ikut tertahan penjaga. Pemulihan: pasang ulang
`power/control=on`, lalu `systemctl restart vpp`.

## 8. Batasan yang perlu diketahui

- Verifikasi memakai **jumlah** policer, bukan attachment per interface atau nama
  persis. Kalau `policer add` berhasil tapi `policer input` ditolak, jumlah tetap
  cocok; periksa manual dengan `vppctl show interface <if> | grep -i policer`.
  Verifikasi juga terikat format baris `policer add name <X>` di `vpptc.vpp`.
- Kalau `vpptc exec` ditolak di tengah file, VPP bisa menyimpan policer yatim
  (policer yang sudah ter-*add* tapi gagal ter-*attach*). Bersihkan dengan
  `vppctl policer del name <nama>` atau `systemctl restart vpp`.
- Tidak ada rekonsiliasi: kalau ada yang menghapus policer secara manual, atau
  `vpptc.vpp` menjadi basi relatif `vpptc.yaml`, konfigurasi tidak kembali
  benar sampai VPP restart. Tidak ada timer maupun guard kedaluwarsa.
- `vpp-vppcfg` yang di-restart manual bisa membuat VPP mengganti interface, dan
  policer yang ter-attach ikut hilang. Pemulihannya butuh `systemctl restart vpp`,
  karena `vpp-vpptc` menolak start manual.
- FRR masih memakai `ExecStartPre=/bin/sleep 8` di `frr.service.d/override.conf`.
  Sebaiknya diganti `Wants=` + `After=vpp-vppcfg.service` agar FRR benar-benar
  menunggu konfigurasi, bukan menebak.

## 9. Ringkasan file

```
/etc/systemd/system/vpp-vppcfg.service   # apply vppcfg + tulis penanda runtime
/etc/systemd/system/vpp-vpptc.service    # apply & verifikasi policer
/run/vpp-vppcfg.applied                  # runtime: MainPID VPP yang sudah di-apply (auto hilang saat reboot)
/etc/vpp/vppcfg.yaml                     # sumber kebenaran (interface/bond/IP)
/etc/vpp/vpptc.yaml  -> vpptc.vpp        # sumber kebenaran (policer)
```