---
read_when:
    - Mengonfigurasi persetujuan exec atau allowlist
    - Mengimplementasikan UX persetujuan exec di aplikasi macOS
    - Meninjau prompt keluar dari sandbox dan implikasinya
summary: Persetujuan exec, allowlist, dan prompt keluar dari sandbox
title: Persetujuan Exec
x-i18n:
    generated_at: "2026-04-06T03:12:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: 39e91cd5c7615bdb9a6b201a85bde7514327910f6f12da5a4b0532bceb229c22
    source_path: tools/exec-approvals.md
    workflow: 15
---

# Persetujuan exec

Persetujuan exec adalah **guardrail aplikasi pendamping / host node** untuk memungkinkan agen yang berada di sandbox menjalankan
perintah pada host nyata (`gateway` atau `node`). Anggap ini seperti interlock keselamatan:
perintah hanya diizinkan ketika kebijakan + allowlist + (opsional) persetujuan pengguna semuanya setuju.
Persetujuan exec berlaku **sebagai tambahan** terhadap kebijakan alat dan elevated gating (kecuali elevated disetel ke `full`, yang melewati persetujuan).
Kebijakan efektif adalah yang **lebih ketat** dari default `tools.exec.*` dan persetujuan; jika field persetujuan dihilangkan, nilai `tools.exec` yang digunakan.
Host exec juga menggunakan status persetujuan lokal pada mesin itu. Nilai lokal host
`ask: "always"` di `~/.openclaw/exec-approvals.json` akan tetap memunculkan prompt meskipun
default sesi atau config meminta `ask: "on-miss"`.
Gunakan `openclaw approvals get`, `openclaw approvals get --gateway`, atau
`openclaw approvals get --node <id|name|ip>` untuk memeriksa kebijakan yang diminta,
sumber kebijakan host, dan hasil efektifnya.

Jika UI aplikasi pendamping **tidak tersedia**, setiap permintaan yang memerlukan prompt akan
diselesaikan oleh **ask fallback** (default: tolak).

Klien persetujuan chat native juga dapat mengekspos affordance khusus channel pada pesan
persetujuan yang menunggu. Misalnya, Matrix dapat mengisi shortcut reaksi pada
prompt persetujuan (`✅` izinkan sekali, `❌` tolak, dan `♾️` izinkan selalu jika tersedia)
sementara tetap membiarkan perintah `/approve ...` di pesan sebagai fallback.

## Di mana ini berlaku

Persetujuan exec diberlakukan secara lokal pada host eksekusi:

- **host gateway** → proses `openclaw` pada mesin gateway
- **host node** → node runner (aplikasi pendamping macOS atau host node headless)

Catatan model kepercayaan:

- Pemanggil yang diautentikasi gateway adalah operator tepercaya untuk Gateway tersebut.
- Node yang telah dipairing memperluas kemampuan operator tepercaya tersebut ke host node.
- Persetujuan exec mengurangi risiko eksekusi yang tidak disengaja, tetapi bukan batas auth per pengguna.
- Eksekusi host node yang disetujui mengikat konteks eksekusi kanonis: cwd kanonis, argv yang persis, pengikatan env
  ketika ada, dan path executable yang dipin bila berlaku.
- Untuk shell script dan invokasi file interpreter/runtime langsung, OpenClaw juga mencoba mengikat
  satu operand file lokal konkret. Jika file yang diikat berubah setelah persetujuan tetapi sebelum eksekusi,
  eksekusi ditolak alih-alih menjalankan konten yang telah bergeser.
- Pengikatan file ini sengaja bersifat best-effort, bukan model semantik lengkap untuk setiap
  jalur loader interpreter/runtime. Jika mode persetujuan tidak dapat mengidentifikasi secara tepat satu
  file lokal konkret untuk diikat, OpenClaw menolak membuat eksekusi berbasis persetujuan alih-alih berpura-pura memiliki cakupan penuh.

Pemisahan macOS:

