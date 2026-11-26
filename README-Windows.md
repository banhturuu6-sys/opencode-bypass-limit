# opencode-wp / opencode-turu2

Clone opencode v2.3.2 + tunnel WARP (`windtf/wireproxy`) yang hanya dinyalakan
saat pesan limit muncul, lalu sesi dilanjutkan otomatis **di dalam TUI yang sama**.

Binary `opencode` di folder ini adalah salinan `../test/opencode` yang di-patch
sama panjang byte (lihat `Banner limit` di Catatan); file asli
`../test/opencode` tidak disentuh.

Dua bentuk distribusi:

- `opencode-wp/` — launcher + `bin/wireproxy` + `wp-plugin/` (untuk diedit).
- `../opencode-turu2` — satu file (stub ELF + payload **tar tanpa kompresi**,
  jadi ukurannya ≈ binary asli + wireproxy + launcher). Saat dijalankan ia
  mengekstrak diri ke `~/.cache/opencode-turu2/<ukuran-mtime>` lalu menjalankan
  launcher dari situ. Build ulang (walau ukurannya sama) membuat cache baru, jadi
  payload lama tidak pernah terpakai.
- `../opencode-turu2-pretest` — build uji state machine dari sumber yang sama:
  `PRETEST=1 ./build-turu2.sh` (tidak menyentuh artifact rilis). Transisinya
  (`START` → `IDLE` → `USAGE_LIMIT` → `RECOVERING` → `HEALTH_CHECK` →
  `RUNNING`/`FAILED`, `WAIT_RETRY_AFTER` untuk 429) ditulis sebagai
  `state=...` di `warp.log`, dan `OPENCODE_WARP_MAX` habis menghasilkan
  `STOPPED` lalu balik ke `START` tanpa mematikan sesi. Lihat dengan
  `tail -f ~/.local/share/opencode/warp/warp.log | grep state=`.

## Pakai

```sh
./opencode-turu2                                   # TUI
./opencode-turu2 --model <provider/model> "..."  # prompt awal (prefill); model pilihanmu
./opencode-turu2 run "tugas kamu"                  # mode scripted (tanpa TUI)
./opencode-turu2 run --continue ...                # lanjut sesi manual
./opencode-turu2 serve --port 4097                 # server API (password di cetak)
./opencode-turu2 service start                     # server latar + retry WARP
./opencode-turu2 ip                                # cek IP langsung + IP tunnel aktif
./opencode-turu2 warp show                         # device WARP milikmu (IP/kunci/peer)
./opencode-turu2 warp new                          # buang identitas lama, daftar device baru
./opencode-turu2 warp gen [warp.conf]              # conf WireGuard standar untuk klien WG mana pun
```

`ip` (alias `checkip`) mencetak IP keluar tanpa tunnel dan IP yang sedang
dipakai tunnel WARP. Tunnel yang sedang jalan diperiksa ulang lewat proxy-nya,
jadi angkanya IP saat itu, bukan catatan lama; kalau tunnel sudah mati
disebutkan URL terakhirnya. Exit code 1 kalau IP langsung pun tidak terbaca.

`warp` mengelola identitas WARP kamu sendiri: `warp new` menghapus device,
pool, dan daftar jaringan burned, lalu mendaftarkan device baru **langsung ke
Cloudflare** (`api.cloudflareclient.com`) dengan keypair yang dibuat di mesin
ini — tidak ada server/perangkat milik pihak lain yang dipakai. `warp show`
mencetak id, IPv4/IPv6, peer, jumlah device di pool, dan lokasi state.
`warp gen` mencetak conf WireGuard biasa (tanpa bagian `[http]` wireproxy) dari
device yang ada — atau mendaftarkannya dulu kalau belum ada — sehingga bisa
diimpor ke klien WireGuard mana pun, sama seperti generator WARP di browser;
beri nama file (`warp gen ~/warp.conf`, mode 0600) untuk menyimpannya. Tidak ada
QR; pakai `qrencode -t ansiutf8 < warp.conf`.
Rotasi mencoba 10 kali per putaran. Setelah 5 percobaan gagal, launcher membuang
identitas lama dan mendaftarkan device WARP baru sendiri langsung ke Cloudflare
(cooldown 429 diabaikan pada percobaan pertama), lalu mencoba tunnel dari device
itu — bukan mengulang device pool. Sisa percobaan rotasi memakai device baru itu
juga. Kalau 10 percobaan habis dan masih tidak ada tunnel, launcher **mendaftarkan
kunci WARP baru lagi** dan melanjutkan percobaan (bukan menyerah) sampai ada
tunnel yang bisa dipakai; `OPENCODE_WARP_ROUNDS` bisa membatasi jumlah putaran
tambahan. Kalau tetap gagal, backup lama yang dipromosikan. Registrasi baru dipakai pada rotasi berikutnya (TUI yang sedang
jalan tetap memakai device di memorinya sampai limit berikutnya).

