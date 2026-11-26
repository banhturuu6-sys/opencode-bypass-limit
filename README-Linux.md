# OpenCode TURU v2.4.0 — Linux x64 & arm64

Rilis perbaikan di atas `v2.3.2-turu.1`. Basis kode tetap OpenCode v2.3.2; nomor versi dinaikkan supaya build ini bisa dibedakan dari rilis sebelumnya dan tidak ditawari downgrade oleh `upgrade`.

**Tag:** `v2.4.0-turu.1`  
**Basis:** OpenCode v2.3.2  
**Tanggal:** 2026-11-22

## Download

| Asset | Arsitektur | Ukuran | SHA-256 |
|---|---|---|---|
| `opencode-turu-linux-x64` | x86-64 | 267.872.714 B | `33b588d1d40cdf89bd1bc79da2ee8ae250c6e13ddb981a715dcca412f45b40f1` |
| `opencode-turu-linux-arm64` | AArch64 | 221.236.216 B | `29626186b27fbde71094e42f79e66d23ea6ec36fbb775187a28266947de7456a` |

Mode `0775` (x64) / `0755` (arm64), ELF dinamik. x64 dibangun ulang di atas basis turu2 (`opencode-wp/`, wrapper + payload), jadi ukurannya lebih besar dari rilis patch byte pertama; nomor versi tetap v2.4.0. arm64 tetap rilis patch byte sama-panjang.

## Perbaikan utama: crash `xi.c.exitCode`

Gejala: `undefined is not an object (evaluating 'xi.c.exitCode')` saat rotasi tunnel dijalankan sebelum ada tunnel cadangan.

Penyebab: nama variabel minify `xi` dideklarasikan dua kali di top-level bundle x64.

| Item | Lokasi (x64) | Isi |
|---|---|---|
| Deklarasi blok rotasi | `176804395` | `var ei=null,Ns=0,xi=null` |
| Deklarasi yang menimpa | `177483306` | `var hi=26214400,xi=15728640` |
| Guard yang crash | `176804434` | `if(!xi\|\|xi.c.exitCode!==null)return 0;` |

Deklarasi kedua dieksekusi belakangan saat modul dimuat, sehingga `xi` berisi angka `15728640` (truthy, tanpa `.c`). Guard lolos dan `xi.c.exitCode` melempar error.

Patch (97 byte, offset `176804434`–`176804530`):

```js
// sebelum
if(!xi||xi.c.exitCode!==null)return 0; ... if(e)try{e.kill()}catch{};
// sesudah
if(!xi?.c||xi.c.exitCode!==null)return 0; ... try{e.kill()}catch{};··
```

Semantik: `Xk()` mengembalikan `0` bila tidak ada cadangan yang bisa dipromosikan — jawaban yang benar untuk kondisi itu. Bila cadangan valid (`{c, p}`), promosi tetap berjalan seperti biasa.

### arm64 tidak perlu patch

Nama minify di arm64 berbeda dan tidak bentrok: blok rotasi memakai `ts`/`qs`/`ks`, sedangkan blok limit gambar memakai `Cd`/`Bd`. Tidak ada deklarasi `ts` setelah blok rotasi, jadi guard `if(!ts||ts.c.exitCode...)` tidak pernah melihat nilai yang salah. File arm64 hanya menerima bump versi.

## Perubahan lain

- **Versi**: `2.3.2` → `2.4.0` pada konstanta aplikasi (`var On="2.4.0"`, offset `167904356`) dan User-Agent (`opencode/hop/2.4.0/cli`, offset `226145739`). Channel internal `el="hop"` **tidak** diubah agar entitlement model di sisi server tetap berlaku.
- **Permission**: `warp-pool.json`, `warp-burned.json`, `warp.log` di `~/.local/share/opencode/turu-warp/` diubah dari `0664` ke `0600`. File-file itu memuat `privateKey` WARP dalam bentuk plaintext.
- **Plugin `turu-stats`**: `src/tui.tsx` 664 → 516 baris, `src/index.ts` 11 → 7 baris, tanpa perubahan angka footer/`/turu`, `readBypass()`, atau `collectTokens()`.

## Verifikasi

```
./opencode-turu-linux-x64 --version        -> opencode v2.4.0 (rc 0)
./opencode-turu-linux-x64 service status   -> http://127.0.0.1:20672 (rc 0)
./opencode-turu-linux-x64 plugin list      -> 3 plugin, turu.stats local (rc 0)
./opencode-turu-linux-x64 stats            -> rc 0
./opencode-turu-linux-x64 models           -> rc 0, daftar model lengkap
./opencode-turu-linux-x64 debug paths      -> rc 0
```

Uji logika guard: `xi=null` → `0`, `xi=15728640` → `0`, `xi={}` → `0`, `xi={c:{exitCode:null}}` → promosi, `xi={c:{exitCode:0}}` → `0`.

## Masalah yang masih terbuka

1. **Akar tabrakan nama belum hilang.** Patch memperbaiki gejalanya; nama `xi` tetap dideklarasikan dua kali. Build ulang dari source fork yang sama akan memunculkan bug lagi. Perbaikan akar butuh rename variabel di source + rebuild.
2. **Tidak ada toolchain build.** Binary 226 MB ini tidak dapat dibangun ulang di mesin ini; semua perbaikan hanya bisa berupa patch byte.
3. **Reliabilitas tunnel.** `warp.log` mencatat `tunnel did not come up`, DNS timeout ke `1.1.1.1`, dan `connection reset by peer` pada beberapa rotasi. Belum dipastikan apakah penyebabnya kode atau jaringan.
4. **arm64 belum diuji runtime.** Tidak ada `qemu-aarch64` di mesin ini; arm64 hanya diverifikasi secara struktur byte.
5. **Fork tertinggal upstream.** Basis v2.3.2; upstream sudah v2.3.3. Fitur TURU (pool 4 device, warm backup) tidak ada di upstream, jadi tidak bisa sekadar diganti.

## Rollback

| File | Backup |
|---|---|
| `opencode-turu-linux-x64` | `opencode-turu-linux-x64.pre-2.4.0` |
| `opencode-turu-linux-arm64` | `opencode-turu-linux-arm64.pre-2.4.0` |
| `~/.local/bin/opencode-turu` | `~/.local/bin/opencode-turu.pre-exitcode-fix` (v2.3.2, belum ada patch) |

Proses TUI/service yang sedang berjalan masih memakai binary lama di memori. Versi baru berlaku saat `opencode-turu` dijalankan ulang.