- **layanan host node** meneruskan `system.run` ke **aplikasi macOS** melalui IPC lokal.
- **aplikasi macOS** menegakkan persetujuan + menjalankan perintah dalam konteks UI.

## Pengaturan dan penyimpanan

Persetujuan disimpan dalam file JSON lokal pada host eksekusi:

`~/.openclaw/exec-approvals.json`

Contoh skema:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

## Mode "YOLO" tanpa persetujuan

Jika Anda ingin host exec berjalan tanpa prompt persetujuan, Anda harus membuka **kedua** lapisan kebijakan:

- kebijakan exec yang diminta dalam config OpenClaw (`tools.exec.*`)
- kebijakan persetujuan lokal host di `~/.openclaw/exec-approvals.json`

Ini sekarang menjadi perilaku host default kecuali Anda memperketatnya secara eksplisit:

- `tools.exec.security`: `full` pada `gateway`/`node`
- `tools.exec.ask`: `off`
- host `askFallback`: `full`

Perbedaan penting:

- `tools.exec.host=auto` memilih tempat exec berjalan: sandbox jika tersedia, jika tidak gateway.
- YOLO memilih bagaimana host exec disetujui: `security=full` plus `ask=off`.
- Dalam mode YOLO, OpenClaw tidak menambahkan gate persetujuan heuristik pengaburan perintah terpisah di atas kebijakan host exec yang dikonfigurasi.
- `auto` tidak menjadikan perutean gateway sebagai override gratis dari sesi yang berada di sandbox. Permintaan per panggilan `host=node` diizinkan dari `auto`, dan `host=gateway` hanya diizinkan dari `auto` ketika tidak ada runtime sandbox yang aktif. Jika Anda menginginkan default non-auto yang stabil, setel `tools.exec.host` atau gunakan `/exec host=...` secara eksplisit.

Jika Anda menginginkan penyiapan yang lebih konservatif, perketat kembali salah satu lapisan ke `allowlist` / `on-miss`
atau `deny`.