Tunnel dicoba **burst**: `OPENCODE_WARP_BURST` (default 3) tunnel dinyalakan
bersamaan dengan endpoint acak yang berbeda, dan tunnel pertama yang terbukti
membawa trafik langsung dipakai; yang gagal atau mendarat di jaringan yang sama
langsung dibuang tanpa jeda menunggu probe tunnel berikutnya. Jadi satu rotasi
gagal tidak menahan rotasi berikutnya, dan device yang tunnelnya gagal tidak
diambil lagi dari pool.

Subcommand opencode (`run`, `serve`, `service`, `models`, `auth`, `debug`, …)
dilewatkan apa adanya — tanpa `--standalone` yang dulu membuat `service status`
malah mencetak halaman bantuan. Hanya TUI/prompt yang mendapat `--standalone`.

### Build di Windows (menghasilkan .exe)

Di Windows, jalankan dari folder `opencode-wp\`:

```powershell
.\build-windows.ps1
```

Script ini:
1. Memzip seluruh folder (kecuali build artifacts) menjadi `payload.zip`.
2. Meng-compile `selfextract.cs` dengan `csc.exe`, menyematkan `payload.zip`
   sebagai resource, menghasilkan `..\opencode-turu2.exe`.
3. Saat dijalankan, stub mengekstrak payload ke
   `%LOCALAPPDATA%\opencode-turu2\<key>` (cache, tidak diekstrak ulang kalau
   sama) lalu menjalankan `opencode-wp` via Node.js, meneruskan semua argumen.

Prasyarat: Node.js, `opencode.exe`, `bin\wireproxy.exe`, dan .NET Framework
(csc.exe). Fitur sama dengan Linux (rotasi WARP, service latar, health check).

### Windows — Step by Step

1. **Install Node.js** (>= 18) dari https://nodejs.org/ dan pastikan `node` ada di PATH.
2. **Dapatkan binary opencode Windows** (`opencode.exe`) dari rilis resmi opencode,
   lalu letakkan di folder `opencode-wp\` (satu level dengan `opencode-wp` launcher).
   - Tanpa file ini, wrapper tidak bisa berjalan (tidak ada yang dibungkus).
   - Untuk rotasi otomatis aktif, `opencode.exe` harus sudah di-patch (banner free-limit
     diblank + dialog limit jadi no-op). Binary asli yang belum di-patch tetap bisa
     dipakai untuk `run`/`serve`/`service`, tapi dialog limit akan muncul.
3. **Dapatkan binary wireproxy Windows** (`wireproxy.exe`) dari `windtf/wireproxy`,
   letakkan di `opencode-wp\bin\wireproxy.exe`.
4. **Jalankan wrapper** dari PowerShell:
   ```powershell
   .\opencode-turu2.ps1
   ```
   atau pakai `.exe` yang sudah disediakan (jika ada):
   ```powershell
   .\opencode-turu2.exe
   ```
5. **(Opsional) Build self-contained .exe** di Windows:
   ```powershell
   cd opencode-wp
   .\build-windows.ps1
   ```
   Hasilnya `..\opencode-turu2.exe` (self-extracting, mengekstrak payload ke
   `%LOCALAPPDATA%\opencode-turu2\<key>` lalu menjalankan launcher).

Catatan: seluruh fitur (rotasi WARP, service latar, health check) berjalan di
Windows selama `opencode.exe` + `wireproxy.exe` tersedia dan (untuk rotasi
otomatis) binary opencode sudah di-patch.

### Windows (PowerShell)

Launcher `opencode-wp` adalah skrip Node.js, jadi bisa jalan di Windows selama
Node.js terpasang. Yang dibutuhkan:

- Node.js >= 18 di PATH.
- `opencode.exe` (binary opencode Windows) di folder `opencode-wp/`.
- `bin\wireproxy.exe` (binary wireproxy Windows) di folder `opencode-wp/`.

Unduh binary Windows dari rilis resmi opencode dan `windtf/wireproxy`, lalu
letakkan sesuai struktur. Jalankan dari PowerShell:

```powershell
.\opencode-turu2.ps1                                   # TUI
.\opencode-turu2.ps1 --model <provider/model> "..."    # prompt awal
.\opencode-turu2.ps1 run "tugas kamu"                  # mode scripted
.\opencode-turu2.ps1 service start                     # server latar + retry WARP
.\opencode-turu2.ps1 ip                               # cek IP
```

Catatan Windows:
- Distribusi single-file (stub ELF) **tidak** berlaku di Windows; gunakan folder
  `opencode-wp/` (zip) sebagai paket portable.
- `console` (pembuatan akun Console) butuh `bash`/`console-run.sh` (Linux/macOS);
  di Windows jalankan `console-gen.py` manual via Python.
- `serviceHasPlugin` tidak bisa cek `/proc` di Windows; asumsi plugin aktif.
- `taskkill` dipakai untuk menghentikan service.

### Service latar

`service start` / `restart` dijalankan oleh launcher, bukan `serve --service`:
binary build ini menulis registrasi service lalu salah menganggap registrasinya
diganti dan langsung keluar, sehingga `service start`/`restart` gagal dengan
`Server process exited with code 130` dan retry WARP tidak pernah aktif.
Launcher menyalakan `serve` biasa (host/port/password/cors/env dibaca dari
konfigurasi `service set ...`), menunggu `/api/info` menjawab, lalu menulis
`service.json` sendiri dengan bentuk yang sama seperti upstream. Sesudah itu
proses kecil `__service-watch` menjalankan loop retry yang sama dengan TUI
(owner `service`). Plugin di dalam service membaca `retry-service.json` saat
prompt pertama, jadi service yang sudah jalan ikut rotasi WARP tanpa restart.
`service stop` mematikan watcher dan service; `status`/`get`/`set`/`unset`
tetap milik opencode asli.

Saat start tidak ada tunnel. Rotasi dipicu oleh dua hal di log opencode: flag
limit (`FreeUsageLimitError`, `QuotaExceeded`, `RateLimitError`,
`provider.free-limit`, `Free usage limit`, dll) dan kegagalan jaringan yang
memunculkan banner `Retry due` (`Connection lost while reading the response`,
`ECONNRESET`/`ECONNREFUSED`/`ETIMEDOUT`/`EPIPE` pada baris `level=ERROR`).
Tidak ada penantian reset kuota atau backoff: baris pemicu terbaca dalam ~0,4 s,
WARP langsung diputar, dan turn yang terpotong dilanjutkan. Reconnect SSE biasa
(`level=INFO ... event stream disconnected`) bukan turn gagal, jadi tidak
memutar WARP. Loop retry tidak punya kondisi berhenti selama
`OPENCODE_WARP_MAX=0` (default).

1. wireproxy dinyalakan dan WARP didaftarkan ulang (identitas baru), jadi IP
   keluar berganti. `OPENCODE_WARP_HOSTS` (default `opencode.ai`) dibelokkan
   lewat tunnel; host lain tetap langsung.
2. Sesi diulang **tanpa menutup TUI**:
   - mode `run`: launcher mematok id sesi (`--session ses_…`, dibuat otomatis
     oleh opencode) lalu mengulang `run --session <id yang sama>` dengan env
     proxy. Jadi yang dilanjutkan adalah sesi yang kena limit, bukan "sesi
     terakhir" di direktori itu. Kalau kamu sendiri memberi `--session` atau
     `--continue`, flag-mu yang dipakai.
   - mode TUI: launcher menulis `retry-<pid>.json` (`{at, owner, url,
     sessionID?}`); plugin server (`wp-plugin/`) di dalam TUI memasang
     `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY` ke `process.env` lalu mengirim ulang
     prompt ke sesi yang sama. TUI tetap hidup, tidak ada restart. Kalau id sesi
     tidak ada di baris log (mis. `QuotaExceeded`) atau id itu tidak dikenal
     server, plugin memakai sesi yang terakhir aktif.
3. Setiap pesan limit langsung **rotasi** (IP keluar baru) lalu turn yang
   terpotong dilanjutkan, dengan pipeline yang sama seperti
   build sebelumnya:
   - registrasi device WARP dicoba ke `v0a2158/reg` dan `v0i1909051800/reg`
     dengan dua versi client; device lama disimpan di `warp-pool.json`
     (maks 4, dedupe `publicKey`) dan dipakai ulang kalau jaringannya belum
     dibakar;
   - tiap tunnel memakai **port lokal acak** (`BindAddress = 127.0.0.1:<port>`).
     Setiap port yang sudah dipakai dicatat sebagai
     `$TMPDIR/opencode-turu2-<uid>-ports/warp-proxy-<port>.conf` dan **tidak
     dipilih lagi selama 24 jam**; setelah
     24 jam catatannya kedaluwarsa dan dibersihkan, port itu boleh dipakai
     lagi. Port yang ditolak wireproxy **atau** hidup tapi tidak membawa trafik
     dihitung sebagai satu percobaan port, lalu port diputar lagi untuk sesi
     yang sama — sampai `OPENCODE_WARP_PORTS` (default 10) port gagal
     berturut-turut. Setelah itu kunci WARP baru didaftarkan dan port diputar
     lagi untuk sesi yang sama; siklus ini berulang tanpa batas sampai request
     limit (429) berubah menjadi 200;
   - satu device WARP dipakai bersama seluruh burst (`OPENCODE_WARP_BURST`,
     default 3) dan percobaan port dalam satu siklus rotasi. Kunci WARP baru
     didaftarkan hanya kalau jaringan device itu sudah burned (IP-nya yang tadi
     kena limit) atau semua port untuk device itu gagal — bukan tiap 5
     percobaan, dan bukan satu registrasi per job burst. Dengan begitu satu
     siklus 429 normalnya hanya butuh satu registrasi;
   - ada `OPENCODE_WARP_ROUNDS` (default 0 = tanpa batas) untuk terus mencoba
     sampai tunnel bisa dipakai; tiap putaran memakai device aktif yang sama
     sampai dinyatakan burned/habis;
   - IP keluarnya diprobe berulang selama 15 s (mulai 5 s, tiap 1,5 s),
     **IPv6 lebih dulu** (`ipv6.icanhazip.com`) karena kuota mengikuti alamat
     IPv6; host IPv4 hanya cadangan. Identitas jaringan IPv6 dibandingkan
     sebagai alamat penuh (/128), bukan /64, supaya device baru dengan alamat
     berbeda tidak dianggap jaringan yang sama;
     balasan `429` dari host probe dibaca `Retry-After`-nya, request dijeda
     lalu diulang (maks `OPENCODE_WARP_COOLDOWN`, default 3 s) — rate limit di
     host probe bukan tunnel mati, dan semua jeda retry memakai cooldown yang
     sama supaya retry ke-10 tidak menunggu lebih lama dari retry pertama;
     rotasi ditolak kalau IP sama dengan sebelumnya atau jaringannya ada di
     `warp-burned.json` (permanen: jaringan yang pernah kena limit tidak
     dipakai lagi dan catatannya tidak dihapus, termasuk oleh `warp new`);
   - tunnel lama tidak langsung dimatikan: satu tunnel hangat disimpan sebagai
     `backup` dan dipromosikan kalau semua percobaan gagal, jadi tidak pernah
     ada momen tanpa egress. IPv6 dibandingkan sebagai alamat penuh (/128).
   **Model tidak pernah diganti oleh sistem**: model tetap milik yang kamu
   pilih; kalau kamu ingin model lain, ganti sendiri lewat `/models`.
   Catatan: kuota free bisa mengikuti **akun**, bukan IP. Bila limit tetap
   muncul, launcher terus merotasi dan melanjutkan; pilih model lain lewat
   `/models` bila kuota model itu memang habis.
3b. Rotasi **tidak menggambar apa pun** di layar: tidak ada baris status di
   atas kotak ketik, tidak ada toast, tidak ada dialog `rotation network`, dan
   tidak ada pesan baru yang ditulis ke riwayat chat. Slot `session.composer.top`
   sudah dilepas dari `wp-plugin/tui.tsx` (plugin tetap didaftarkan lewat
   `OPENCODE_CLI_CONFIG_CONTENT`, jadi `~/.config/opencode/cli.json` milikmu
   tidak diubah), sehingga kotak ketik tetap kosong selama rotasi. Detail proses
   rotasi (registrasi, tunnel, probe, IP) tetap lengkap di `warp.log`, dan
   ringkasan fase terakhir tetap ditulis ke `status-<owner>.json` di
   `OPENCODE_WARP_HOME` (dipicu sekali oleh `rotate()`, jadi pemicu bersamaan
   memakai promise yang sama).

3c. **Satu sesi, satu tunnel + IP sendiri.** Sesi yang dibuat setelah plugin
   hidup memicu satu rotasi WARP khusus sesi itu: plugin menulis
   `fresh-<owner>.json` (`{at, owner, sessionID}`), launcher memutar WARP lewat
   jalur yang sama dengan rotasi limit, lalu membalas `retry-<owner>.json`
   dengan `silent: true` sehingga plugin hanya memasang proxy baru (tidak ada
   pesan lanjutan yang dikirim). Subagent (`parentID`) tidak memicu rotasi
   karena ikut tunnel sesi induk. Sesi yang sudah ada saat plugin hidup tidak
   diputar (tidak ada rotasi untuk sesi yang sedang berjalan). Batasannya:
   binary 2.4.0 hanya punya satu env proxy untuk request AI —
   `/api/session/:id/environment` hanya mengganti env untuk shell command —
   jadi tunnel dipakai bergantian: sesi yang aktif memakai tunnel miliknya,
   dan membuka sesi lama memakai tunnel terakhir yang dipasang. Rotasi mulai
   begitu sesi baru terdeteksi; kalau pesan pertama terlanjur berjalan saat
   WARP masih diputar, turn itu masih lewat tunnel sebelumnya dan tunnel baru
   dipakai sejak request berikutnya.

   Lanjutan turn dikirim plugin server sebagai pesan `synthetic` **tanpa
   `description`**; TUI 2.4.0 melewati pesan jenis itu (`if (type === "synthetic"
   && !description?.trim()) return`), jadi perintah yang sudah kamu kirim tidak
   pernah muncul kembali di kolom chat. Launcher tidak pernah mengirim ulang
   prompt user: `ctx.session.prompt` tidak dipakai sama sekali. Setelah IP yang
   tidak kena limit didapat, turn yang terpotong dilanjutkan dan sesi berjalan
   dengan IP itu sampai limit berikutnya memicu rotasi lagi. Pesan
   limit/model-switch bawaan opencode juga tidak dirender (gate `lye` di binary
   dimatikan), jadi saat kena limit layar tetap bersih.
4. Watchdog: setelah retry, kalau server tidak menulis satu baris log pun
   selama `OPENCODE_WARP_STALL` (default 90 s), retry dipaksa sekali lagi
   dengan rotasi baru (model tidak disentuh). Baris log apa pun (termasuk balasan normal) membatalkan
   watchdog, jadi TUI yang diam setelah retry sukses tidak di-"continue".
   Ini mencegah request yang menggantung membiarkan TUI menunggu selamanya.
5. Kalau helper mati di tengah sesi, launcher menulis `{url:null, silent:true}`
   → env proxy dibersihkan tanpa memicu turn baru (tidak menunjuk port mati).
   Turn yang sedang jalan gagal `ECONNRESET` lalu ditangani rotasi limit. Saat
   TUI ditutup, `{url:null, silent:true}` ditulis → proxy dibersihkan tanpa prompt.
6. Berhenti hanya kalau `OPENCODE_WARP_MAX` diisi dan habis (hanya mode `run`),
   atau Ctrl-C. Mode TUI tidak pernah berhenti sendiri: rotasi berulang selama
   pesan limit masih muncul.

Kode keluar: `0` sukses, `1` error launcher, `2` batas `OPENCODE_WARP_MAX`
tercapai, `130` dihentikan (SIGINT/SIGTERM).

## Env

| Variabel | Default | Arti |
|---|---|---|
| `OPENCODE_WARP_MAX` | `0` | batas retry mode `run`; `0` = tanpa batas (TUI selalu lanjut) |
| `OPENCODE_WARP_STALL` | `90000` | ms tanpa baris log setelah retry → paksa rotasi + retry lagi (min 15000) |
| `OPENCODE_WARP_BURST` | `3` | tunnel yang dinyalakan bersamaan tiap putaran rotasi; yang pertama membawa trafik dipakai |
| `OPENCODE_WARP_ACCOUNT_LIMIT` | `3` | rotasi berturut dengan IP berbeda yang tetap 429 → dianggap kuota akun habis (dicatat, tidak berhenti) |
| `OPENCODE_WARP_NEW_ACCOUNT_AFTER` | `300` | jumlah percobaan tunnel dengan key pair yang sama sebelum akun WARP baru didaftarkan |
| `OPENCODE_WARP_PORTS` | `10` | port lokal yang diputar per tunnel sebelum kunci WARP baru didaftarkan; setelah kunci baru, port diputar lagi untuk sesi yang sama |
| `OPENCODE_WARP_COOLDOWN` | `3000` | ms jeda semua retry (registrasi 429 dan `Retry-After` probe); tidak bertambah di retry berikutnya, `0` = tanpa jeda |
| `OPENCODE_WARP_REG_ENDPOINTS` | `https://api.cloudflareclient.com/v0a2158/reg,https://api.cloudflareclient.com/v0i1909051800/reg` | endpoint registrasi WARP (comma-separated); coba berurutan; override tanpa rebuild |
| `OPENCODE_WARP_ROUNDS` | `0` | putaran tambahan (masing-masing 10 percobaan + kunci WARP baru) setelah putaran pertama gagal; `0` = terus mencoba sampai ada tunnel |
| `OPENCODE_WARP_HOSTS` | `opencode.ai` | domain yang dilewatkan tunnel (pisah koma) |
| `OPENCODE_WARP_CONTINUE` | `continue` | pesan yang dikirim saat melanjutkan |
| `OPENCODE_WARP_HOME` | `~/.local/share/opencode/warp` | state: device, `warp.log`, `warp-burned.json`, `retry-<owner>.json`, `status-<owner>.json`. Catatan port tidak disimpan di sini, melainkan di `$TMPDIR/opencode-turu2-<uid>-ports/` (mode 0700) sebagai `warp-proxy-<port>.conf`: port tidak dipakai lagi selama 24 jam, dan folder itu otomatis hilang saat reboot. Catatan lama di `OPENCODE_WARP_HOME` / `used-ports/` dipindahkan ke sana saat start. Folder lama `turu-warp` disalin sekali kalau file di atas belum ada |
| `OPENCODE_WARP_PROXY` | `<dir>/bin/wireproxy` | path binary wireproxy |
| `OPENCODE_WARP_LOG_DIR` | `~/.local/share/opencode/log` | lokasi `opencode.log` |
| `OPENCODE_WARP_DEBUG` | – | `1` = tulis jejak deteksi ke `warp.log` |
| `WP_OFF` | – | `1` = jalankan opencode langsung tanpa wrapper |

