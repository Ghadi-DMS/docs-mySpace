---
read_when:
    - Menambahkan atau memodifikasi migrasi doctor
    - Memperkenalkan perubahan config yang breaking
summary: 'Perintah Doctor: pemeriksaan kesehatan, migrasi config, dan langkah perbaikan'
title: Doctor
x-i18n:
    generated_at: "2026-04-06T03:07:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6c0a15c522994552a1eef39206bed71fc5bf45746776372f24f31c101bfbd411
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` adalah tool perbaikan + migrasi untuk OpenClaw. Tool ini memperbaiki
config/state yang usang, memeriksa kesehatan, dan memberikan langkah perbaikan yang dapat ditindaklanjuti.

## Mulai cepat

```bash
openclaw doctor
```

### Headless / otomatisasi

```bash
openclaw doctor --yes
```

Terima default tanpa prompt (termasuk langkah perbaikan restart/service/sandbox jika berlaku).

```bash
openclaw doctor --repair
```

Terapkan perbaikan yang direkomendasikan tanpa prompt (perbaikan + restart jika aman).

```bash
openclaw doctor --repair --force
```

Terapkan juga perbaikan agresif (menimpa config supervisor kustom).

```bash
openclaw doctor --non-interactive
```

Jalankan tanpa prompt dan hanya terapkan migrasi yang aman (normalisasi config + pemindahan state di disk). Melewati tindakan restart/service/sandbox yang memerlukan konfirmasi manusia.
Migrasi state legacy berjalan otomatis saat terdeteksi.

```bash
openclaw doctor --deep
```

Pindai service sistem untuk instalasi gateway tambahan (launchd/systemd/schtasks).

Jika Anda ingin meninjau perubahan sebelum menulis, buka file config terlebih dahulu:

```bash
cat ~/.openclaw/openclaw.json
```

## Apa yang dilakukan (ringkasan)

- Pembaruan pre-flight opsional untuk instalasi git (hanya interaktif).
- Pemeriksaan kesegaran protokol UI (membangun ulang Control UI saat skema protokol lebih baru).
- Pemeriksaan kesehatan + prompt restart.
- Ringkasan status Skills (eligible/missing/blocked) dan status plugin.
- Normalisasi config untuk nilai legacy.
- Migrasi config Talk dari field datar legacy `talk.*` ke `talk.provider` + `talk.providers.<provider>`.
- Pemeriksaan migrasi browser untuk config extension Chrome legacy dan kesiapan Chrome MCP.
- Peringatan override provider OpenCode (`models.providers.opencode` / `models.providers.opencode-go`).
- Pemeriksaan prasyarat OAuth TLS untuk profil OAuth OpenAI Codex.
- Migrasi state legacy di disk (sessions/dir agen/auth WhatsApp).
- Migrasi key kontrak manifes plugin legacy (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Migrasi cron store legacy (`jobId`, `schedule.cron`, field delivery/payload tingkat atas, payload `provider`, pekerjaan fallback webhook sederhana `notify: true`).
- Inspeksi file lock sesi dan pembersihan lock usang.
- Pemeriksaan integritas state dan izin (sessions, transcripts, state dir).
- Pemeriksaan izin file config (chmod 600) saat berjalan secara lokal.
- Kesehatan auth model: memeriksa kedaluwarsa OAuth, dapat me-refresh token yang akan kedaluwarsa, dan melaporkan status cooldown/nonaktif profil auth.
- Deteksi workspace dir tambahan (`~/openclaw`).
- Perbaikan image sandbox saat sandboxing diaktifkan.
- Migrasi service legacy dan deteksi gateway tambahan.
- Migrasi state legacy channel Matrix (dalam mode `--fix` / `--repair`).
- Pemeriksaan runtime gateway (service terinstal tetapi tidak berjalan; label launchd yang di-cache).
- Peringatan status channel (diprobe dari gateway yang sedang berjalan).
- Audit config supervisor (launchd/systemd/schtasks) dengan perbaikan opsional.
- Pemeriksaan best practice runtime gateway (Node vs Bun, path version manager).
- Diagnostik benturan port gateway (default `18789`).
- Peringatan keamanan untuk kebijakan DM terbuka.
- Pemeriksaan auth gateway untuk mode token lokal (menawarkan pembuatan token saat tidak ada sumber token; tidak menimpa config SecretRef token).
- Pemeriksaan linger systemd di Linux.
- Pemeriksaan ukuran file bootstrap workspace (peringatan truncation/hampir batas untuk file konteks).
- Pemeriksaan status shell completion dan instalasi/upgrade otomatis.
- Pemeriksaan kesiapan provider embedding memory search (model lokal, API key remote, atau biner QMD).
- Pemeriksaan instalasi source (ketidakcocokan workspace pnpm, aset UI yang hilang, biner tsx yang hilang).
- Menulis config yang diperbarui + metadata wizard.

## Perilaku dan alasan terperinci

### 0) Pembaruan opsional (instalasi git)

Jika ini adalah checkout git dan doctor berjalan secara interaktif, doctor akan menawarkan untuk
memperbarui (fetch/rebase/build) sebelum menjalankan doctor.

### 1) Normalisasi config

Jika config berisi bentuk nilai legacy (misalnya `messages.ackReaction`
tanpa override khusus channel), doctor akan menormalkannya ke dalam
skema saat ini.

Itu mencakup field datar Talk legacy. Config Talk publik saat ini adalah
`talk.provider` + `talk.providers.<provider>`. Doctor menulis ulang bentuk lama
`talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` ke dalam peta provider.

### 2) Migrasi key config legacy

Saat config berisi key yang sudah deprecated, perintah lain menolak berjalan dan meminta
Anda menjalankan `openclaw doctor`.

Doctor akan:

- Menjelaskan key legacy mana yang ditemukan.
- Menampilkan migrasi yang diterapkannya.
- Menulis ulang `~/.openclaw/openclaw.json` dengan skema yang diperbarui.

Gateway juga otomatis menjalankan migrasi doctor saat startup ketika mendeteksi
format config legacy, sehingga config usang diperbaiki tanpa intervensi manual.
Migrasi cron job store ditangani oleh `openclaw doctor --fix`.

Migrasi saat ini:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` tingkat atas
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- legacy `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- Untuk channel dengan `accounts` bernama tetapi masih memiliki nilai channel tingkat atas akun-tunggal, pindahkan nilai berlingkup akun tersebut ke akun yang dipromosikan yang dipilih untuk channel itu (`accounts.default` untuk sebagian besar channel; Matrix dapat mempertahankan target bernama/default yang cocok yang sudah ada)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- hapus `browser.relayBindHost` (pengaturan relay extension legacy)

Peringatan doctor juga mencakup panduan default akun untuk channel multi-akun:

- Jika dua atau lebih entri `channels.<channel>.accounts` dikonfigurasi tanpa `channels.<channel>.defaultAccount` atau `accounts.default`, doctor memperingatkan bahwa routing fallback dapat memilih akun yang tidak diharapkan.
- Jika `channels.<channel>.defaultAccount` disetel ke ID akun yang tidak dikenal, doctor memperingatkan dan mencantumkan ID akun yang dikonfigurasi.

### 2b) Override provider OpenCode

Jika Anda menambahkan `models.providers.opencode`, `opencode-zen`, atau `opencode-go`
secara manual, itu akan menimpa katalog OpenCode bawaan dari `@mariozechner/pi-ai`.
Hal itu dapat memaksa model ke API yang salah atau membuat biaya menjadi nol. Doctor memperingatkan agar Anda
dapat menghapus override tersebut dan memulihkan routing API + biaya per model.

### 2c) Migrasi browser dan kesiapan Chrome MCP

Jika config browser Anda masih menunjuk ke path extension Chrome yang telah dihapus, doctor
menormalkannya ke model attach Chrome MCP host-lokal saat ini:

- `browser.profiles.*.driver: "extension"` menjadi `"existing-session"`
- `browser.relayBindHost` dihapus

Doctor juga mengaudit path Chrome MCP host-lokal saat Anda menggunakan `defaultProfile:
"user"` atau profil `existing-session` yang dikonfigurasi:

- memeriksa apakah Google Chrome terinstal pada host yang sama untuk profil
  auto-connect default
- memeriksa versi Chrome yang terdeteksi dan memperingatkan saat versinya di bawah Chrome 144
- mengingatkan Anda untuk mengaktifkan remote debugging di halaman inspect browser (misalnya
  `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  atau `edge://inspect/#remote-debugging`)