Penyiapan persisten "jangan pernah memunculkan prompt" pada host gateway:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
openclaw gateway restart
```

Lalu setel file persetujuan host agar sesuai:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Untuk host node, terapkan file persetujuan yang sama pada node tersebut sebagai gantinya:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Shortcut khusus sesi:

- `/exec security=full ask=off` hanya mengubah sesi saat ini.
- `/elevated full` adalah shortcut break-glass yang juga melewati persetujuan exec untuk sesi tersebut.

Jika file persetujuan host tetap lebih ketat daripada config, kebijakan host yang lebih ketat tetap menang.

## Knob kebijakan

### Keamanan (`exec.security`)

- **deny**: blokir semua permintaan host exec.
- **allowlist**: izinkan hanya perintah yang ada di allowlist.
- **full**: izinkan semuanya (setara dengan elevated).

### Ask (`exec.ask`)

- **off**: jangan pernah memunculkan prompt.
- **on-miss**: munculkan prompt hanya ketika allowlist tidak cocok.
- **always**: munculkan prompt pada setiap perintah.
- kepercayaan tahan lama `allow-always` tidak menekan prompt ketika mode ask efektif adalah `always`

### Ask fallback (`askFallback`)

Jika prompt diperlukan tetapi tidak ada UI yang dapat dijangkau, fallback akan memutuskan:

- **deny**: blokir.
- **allowlist**: izinkan hanya jika allowlist cocok.
- **full**: izinkan.

### Hardening eval interpreter inline (`tools.exec.strictInlineEval`)

Ketika `tools.exec.strictInlineEval=true`, OpenClaw memperlakukan bentuk code-eval inline sebagai khusus-persetujuan bahkan jika binary interpreter itu sendiri ada di allowlist.

Contoh:

- `python -c`
- `node -e`, `node --eval`, `node -p`
- `ruby -e`
- `perl -e`, `perl -E`
- `php -r`
- `lua -e`
- `osascript -e`

Ini adalah defense-in-depth untuk loader interpreter yang tidak dipetakan dengan rapi ke satu operand file yang stabil. Dalam mode strict:

- perintah-perintah ini tetap memerlukan persetujuan eksplisit;
- `allow-always` tidak secara otomatis mempertahankan entri allowlist baru untuk perintah-perintah ini.

## Allowlist (per agen)

Allowlist bersifat **per agen**. Jika ada beberapa agen, ubah agen yang sedang Anda
edit di aplikasi macOS. Pattern adalah **glob case-insensitive yang cocok**.
Pattern harus di-resolve menjadi **path binary** (entri yang hanya basename akan diabaikan).
Entri lama `agents.default` dimigrasikan ke `agents.main` saat dimuat.
Rangkaian shell seperti `echo ok && pwd` tetap memerlukan setiap segmen tingkat atas memenuhi aturan allowlist.

Contoh:

- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

Setiap entri allowlist melacak:

- **id** UUID stabil yang digunakan untuk identitas UI (opsional)
- **terakhir digunakan** timestamp
- **perintah terakhir digunakan**
- **path terakhir yang di-resolve**

## Auto-allow Skills CLI

Saat **Auto-allow Skills CLI** diaktifkan, executable yang dirujuk oleh Skills yang diketahui
dianggap berada dalam allowlist pada node (node macOS atau host node headless). Fitur ini menggunakan
`skills.bins` melalui Gateway RPC untuk mengambil daftar bin skill. Nonaktifkan ini jika Anda menginginkan allowlist manual yang ketat.

Catatan kepercayaan penting:

- Ini adalah **allowlist kenyamanan implisit**, terpisah dari entri allowlist path manual.
- Ini dimaksudkan untuk lingkungan operator tepercaya di mana Gateway dan node berada dalam batas kepercayaan yang sama.
- Jika Anda memerlukan kepercayaan eksplisit yang ketat, pertahankan `autoAllowSkills: false` dan gunakan hanya entri allowlist path manual.

## Safe bins (khusus stdin)

`tools.exec.safeBins` mendefinisikan daftar kecil binary **khusus stdin** (misalnya `cut`)
yang dapat berjalan dalam mode allowlist **tanpa** entri allowlist eksplisit. Safe bins menolak
argumen file posisional dan token mirip path, sehingga hanya dapat beroperasi pada stream masuk.
Perlakukan ini sebagai jalur cepat yang sempit untuk filter stream, bukan daftar kepercayaan umum.
**Jangan** menambahkan interpreter atau binary runtime (misalnya `python3`, `node`, `ruby`, `bash`, `sh`, `zsh`) ke `safeBins`.
Jika sebuah perintah dapat mengevaluasi kode, menjalankan subperintah, atau membaca file secara desain, pilih entri allowlist eksplisit dan tetap aktifkan prompt persetujuan.
Safe bin kustom harus mendefinisikan profil eksplisit di `tools.exec.safeBinProfiles.<bin>`.
Validasi bersifat deterministik hanya dari bentuk argv (tanpa pemeriksaan keberadaan filesystem host), yang
mencegah perilaku oracle keberadaan file dari perbedaan allow/deny.
Opsi yang berorientasi file ditolak untuk safe bin default (misalnya `sort -o`, `sort --output`,
`sort --files0-from`, `sort --compress-program`, `sort --random-source`,
`sort --temporary-directory`/`-T`, `wc --files0-from`, `jq -f/--from-file`,
`grep -f/--file`).
Safe bins juga menegakkan kebijakan flag per-binary yang eksplisit untuk opsi yang merusak perilaku
khusus stdin (misalnya `sort -o/--output/--compress-program` dan flag rekursif grep).
Opsi panjang divalidasi fail-closed dalam mode safe-bin: flag yang tidak dikenal dan singkatan ambigu ditolak.
Flag yang ditolak menurut profil safe-bin:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Safe bins juga memaksa token argv diperlakukan sebagai **teks literal** saat eksekusi (tanpa globbing
dan tanpa ekspansi `$VARS`) untuk segmen khusus stdin, sehingga pola seperti `*` atau `$HOME/...` tidak dapat
digunakan untuk menyelundupkan pembacaan file.
Safe bins juga harus di-resolve dari direktori binary tepercaya (default sistem plus
opsional `tools.exec.safeBinTrustedDirs`). Entri `PATH` tidak pernah otomatis dipercaya.
Direktori safe-bin tepercaya default sengaja minimal: `/bin`, `/usr/bin`.
Jika executable safe-bin Anda berada di path package-manager/pengguna (misalnya
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), tambahkan secara eksplisit
ke `tools.exec.safeBinTrustedDirs`.
Rangkaian shell dan redirection tidak diizinkan otomatis dalam mode allowlist.

Rangkaian shell (`&&`, `||`, `;`) diizinkan ketika setiap segmen tingkat atas memenuhi allowlist
(termasuk safe bin atau skill auto-allow). Redirection tetap tidak didukung dalam mode allowlist.
Substitusi perintah (`$()` / backticks) ditolak selama parsing allowlist, termasuk di dalam
tanda kutip ganda; gunakan tanda kutip tunggal jika Anda membutuhkan teks literal `$()`.
Pada persetujuan aplikasi pendamping macOS, teks shell mentah yang berisi sintaks kontrol atau ekspansi shell
(`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) diperlakukan sebagai miss allowlist kecuali
binary shell itu sendiri ada di allowlist.
Untuk shell wrapper (`bash|sh|zsh ... -c/-lc`), override env berscope permintaan dikurangi menjadi
allowlist kecil yang eksplisit (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
Untuk keputusan allow-always dalam mode allowlist, wrapper dispatch yang dikenal
(`env`, `nice`, `nohup`, `stdbuf`, `timeout`) mempertahankan path executable internal alih-alih path wrapper. Shell multiplexer (`busybox`, `toybox`) juga di-unwarp untuk applet shell (`sh`, `ash`,
dan seterusnya) sehingga executable internal dipertahankan alih-alih binary multiplexer. Jika wrapper atau
multiplexer tidak dapat di-unwarp dengan aman, tidak ada entri allowlist yang dipertahankan secara otomatis.
Jika Anda mengallowlist interpreter seperti `python3` atau `node`, sebaiknya gunakan `tools.exec.strictInlineEval=true` agar eval inline tetap memerlukan persetujuan eksplisit. Dalam mode strict, `allow-always` tetap dapat mempertahankan invokasi interpreter/script yang aman, tetapi carrier inline-eval tidak dipertahankan secara otomatis.

Safe bin default:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` dan `sort` tidak ada dalam daftar default. Jika Anda melakukan opt in, pertahankan entri allowlist eksplisit untuk
alur kerja non-stdin mereka.
Untuk `grep` dalam mode safe-bin, berikan pattern dengan `-e`/`--regexp`; bentuk pattern posisional
ditolak agar operand file tidak dapat diselundupkan sebagai posisional ambigu.

### Safe bins versus allowlist

| Topik            | `tools.exec.safeBins`                                  | Allowlist (`exec-approvals.json`)                            |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| Tujuan           | Auto-allow filter stdin yang sempit                    | Secara eksplisit mempercayai executable tertentu             |
| Jenis pencocokan | Nama executable + kebijakan argv safe-bin              | Glob pattern path executable yang di-resolve                 |
| Cakupan argumen  | Dibatasi oleh profil safe-bin dan aturan token literal | Hanya pencocokan path; argumen selain itu adalah tanggung jawab Anda |
| Contoh umum      | `head`, `tail`, `tr`, `wc`                             | `jq`, `python3`, `node`, `ffmpeg`, CLI kustom                |
| Penggunaan terbaik | Transformasi teks berisiko rendah dalam pipeline     | Alat apa pun dengan perilaku atau efek samping yang lebih luas |

Lokasi konfigurasi:

- `safeBins` berasal dari config (`tools.exec.safeBins` atau per-agent `agents.list[].tools.exec.safeBins`).
- `safeBinTrustedDirs` berasal dari config (`tools.exec.safeBinTrustedDirs` atau per-agent `agents.list[].tools.exec.safeBinTrustedDirs`).
- `safeBinProfiles` berasal dari config (`tools.exec.safeBinProfiles` atau per-agent `agents.list[].tools.exec.safeBinProfiles`). Key profil per-agent mengoverride key global.
- entri allowlist berada di `~/.openclaw/exec-approvals.json` lokal host di bawah `agents.<id>.allowlist` (atau melalui UI Control / `openclaw approvals allowlist ...`).
- `openclaw security audit` memperingatkan dengan `tools.exec.safe_bins_interpreter_unprofiled` ketika bin interpreter/runtime muncul di `safeBins` tanpa profil eksplisit.
- `openclaw doctor --fix` dapat membuat scaffold entri `safeBinProfiles.<bin>` kustom yang hilang sebagai `{}` (tinjau dan perketat setelahnya). Bin interpreter/runtime tidak dibuat scaffold otomatis.

Contoh profil kustom:
__OC_I18N_900004__
Jika Anda secara eksplisit melakukan opt in `jq` ke `safeBins`, OpenClaw tetap menolak builtin `env` dalam mode safe-bin
sehingga `jq -n env` tidak dapat membuang env host process tanpa path allowlist eksplisit
atau prompt persetujuan.

## Pengeditan UI Control

Gunakan kartu **UI Control → Nodes → Exec approvals** untuk mengedit default, override
per agen, dan allowlist. Pilih scope (Defaults atau sebuah agen), ubah kebijakan,
tambahkan/hapus pattern allowlist, lalu **Save**. UI menampilkan metadata **terakhir digunakan**
per pattern sehingga Anda dapat menjaga daftar tetap rapi.

Pemilih target memilih **Gateway** (persetujuan lokal) atau **Node**. Node
harus mengiklankan `system.execApprovals.get/set` (aplikasi macOS atau host node headless).
Jika sebuah node belum mengiklankan persetujuan exec, edit
`~/.openclaw/exec-approvals.json` lokalnya secara langsung.

CLI: `openclaw approvals` mendukung pengeditan gateway atau node (lihat [Approvals CLI](/cli/approvals)).

## Alur persetujuan

Ketika prompt diperlukan, gateway menyiarkan `exec.approval.requested` ke klien operator.
UI Control dan aplikasi macOS menyelesaikannya melalui `exec.approval.resolve`, lalu gateway meneruskan
permintaan yang disetujui ke host node.

Untuk `host=node`, permintaan persetujuan menyertakan payload `systemRunPlan` kanonis. Gateway menggunakan
plan tersebut sebagai konteks perintah/cwd/sesi otoritatif saat meneruskan permintaan `system.run`
yang telah disetujui.

Ini penting untuk latensi persetujuan asinkron:

- jalur node exec menyiapkan satu plan kanonis di awal
- catatan persetujuan menyimpan plan tersebut beserta metadata pengikatannya
- setelah disetujui, panggilan akhir `system.run` yang diteruskan menggunakan kembali plan yang tersimpan
  alih-alih mempercayai edit pemanggil berikutnya
- jika pemanggil mengubah `command`, `rawCommand`, `cwd`, `agentId`, atau
  `sessionKey` setelah permintaan persetujuan dibuat, gateway menolak eksekusi
  yang diteruskan sebagai ketidakcocokan persetujuan

## Perintah interpreter/runtime

Eksekusi interpreter/runtime berbasis persetujuan sengaja konservatif:

- Konteks argv/cwd/env yang persis selalu diikat.
- Bentuk shell script langsung dan file runtime langsung diikat secara best-effort ke satu snapshot file lokal konkret.
- Bentuk wrapper package-manager umum yang tetap di-resolve ke satu file lokal langsung (misalnya
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`) di-unwarp sebelum pengikatan.
- Jika OpenClaw tidak dapat mengidentifikasi secara tepat satu file lokal konkret untuk perintah interpreter/runtime
  (misalnya script package, bentuk eval, rangkaian loader khusus runtime, atau bentuk multi-file yang ambigu),
  eksekusi berbasis persetujuan ditolak alih-alih mengklaim cakupan semantik yang tidak dimilikinya.
