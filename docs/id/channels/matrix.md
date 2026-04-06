---
read_when:
    - Menyiapkan Matrix di OpenClaw
    - Mengonfigurasi E2EE dan verifikasi Matrix
summary: Status dukungan Matrix, penyiapan, dan contoh konfigurasi
title: Matrix
x-i18n:
    generated_at: "2026-04-06T03:07:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3e2d84c08d7d5b96db14b914e54f08d25334401cdd92eb890bc8dfb37b0ca2dc
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix adalah plugin channel bawaan Matrix untuk OpenClaw.
Plugin ini menggunakan `matrix-js-sdk` resmi dan mendukung DM, room, thread, media, reaksi, polling, lokasi, dan E2EE.

## Plugin bawaan

Matrix disertakan sebagai plugin bawaan dalam rilis OpenClaw saat ini, sehingga build paket normal tidak memerlukan instalasi terpisah.

Jika Anda menggunakan build lama atau instalasi kustom yang tidak menyertakan Matrix, instal secara manual:

Instal dari npm:

```bash
openclaw plugins install @openclaw/matrix
```

Instal dari checkout lokal:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Lihat [Plugins](/id/tools/plugin) untuk perilaku plugin dan aturan instalasi.

## Penyiapan

1. Pastikan plugin Matrix tersedia.
   - Rilis OpenClaw paket saat ini sudah menyertakannya.
   - Instalasi lama/kustom dapat menambahkannya secara manual dengan perintah di atas.
2. Buat akun Matrix di homeserver Anda.
3. Konfigurasikan `channels.matrix` dengan salah satu dari:
   - `homeserver` + `accessToken`, atau
   - `homeserver` + `userId` + `password`.
4. Mulai ulang gateway.
5. Mulai DM dengan bot atau undang bot ke room.

Jalur penyiapan interaktif:

```bash
openclaw channels add
openclaw configure --section channels
```

Yang sebenarnya ditanyakan wizard Matrix:

- URL homeserver
- metode auth: access token atau password
- ID pengguna hanya saat Anda memilih auth password
- nama perangkat opsional
- apakah akan mengaktifkan E2EE
- apakah akan mengonfigurasi akses room Matrix sekarang

Perilaku wizard yang penting:

- Jika variabel env auth Matrix sudah ada untuk akun yang dipilih, dan akun tersebut belum memiliki auth yang disimpan di config, wizard menawarkan pintasan env dan hanya menulis `enabled: true` untuk akun tersebut.
- Saat Anda menambahkan akun Matrix lain secara interaktif, nama akun yang dimasukkan dinormalisasi menjadi ID akun yang digunakan di config dan variabel env. Misalnya, `Ops Bot` menjadi `ops-bot`.
- Prompt allowlist DM langsung menerima nilai `@user:server` lengkap. Nama tampilan hanya berfungsi saat pencarian direktori live menemukan satu kecocokan yang persis; jika tidak, wizard meminta Anda mencoba lagi dengan ID Matrix lengkap.
- Prompt allowlist room langsung menerima ID room dan alias. Prompt ini juga dapat menyelesaikan nama room yang sudah diikuti secara live, tetapi nama yang tidak dapat diselesaikan hanya disimpan sebagaimana diketik saat penyiapan dan diabaikan nanti oleh resolusi allowlist runtime. Gunakan `!room:server` atau `#alias:server`.
- Identitas room/sesi runtime menggunakan ID room Matrix yang stabil. Alias yang dideklarasikan room hanya digunakan sebagai input pencarian, bukan sebagai kunci sesi jangka panjang atau identitas grup yang stabil.
- Untuk menyelesaikan nama room sebelum menyimpannya, gunakan `openclaw channels resolve --channel matrix "Project Room"`.

Penyiapan minimal berbasis token:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Penyiapan berbasis password (token di-cache setelah login):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

Matrix menyimpan kredensial cache di `~/.openclaw/credentials/matrix/`.
Akun default menggunakan `credentials.json`; akun bernama menggunakan `credentials-<account>.json`.
Ketika kredensial cache ada di sana, OpenClaw menganggap Matrix sudah dikonfigurasi untuk penyiapan, doctor, dan penemuan status channel meskipun auth saat ini tidak disetel langsung di config.

Padanan variabel env (digunakan saat kunci config tidak disetel):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Untuk akun non-default, gunakan variabel env berlingkup akun:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

Contoh untuk akun `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

Untuk ID akun yang dinormalisasi `ops-bot`, gunakan:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix melakukan escape pada tanda baca dalam ID akun agar variabel env berlingkup akun bebas tabrakan.
Misalnya, `-` menjadi `_X2D_`, sehingga `ops-prod` dipetakan ke `MATRIX_OPS_X2D_PROD_*`.

Wizard interaktif hanya menawarkan pintasan variabel env saat variabel env auth tersebut sudah ada dan akun yang dipilih belum memiliki auth Matrix yang disimpan di config.

## Contoh konfigurasi

Ini adalah config baseline praktis dengan pairing DM, allowlist room, dan E2EE diaktifkan:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

## Pratinjau streaming

Streaming balasan Matrix bersifat opt-in.

Setel `channels.matrix.streaming` ke `"partial"` saat Anda ingin OpenClaw mengirim satu balasan pratinjau live,
mengedit pratinjau itu di tempat saat model sedang menghasilkan teks, lalu memfinalkannya saat
balasan selesai:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` adalah default. OpenClaw menunggu balasan final dan mengirimkannya sekali.
- `streaming: "partial"` membuat satu pesan pratinjau yang dapat diedit untuk blok assistant saat ini menggunakan pesan teks Matrix normal. Ini mempertahankan perilaku notifikasi pratinjau-dahulu Matrix lama, sehingga klien bawaan mungkin memberi notifikasi pada teks pratinjau streaming pertama, bukan pada blok yang sudah selesai.
- `streaming: "quiet"` membuat satu pemberitahuan pratinjau senyap yang dapat diedit untuk blok assistant saat ini. Gunakan ini hanya jika Anda juga mengonfigurasi aturan push penerima untuk edit pratinjau yang sudah difinalkan.
- `blockStreaming: true` mengaktifkan pesan progres Matrix terpisah. Dengan streaming pratinjau aktif, Matrix mempertahankan draf live untuk blok saat ini dan mempertahankan blok yang selesai sebagai pesan terpisah.
- Saat streaming pratinjau aktif dan `blockStreaming` nonaktif, Matrix mengedit draf live di tempat dan memfinalkan event yang sama saat blok atau giliran selesai.
- Jika pratinjau tidak lagi muat dalam satu event Matrix, OpenClaw menghentikan streaming pratinjau dan kembali ke pengiriman final normal.
- Balasan media tetap mengirim lampiran seperti biasa. Jika pratinjau lama tidak lagi dapat digunakan ulang dengan aman, OpenClaw menghapusnya sebelum mengirim balasan media final.
- Edit pratinjau menambah panggilan API Matrix. Biarkan streaming nonaktif jika Anda menginginkan perilaku rate limit yang paling konservatif.

