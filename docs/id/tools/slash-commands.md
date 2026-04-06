---
read_when:
    - Menggunakan atau mengonfigurasi perintah chat
    - Men-debug perutean atau izin perintah
summary: 'Slash command: teks vs native, config, dan perintah yang didukung'
title: Slash Commands
x-i18n:
    generated_at: "2026-04-06T03:13:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: 417e35b9ddd87f25f6c019111b55b741046ea11039dde89210948185ced5696d
    source_path: tools/slash-commands.md
    workflow: 15
---

# Slash command

Perintah ditangani oleh Gateway. Sebagian besar perintah harus dikirim sebagai pesan **mandiri** yang diawali dengan `/`.
Perintah chat bash khusus host menggunakan `! <cmd>` (dengan `/bash <cmd>` sebagai alias).

Ada dua sistem yang terkait:

- **Perintah**: pesan mandiri `/...`.
- **Directive**: `/think`, `/fast`, `/verbose`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`.
  - Directive dihapus dari pesan sebelum model melihatnya.
  - Dalam pesan chat normal (bukan hanya directive), directive diperlakukan sebagai “petunjuk inline” dan **tidak** menyimpan setelan sesi.
  - Dalam pesan yang hanya berisi directive (pesan hanya berisi directive), directive disimpan ke sesi dan membalas dengan pengakuan.
  - Directive hanya diterapkan untuk **pengirim yang diotorisasi**. Jika `commands.allowFrom` disetel, itu adalah satu-satunya
    allowlist yang digunakan; jika tidak, otorisasi berasal dari allowlist/pairing channel plus `commands.useAccessGroups`.
    Pengirim yang tidak diotorisasi akan melihat directive diperlakukan sebagai teks biasa.

Ada juga beberapa **pintasan inline** (hanya untuk pengirim yang di-allowlist/diotorisasi): `/help`, `/commands`, `/status`, `/whoami` (`/id`).
Pintasan ini langsung dijalankan, dihapus sebelum model melihat pesan, dan teks yang tersisa melanjutkan alur normal.

## Config

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: false,
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text` (default `true`) mengaktifkan parsing `/...` dalam pesan chat.
  - Pada permukaan tanpa native command (WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams), perintah teks tetap berfungsi bahkan jika Anda menyetelnya ke `false`.
- `commands.native` (default `"auto"`) mendaftarkan native command.
  - Auto: aktif untuk Discord/Telegram; nonaktif untuk Slack (sampai Anda menambahkan slash command); diabaikan untuk provider tanpa dukungan native.
  - Setel `channels.discord.commands.native`, `channels.telegram.commands.native`, atau `channels.slack.commands.native` untuk override per provider (bool atau `"auto"`).
  - `false` menghapus perintah yang sebelumnya terdaftar di Discord/Telegram saat startup. Perintah Slack dikelola di app Slack dan tidak dihapus secara otomatis.
- `commands.nativeSkills` (default `"auto"`) mendaftarkan perintah **skill** secara native saat didukung.
  - Auto: aktif untuk Discord/Telegram; nonaktif untuk Slack (Slack mengharuskan pembuatan satu slash command per skill).
  - Setel `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills`, atau `channels.slack.commands.nativeSkills` untuk override per provider (bool atau `"auto"`).
- `commands.bash` (default `false`) mengaktifkan `! <cmd>` untuk menjalankan perintah shell host (`/bash <cmd>` adalah alias; memerlukan allowlist `tools.elevated`).
- `commands.bashForegroundMs` (default `2000`) mengontrol berapa lama bash menunggu sebelum beralih ke mode background (`0` langsung ke background).
- `commands.config` (default `false`) mengaktifkan `/config` (membaca/menulis `openclaw.json`).
- `commands.mcp` (default `false`) mengaktifkan `/mcp` (membaca/menulis config MCP yang dikelola OpenClaw di bawah `mcp.servers`).
- `commands.plugins` (default `false`) mengaktifkan `/plugins` (penemuan/status plugin plus kontrol install + enable/disable).
- `commands.debug` (default `false`) mengaktifkan `/debug` (override khusus runtime).
- `commands.allowFrom` (opsional) menetapkan allowlist per provider untuk otorisasi perintah. Saat dikonfigurasi, ini menjadi
  satu-satunya sumber otorisasi untuk perintah dan directive (`commands.useAccessGroups`
  dan allowlist/pairing channel diabaikan). Gunakan `"*"` untuk default global; kunci khusus provider menimpanya.