- Untuk alur kerja tersebut, pilih sandboxing, batas host terpisah, atau alur allowlist/full tepercaya yang eksplisit di mana operator menerima semantik runtime yang lebih luas.

Ketika persetujuan diperlukan, alat exec segera mengembalikan id persetujuan. Gunakan id tersebut untuk
mengorelasikan event system berikutnya (`Exec finished` / `Exec denied`). Jika tidak ada keputusan yang datang sebelum timeout, permintaan diperlakukan sebagai timeout persetujuan dan ditampilkan sebagai alasan penolakan.

### Perilaku pengiriman tindak lanjut

Setelah exec asinkron yang disetujui selesai, OpenClaw mengirim giliran `agent` tindak lanjut ke sesi yang sama.

- Jika ada target pengiriman eksternal yang valid (channel yang dapat mengirim plus target `to`), pengiriman tindak lanjut menggunakan channel tersebut.
- Dalam alur webchat-only atau sesi internal tanpa target eksternal, pengiriman tindak lanjut tetap khusus sesi (`deliver: false`).
- Jika pemanggil secara eksplisit meminta pengiriman eksternal yang ketat tanpa channel eksternal yang dapat di-resolve, permintaan gagal dengan `INVALID_REQUEST`.
- Jika `bestEffortDeliver` diaktifkan dan tidak ada channel eksternal yang dapat di-resolve, pengiriman diturunkan menjadi khusus sesi alih-alih gagal.