`blockStreaming` tidak mengaktifkan pratinjau draf dengan sendirinya.
Gunakan `streaming: "partial"` atau `streaming: "quiet"` untuk edit pratinjau; lalu tambahkan `blockStreaming: true` hanya jika Anda juga ingin blok assistant yang sudah selesai tetap terlihat sebagai pesan progres terpisah.

Jika Anda memerlukan notifikasi Matrix bawaan tanpa aturan push kustom, gunakan `streaming: "partial"` untuk perilaku pratinjau-dahulu atau biarkan `streaming` nonaktif untuk pengiriman final saja. Dengan `streaming: "off"`:

- `blockStreaming: true` mengirim setiap blok yang selesai sebagai pesan Matrix normal yang memicu notifikasi.
- `blockStreaming: false` hanya mengirim balasan akhir yang sudah selesai sebagai pesan Matrix normal yang memicu notifikasi.

### Aturan push self-hosted untuk pratinjau final senyap

Jika Anda menjalankan infrastruktur Matrix sendiri dan ingin pratinjau senyap hanya memberi notifikasi saat suatu blok atau
balasan final selesai, setel `streaming: "quiet"` dan tambahkan aturan push per pengguna untuk edit pratinjau yang sudah difinalkan.

Ini biasanya merupakan penyiapan pengguna penerima, bukan perubahan config global homeserver:

Pemetaan singkat sebelum mulai:

- pengguna penerima = orang yang harus menerima notifikasi
- pengguna bot = akun Matrix OpenClaw yang mengirim balasan
- gunakan access token pengguna penerima untuk panggilan API di bawah
- cocokkan `sender` dalam aturan push dengan MXID lengkap pengguna bot

1. Konfigurasikan OpenClaw agar menggunakan pratinjau senyap:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. Pastikan akun penerima sudah menerima notifikasi push Matrix normal. Aturan
   pratinjau senyap hanya berfungsi jika pengguna tersebut sudah memiliki pusher/perangkat yang berfungsi.

3. Dapatkan access token pengguna penerima.
   - Gunakan token pengguna penerima, bukan token bot.
   - Menggunakan ulang token sesi klien yang sudah ada biasanya paling mudah.
   - Jika Anda perlu membuat token baru, Anda dapat login melalui API Client-Server Matrix standar:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. Verifikasi bahwa akun penerima sudah memiliki pusher:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Jika ini tidak mengembalikan pusher/perangkat aktif, perbaiki dulu notifikasi Matrix normal sebelum menambahkan
aturan OpenClaw di bawah.

OpenClaw menandai edit pratinjau khusus teks yang sudah difinalkan dengan:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Buat aturan push override untuk setiap akun penerima yang harus menerima notifikasi ini:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

Ganti nilai berikut sebelum menjalankan perintah:

- `https://matrix.example.org`: URL dasar homeserver Anda
- `$USER_ACCESS_TOKEN`: access token pengguna penerima
- `openclaw-finalized-preview-botname`: ID aturan yang unik untuk bot ini bagi pengguna penerima ini
- `@bot:example.org`: MXID bot Matrix OpenClaw Anda, bukan MXID pengguna penerima

Penting untuk penyiapan multi-bot:

- Aturan push menggunakan kunci `ruleId`. Menjalankan ulang `PUT` terhadap rule ID yang sama akan memperbarui aturan itu.
- Jika satu pengguna penerima harus menerima notifikasi dari beberapa akun bot Matrix OpenClaw, buat satu aturan per bot dengan rule ID unik untuk setiap kecocokan sender.
- Pola sederhana adalah `openclaw-finalized-preview-<botname>`, misalnya `openclaw-finalized-preview-ops` atau `openclaw-finalized-preview-support`.

Aturan dievaluasi terhadap pengirim event:

- autentikasi dengan token pengguna penerima
- cocokkan `sender` dengan MXID bot OpenClaw

6. Verifikasi bahwa aturan ada:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Uji balasan streaming. Dalam mode senyap, room seharusnya menampilkan pratinjau draf senyap dan edit final
   di tempat seharusnya memberi notifikasi sekali saat blok atau giliran selesai.

Jika nanti Anda perlu menghapus aturan tersebut, hapus rule ID yang sama dengan token pengguna penerima:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Catatan:

- Buat aturan dengan access token pengguna penerima, bukan token bot.
- Aturan `override` baru yang ditentukan pengguna disisipkan di depan aturan suppress default, jadi tidak diperlukan parameter urutan tambahan.
- Ini hanya memengaruhi edit pratinjau khusus teks yang dapat difinalkan OpenClaw dengan aman di tempat. Fallback media dan fallback pratinjau lama tetap menggunakan pengiriman Matrix normal.
- Jika `GET /_matrix/client/v3/pushers` tidak menampilkan pusher, pengguna tersebut belum memiliki pengiriman push Matrix yang berfungsi untuk akun/perangkat ini.

#### Synapse

Untuk Synapse, penyiapan di atas biasanya sudah cukup dengan sendirinya:

- Tidak diperlukan perubahan `homeserver.yaml` khusus untuk notifikasi pratinjau OpenClaw yang sudah difinalkan.
- Jika deployment Synapse Anda sudah mengirim notifikasi push Matrix normal, token pengguna + panggilan `pushrules` di atas adalah langkah penyiapan utama.
- Jika Anda menjalankan Synapse di belakang reverse proxy atau worker, pastikan `/_matrix/client/.../pushrules/` menjangkau Synapse dengan benar.
- Jika Anda menjalankan worker Synapse, pastikan pusher dalam kondisi sehat. Pengiriman push ditangani oleh proses utama atau `synapse.app.pusher` / worker pusher yang dikonfigurasi.