## Catatan

- Retry WARP (rotasi + lanjutan turn) berjalan di mode TUI dan juga di mode
  `service` (`serve`): plugin dimuat dari cache payload (`wp-plugin`) yang
  terdaftar di konfigurasi service, sehingga watcher memutar WARP dan plugin
  menerapkannya. Jika service lama (tanpa env plugin) terdeteksi, jalankan
  `service restart` agar retry aktif.

- Banner limit: `You've used today's free messages.` → `rotation network`, dan
  baris reset/upsell OpenCode Go dikosongkan (spasi), semuanya lewat substitusi
  byte **sama panjang** di binary (`./opencode` di folder ini sudah ter-patch;
  `build-turu2.sh` hanya membungkusnya ulang, patch tidak diulang; offset &
  ukuran payload tidak berubah). Yang dipatch: 4 salinan
  banner, 2 template `` ` Resets at ${r.reset}.` ``, 1 template
  `` ` They reset at ${f()}.` ``, paragraf `OpenCode Go gives you reliable …`
  (2 salinan JS + 1 tabel string), serta label `Upgrade to OpenCode Go →` dan
  `· Switch model →` (UTF-16 + salinan JS). Label tautan `F("/connect")` dan
  `F("/models")` di baris banner juga dikosongkan (hanya pemanggilan pertama;
  `F("/connect")` kedua milik dialog "Connect provider" dibiarkan). Baris
  reset tidak dirender karena template JS-nya sudah jadi string kosong.
  Dialog free-limit `Fa` ("Free limit reached" + `rotation network`) juga
  dimatikan: pemanggilan `Ht.replace(()=>s(Fa,…))` diubah jadi ekspresi no-op,
  jadi tidak ada kartu yang menumpuk di transkrip saat rotasi berulang.
  Renderer notice limit (`lye`) juga di-gate `return !1;`, jadi
  `Free usage limit reached…`, `You've reached the spend limit for this model.
  Switch models…`, notice Go-limit dan budget-limit tidak muncul sama sekali —
  tanpa baris `Error:` kosong, karena pesan error per-turn (`md`) juga dimatikan.
  Model tetap milik pengguna; wrapper tidak pernah mengganti model.