Dialog konfirmasi mencakup:

- perintah + argumen
- cwd
- id agen
- path executable yang di-resolve
- host + metadata kebijakan

Tindakan:

- **Allow once** → jalankan sekarang
- **Always allow** → tambahkan ke allowlist + jalankan
- **Deny** → blokir

## Penerusan persetujuan ke chat channel

Anda dapat meneruskan prompt persetujuan exec ke channel chat mana pun (termasuk plugin channel) dan menyetujuinya
dengan `/approve`. Ini menggunakan pipeline pengiriman keluar normal.

Config:
__OC_I18N_900005__
Balas di chat:
__OC_I18N_900006__
Perintah `/approve` menangani persetujuan exec dan persetujuan plugin. Jika ID tidak cocok dengan persetujuan exec yang menunggu, perintah ini otomatis memeriksa persetujuan plugin sebagai gantinya.

### Penerusan persetujuan plugin

Penerusan persetujuan plugin menggunakan pipeline pengiriman yang sama seperti persetujuan exec tetapi memiliki
config independennya sendiri di bawah `approvals.plugin`. Mengaktifkan atau menonaktifkan salah satunya tidak memengaruhi yang lain.
__OC_I18N_900007__
Bentuk config identik dengan `approvals.exec`: `enabled`, `mode`, `agentFilter`,
`sessionFilter`, dan `targets` bekerja dengan cara yang sama.