#### Tuwunel

Untuk Tuwunel, gunakan alur penyiapan dan panggilan API push-rule yang sama seperti di atas:

- Tidak diperlukan config khusus Tuwunel untuk penanda pratinjau yang sudah difinalkan itu sendiri.
- Jika notifikasi Matrix normal sudah berfungsi untuk pengguna tersebut, token pengguna + panggilan `pushrules` di atas adalah langkah penyiapan utama.
- Jika notifikasi tampak hilang saat pengguna aktif di perangkat lain, periksa apakah `suppress_push_when_active` diaktifkan. Tuwunel menambahkan opsi ini di Tuwunel 1.4.2 pada 12 September 2025, dan ini dapat dengan sengaja menekan push ke perangkat lain saat satu perangkat aktif.

## Enkripsi dan verifikasi

Di room terenkripsi (E2EE), event gambar keluar menggunakan `thumbnail_file` sehingga pratinjau gambar dienkripsi bersama lampiran penuh. Room yang tidak terenkripsi tetap menggunakan `thumbnail_url` biasa. Tidak diperlukan konfigurasi — plugin mendeteksi status E2EE secara otomatis.

### Room bot ke bot

Secara default, pesan Matrix dari akun Matrix OpenClaw lain yang dikonfigurasi diabaikan.