- `commands.useAccessGroups` (default `true`) menegakkan allowlist/kebijakan untuk perintah saat `commands.allowFrom` tidak disetel.

## Daftar perintah

Teks + native (saat diaktifkan):

- `/help`
- `/commands`
- `/tools [compact|verbose]` (tampilkan apa yang dapat digunakan agent saat ini sekarang juga; `verbose` menambahkan deskripsi)
- `/skill <name> [input]` (jalankan skill berdasarkan nama)
- `/status` (tampilkan status saat ini; mencakup penggunaan/kuota provider untuk provider model saat ini bila tersedia)
- `/tasks` (daftarkan background task untuk sesi saat ini; menampilkan detail task aktif dan terbaru dengan hitungan fallback lokal agent)
- `/allowlist` (daftarkan/tambahkan/hapus entri allowlist)
- `/approve <id> <decision>` (selesaikan prompt persetujuan exec; gunakan pesan persetujuan tertunda untuk keputusan yang tersedia)
- `/context [list|detail|json]` (jelaskan “context”; `detail` menampilkan ukuran per-file + per-tool + per-skill + system prompt)
- `/btw <question>` (ajukan pertanyaan sampingan sementara tentang sesi saat ini tanpa mengubah konteks sesi di masa depan; lihat [/tools/btw](/id/tools/btw))
- `/export-session [path]` (alias: `/export`) (ekspor sesi saat ini ke HTML dengan system prompt lengkap)
- `/whoami` (tampilkan id pengirim Anda; alias: `/id`)
- `/session idle <duration|off>` (kelola auto-unfocus karena tidak aktif untuk thread binding yang difokuskan)
- `/session max-age <duration|off>` (kelola auto-unfocus hard max-age untuk thread binding yang difokuskan)
- `/subagents list|kill|log|info|send|steer|spawn` (inspeksi, kontrol, atau spawn run sub-agent untuk sesi saat ini)
- `/acp spawn|cancel|steer|close|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|sessions` (inspeksi dan kontrol sesi runtime ACP)
- `/agents` (daftar agent yang dibinding ke thread untuk sesi ini)
- `/focus <target>` (Discord: bind thread ini, atau thread baru, ke target sesi/subagent)
- `/unfocus` (Discord: hapus thread binding saat ini)
- `/kill <id|#|all>` (segera hentikan satu atau semua sub-agent yang sedang berjalan untuk sesi ini; tanpa pesan konfirmasi)
- `/steer <id|#> <message>` (arahkan sub-agent yang sedang berjalan segera: di dalam run bila memungkinkan, jika tidak hentikan pekerjaan saat ini dan mulai ulang dengan pesan pengarahan)
- `/tell <id|#> <message>` (alias untuk `/steer`)
- `/config show|get|set|unset` (persistenkan config ke disk, hanya pemilik; memerlukan `commands.config: true`)
- `/mcp show|get|set|unset` (kelola config server MCP OpenClaw, hanya pemilik; memerlukan `commands.mcp: true`)
- `/plugins list|show|get|install|enable|disable` (inspeksi plugin yang ditemukan, instal plugin baru, dan ubah status enable; hanya pemilik untuk penulisan; memerlukan `commands.plugins: true`)
  - `/plugin` adalah alias untuk `/plugins`.
  - `/plugin install <spec>` menerima spesifikasi plugin yang sama seperti `openclaw plugins install`: path/arsip lokal, package npm, atau `clawhub:<pkg>`.
  - Penulisan enable/disable tetap membalas dengan petunjuk restart. Pada gateway foreground yang diawasi, OpenClaw mungkin melakukan restart itu secara otomatis tepat setelah penulisan.
- `/debug show|set|unset|reset` (override runtime, hanya pemilik; memerlukan `commands.debug: true`)
- `/usage off|tokens|full|cost` (footer penggunaan per respons atau ringkasan biaya lokal)
- `/tts off|always|inbound|tagged|status|provider|limit|summary|audio` (kontrol TTS; lihat [/tts](/id/tools/tts))
  - Discord: native command adalah `/voice` (Discord mencadangkan `/tts`); teks `/tts` tetap berfungsi.