Channel yang mendukung balasan interaktif bersama merender tombol persetujuan yang sama untuk persetujuan exec dan
plugin. Channel tanpa UI interaktif bersama akan fallback ke teks biasa dengan instruksi `/approve`.

### Persetujuan chat yang sama pada channel apa pun

Ketika permintaan persetujuan exec atau plugin berasal dari permukaan chat yang dapat dikirim, chat yang sama
sekarang dapat menyetujuinya dengan `/approve` secara default. Ini berlaku untuk channel seperti Slack, Matrix, dan
Microsoft Teams selain alur UI Web dan UI terminal yang sudah ada.

Jalur perintah teks bersama ini menggunakan model auth channel normal untuk percakapan tersebut. Jika chat asal
sudah dapat mengirim perintah dan menerima balasan, permintaan persetujuan tidak lagi memerlukan
adapter pengiriman native terpisah hanya agar tetap tertunda.

Discord dan Telegram juga mendukung `/approve` pada chat yang sama, tetapi channel tersebut tetap menggunakan
daftar approver yang telah di-resolve untuk otorisasi bahkan saat pengiriman persetujuan native dinonaktifkan.

Untuk Telegram dan klien persetujuan native lain yang memanggil Gateway secara langsung,
fallback ini sengaja dibatasi pada kegagalan "approval not found". Penolakan/error
persetujuan exec yang nyata tidak diam-diam dicoba ulang sebagai persetujuan plugin.