Gunakan `allowBots` saat Anda memang ingin lalu lintas Matrix antar-agent:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` menerima pesan dari akun bot Matrix lain yang dikonfigurasi di room dan DM yang diizinkan.
- `allowBots: "mentions"` menerima pesan tersebut hanya saat secara terlihat menyebut bot ini di room. DM tetap diizinkan.
- `groups.<room>.allowBots` menimpa setelan tingkat akun untuk satu room.
- OpenClaw tetap mengabaikan pesan dari ID pengguna Matrix yang sama untuk menghindari loop balasan ke diri sendiri.
- Matrix tidak mengekspos penanda bot bawaan di sini; OpenClaw menganggap "ditulis bot" sebagai "dikirim oleh akun Matrix lain yang dikonfigurasi pada gateway OpenClaw ini".

Gunakan allowlist room yang ketat dan persyaratan mention saat mengaktifkan lalu lintas bot ke bot di room bersama.

Aktifkan enkripsi:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

Periksa status verifikasi:

```bash
openclaw matrix verify status
```

Status verbose (diagnostik lengkap):

```bash
openclaw matrix verify status --verbose
```

Sertakan recovery key yang disimpan dalam output yang dapat dibaca mesin:

```bash
openclaw matrix verify status --include-recovery-key --json
```

Bootstrap cross-signing dan status verifikasi:

```bash
openclaw matrix verify bootstrap
```

Dukungan multi-akun: gunakan `channels.matrix.accounts` dengan kredensial per akun dan `name` opsional. Lihat [Configuration reference](/id/gateway/configuration-reference#multi-account-all-channels) untuk pola bersama.

Diagnostik bootstrap verbose:

```bash
openclaw matrix verify bootstrap --verbose
```

Paksa reset identitas cross-signing baru sebelum bootstrap:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

Verifikasi perangkat ini dengan recovery key:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

Detail verifikasi perangkat verbose:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

Periksa kesehatan backup room-key:

```bash
openclaw matrix verify backup status
```

Diagnostik kesehatan backup verbose:

```bash
openclaw matrix verify backup status --verbose
```

Pulihkan room key dari backup server:

```bash
openclaw matrix verify backup restore
```

Diagnostik pemulihan verbose:

```bash
openclaw matrix verify backup restore --verbose
```

Hapus backup server saat ini dan buat baseline backup baru. Jika backup key yang tersimpan
tidak dapat dimuat dengan bersih, reset ini juga dapat membuat ulang secret storage sehingga
cold start berikutnya dapat memuat backup key yang baru:

```bash
openclaw matrix verify backup reset --yes
```

Semua perintah `verify` ringkas secara default (termasuk logging SDK internal yang senyap) dan hanya menampilkan diagnostik detail dengan `--verbose`.
Gunakan `--json` untuk output lengkap yang dapat dibaca mesin saat melakukan scripting.

Dalam penyiapan multi-akun, perintah CLI Matrix menggunakan akun default Matrix implisit kecuali Anda meneruskan `--account <id>`.
Jika Anda mengonfigurasi beberapa akun bernama, setel `channels.matrix.defaultAccount` terlebih dahulu atau operasi CLI implisit tersebut akan berhenti dan meminta Anda memilih akun secara eksplisit.
Gunakan `--account` kapan pun Anda ingin operasi verifikasi atau perangkat menargetkan akun bernama secara eksplisit:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Saat enkripsi dinonaktifkan atau tidak tersedia untuk akun bernama, peringatan Matrix dan error verifikasi mengarah ke kunci config akun tersebut, misalnya `channels.matrix.accounts.assistant.encryption`.

### Arti "terverifikasi"

OpenClaw menganggap perangkat Matrix ini terverifikasi hanya saat perangkat itu diverifikasi oleh identitas cross-signing Anda sendiri.
Dalam praktiknya, `openclaw matrix verify status --verbose` mengekspos tiga sinyal kepercayaan:

- `Locally trusted`: perangkat ini dipercaya hanya oleh klien saat ini
- `Cross-signing verified`: SDK melaporkan perangkat ini sebagai terverifikasi melalui cross-signing
- `Signed by owner`: perangkat ditandatangani oleh self-signing key Anda sendiri

`Verified by owner` menjadi `yes` hanya saat verifikasi cross-signing atau owner-signing ada.
Kepercayaan lokal saja tidak cukup bagi OpenClaw untuk menganggap perangkat ini sepenuhnya terverifikasi.

### Apa yang dilakukan bootstrap

`openclaw matrix verify bootstrap` adalah perintah perbaikan dan penyiapan untuk akun Matrix terenkripsi.
Perintah ini melakukan semua hal berikut secara berurutan:

- mem-bootstrap secret storage, menggunakan ulang recovery key yang ada bila memungkinkan
- mem-bootstrap cross-signing dan mengunggah public cross-signing key yang belum ada
- mencoba menandai dan cross-sign perangkat saat ini
- membuat backup room-key sisi server baru jika belum ada

Jika homeserver memerlukan auth interaktif untuk mengunggah cross-signing key, OpenClaw mencoba unggahan tanpa auth terlebih dahulu, lalu dengan `m.login.dummy`, lalu dengan `m.login.password` saat `channels.matrix.password` dikonfigurasi.

Gunakan `--force-reset-cross-signing` hanya jika Anda memang ingin membuang identitas cross-signing saat ini dan membuat yang baru.

Jika Anda memang ingin membuang backup room-key saat ini dan memulai baseline
backup baru untuk pesan mendatang, gunakan `openclaw matrix verify backup reset --yes`.
Lakukan ini hanya jika Anda menerima bahwa riwayat terenkripsi lama yang tidak dapat dipulihkan akan tetap
tidak tersedia dan bahwa OpenClaw mungkin akan membuat ulang secret storage jika secret backup saat ini
tidak dapat dimuat dengan aman.

### Baseline backup baru

Jika Anda ingin menjaga agar pesan terenkripsi mendatang tetap berfungsi dan menerima kehilangan riwayat lama yang tidak dapat dipulihkan, jalankan perintah berikut secara berurutan:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Tambahkan `--account <id>` ke setiap perintah jika Anda ingin menargetkan akun Matrix bernama secara eksplisit.

### Perilaku saat startup

Saat `encryption: true`, Matrix secara default menetapkan `startupVerification` ke `"if-unverified"`.
Saat startup, jika perangkat ini masih belum terverifikasi, Matrix akan meminta verifikasi mandiri di klien Matrix lain,
melewati permintaan duplikat saat satu permintaan sudah tertunda, dan menerapkan cooldown lokal sebelum mencoba lagi setelah restart.
Secara default, percobaan permintaan yang gagal diulang lebih cepat daripada pembuatan permintaan yang berhasil.
Setel `startupVerification: "off"` untuk menonaktifkan permintaan startup otomatis, atau sesuaikan `startupVerificationCooldownHours`
jika Anda ingin jendela percobaan ulang yang lebih pendek atau lebih panjang.

Startup juga secara otomatis melakukan proses bootstrap crypto konservatif.
Proses itu mencoba menggunakan ulang secret storage dan identitas cross-signing saat ini terlebih dahulu, dan menghindari reset cross-signing kecuali Anda menjalankan alur perbaikan bootstrap yang eksplisit.

Jika startup menemukan status bootstrap yang rusak dan `channels.matrix.password` dikonfigurasi, OpenClaw dapat mencoba jalur perbaikan yang lebih ketat.
Jika perangkat saat ini sudah ditandatangani pemilik, OpenClaw mempertahankan identitas itu alih-alih meresetnya secara otomatis.

Upgrade dari plugin Matrix publik sebelumnya:

- OpenClaw secara otomatis menggunakan ulang akun Matrix, access token, dan identitas perangkat yang sama bila memungkinkan.
- Sebelum perubahan migrasi Matrix yang dapat ditindaklanjuti dijalankan, OpenClaw membuat atau menggunakan ulang snapshot pemulihan di `~/Backups/openclaw-migrations/`.
- Jika Anda menggunakan beberapa akun Matrix, setel `channels.matrix.defaultAccount` sebelum upgrade dari tata letak flat-store lama agar OpenClaw tahu akun mana yang harus menerima state lawas bersama tersebut.
- Jika plugin sebelumnya menyimpan key dekripsi backup room-key Matrix secara lokal, startup atau `openclaw doctor --fix` akan mengimpornya ke alur recovery-key baru secara otomatis.
- Jika access token Matrix berubah setelah migrasi disiapkan, startup sekarang memindai root penyimpanan hash token saudara untuk state pemulihan lawas yang tertunda sebelum menyerah pada pemulihan backup otomatis.
- Jika access token Matrix berubah kemudian untuk akun, homeserver, dan pengguna yang sama, OpenClaw sekarang lebih memilih menggunakan ulang root penyimpanan hash token yang paling lengkap yang ada alih-alih memulai dari direktori state Matrix kosong.
- Pada startup gateway berikutnya, room key yang dicadangkan akan dipulihkan secara otomatis ke crypto store baru.
- Jika plugin lama memiliki room key lokal saja yang tidak pernah dicadangkan, OpenClaw akan memberi peringatan dengan jelas. Key tersebut tidak dapat diekspor secara otomatis dari rust crypto store sebelumnya, sehingga beberapa riwayat terenkripsi lama mungkin tetap tidak tersedia sampai dipulihkan secara manual.
- Lihat [Matrix migration](/id/install/migrating-matrix) untuk alur upgrade lengkap, batasan, perintah pemulihan, dan pesan migrasi umum.

State runtime terenkripsi diatur dalam root hash token per akun dan per pengguna di
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Direktori tersebut berisi sync store (`bot-storage.json`), crypto store (`crypto/`),
file recovery key (`recovery-key.json`), snapshot IndexedDB (`crypto-idb-snapshot.json`),
binding thread (`thread-bindings.json`), dan state verifikasi startup (`startup-verification.json`)
saat fitur tersebut digunakan.
Saat token berubah tetapi identitas akun tetap sama, OpenClaw menggunakan ulang root terbaik yang ada
untuk tuple akun/homeserver/pengguna tersebut sehingga state sync sebelumnya, state crypto, binding thread,
dan state verifikasi startup tetap terlihat.

### Model crypto store Node

E2EE Matrix dalam plugin ini menggunakan jalur Rust crypto `matrix-js-sdk` resmi di Node.
Jalur itu mengharapkan persistensi berbasis IndexedDB saat Anda ingin state crypto bertahan setelah restart.

Saat ini OpenClaw menyediakannya di Node dengan cara:

- menggunakan `fake-indexeddb` sebagai shim API IndexedDB yang diharapkan SDK
- memulihkan konten IndexedDB Rust crypto dari `crypto-idb-snapshot.json` sebelum `initRustCrypto`
- menyimpan kembali konten IndexedDB yang diperbarui ke `crypto-idb-snapshot.json` setelah init dan selama runtime
- menserialkan pemulihan dan penyimpanan snapshot terhadap `crypto-idb-snapshot.json` dengan file lock advisori agar persistensi runtime gateway dan pemeliharaan CLI tidak saling berpacu pada file snapshot yang sama

Ini adalah plumbing kompatibilitas/penyimpanan, bukan implementasi crypto kustom.
File snapshot adalah state runtime sensitif dan disimpan dengan izin file yang ketat.
Dalam model keamanan OpenClaw, host gateway dan direktori state OpenClaw lokal sudah berada di dalam batas operator tepercaya, sehingga ini terutama merupakan persoalan ketahanan operasional, bukan batas kepercayaan jarak jauh yang terpisah.

Peningkatan yang direncanakan:

- menambahkan dukungan SecretRef untuk material key Matrix persisten sehingga recovery key dan secret enkripsi store terkait dapat diambil dari provider secret OpenClaw, bukan hanya file lokal

## Pengelolaan profil

Perbarui profil mandiri Matrix untuk akun yang dipilih dengan:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Tambahkan `--account <id>` saat Anda ingin menargetkan akun Matrix bernama secara eksplisit.

Matrix menerima URL avatar `mxc://` secara langsung. Saat Anda meneruskan URL avatar `http://` atau `https://`, OpenClaw terlebih dahulu mengunggahnya ke Matrix dan menyimpan kembali URL `mxc://` yang telah diselesaikan ke `channels.matrix.avatarUrl` (atau override akun yang dipilih).