Doctor tidak dapat mengaktifkan pengaturan sisi Chrome untuk Anda. Chrome MCP host-lokal
tetap memerlukan:

- browser berbasis Chromium 144+ di host gateway/node
- browser berjalan secara lokal
- remote debugging diaktifkan di browser tersebut
- menyetujui prompt persetujuan attach pertama di browser

Kesiapan di sini hanya terkait prasyarat attach lokal. Existing-session mempertahankan
batas route Chrome MCP saat ini; route lanjutan seperti `responsebody`, ekspor PDF,
intersepsi unduhan, dan aksi batch tetap memerlukan browser terkelola
atau profil CDP mentah.

Pemeriksaan ini **tidak** berlaku untuk Docker, sandbox, remote-browser, atau alur
headless lainnya. Semua itu tetap menggunakan CDP mentah.

### 2d) Prasyarat OAuth TLS

Saat profil OAuth OpenAI Codex dikonfigurasi, doctor mem-probe endpoint otorisasi OpenAI
untuk memverifikasi bahwa stack TLS Node/OpenSSL lokal dapat
memvalidasi rantai sertifikat. Jika probe gagal dengan error sertifikat (misalnya
`UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, sertifikat kedaluwarsa, atau sertifikat self-signed),
doctor mencetak panduan perbaikan khusus platform. Di macOS dengan Node Homebrew, perbaikannya
biasanya adalah `brew postinstall ca-certificates`. Dengan `--deep`, probe dijalankan
bahkan jika gateway sehat.

### 3) Migrasi state legacy (layout disk)

Doctor dapat memigrasikan layout di disk yang lebih lama ke struktur saat ini:

- Store sessions + transcripts:
  - dari `~/.openclaw/sessions/` ke `~/.openclaw/agents/<agentId>/sessions/`
- Dir agen:
  - dari `~/.openclaw/agent/` ke `~/.openclaw/agents/<agentId>/agent/`
- State auth WhatsApp (Baileys):
  - dari legacy `~/.openclaw/credentials/*.json` (kecuali `oauth.json`)
  - ke `~/.openclaw/credentials/whatsapp/<accountId>/...` (ID akun default: `default`)

Migrasi ini bersifat best-effort dan idempoten; doctor akan mengeluarkan peringatan saat
masih meninggalkan folder legacy sebagai cadangan. Gateway/CLI juga otomatis memigrasikan
sessions legacy + dir agen saat startup sehingga history/auth/models masuk ke
path per-agent tanpa perlu menjalankan doctor secara manual. Auth WhatsApp sengaja hanya
dimigrasikan melalui `openclaw doctor`. Normalisasi provider/peta provider Talk kini
membandingkan berdasarkan kesetaraan struktural, sehingga perbedaan urutan key saja tidak lagi memicu
perubahan no-op `doctor --fix` berulang.

### 3a) Migrasi manifes plugin legacy

Doctor memindai semua manifes plugin yang terinstal untuk key capability tingkat atas
yang deprecated (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). Saat ditemukan, doctor menawarkan untuk memindahkannya ke objek `contracts`
dan menulis ulang file manifes di tempat. Migrasi ini idempoten;
jika key `contracts` sudah memiliki nilai yang sama, key legacy akan dihapus
tanpa menduplikasi data.

### 3b) Migrasi cron store legacy

Doctor juga memeriksa cron job store (`~/.openclaw/cron/jobs.json` secara default,
atau `cron.store` saat dioverride) untuk bentuk job lama yang masih
diterima scheduler demi kompatibilitas.

Pembersihan cron saat ini mencakup:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- field payload tingkat atas (`message`, `model`, `thinking`, ...) → `payload`
- field delivery tingkat atas (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- alias delivery `provider` payload → `delivery.channel` eksplisit
- job fallback webhook legacy sederhana `notify: true` → `delivery.mode="webhook"` eksplisit dengan `delivery.to=cron.webhook`

Doctor hanya memigrasikan job `notify: true` secara otomatis bila dapat dilakukan tanpa
mengubah perilaku. Jika sebuah job menggabungkan fallback notify legacy dengan
mode delivery non-webhook yang sudah ada, doctor akan memperingatkan dan membiarkan job itu untuk tinjauan manual.

### 3c) Pembersihan lock sesi

Doctor memindai setiap direktori sesi agen untuk file write-lock usang — file yang tertinggal
saat sesi berakhir secara abnormal. Untuk setiap file lock yang ditemukan, doctor melaporkan:
path, PID, apakah PID masih hidup, usia lock, dan apakah lock tersebut
dianggap usang (PID mati atau lebih tua dari 30 menit). Dalam mode `--fix` / `--repair`
doctor menghapus file lock usang secara otomatis; jika tidak, doctor akan mencetak catatan dan
menginstruksikan Anda untuk menjalankan ulang dengan `--fix`.

### 4) Pemeriksaan integritas state (persistensi sesi, routing, dan keamanan)

Direktori state adalah batang otak operasional. Jika direktori ini hilang, Anda kehilangan
sesi, kredensial, log, dan config (kecuali Anda memiliki cadangan di tempat lain).

Doctor memeriksa:

- **State dir hilang**: memperingatkan tentang kehilangan state yang katastrofik, menawarkan untuk membuat ulang
  direktori, dan mengingatkan bahwa doctor tidak dapat memulihkan data yang hilang.
- **Izin state dir**: memverifikasi dapat ditulis; menawarkan untuk memperbaiki izin
  (dan mengeluarkan petunjuk `chown` saat terdeteksi ketidakcocokan owner/group).
- **State dir macOS yang tersinkron cloud**: memperingatkan saat state di-resolve di bawah iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) atau
  `~/Library/CloudStorage/...` karena path berbasis sinkronisasi dapat menyebabkan I/O lebih lambat
  serta race lock/sinkronisasi.
- **State dir Linux di SD atau eMMC**: memperingatkan saat state di-resolve ke sumber mount `mmcblk*`,
  karena I/O acak berbasis SD atau eMMC dapat lebih lambat dan lebih cepat aus
  di bawah penulisan sesi dan kredensial.
- **Dir sesi hilang**: `sessions/` dan direktori session store diperlukan untuk
  mempertahankan history dan menghindari crash `ENOENT`.
- **Ketidakcocokan transcript**: memperingatkan saat entri sesi terbaru memiliki
  file transcript yang hilang.
- **Sesi utama “JSONL 1 baris”**: menandai saat transcript utama hanya memiliki satu
  baris (history tidak terakumulasi).
- **Beberapa state dir**: memperingatkan saat beberapa folder `~/.openclaw` ada di berbagai
  direktori home atau saat `OPENCLAW_STATE_DIR` menunjuk ke tempat lain (history dapat
  terpecah antarinstalasi).
- **Pengingat mode remote**: jika `gateway.mode=remote`, doctor mengingatkan Anda untuk menjalankannya
  di host remote (state berada di sana).
- **Izin file config**: memperingatkan jika `~/.openclaw/openclaw.json` dapat dibaca
  oleh group/dunia dan menawarkan untuk memperketat ke `600`.

### 5) Kesehatan auth model (kedaluwarsa OAuth)

Doctor memeriksa profil OAuth di auth store, memperingatkan saat token akan
kedaluwarsa/sudah kedaluwarsa, dan dapat me-refresh-nya saat aman. Jika profil
OAuth/token Anthropic usang, doctor menyarankan Anthropic API key atau jalur
setup-token Anthropic legacy.
Prompt refresh hanya muncul saat berjalan interaktif (TTY); `--non-interactive`
melewati upaya refresh.

Doctor juga mendeteksi state Anthropic Claude CLI lama yang telah dihapus. Jika byte kredensial
`anthropic:claude-cli` lama masih ada di `auth-profiles.json`,
doctor mengonversinya kembali menjadi profil token/OAuth Anthropic dan menulis ulang
referensi model `claude-cli/...` yang usang.
Jika byte-nya sudah hilang, doctor menghapus config usang tersebut dan mencetak
perintah pemulihan.

Doctor juga melaporkan profil auth yang sementara tidak dapat digunakan karena:

- cooldown singkat (rate limit/timeout/kegagalan auth)
- penonaktifan lebih lama (kegagalan billing/kredit)

### 6) Validasi model hooks

Jika `hooks.gmail.model` disetel, doctor memvalidasi referensi model terhadap
katalog dan allowlist serta memperingatkan saat model tidak dapat di-resolve atau tidak diizinkan.

### 7) Perbaikan image sandbox

Saat sandboxing diaktifkan, doctor memeriksa image Docker dan menawarkan untuk membangun atau
beralih ke nama legacy jika image saat ini hilang.

### 7b) Dependensi runtime plugin bundel

Doctor memverifikasi bahwa dependensi runtime plugin bundel (misalnya paket
runtime plugin Discord) ada di root instalasi OpenClaw.
Jika ada yang hilang, doctor melaporkan paket tersebut dan menginstalnya dalam
mode `openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) Migrasi service gateway dan petunjuk pembersihan

Doctor mendeteksi service gateway legacy (launchd/systemd/schtasks) dan
menawarkan untuk menghapusnya serta menginstal service OpenClaw menggunakan port gateway
saat ini. Doctor juga dapat memindai service tambahan yang mirip gateway dan mencetak petunjuk pembersihan.
Service gateway OpenClaw bernama profil dianggap sebagai kelas utama dan tidak
ditandai sebagai "tambahan".

### 8b) Migrasi Matrix saat startup

Saat sebuah akun channel Matrix memiliki migrasi state legacy yang tertunda atau dapat ditindaklanjuti,
doctor (dalam mode `--fix` / `--repair`) membuat snapshot pra-migrasi lalu
menjalankan langkah migrasi best-effort: migrasi state Matrix legacy dan persiapan
encrypted-state legacy. Kedua langkah ini tidak fatal; error dicatat dan
startup tetap berlanjut. Dalam mode read-only (`openclaw doctor` tanpa `--fix`) pemeriksaan ini
dilewati sepenuhnya.

### 9) Peringatan keamanan

Doctor mengeluarkan peringatan saat sebuah provider terbuka untuk DM tanpa allowlist, atau
saat sebuah kebijakan dikonfigurasi dengan cara yang berbahaya.

### 10) Linger systemd (Linux)

Jika berjalan sebagai service pengguna systemd, doctor memastikan lingering diaktifkan agar
gateway tetap hidup setelah logout.

### 11) Status workspace (Skills, plugin, dan dir legacy)

Doctor mencetak ringkasan state workspace untuk agen default:

- **Status Skills**: menghitung skill yang eligible, missing-requirements, dan diblokir allowlist.
- **Dir workspace legacy**: memperingatkan saat `~/openclaw` atau direktori workspace legacy lain
  ada berdampingan dengan workspace saat ini.
- **Status plugin**: menghitung plugin yang loaded/disabled/errored; mencantumkan ID plugin untuk setiap
  error; melaporkan capability plugin bundel.
- **Peringatan kompatibilitas plugin**: menandai plugin yang memiliki masalah kompatibilitas dengan
  runtime saat ini.
- **Diagnostik plugin**: menampilkan setiap peringatan atau error saat load yang dikeluarkan oleh
  registry plugin.

### 11b) Ukuran file bootstrap

Doctor memeriksa apakah file bootstrap workspace (misalnya `AGENTS.md`,
`CLAUDE.md`, atau file konteks lain yang disuntikkan) mendekati atau melebihi
anggaran karakter yang dikonfigurasi. Doctor melaporkan jumlah karakter mentah vs. karakter yang disuntikkan per file, persentase truncation,
penyebab truncation (`max/file` atau `max/total`), dan total karakter yang disuntikkan
sebagai fraksi dari total anggaran. Saat file dipotong atau mendekati batas,
doctor mencetak tips untuk menyetel `agents.defaults.bootstrapMaxChars`
dan `agents.defaults.bootstrapTotalMaxChars`.

### 11c) Shell completion

Doctor memeriksa apakah tab completion sudah terinstal untuk shell saat ini
(zsh, bash, fish, atau PowerShell):

- Jika profil shell menggunakan pola completion dinamis yang lambat
  (`source <(openclaw completion ...)`), doctor meningkatkannya ke
  varian file cache yang lebih cepat.
- Jika completion dikonfigurasi di profil tetapi file cache hilang,
  doctor membuat ulang cache secara otomatis.
- Jika tidak ada completion yang dikonfigurasi sama sekali, doctor meminta untuk menginstalnya
  (hanya mode interaktif; dilewati dengan `--non-interactive`).

Jalankan `openclaw completion --write-state` untuk membuat ulang cache secara manual.

### 12) Pemeriksaan auth gateway (token lokal)

Doctor memeriksa kesiapan auth token gateway lokal.

- Jika mode token memerlukan token dan tidak ada sumber token, doctor menawarkan untuk membuatnya.
- Jika `gateway.auth.token` dikelola SecretRef tetapi tidak tersedia, doctor memperingatkan dan tidak menimpanya dengan plaintext.
- `openclaw doctor --generate-gateway-token` memaksa pembuatan hanya saat tidak ada SecretRef token yang dikonfigurasi.

### 12b) Perbaikan read-only yang sadar SecretRef

Beberapa alur perbaikan perlu memeriksa kredensial yang dikonfigurasi tanpa melemahkan perilaku fail-fast runtime.

- `openclaw doctor --fix` kini menggunakan model ringkasan SecretRef read-only yang sama seperti perintah keluarga status untuk perbaikan config yang terarah.
- Contoh: perbaikan `allowFrom` / `groupAllowFrom` Telegram `@username` mencoba menggunakan kredensial bot yang dikonfigurasi jika tersedia.
- Jika token bot Telegram dikonfigurasi melalui SecretRef tetapi tidak tersedia pada jalur perintah saat ini, doctor melaporkan bahwa kredensial tersebut configured-but-unavailable dan melewati auto-resolution alih-alih crash atau salah melaporkan token sebagai hilang.

### 13) Pemeriksaan kesehatan gateway + restart

Doctor menjalankan pemeriksaan kesehatan dan menawarkan untuk me-restart gateway saat terlihat
tidak sehat.

### 13b) Kesiapan memory search

Doctor memeriksa apakah provider embedding memory search yang dikonfigurasi siap
untuk agen default. Perilakunya bergantung pada backend dan provider yang dikonfigurasi:

- **Backend QMD**: mem-probe apakah biner `qmd` tersedia dan dapat dijalankan.
  Jika tidak, doctor mencetak panduan perbaikan termasuk paket npm dan opsi path biner manual.
- **Provider lokal eksplisit**: memeriksa file model lokal atau URL model
  remote/dapat diunduh yang dikenali. Jika hilang, doctor menyarankan beralih ke provider remote.
- **Provider remote eksplisit** (`openai`, `voyage`, dll.): memverifikasi bahwa API key
  ada di environment atau auth store. Mencetak petunjuk perbaikan yang dapat ditindaklanjuti jika tidak ada.
- **Provider auto**: memeriksa ketersediaan model lokal terlebih dahulu, lalu mencoba setiap provider remote
  sesuai urutan pemilihan otomatis.

Saat hasil probe gateway tersedia (gateway sehat pada saat
pemeriksaan), doctor melakukan referensi silang hasilnya dengan config yang terlihat oleh CLI dan mencatat
setiap ketidaksesuaian.

Gunakan `openclaw memory status --deep` untuk memverifikasi kesiapan embedding saat runtime.

### 14) Peringatan status channel

Jika gateway sehat, doctor menjalankan probe status channel dan melaporkan
peringatan dengan perbaikan yang disarankan.

### 15) Audit config supervisor + perbaikan

Doctor memeriksa config supervisor yang terinstal (launchd/systemd/schtasks) untuk
default yang hilang atau usang (misalnya dependensi network-online systemd dan
delay restart). Saat menemukan ketidakcocokan, doctor merekomendasikan pembaruan dan dapat
menulis ulang file service/task ke default saat ini.

Catatan:

- `openclaw doctor` meminta konfirmasi sebelum menulis ulang config supervisor.
- `openclaw doctor --yes` menerima prompt perbaikan default.
- `openclaw doctor --repair` menerapkan perbaikan yang direkomendasikan tanpa prompt.
- `openclaw doctor --repair --force` menimpa config supervisor kustom.
- Jika auth token memerlukan token dan `gateway.auth.token` dikelola SecretRef, install/perbaikan service doctor memvalidasi SecretRef tetapi tidak menyimpan nilai token plaintext yang sudah di-resolve ke metadata environment service supervisor.
- Jika auth token memerlukan token dan SecretRef token yang dikonfigurasi tidak ter-resolve, doctor memblokir jalur install/perbaikan dengan panduan yang dapat ditindaklanjuti.
- Jika `gateway.auth.token` dan `gateway.auth.password` sama-sama dikonfigurasi dan `gateway.auth.mode` tidak disetel, doctor memblokir install/perbaikan sampai mode disetel secara eksplisit.
- Untuk unit user-systemd Linux, pemeriksaan drift token doctor kini mencakup sumber `Environment=` dan `EnvironmentFile=` saat membandingkan metadata auth service.
- Anda selalu dapat memaksa penulisan ulang penuh melalui `openclaw gateway install --force`.

### 16) Diagnostik runtime gateway + port

Doctor memeriksa runtime service (PID, status keluar terakhir) dan memperingatkan saat
service terinstal tetapi sebenarnya tidak berjalan. Doctor juga memeriksa benturan port
pada port gateway (default `18789`) dan melaporkan kemungkinan penyebabnya (gateway sudah
berjalan, SSH tunnel).

### 17) Best practice runtime gateway

Doctor memperingatkan saat service gateway berjalan di Bun atau path Node yang dikelola version manager
(`nvm`, `fnm`, `volta`, `asdf`, dll.). Channel WhatsApp + Telegram memerlukan Node,
dan path version manager dapat rusak setelah upgrade karena service tidak
memuat init shell Anda. Doctor menawarkan untuk memigrasikan ke instalasi Node sistem saat
tersedia (Homebrew/apt/choco).

### 18) Penulisan config + metadata wizard

Doctor menyimpan setiap perubahan config dan memberi cap metadata wizard untuk mencatat
jalannya doctor.

### 19) Tips workspace (backup + sistem memori)

Doctor menyarankan sistem memori workspace jika belum ada dan mencetak tip backup
jika workspace belum berada di bawah git.

Lihat [/concepts/agent-workspace](/id/concepts/agent-workspace) untuk panduan lengkap tentang
struktur workspace dan backup git (disarankan GitHub atau GitLab privat).