### Pengiriman persetujuan native

Beberapa channel juga dapat bertindak sebagai klien persetujuan native. Klien native menambahkan DM approver, fanout chat asal,
dan UX persetujuan interaktif khusus channel di atas
alur `/approve` chat yang sama bersama.

Saat kartu/tombol persetujuan native tersedia, UI native tersebut adalah jalur utama yang
berhadapan dengan agen. Agen tidak boleh juga menggemakan perintah
`/approve` chat biasa yang duplikat kecuali hasil alat menyatakan persetujuan chat tidak tersedia atau
persetujuan manual adalah satu-satunya jalur yang tersisa.

Model generik:

- kebijakan host exec tetap menentukan apakah persetujuan exec diperlukan
- `approvals.exec` mengontrol penerusan prompt persetujuan ke tujuan chat lain
- `channels.<channel>.execApprovals` mengontrol apakah channel tersebut bertindak sebagai klien persetujuan native

Klien persetujuan native otomatis mengaktifkan pengiriman DM-first ketika semua hal berikut benar:

- channel mendukung pengiriman persetujuan native
- approver dapat di-resolve dari `execApprovals.approvers` yang eksplisit atau sumber fallback yang didokumentasikan channel tersebut
- `channels.<channel>.execApprovals.enabled` tidak disetel atau `"auto"`

Setel `enabled: false` untuk menonaktifkan klien persetujuan native secara eksplisit. Setel `enabled: true` untuk memaksanya
aktif saat approver berhasil di-resolve. Pengiriman chat asal publik tetap eksplisit melalui
`channels.<channel>.execApprovals.target`.

FAQ: [Mengapa ada dua config persetujuan exec untuk persetujuan chat?](/help/faq#why-are-there-two-exec-approval-configs-for-chat-approvals)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`

Klien persetujuan native ini menambahkan perutean DM dan fanout channel opsional di atas
alur `/approve` chat yang sama bersama dan tombol persetujuan bersama.

Perilaku bersama:

- Slack, Matrix, Microsoft Teams, dan chat lain yang dapat dikirim menggunakan model auth channel normal
  untuk `/approve` pada chat yang sama
- ketika klien persetujuan native aktif otomatis, target pengiriman native default adalah DM approver
- untuk Discord dan Telegram, hanya approver yang telah di-resolve yang dapat menyetujui atau menolak
- approver Discord dapat bersifat eksplisit (`execApprovals.approvers`) atau disimpulkan dari `commands.ownerAllowFrom`
- approver Telegram dapat bersifat eksplisit (`execApprovals.approvers`) atau disimpulkan dari config owner yang sudah ada (`allowFrom`, ditambah direct-message `defaultTo` bila didukung)
- approver Slack dapat bersifat eksplisit (`execApprovals.approvers`) atau disimpulkan dari `commands.ownerAllowFrom`
- tombol native Slack mempertahankan jenis id persetujuan, sehingga id `plugin:` dapat me-resolve persetujuan plugin
  tanpa lapisan fallback lokal Slack kedua
- perutean DM/channel native Matrix hanya untuk exec; persetujuan plugin Matrix tetap berada pada
  `/approve` chat yang sama bersama dan jalur penerusan `approvals.plugin` opsional
- peminta tidak perlu menjadi approver
- chat asal dapat menyetujui langsung dengan `/approve` ketika chat tersebut sudah mendukung perintah dan balasan
- tombol persetujuan Discord native merutekan berdasarkan jenis id persetujuan: id `plugin:` langsung menuju
  persetujuan plugin, selain itu menuju persetujuan exec
- tombol persetujuan Telegram native mengikuti fallback exec-ke-plugin terbatas yang sama seperti `/approve`
- ketika `target` native mengaktifkan pengiriman chat asal, prompt persetujuan menyertakan teks perintah
- persetujuan exec yang menunggu kedaluwarsa setelah 30 menit secara default
- jika tidak ada UI operator atau klien persetujuan yang dikonfigurasi yang dapat menerima permintaan, prompt akan fallback ke `askFallback`

Telegram secara default menggunakan DM approver (`target: "dm"`). Anda dapat beralih ke `channel` atau `both` ketika
ingin prompt persetujuan muncul di chat/topic Telegram asal juga. Untuk topic forum Telegram, OpenClaw mempertahankan topic untuk prompt persetujuan dan tindak lanjut pasca-persetujuan.

Lihat:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)