## Pemberitahuan verifikasi otomatis

Matrix sekarang memposting pemberitahuan siklus hidup verifikasi langsung ke room verifikasi DM ketat sebagai pesan `m.notice`.
Itu mencakup:

- pemberitahuan permintaan verifikasi
- pemberitahuan verifikasi siap (dengan panduan eksplisit "Verify by emoji")
- pemberitahuan mulai dan selesai verifikasi
- detail SAS (emoji dan desimal) bila tersedia

Permintaan verifikasi masuk dari klien Matrix lain dilacak dan diterima otomatis oleh OpenClaw.
Untuk alur verifikasi mandiri, OpenClaw juga memulai alur SAS secara otomatis saat verifikasi emoji tersedia dan mengonfirmasi sisinya sendiri.
Untuk permintaan verifikasi dari pengguna/perangkat Matrix lain, OpenClaw menerima permintaan secara otomatis lalu menunggu alur SAS berjalan seperti biasa.
Anda tetap perlu membandingkan emoji atau SAS desimal di klien Matrix Anda dan mengonfirmasi "They match" di sana untuk menyelesaikan verifikasi.

OpenClaw tidak menerima otomatis alur duplikat yang dimulai sendiri secara membabi buta. Startup melewati pembuatan permintaan baru saat permintaan verifikasi mandiri sudah tertunda.

Pemberitahuan protokol/sistem verifikasi tidak diteruskan ke pipeline chat agent, sehingga tidak menghasilkan `NO_REPLY`.

### Kebersihan perangkat

Perangkat Matrix yang dikelola OpenClaw lama dapat menumpuk di akun dan membuat kepercayaan room terenkripsi lebih sulit dipahami.
Daftarkan perangkat dengan:

```bash
openclaw matrix devices list
```

Hapus perangkat OpenClaw lama yang sudah tidak aktif dengan:

```bash
openclaw matrix devices prune-stale
```

### Perbaikan Room Langsung

Jika state direct-message tidak sinkron, OpenClaw dapat berakhir dengan pemetaan `m.direct` lama yang menunjuk ke room solo lama alih-alih DM live. Periksa pemetaan saat ini untuk rekan dengan:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Perbaiki dengan:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

Perbaikan menjaga logika khusus Matrix tetap berada di dalam plugin:

- perbaikan ini lebih memilih DM 1:1 ketat yang sudah dipetakan di `m.direct`
- jika tidak, perbaikan ini kembali ke DM 1:1 ketat mana pun yang saat ini diikuti dengan pengguna tersebut
- jika tidak ada DM sehat, perbaikan ini membuat direct room baru dan menulis ulang `m.direct` agar menunjuk ke room itu

Alur perbaikan tidak menghapus room lama secara otomatis. Alur ini hanya memilih DM yang sehat dan memperbarui pemetaan agar pengiriman Matrix baru, pemberitahuan verifikasi, dan alur direct-message lain kembali menargetkan room yang benar.

## Thread

Matrix mendukung thread Matrix bawaan untuk balasan otomatis maupun pengiriman message-tool.