- Agen `hacking` (doktrin pentest/offensive-security) ikut dibundel di
  `wp-agent.json` dan didaftarkan lewat `OPENCODE_CONFIG_CONTENT.agent`; tidak
  menulis ke `~/.config/opencode`, dan config pengguna tetap menang. Pakai:
  `./opencode-turu2 --agent hacking` atau lewat tool subagent.
- Deteksi memakai log bersama: sesi opencode lain yang kena limit saat wrapper
  berjalan bisa memicu retry (tunnel menyala tanpa perlu = tidak berbahaya).
- Berkas retry/status dipisah per launcher (`<owner>` = pid, atau `service`),
  jadi dua TUI di HOME yang sama tidak saling menimpa; berkas sesi lama
  dibiarkan (kecil), pemilih IP memakai yang terbaru.
- `serve` tidak mendapat `--standalone` (ditolak server); saat dijalankan di
  depan, launcher yang mengurus retry-nya. TUI mendapat `--standalone` kecuali
  kamu memberi `--server` sendiri.
- `--prompt` hanya mengisi kotak input; Enter tetap harus ditekan.
- Tunnel hanya untuk `OPENCODE_WARP_HOSTS`; kalau tunnel mati, host lain jalan.

## Copyright

Launcher, stub, build script, plugin, dan dokumen di folder ini: © 2026 ALDO.
`opencode` (OpenCode v2.3.2) dan `bin/wireproxy` tetap milik proyek aslinya
masing-masing; build ini hanya membungkus dan menambal keduanya.