### Alur IPC macOS
__OC_I18N_900008__
Catatan keamanan:

- Mode Unix socket `0600`, token disimpan di `exec-approvals.json`.
- Pemeriksaan peer dengan UID yang sama.
- Challenge/response (nonce + token HMAC + hash permintaan) + TTL singkat.

## Event system

Siklus hidup exec ditampilkan sebagai pesan system:

- `Exec running` (hanya jika perintah melebihi ambang pemberitahuan running)
- `Exec finished`
- `Exec denied`

Pesan-pesan ini diposting ke sesi agen setelah node melaporkan event tersebut.
Persetujuan exec host-gateway memunculkan event siklus hidup yang sama ketika perintah selesai (dan opsional ketika berjalan lebih lama dari ambang batas).
Exec yang digate oleh persetujuan menggunakan kembali id persetujuan sebagai `runId` dalam pesan-pesan ini agar mudah dikorelasikan.

## Perilaku saat persetujuan ditolak

Ketika persetujuan exec asinkron ditolak, OpenClaw mencegah agen menggunakan kembali
keluaran dari eksekusi sebelumnya atas perintah yang sama dalam sesi. Alasan penolakan
diteruskan dengan panduan eksplisit bahwa tidak ada keluaran perintah yang tersedia, yang mencegah
agen mengklaim ada keluaran baru atau mengulangi perintah yang ditolak dengan
hasil usang dari eksekusi sukses sebelumnya.

## Implikasi

- **full** sangat kuat; pilih allowlist bila memungkinkan.
- **ask** membuat Anda tetap terlibat sambil tetap memungkinkan persetujuan cepat.
- Allowlist per agen mencegah persetujuan satu agen bocor ke agen lain.
- Persetujuan hanya berlaku untuk permintaan host exec dari **pengirim yang terotorisasi**. Pengirim yang tidak terotorisasi tidak dapat mengeluarkan `/exec`.
- `/exec security=full` adalah kenyamanan tingkat sesi untuk operator yang terotorisasi dan dirancang untuk melewati persetujuan.
  Untuk memblokir paksa host exec, setel keamanan persetujuan ke `deny` atau tolak alat `exec` melalui kebijakan alat.

Terkait:

- [Alat Exec](/id/tools/exec)
- [Mode Elevated](/id/tools/elevated)
- [Skills](/id/tools/skills)

## Terkait

- [Exec](/id/tools/exec) — alat eksekusi perintah shell
- [Sandboxing](/id/gateway/sandboxing) — mode sandbox dan akses workspace
- [Security](/id/gateway/security) — model keamanan dan penguatan
- [Sandbox vs Tool Policy vs Elevated](/id/gateway/sandbox-vs-tool-policy-vs-elevated) — kapan menggunakan masing-masing