- `dm.sessionScope: "per-user"` (default) menjaga perutean DM Matrix tetap berbasis pengirim, sehingga beberapa room DM dapat berbagi satu sesi saat semuanya diselesaikan ke rekan yang sama.
- `dm.sessionScope: "per-room"` mengisolasi setiap room DM Matrix ke kunci sesi masing-masing sambil tetap menggunakan auth DM normal dan pemeriksaan allowlist.
- Binding percakapan Matrix eksplisit tetap lebih diutamakan daripada `dm.sessionScope`, sehingga room dan thread yang dibinding tetap menggunakan sesi target yang dipilih.
- `threadReplies: "off"` menjaga balasan tetap di level atas dan menjaga pesan threaded masuk pada sesi induk.
- `threadReplies: "inbound"` membalas di dalam thread hanya ketika pesan masuk sudah berada di thread itu.
- `threadReplies: "always"` menjaga balasan room di thread yang berakar pada pesan pemicu dan merutekan percakapan itu melalui sesi berlingkup thread yang cocok dari pesan pemicu pertama.
- `dm.threadReplies` menimpa setelan tingkat atas untuk DM saja. Misalnya, Anda dapat menjaga thread room tetap terisolasi sambil menjaga DM tetap datar.
- Pesan threaded masuk menyertakan pesan root thread sebagai konteks agent tambahan.
- Pengiriman message-tool sekarang otomatis mewarisi thread Matrix saat ini ketika targetnya adalah room yang sama, atau target pengguna DM yang sama, kecuali `threadId` eksplisit diberikan.
- Penggunaan ulang target pengguna DM dengan sesi yang sama hanya aktif saat metadata sesi saat ini membuktikan rekan DM yang sama pada akun Matrix yang sama; jika tidak, OpenClaw kembali ke perutean normal berlingkup pengguna.
- Saat OpenClaw melihat room DM Matrix bertabrakan dengan room DM lain pada sesi DM Matrix bersama yang sama, OpenClaw memposting `m.notice` satu kali di room itu dengan jalur keluar `/focus` saat binding thread diaktifkan dan petunjuk `dm.sessionScope`.
- Binding thread runtime didukung untuk Matrix. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`, dan `/acp spawn` yang dibinding ke thread kini berfungsi di room dan DM Matrix.
- `/focus` room/DM Matrix level atas membuat thread Matrix baru dan membindingnya ke sesi target saat `threadBindings.spawnSubagentSessions=true`.
- Menjalankan `/focus` atau `/acp spawn --thread here` di dalam thread Matrix yang sudah ada akan membinding thread saat ini tersebut.

## ACP conversation bindings

Room, DM, dan thread Matrix yang sudah ada dapat diubah menjadi workspace ACP yang tahan lama tanpa mengubah permukaan chat.

Alur operator cepat:

- Jalankan `/acp spawn codex --bind here` di dalam DM, room, atau thread Matrix yang sudah ada dan ingin terus Anda gunakan.
- Di DM atau room Matrix level atas, DM/room saat ini tetap menjadi permukaan chat dan pesan berikutnya dirutekan ke sesi ACP yang di-spawn.
- Di dalam thread Matrix yang sudah ada, `--bind here` membinding thread saat ini di tempat.
- `/new` dan `/reset` mereset sesi ACP yang sama yang dibinding di tempat.
- `/acp close` menutup sesi ACP dan menghapus binding.

Catatan:

- `--bind here` tidak membuat child thread Matrix.
- `threadBindings.spawnAcpSessions` hanya diperlukan untuk `/acp spawn --thread auto|here`, saat OpenClaw perlu membuat atau membinding child thread Matrix.

### Config Thread Binding

Matrix mewarisi default global dari `session.threadBindings`, dan juga mendukung override per channel:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Flag spawn yang dibinding ke thread Matrix bersifat opt-in:

- Setel `threadBindings.spawnSubagentSessions: true` untuk mengizinkan `/focus` level atas membuat dan membinding thread Matrix baru.
- Setel `threadBindings.spawnAcpSessions: true` untuk mengizinkan `/acp spawn --thread auto|here` membinding sesi ACP ke thread Matrix.

## Reaksi

Matrix mendukung aksi reaksi keluar, notifikasi reaksi masuk, dan reaksi ack masuk.

- Tooling reaksi keluar dikendalikan oleh `channels["matrix"].actions.reactions`.
- `react` menambahkan reaksi ke event Matrix tertentu.
- `reactions` mencantumkan ringkasan reaksi saat ini untuk event Matrix tertentu.
- `emoji=""` menghapus reaksi milik akun bot sendiri pada event itu.
- `remove: true` hanya menghapus reaksi emoji yang ditentukan dari akun bot.

Cakupan reaksi ack diselesaikan dengan urutan resolusi OpenClaw standar:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- fallback emoji identitas agent

Cakupan reaksi ack diselesaikan dalam urutan ini:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

Mode notifikasi reaksi diselesaikan dalam urutan ini:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- default: `own`

Perilaku saat ini:

- `reactionNotifications: "own"` meneruskan event `m.reaction` yang ditambahkan saat menargetkan pesan Matrix yang ditulis bot.
- `reactionNotifications: "off"` menonaktifkan event sistem reaksi.
- Penghapusan reaksi masih belum disintesis menjadi event sistem karena Matrix menampilkannya sebagai redaksi, bukan sebagai penghapusan `m.reaction` mandiri.

## Konteks riwayat

- `channels.matrix.historyLimit` mengontrol berapa banyak pesan room terbaru yang disertakan sebagai `InboundHistory` saat pesan room Matrix memicu agent.
- Nilai ini fallback ke `messages.groupChat.historyLimit`. Setel `0` untuk menonaktifkan.
- Riwayat room Matrix hanya untuk room. DM tetap menggunakan riwayat sesi normal.
- Riwayat room Matrix hanya pending: OpenClaw menampung pesan room yang belum memicu balasan, lalu mengambil snapshot jendela itu saat mention atau pemicu lain tiba.
- Pesan pemicu saat ini tidak disertakan dalam `InboundHistory`; pesan itu tetap berada di body masuk utama untuk giliran tersebut.
- Percobaan ulang event Matrix yang sama menggunakan ulang snapshot riwayat asli alih-alih bergeser maju ke pesan room yang lebih baru.

## Visibilitas konteks

Matrix mendukung kontrol `contextVisibility` bersama untuk konteks room tambahan seperti teks balasan yang diambil, root thread, dan riwayat tertunda.

- `contextVisibility: "all"` adalah default. Konteks tambahan dipertahankan sebagaimana diterima.
- `contextVisibility: "allowlist"` memfilter konteks tambahan ke pengirim yang diizinkan oleh pemeriksaan allowlist room/pengguna yang aktif.
- `contextVisibility: "allowlist_quote"` berperilaku seperti `allowlist`, tetapi tetap mempertahankan satu balasan kutipan eksplisit.

Setelan ini memengaruhi visibilitas konteks tambahan, bukan apakah pesan masuk itu sendiri dapat memicu balasan.
Otorisasi pemicu tetap berasal dari setelan `groupPolicy`, `groups`, `groupAllowFrom`, dan kebijakan DM.

## Contoh kebijakan DM dan room

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Lihat [Groups](/id/channels/groups) untuk perilaku mention-gating dan allowlist.

Contoh pairing untuk DM Matrix:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Jika pengguna Matrix yang belum disetujui terus mengirimi Anda pesan sebelum persetujuan, OpenClaw menggunakan ulang kode pairing tertunda yang sama dan dapat mengirim balasan pengingat lagi setelah cooldown singkat alih-alih membuat kode baru.

Lihat [Pairing](/id/channels/pairing) untuk alur pairing DM bersama dan tata letak penyimpanan.

## Persetujuan exec

Matrix dapat bertindak sebagai klien persetujuan exec untuk akun Matrix.

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (opsional; fallback ke `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, default: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Approver harus berupa ID pengguna Matrix seperti `@owner:example.org`. Matrix otomatis mengaktifkan persetujuan exec bawaan saat `enabled` tidak disetel atau `"auto"` dan setidaknya satu approver dapat diselesaikan, baik dari `execApprovals.approvers` maupun dari `channels.matrix.dm.allowFrom`. Setel `enabled: false` untuk menonaktifkan Matrix sebagai klien persetujuan bawaan secara eksplisit. Jika tidak, permintaan persetujuan akan fallback ke rute persetujuan lain yang dikonfigurasi atau kebijakan fallback persetujuan exec.