- `/stop`
- `/restart`
- `/dock-telegram` (alias: `/dock_telegram`) (alihkah balasan ke Telegram)
- `/dock-discord` (alias: `/dock_discord`) (alihkah balasan ke Discord)
- `/dock-slack` (alias: `/dock_slack`) (alihkah balasan ke Slack)
- `/activation mention|always` (khusus grup)
- `/send on|off|inherit` (hanya pemilik)
- `/reset` atau `/new [model]` (petunjuk model opsional; sisa pesan diteruskan)
- `/think <off|minimal|low|medium|high|xhigh>` (pilihan dinamis menurut model/provider; alias: `/thinking`, `/t`)
- `/fast status|on|off` (menghilangkan argumen akan menampilkan status mode cepat efektif saat ini)
- `/verbose on|full|off` (alias: `/v`)
- `/reasoning on|off|stream` (alias: `/reason`; saat aktif, mengirim pesan terpisah dengan prefiks `Reasoning:`; `stream` = hanya draf Telegram)
- `/elevated on|off|ask|full` (alias: `/elev`; `full` melewati persetujuan exec)
- `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` (kirim `/exec` untuk menampilkan status saat ini)
- `/model <name>` (alias: `/models`; atau `/<alias>` dari `agents.defaults.models.*.alias`)
- `/queue <mode>` (ditambah opsi seperti `debounce:2s cap:25 drop:summarize`; kirim `/queue` untuk melihat setelan saat ini)
- `/bash <command>` (khusus host; alias untuk `! <command>`; memerlukan `commands.bash: true` + allowlist `tools.elevated`)
- `/dreaming [on|off|status|help]` (aktifkan/nonaktifkan dreaming global atau tampilkan status; lihat [Dreaming](/concepts/dreaming))

Hanya teks:

- `/compact [instructions]` (lihat [/concepts/compaction](/id/concepts/compaction))
- `! <command>` (khusus host; satu per satu; gunakan `!poll` + `!stop` untuk pekerjaan berjalan lama)
- `!poll` (periksa output / status; menerima `sessionId` opsional; `/bash poll` juga berfungsi)
- `!stop` (hentikan pekerjaan bash yang sedang berjalan; menerima `sessionId` opsional; `/bash stop` juga berfungsi)

Catatan:

- Perintah menerima `:` opsional antara perintah dan argumen (misalnya `/think: high`, `/send: on`, `/help:`).
- `/new <model>` menerima alias model, `provider/model`, atau nama provider (fuzzy match); jika tidak ada kecocokan, teks diperlakukan sebagai body pesan.
- Untuk rincian lengkap penggunaan provider, gunakan `openclaw status --usage`.
- `/allowlist add|remove` memerlukan `commands.config=true` dan menghormati `configWrites` channel.
- Dalam channel multi-akun, `/allowlist --account <id>` yang menargetkan config dan `/config set channels.<provider>.accounts.<id>...` juga menghormati `configWrites` akun target.
- `/usage` mengontrol footer penggunaan per respons; `/usage cost` mencetak ringkasan biaya lokal dari log sesi OpenClaw.
- `/restart` aktif secara default; setel `commands.restart: false` untuk menonaktifkannya.
- Native command khusus Discord: `/vc join|leave|status` mengontrol voice channel (memerlukan `channels.discord.voice` dan native command; tidak tersedia sebagai teks).
- Perintah Discord thread-binding (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`) memerlukan thread binding efektif diaktifkan (`session.threadBindings.enabled` dan/atau `channels.discord.threadBindings.enabled`).
- Referensi perintah ACP dan perilaku runtime: [ACP Agents](/id/tools/acp-agents).
- `/verbose` ditujukan untuk debugging dan visibilitas tambahan; biarkan **nonaktif** dalam penggunaan normal.
- `/fast on|off` menyimpan override sesi. Gunakan opsi `inherit` di UI Sessions untuk menghapusnya dan fallback ke default config.
- `/fast` bersifat khusus provider: OpenAI/OpenAI Codex memetakannya ke `service_tier=priority` pada endpoint Responses native, sedangkan permintaan Anthropic publik langsung, termasuk lalu lintas terautentikasi OAuth yang dikirim ke `api.anthropic.com`, memetakannya ke `service_tier=auto` atau `standard_only`. Lihat [OpenAI](/id/providers/openai) dan [Anthropic](/id/providers/anthropic).
- Ringkasan kegagalan tool tetap ditampilkan saat relevan, tetapi teks kegagalan rinci hanya disertakan saat `/verbose` bernilai `on` atau `full`.
- `/reasoning` (dan `/verbose`) berisiko dalam pengaturan grup: keduanya dapat mengungkap reasoning internal atau output tool yang tidak Anda maksudkan untuk ditampilkan. Sebaiknya biarkan nonaktif, terutama di chat grup.
- `/model` segera menyimpan model sesi baru.
- Jika agent sedang idle, run berikutnya langsung menggunakannya.
- Jika run sudah aktif, OpenClaw menandai live switch sebagai tertunda dan hanya memulai ulang ke model baru pada titik retry yang bersih.
- Jika aktivitas tool atau output balasan sudah dimulai, live switch tertunda dapat tetap antre sampai kesempatan retry berikutnya atau giliran pengguna berikutnya.
- **Jalur cepat:** pesan khusus perintah dari pengirim yang di-allowlist ditangani segera (melewati antrean + model).
- **Mention gating grup:** pesan khusus perintah dari pengirim yang di-allowlist melewati persyaratan mention.
- **Pintasan inline (hanya pengirim yang di-allowlist):** beberapa perintah juga berfungsi saat disematkan dalam pesan normal dan dihapus sebelum model melihat teks yang tersisa.
  - Contoh: `hey /status` memicu balasan status, dan teks yang tersisa melanjutkan alur normal.
- Saat ini: `/help`, `/commands`, `/status`, `/whoami` (`/id`).
- Pesan khusus perintah dari pengirim yang tidak diotorisasi diabaikan secara senyap, dan token inline `/...` diperlakukan sebagai teks biasa.
- **Perintah skill:** skill `user-invocable` diekspos sebagai slash command. Nama dibersihkan menjadi `a-z0-9_` (maks 32 karakter); bentrokan akan diberi sufiks numerik (misalnya `_2`).
  - `/skill <name> [input]` menjalankan skill berdasarkan nama (berguna saat batas native command mencegah perintah per skill).
  - Secara default, perintah skill diteruskan ke model sebagai permintaan normal.
  - Skill secara opsional dapat mendeklarasikan `command-dispatch: tool` untuk merutekan perintah langsung ke tool (deterministik, tanpa model).
  - Contoh: `/prose` (plugin OpenProse) — lihat [OpenProse](/id/prose).
- **Argumen native command:** Discord menggunakan autocomplete untuk opsi dinamis (dan menu tombol saat Anda menghilangkan argumen wajib). Telegram dan Slack menampilkan menu tombol saat sebuah perintah mendukung pilihan dan Anda menghilangkan argumen.

## `/tools`

`/tools` menjawab pertanyaan runtime, bukan pertanyaan config: **apa yang dapat digunakan agent ini saat ini dalam
percakapan ini**.

- Default `/tools` ringkas dan dioptimalkan untuk pemindaian cepat.
- `/tools verbose` menambahkan deskripsi singkat.
- Permukaan native command yang mendukung argumen mengekspos sakelar mode yang sama seperti `compact|verbose`.
- Hasil dibatasi ke sesi, sehingga mengubah agent, channel, thread, otorisasi pengirim, atau model dapat
  mengubah output.
- `/tools` mencakup tool yang benar-benar dapat dijangkau saat runtime, termasuk tool inti, tool plugin
  yang terhubung, dan tool milik channel.

Untuk pengeditan profil dan override, gunakan panel Tools di Control UI atau permukaan config/katalog
alih-alih memperlakukan `/tools` sebagai katalog statis.

## Permukaan penggunaan (apa yang tampil di mana)

- **Penggunaan/kuota provider** (contoh: “Claude 80% tersisa”) muncul di `/status` untuk provider model saat ini ketika pelacakan penggunaan diaktifkan. OpenClaw menormalkan jendela provider menjadi `% tersisa`; untuk MiniMax, field persentase remaining-only dibalik sebelum ditampilkan, dan respons `model_remains` mengutamakan entri chat-model ditambah label paket bertag model.
- **Baris token/cache** di `/status` dapat fallback ke entri penggunaan transkrip terbaru saat snapshot sesi live jarang. Nilai live bukan nol yang ada tetap diutamakan, dan fallback transkrip juga dapat memulihkan label model runtime aktif plus total yang lebih besar berorientasi prompt saat total tersimpan hilang atau lebih kecil.
- **Token/biaya per respons** dikendalikan oleh `/usage off|tokens|full` (ditambahkan ke balasan normal).
- `/model status` adalah tentang **model/auth/endpoint**, bukan penggunaan.

## Pemilihan model (`/model`)

`/model` diimplementasikan sebagai directive.

Contoh:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

Catatan:

- `/model` dan `/model list` menampilkan pemilih ringkas bernomor (keluarga model + provider yang tersedia).
- Di Discord, `/model` dan `/models` membuka pemilih interaktif dengan dropdown provider dan model plus langkah Submit.
- `/model <#>` memilih dari pemilih tersebut (dan lebih memilih provider saat ini jika memungkinkan).
- `/model status` menampilkan tampilan detail, termasuk endpoint provider yang dikonfigurasi (`baseUrl`) dan mode API (`api`) bila tersedia.

## Override debug

`/debug` memungkinkan Anda menetapkan override config **khusus runtime** (memori, bukan disk). Hanya pemilik. Nonaktif secara default; aktifkan dengan `commands.debug: true`.

Contoh:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

Catatan:

- Override segera berlaku untuk pembacaan config baru, tetapi **tidak** menulis ke `openclaw.json`.
- Gunakan `/debug reset` untuk menghapus semua override dan kembali ke config di disk.

## Pembaruan config

`/config` menulis ke config di disk Anda (`openclaw.json`). Hanya pemilik. Nonaktif secara default; aktifkan dengan `commands.config: true`.

Contoh:

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

Catatan:

- Config divalidasi sebelum penulisan; perubahan yang tidak valid ditolak.
- Pembaruan `/config` tetap bertahan setelah restart.

## Pembaruan MCP

`/mcp` menulis definisi server MCP yang dikelola OpenClaw di bawah `mcp.servers`. Hanya pemilik. Nonaktif secara default; aktifkan dengan `commands.mcp: true`.

Contoh:

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

Catatan:

- `/mcp` menyimpan config di config OpenClaw, bukan setelan proyek milik Pi.
- Adapter runtime memutuskan transport mana yang benar-benar dapat dieksekusi.

## Pembaruan plugin

`/plugins` memungkinkan operator menginspeksi plugin yang ditemukan dan mengubah status enable di config. Alur read-only dapat menggunakan `/plugin` sebagai alias. Nonaktif secara default; aktifkan dengan `commands.plugins: true`.

Contoh:

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

Catatan:

- `/plugins list` dan `/plugins show` menggunakan penemuan plugin nyata terhadap workspace saat ini plus config di disk.
- `/plugins enable|disable` hanya memperbarui config plugin; perintah ini tidak menginstal atau menghapus plugin.
- Setelah perubahan enable/disable, mulai ulang gateway untuk menerapkannya.

## Catatan permukaan

- **Perintah teks** berjalan di sesi chat normal (DM berbagi `main`, grup memiliki sesi sendiri).
- **Native command** menggunakan sesi terisolasi:
  - Discord: `agent:<agentId>:discord:slash:<userId>`
  - Slack: `agent:<agentId>:slack:slash:<userId>` (prefiks dapat dikonfigurasi melalui `channels.slack.slashCommand.sessionPrefix`)
  - Telegram: `telegram:slash:<userId>` (menargetkan sesi chat melalui `CommandTargetSessionKey`)
- **`/stop`** menargetkan sesi chat aktif sehingga dapat menghentikan run saat ini.
- **Slack:** `channels.slack.slashCommand` masih didukung untuk satu perintah gaya `/openclaw`. Jika Anda mengaktifkan `commands.native`, Anda harus membuat satu slash command Slack per perintah bawaan (nama sama seperti `/help`). Menu argumen perintah untuk Slack dikirim sebagai tombol Block Kit ephemeral.
  - Pengecualian native Slack: daftarkan `/agentstatus` (bukan `/status`) karena Slack mencadangkan `/status`. Teks `/status` tetap berfungsi dalam pesan Slack.

## Pertanyaan sampingan BTW

`/btw` adalah **pertanyaan sampingan** singkat tentang sesi saat ini.

Tidak seperti chat normal:

- perintah ini menggunakan sesi saat ini sebagai konteks latar,
- berjalan sebagai panggilan one-shot **tanpa tool** yang terpisah,
- tidak mengubah konteks sesi di masa depan,
- tidak ditulis ke riwayat transkrip,
- dikirim sebagai hasil sampingan live alih-alih sebagai pesan assistant normal.

Ini membuat `/btw` berguna saat Anda menginginkan klarifikasi sementara sementara
tugas utama tetap berjalan.

Contoh:

```text
/btw apa yang sedang kita lakukan sekarang?
```

Lihat [BTW Side Questions](/id/tools/btw) untuk perilaku lengkap dan detail UX
klien.