Perutean bawaan Matrix saat ini hanya untuk exec:

- `channels.matrix.execApprovals.*` mengontrol perutean DM/channel bawaan untuk persetujuan exec saja.
- Persetujuan plugin tetap menggunakan `/approve` same-chat bersama ditambah penerusan `approvals.plugin` yang dikonfigurasi.
- Matrix masih dapat menggunakan ulang `channels.matrix.dm.allowFrom` untuk otorisasi persetujuan plugin ketika approver dapat diinferensikan dengan aman, tetapi tidak mengekspos jalur fanout DM/channel persetujuan plugin bawaan yang terpisah.

Aturan pengiriman:

- `target: "dm"` mengirim prompt persetujuan ke DM approver
- `target: "channel"` mengirim prompt kembali ke room atau DM Matrix asal
- `target: "both"` mengirim ke DM approver dan room atau DM Matrix asal

Prompt persetujuan Matrix menanamkan pintasan reaksi pada pesan persetujuan utama:

- `✅` = izinkan sekali
- `❌` = tolak
- `♾️` = selalu izinkan jika keputusan itu diizinkan oleh kebijakan exec efektif

Approver dapat bereaksi pada pesan itu atau menggunakan slash command fallback: `/approve <id> allow-once`, `/approve <id> allow-always`, atau `/approve <id> deny`.

Hanya approver yang diselesaikan yang dapat menyetujui atau menolak. Pengiriman channel menyertakan teks perintah, jadi aktifkan `channel` atau `both` hanya di room tepercaya.

Prompt persetujuan Matrix menggunakan ulang shared core approval planner. Permukaan bawaan khusus Matrix hanya merupakan transport untuk persetujuan exec: perutean room/DM dan perilaku kirim/perbarui/hapus pesan.

Override per akun:

- `channels.matrix.accounts.<account>.execApprovals`

Dokumentasi terkait: [Exec approvals](/id/tools/exec-approvals)

## Contoh multi-akun

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

Nilai `channels.matrix` tingkat atas bertindak sebagai default untuk akun bernama kecuali suatu akun menimpanya.
Anda dapat memberi cakupan entri room turunan ke satu akun Matrix dengan `groups.<room>.account` (atau `rooms.<room>.account` lama).
Entri tanpa `account` tetap dibagikan ke semua akun Matrix, dan entri dengan `account: "default"` tetap berfungsi saat akun default dikonfigurasi langsung pada `channels.matrix.*` tingkat atas.
Default auth bersama parsial tidak dengan sendirinya membuat akun default implisit terpisah. OpenClaw hanya mensintesis akun `default` tingkat atas saat default tersebut memiliki auth baru (`homeserver` plus `accessToken`, atau `homeserver` plus `userId` dan `password`); akun bernama masih dapat tetap dapat ditemukan dari `homeserver` plus `userId` ketika kredensial cache memenuhi auth nanti.
Jika Matrix sudah memiliki tepat satu akun bernama, atau `defaultAccount` menunjuk ke kunci akun bernama yang ada, promosi perbaikan/penyiapan dari akun tunggal ke multi-akun mempertahankan akun itu alih-alih membuat entri `accounts.default` baru. Hanya kunci auth/bootstrap Matrix yang dipindahkan ke akun yang dipromosikan itu; kunci kebijakan pengiriman bersama tetap berada di tingkat atas.
Setel `defaultAccount` saat Anda ingin OpenClaw lebih memilih satu akun Matrix bernama untuk perutean implisit, probing, dan operasi CLI.
Jika Anda mengonfigurasi beberapa akun bernama, setel `defaultAccount` atau teruskan `--account <id>` untuk perintah CLI yang bergantung pada pemilihan akun implisit.
Teruskan `--account <id>` ke `openclaw matrix verify ...` dan `openclaw matrix devices ...` saat Anda ingin menimpa pemilihan implisit itu untuk satu perintah.

## Homeserver privat/LAN

Secara default, OpenClaw memblokir homeserver Matrix privat/internal untuk perlindungan SSRF kecuali Anda
secara eksplisit ikut serta per akun.

Jika homeserver Anda berjalan di localhost, IP LAN/Tailscale, atau hostname internal, aktifkan
`allowPrivateNetwork` untuk akun Matrix tersebut:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      allowPrivateNetwork: true,
      accessToken: "syt_internal_xxx",
    },
  },
}
```

Contoh penyiapan CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Opt-in ini hanya mengizinkan target privat/internal tepercaya. Homeserver cleartext publik seperti
`http://matrix.example.org:8008` tetap diblokir. Gunakan `https://` bila memungkinkan.

## Mem-proxy lalu lintas Matrix

Jika deployment Matrix Anda memerlukan proxy HTTP(S) keluar eksplisit, setel `channels.matrix.proxy`:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Akun bernama dapat menimpa default tingkat atas dengan `channels.matrix.accounts.<id>.proxy` mereka sendiri.
OpenClaw menggunakan setelan proxy yang sama untuk lalu lintas Matrix runtime dan probe status akun.

## Resolusi target

Matrix menerima bentuk target berikut di mana pun OpenClaw meminta target room atau pengguna:

- Pengguna: `@user:server`, `user:@user:server`, atau `matrix:user:@user:server`
- Room: `!room:server`, `room:!room:server`, atau `matrix:room:!room:server`
- Alias: `#alias:server`, `channel:#alias:server`, atau `matrix:channel:#alias:server`

Pencarian direktori live menggunakan akun Matrix yang sedang login:

- Pencarian pengguna mengkueri direktori pengguna Matrix pada homeserver itu.
- Pencarian room menerima ID room dan alias eksplisit secara langsung, lalu fallback ke pencarian nama room yang diikuti untuk akun tersebut.
- Pencarian nama joined-room bersifat upaya terbaik. Jika nama room tidak dapat diselesaikan ke ID atau alias, nama itu diabaikan oleh resolusi allowlist runtime.

## Referensi konfigurasi

- `enabled`: aktifkan atau nonaktifkan channel.
- `name`: label opsional untuk akun.
- `defaultAccount`: ID akun pilihan saat beberapa akun Matrix dikonfigurasi.
- `homeserver`: URL homeserver, misalnya `https://matrix.example.org`.
- `allowPrivateNetwork`: izinkan akun Matrix ini terhubung ke homeserver privat/internal. Aktifkan ini saat homeserver diselesaikan ke `localhost`, IP LAN/Tailscale, atau host internal seperti `matrix-synapse`.
- `proxy`: URL proxy HTTP(S) opsional untuk lalu lintas Matrix. Akun bernama dapat menimpa default tingkat atas dengan `proxy` mereka sendiri.
- `userId`: ID pengguna Matrix lengkap, misalnya `@bot:example.org`.
- `accessToken`: access token untuk auth berbasis token. Nilai plaintext dan nilai SecretRef didukung untuk `channels.matrix.accessToken` dan `channels.matrix.accounts.<id>.accessToken` di seluruh provider env/file/exec. Lihat [Secrets Management](/id/gateway/secrets).
- `password`: password untuk login berbasis password. Nilai plaintext dan nilai SecretRef didukung.
- `deviceId`: ID perangkat Matrix eksplisit.
- `deviceName`: nama tampilan perangkat untuk login password.
- `avatarUrl`: URL avatar mandiri yang disimpan untuk sinkronisasi profil dan pembaruan `set-profile`.
- `initialSyncLimit`: batas event sinkronisasi saat startup.
- `encryption`: aktifkan E2EE.
- `allowlistOnly`: paksa perilaku allowlist-only untuk DM dan room.
- `allowBots`: izinkan pesan dari akun Matrix OpenClaw lain yang dikonfigurasi (`true` atau `"mentions"`).
- `groupPolicy`: `open`, `allowlist`, atau `disabled`.
- `contextVisibility`: mode visibilitas konteks room tambahan (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: allowlist ID pengguna untuk lalu lintas room.
- Entri `groupAllowFrom` harus berupa ID pengguna Matrix lengkap. Nama yang tidak dapat diselesaikan diabaikan saat runtime.
- `historyLimit`: jumlah maksimum pesan room yang disertakan sebagai konteks riwayat grup. Fallback ke `messages.groupChat.historyLimit`. Setel `0` untuk menonaktifkan.
- `replyToMode`: `off`, `first`, atau `all`.
- `markdown`: konfigurasi rendering Markdown opsional untuk teks Matrix keluar.
- `streaming`: `off` (default), `partial`, `quiet`, `true`, atau `false`. `partial` dan `true` mengaktifkan pembaruan draf pratinjau-dahulu dengan pesan teks Matrix normal. `quiet` menggunakan pemberitahuan pratinjau tanpa notifikasi untuk penyiapan push-rule self-hosted.
- `blockStreaming`: `true` mengaktifkan pesan progres terpisah untuk blok assistant yang sudah selesai saat streaming pratinjau draf aktif.
- `threadReplies`: `off`, `inbound`, atau `always`.
- `threadBindings`: override per channel untuk perutean dan siklus hidup sesi yang dibinding ke thread.
- `startupVerification`: mode permintaan verifikasi mandiri otomatis saat startup (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: cooldown sebelum mencoba lagi permintaan verifikasi startup otomatis.
- `textChunkLimit`: ukuran chunk pesan keluar.
- `chunkMode`: `length` atau `newline`.
- `responsePrefix`: prefiks pesan opsional untuk balasan keluar.
- `ackReaction`: override reaksi ack opsional untuk channel/akun ini.
- `ackReactionScope`: override cakupan reaksi ack opsional (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: mode notifikasi reaksi masuk (`own`, `off`).
- `mediaMaxMb`: batas ukuran media dalam MB untuk penanganan media Matrix. Ini berlaku untuk pengiriman keluar dan pemrosesan media masuk.
- `autoJoin`: kebijakan auto-join undangan (`always`, `allowlist`, `off`). Default: `off`.
- `autoJoinAllowlist`: room/alias yang diizinkan saat `autoJoin` adalah `allowlist`. Entri alias diselesaikan ke ID room selama penanganan undangan; OpenClaw tidak mempercayai state alias yang diklaim oleh room yang mengundang.
- `dm`: blok kebijakan DM (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- Entri `dm.allowFrom` harus berupa ID pengguna Matrix lengkap kecuali Anda sudah menyelesaikannya melalui pencarian direktori live.
- `dm.sessionScope`: `per-user` (default) atau `per-room`. Gunakan `per-room` saat Anda ingin setiap room DM Matrix mempertahankan konteks terpisah meskipun rekannya sama.
- `dm.threadReplies`: override kebijakan thread khusus DM (`off`, `inbound`, `always`). Ini menimpa setelan `threadReplies` tingkat atas untuk penempatan balasan dan isolasi sesi di DM.
- `execApprovals`: pengiriman persetujuan exec bawaan Matrix (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: ID pengguna Matrix yang diizinkan menyetujui permintaan exec. Opsional saat `dm.allowFrom` sudah mengidentifikasi para approver.
- `execApprovals.target`: `dm | channel | both` (default: `dm`).
- `accounts`: override per akun bernama. Nilai `channels.matrix` tingkat atas bertindak sebagai default untuk entri ini.
- `groups`: peta kebijakan per room. Gunakan ID room atau alias; nama room yang tidak dapat diselesaikan diabaikan saat runtime. Identitas sesi/grup menggunakan ID room stabil setelah resolusi, sementara label yang mudah dibaca manusia tetap berasal dari nama room.
- `groups.<room>.account`: batasi satu entri room turunan ke akun Matrix tertentu dalam penyiapan multi-akun.
- `groups.<room>.allowBots`: override tingkat room untuk pengirim bot yang dikonfigurasi (`true` atau `"mentions"`).
- `groups.<room>.users`: allowlist pengirim per room.
- `groups.<room>.tools`: override izin/tolak tool per room.
- `groups.<room>.autoReply`: override mention-gating tingkat room. `true` menonaktifkan persyaratan mention untuk room itu; `false` memaksanya aktif kembali.
- `groups.<room>.skills`: filter skill tingkat room opsional.
- `groups.<room>.systemPrompt`: potongan system prompt tingkat room opsional.
- `rooms`: alias lama untuk `groups`.
- `actions`: pengendalian tool per aksi (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Terkait

- [Channels Overview](/id/channels) — semua channel yang didukung
- [Pairing](/id/channels/pairing) — auth DM dan alur pairing
- [Groups](/id/channels/groups) — perilaku chat grup dan mention gating
- [Channel Routing](/id/channels/channel-routing) — perutean sesi untuk pesan
- [Security](/id/gateway/security) — model akses dan hardening
