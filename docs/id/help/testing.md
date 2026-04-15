---
read_when:
    - Menjalankan pengujian secara lokal atau di CI
    - Menambahkan regresi untuk bug model/provider
    - Men-debug perilaku Gateway + agen
summary: 'Kit pengujian: suite unit/e2e/live, runner Docker, dan cakupan masing-masing pengujian'
title: Pengujian
x-i18n:
    generated_at: "2026-04-15T09:15:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: fbf647a5cf13b5861a3ba0cb367dc816c57f0e9c60d3cd6320da193bfadf5609
    source_path: help/testing.md
    workflow: 15
---

# Pengujian

OpenClaw memiliki tiga suite Vitest (unit/integrasi, e2e, live) dan sekumpulan kecil runner Docker.

Dokumen ini adalah panduan “cara kami menguji”:

- Apa yang dicakup tiap suite (dan apa yang dengan sengaja _tidak_ dicakup)
- Perintah mana yang dijalankan untuk alur kerja umum (lokal, sebelum push, debugging)
- Bagaimana pengujian live menemukan kredensial dan memilih model/provider
- Cara menambahkan regresi untuk masalah model/provider di dunia nyata

## Mulai cepat

Pada kebanyakan hari:

- Gate penuh (diharapkan sebelum push): `pnpm build && pnpm check && pnpm test`
- Menjalankan suite penuh lokal yang lebih cepat pada mesin yang lega: `pnpm test:max`
- Loop watch Vitest langsung: `pnpm test:watch`
- Penargetan file langsung sekarang juga merutekan path extension/channel: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Utamakan menjalankan target yang spesifik lebih dulu saat Anda sedang mengiterasi satu kegagalan.
- Situs QA berbasis Docker: `pnpm qa:lab:up`
- Lane QA berbasis VM Linux: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Saat Anda menyentuh pengujian atau menginginkan keyakinan tambahan:

- Gate cakupan: `pnpm test:coverage`
- Suite E2E: `pnpm test:e2e`

Saat men-debug provider/model nyata (memerlukan kredensial nyata):

- Suite live (probe model + alat/gambar Gateway): `pnpm test:live`
- Target satu file live secara senyap: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Tip: saat Anda hanya membutuhkan satu kasus yang gagal, utamakan mempersempit pengujian live melalui env var allowlist yang dijelaskan di bawah.

## Runner khusus QA

Perintah-perintah ini berada di samping suite pengujian utama saat Anda membutuhkan realisme QA-lab:

- `pnpm openclaw qa suite`
  - Menjalankan skenario QA berbasis repo langsung di host.
  - Secara bawaan menjalankan beberapa skenario terpilih secara paralel dengan worker Gateway terisolasi, hingga 64 worker atau jumlah skenario yang dipilih. Gunakan `--concurrency <count>` untuk menyesuaikan jumlah worker, atau `--concurrency 1` untuk lane serial yang lebih lama.
- `pnpm openclaw qa suite --runner multipass`
  - Menjalankan suite QA yang sama di dalam VM Linux Multipass sekali pakai.
  - Mempertahankan perilaku pemilihan skenario yang sama seperti `qa suite` di host.
  - Menggunakan ulang flag pemilihan provider/model yang sama seperti `qa suite`.
  - Run live meneruskan input auth QA yang didukung dan praktis untuk guest:
    kunci provider berbasis env, path konfigurasi provider live QA, dan `CODEX_HOME` bila ada.
  - Direktori output harus tetap berada di bawah root repo agar guest dapat menulis balik melalui workspace yang di-mount.
  - Menulis laporan + ringkasan QA normal serta log Multipass di bawah
    `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Memulai situs QA berbasis Docker untuk pekerjaan QA gaya operator.
- `pnpm openclaw qa matrix`
  - Menjalankan lane QA live Matrix terhadap homeserver Tuwunel berbasis Docker sekali pakai.
  - Host QA ini saat ini hanya untuk repo/dev. Instalasi OpenClaw yang dipaketkan tidak mengirimkan `qa-lab`, jadi tidak mengekspos `openclaw qa`.
  - Checkout repo memuat runner bawaan secara langsung; tidak diperlukan langkah instalasi plugin terpisah.
  - Menyediakan tiga pengguna Matrix sementara (`driver`, `sut`, `observer`) ditambah satu ruang privat, lalu memulai child Gateway QA dengan plugin Matrix nyata sebagai transport SUT.
  - Secara bawaan menggunakan image Tuwunel stabil yang dipin `ghcr.io/matrix-construct/tuwunel:v1.5.1`. Override dengan `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` saat Anda perlu menguji image lain.
  - Matrix tidak mengekspos flag sumber kredensial bersama karena lane ini menyediakan pengguna sekali pakai secara lokal.
  - Menulis laporan QA Matrix, ringkasan, dan artefak peristiwa-teramati di bawah `.artifacts/qa-e2e/...`.
- `pnpm openclaw qa telegram`
  - Menjalankan lane QA live Telegram terhadap grup privat nyata menggunakan token bot driver dan SUT dari env.
  - Memerlukan `OPENCLAW_QA_TELEGRAM_GROUP_ID`, `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`, dan `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. ID grup harus berupa ID chat Telegram numerik.
  - Mendukung `--credential-source convex` untuk kredensial bersama yang dipool. Gunakan mode env secara bawaan, atau set `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` untuk memilih lease pool.
  - Memerlukan dua bot berbeda di grup privat yang sama, dengan bot SUT mengekspos username Telegram.
  - Untuk observasi bot-ke-bot yang stabil, aktifkan Bot-to-Bot Communication Mode di `@BotFather` untuk kedua bot dan pastikan bot driver dapat mengamati lalu lintas bot grup.
  - Menulis laporan QA Telegram, ringkasan, dan artefak pesan-teramati di bawah `.artifacts/qa-e2e/...`.

Lane transport live berbagi satu kontrak standar agar transport baru tidak menyimpang:

`qa-channel` tetap menjadi suite QA sintetis yang luas dan bukan bagian dari matriks cakupan transport live.

| Lane     | Canary | Penyaringan mention | Blok allowlist | Balasan tingkat atas | Lanjut setelah restart | Tindak lanjut thread | Isolasi thread | Observasi reaksi | Perintah bantuan |
| -------- | ------ | ------------------- | -------------- | -------------------- | ---------------------- | -------------------- | -------------- | ---------------- | ---------------- |
| Matrix   | x      | x                   | x              | x                    | x                      | x                    | x              | x                |                  |
| Telegram | x      |                     |                |                      |                        |                      |                |                  | x                |

### Kredensial Telegram bersama melalui Convex (v1)

Saat `--credential-source convex` (atau `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`) diaktifkan untuk
`openclaw qa telegram`, QA lab memperoleh lease eksklusif dari pool berbasis Convex, mengirim heartbeat
untuk lease tersebut selama lane berjalan, dan melepaskan lease saat shutdown.

Scaffold proyek Convex referensi:

- `qa/convex-credential-broker/`

Env var yang diperlukan:

- `OPENCLAW_QA_CONVEX_SITE_URL` (misalnya `https://your-deployment.convex.site`)
- Satu secret untuk peran yang dipilih:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` untuk `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` untuk `ci`
- Pemilihan peran kredensial:
  - CLI: `--credential-role maintainer|ci`
  - Bawaan env: `OPENCLAW_QA_CREDENTIAL_ROLE` (bawaan `maintainer`)

Env var opsional:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (bawaan `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (bawaan `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (bawaan `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (bawaan `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (bawaan `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (ID pelacakan opsional)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` mengizinkan URL Convex `http://` loopback hanya untuk pengembangan lokal.

`OPENCLAW_QA_CONVEX_SITE_URL` sebaiknya menggunakan `https://` dalam operasi normal.

Perintah admin maintainer (tambah/hapus/daftar pool) secara khusus memerlukan
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`.

Helper CLI untuk maintainer:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

Gunakan `--json` untuk output yang dapat dibaca mesin dalam skrip dan utilitas CI.

Kontrak endpoint bawaan (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`):

- `POST /acquire`
  - Permintaan: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - Berhasil: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - Habis/dapat dicoba lagi: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /heartbeat`
  - Permintaan: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - Berhasil: `{ status: "ok" }` (atau `2xx` kosong)
- `POST /release`
  - Permintaan: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - Berhasil: `{ status: "ok" }` (atau `2xx` kosong)
- `POST /admin/add` (khusus secret maintainer)
  - Permintaan: `{ kind, actorId, payload, note?, status? }`
  - Berhasil: `{ status: "ok", credential }`
- `POST /admin/remove` (khusus secret maintainer)
  - Permintaan: `{ credentialId, actorId }`
  - Berhasil: `{ status: "ok", changed, credential }`
  - Penjaga lease aktif: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (khusus secret maintainer)
  - Permintaan: `{ kind?, status?, includePayload?, limit? }`
  - Berhasil: `{ status: "ok", credentials, count }`

Bentuk payload untuk jenis Telegram:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` harus berupa string ID chat Telegram numerik.
- `admin/add` memvalidasi bentuk ini untuk `kind: "telegram"` dan menolak payload yang malformed.

### Menambahkan channel ke QA

Menambahkan channel ke sistem QA markdown memerlukan tepat dua hal:

1. Adapter transport untuk channel tersebut.
2. Paket skenario yang menjalankan kontrak channel.

Jangan menambahkan root perintah QA tingkat atas baru ketika host bersama `qa-lab` dapat
memiliki alur tersebut.

`qa-lab` memiliki mekanisme host bersama:

- root perintah `openclaw qa`
- startup dan teardown suite
- konkurensi worker
- penulisan artefak
- pembuatan laporan
- eksekusi skenario
- alias kompatibilitas untuk skenario `qa-channel` yang lebih lama

Plugin runner memiliki kontrak transport:

- bagaimana `openclaw qa <runner>` dipasang di bawah root `qa` bersama
- bagaimana Gateway dikonfigurasi untuk transport tersebut
- bagaimana kesiapan diperiksa
- bagaimana event masuk diinjeksi
- bagaimana pesan keluar diamati
- bagaimana transkrip dan status transport yang dinormalisasi diekspos
- bagaimana aksi berbasis transport dijalankan
- bagaimana reset atau cleanup khusus transport ditangani

Batas adopsi minimum untuk channel baru adalah:

1. Tetap jadikan `qa-lab` sebagai pemilik root `qa` bersama.
2. Implementasikan runner transport pada seam host bersama `qa-lab`.
3. Simpan mekanisme khusus transport di dalam plugin runner atau harness channel.
4. Pasang runner sebagai `openclaw qa <runner>` alih-alih mendaftarkan root perintah pesaing.
   Plugin runner harus mendeklarasikan `qaRunners` di `openclaw.plugin.json` dan mengekspor array `qaRunnerCliRegistrations` yang sesuai dari `runtime-api.ts`.
   Jaga `runtime-api.ts` tetap ringan; CLI malas dan eksekusi runner harus tetap berada di balik entrypoint terpisah.
5. Tulis atau adaptasi skenario markdown di bawah `qa/scenarios/`.
6. Gunakan helper skenario generik untuk skenario baru.
7. Pertahankan alias kompatibilitas yang ada tetap berfungsi kecuali repo sedang melakukan migrasi yang disengaja.

Aturan keputusannya ketat:

- Jika perilaku bisa diekspresikan sekali di `qa-lab`, taruh di `qa-lab`.
- Jika perilaku bergantung pada satu transport channel, simpan di plugin runner atau harness plugin tersebut.
- Jika suatu skenario memerlukan kapabilitas baru yang dapat digunakan lebih dari satu channel, tambahkan helper generik alih-alih cabang khusus channel di `suite.ts`.
- Jika suatu perilaku hanya bermakna untuk satu transport, buat skenario tetap khusus transport dan nyatakan itu secara eksplisit dalam kontrak skenario.

Nama helper generik yang disarankan untuk skenario baru adalah:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Alias kompatibilitas tetap tersedia untuk skenario yang ada, termasuk:

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

Pekerjaan channel baru sebaiknya menggunakan nama helper generik.
Alias kompatibilitas ada untuk menghindari migrasi besar sekaligus, bukan sebagai model untuk
penulisan skenario baru.

## Suite pengujian (apa yang berjalan di mana)

Bayangkan suite sebagai “realisme yang meningkat” (serta meningkatnya flakiness/biaya):

### Unit / integrasi (bawaan)

- Perintah: `pnpm test`
- Konfigurasi: sepuluh run shard berurutan (`vitest.full-*.config.ts`) di atas project Vitest terlingkup yang sudah ada
- File: inventaris core/unit di bawah `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts`, dan pengujian node `ui` yang di-whitelist yang dicakup oleh `vitest.unit.config.ts`
- Cakupan:
  - Pengujian unit murni
  - Pengujian integrasi in-process (auth Gateway, routing, tooling, parsing, config)
  - Regresi deterministik untuk bug yang sudah diketahui
- Ekspektasi:
  - Berjalan di CI
  - Tidak memerlukan kunci nyata
  - Harus cepat dan stabil
- Catatan project:
  - `pnpm test` tanpa target sekarang menjalankan sebelas konfigurasi shard yang lebih kecil (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) alih-alih satu proses root-project native yang besar. Ini mengurangi RSS puncak pada mesin yang sibuk dan mencegah pekerjaan auto-reply/extension menghambat suite yang tidak terkait.
  - `pnpm test --watch` tetap menggunakan graf project native root `vitest.config.ts`, karena loop watch multi-shard tidak praktis.
  - `pnpm test`, `pnpm test:watch`, dan `pnpm test:perf:imports` merutekan target file/direktori eksplisit melalui lane terlingkup terlebih dahulu, sehingga `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` tidak perlu membayar biaya startup penuh root project.
  - `pnpm test:changed` memperluas path git yang berubah ke lane terlingkup yang sama saat diff hanya menyentuh file sumber/pengujian yang dapat dirutekan; edit config/setup tetap fallback ke rerun root-project yang luas.
  - Pengujian unit ringan impor dari agents, commands, plugins, helper auto-reply, `plugin-sdk`, dan area utilitas murni serupa dirutekan melalui lane `unit-fast`, yang melewati `test/setup-openclaw-runtime.ts`; file stateful/berat saat runtime tetap berada di lane yang ada.
  - Beberapa file sumber helper `plugin-sdk` dan `commands` yang dipilih juga memetakan run mode-changed ke pengujian sibling eksplisit di lane ringan tersebut, sehingga edit helper menghindari rerun suite berat penuh untuk direktori itu.
  - `auto-reply` sekarang memiliki tiga bucket khusus: helper core tingkat atas, pengujian integrasi `reply.*` tingkat atas, dan subtree `src/auto-reply/reply/**`. Ini menjaga pekerjaan harness balasan terberat agar tidak masuk ke pengujian status/chunk/token yang murah.
- Catatan embedded runner:
  - Saat Anda mengubah input discovery message-tool atau konteks runtime Compaction,
    pertahankan kedua tingkat cakupan.
  - Tambahkan regresi helper yang terfokus untuk batas routing/normalisasi murni.
  - Juga jaga suite integrasi embedded runner tetap sehat:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`, dan
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Suite tersebut memverifikasi bahwa id terlingkup dan perilaku Compaction tetap mengalir
    melalui path `run.ts` / `compact.ts` yang nyata; pengujian helper saja bukan
    pengganti yang memadai untuk path integrasi tersebut.
- Catatan pool:
  - Konfigurasi dasar Vitest sekarang bawaan ke `threads`.
  - Konfigurasi Vitest bersama juga menetapkan `isolate: false` dan menggunakan runner non-isolated di seluruh konfigurasi root projects, e2e, dan live.
  - Lane UI root mempertahankan setup dan optimizer `jsdom`-nya, tetapi sekarang juga berjalan pada runner non-isolated bersama.
  - Tiap shard `pnpm test` mewarisi bawaan `threads` + `isolate: false` yang sama dari konfigurasi Vitest bersama.
  - Launcher bersama `scripts/run-vitest.mjs` sekarang juga menambahkan `--no-maglev` untuk proses child Node Vitest secara bawaan guna mengurangi churn kompilasi V8 selama run lokal besar. Set `OPENCLAW_VITEST_ENABLE_MAGLEV=1` jika Anda perlu membandingkannya dengan perilaku V8 bawaan.
- Catatan iterasi lokal cepat:
  - `pnpm test:changed` merutekan melalui lane terlingkup saat path yang berubah terpetakan dengan rapi ke suite yang lebih kecil.
  - `pnpm test:max` dan `pnpm test:changed:max` mempertahankan perilaku routing yang sama, hanya dengan batas worker yang lebih tinggi.
  - Auto-scaling worker lokal sekarang memang dibuat konservatif dan juga mundur saat load average host sudah tinggi, sehingga beberapa run Vitest serentak secara bawaan mengurangi dampak.
  - Konfigurasi dasar Vitest menandai project/file config sebagai `forceRerunTriggers` agar rerun mode-changed tetap benar saat wiring pengujian berubah.
  - Konfigurasi mempertahankan `OPENCLAW_VITEST_FS_MODULE_CACHE` tetap aktif pada host yang didukung; set `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` jika Anda menginginkan satu lokasi cache eksplisit untuk profiling langsung.
- Catatan debug performa:
  - `pnpm test:perf:imports` mengaktifkan pelaporan durasi impor Vitest plus output rincian impor.
  - `pnpm test:perf:imports:changed` mencakup tampilan profiling yang sama ke file yang berubah sejak `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` membandingkan `test:changed` yang dirutekan terhadap path native root-project untuk diff yang sudah di-commit tersebut dan mencetak wall time plus RSS maksimum macOS.
- `pnpm test:perf:changed:bench -- --worktree` membenchmark tree kotor saat ini dengan merutekan daftar file yang berubah melalui `scripts/test-projects.mjs` dan konfigurasi root Vitest.
  - `pnpm test:perf:profile:main` menulis profil CPU thread utama untuk overhead startup dan transform Vitest/Vite.
  - `pnpm test:perf:profile:runner` menulis profil CPU+heap runner untuk suite unit dengan paralelisme file dinonaktifkan.

### E2E (smoke Gateway)

- Perintah: `pnpm test:e2e`
- Konfigurasi: `vitest.e2e.config.ts`
- File: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Bawaan runtime:
  - Menggunakan Vitest `threads` dengan `isolate: false`, sesuai dengan bagian repo lainnya.
  - Menggunakan worker adaptif (CI: hingga 2, lokal: 1 secara bawaan).
  - Berjalan dalam mode senyap secara bawaan untuk mengurangi overhead I/O konsol.
- Override yang berguna:
  - `OPENCLAW_E2E_WORKERS=<n>` untuk memaksa jumlah worker (dibatasi hingga 16).
  - `OPENCLAW_E2E_VERBOSE=1` untuk mengaktifkan kembali output konsol verbose.
- Cakupan:
  - Perilaku end-to-end Gateway multi-instance
  - Surface WebSocket/HTTP, pairing Node, dan jaringan yang lebih berat
- Ekspektasi:
  - Berjalan di CI (saat diaktifkan dalam pipeline)
  - Tidak memerlukan kunci nyata
  - Lebih banyak komponen bergerak dibanding pengujian unit (bisa lebih lambat)

### E2E: smoke backend OpenShell

- Perintah: `pnpm test:e2e:openshell`
- File: `test/openshell-sandbox.e2e.test.ts`
- Cakupan:
  - Memulai Gateway OpenShell terisolasi di host melalui Docker
  - Membuat sandbox dari Dockerfile lokal sementara
  - Menjalankan backend OpenShell OpenClaw melalui `sandbox ssh-config` + exec SSH nyata
  - Memverifikasi perilaku filesystem remote-canonical melalui bridge fs sandbox
- Ekspektasi:
  - Hanya opt-in; bukan bagian dari run bawaan `pnpm test:e2e`
  - Memerlukan CLI `openshell` lokal plus daemon Docker yang berfungsi
  - Menggunakan `HOME` / `XDG_CONFIG_HOME` terisolasi, lalu menghancurkan Gateway dan sandbox pengujian
- Override yang berguna:
  - `OPENCLAW_E2E_OPENSHELL=1` untuk mengaktifkan pengujian saat menjalankan suite e2e yang lebih luas secara manual
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` untuk menunjuk ke biner CLI nonbawaan atau skrip wrapper

### Live (provider nyata + model nyata)

- Perintah: `pnpm test:live`
- Konfigurasi: `vitest.live.config.ts`
- File: `src/**/*.live.test.ts`
- Bawaan: **diaktifkan** oleh `pnpm test:live` (mengatur `OPENCLAW_LIVE_TEST=1`)
- Cakupan:
  - “Apakah provider/model ini benar-benar berfungsi _hari ini_ dengan kredensial nyata?”
  - Menangkap perubahan format provider, keanehan pemanggilan tool, masalah auth, dan perilaku rate limit
- Ekspektasi:
  - Secara desain tidak stabil di CI (jaringan nyata, kebijakan provider nyata, kuota, gangguan)
  - Menghabiskan biaya / menggunakan rate limit
  - Utamakan menjalankan subset yang dipersempit alih-alih “semuanya”
- Run live mengambil sumber `~/.profile` untuk mengambil kunci API yang belum ada.
- Secara bawaan, run live tetap mengisolasi `HOME` dan menyalin materi config/auth ke home pengujian sementara agar fixture unit tidak dapat memodifikasi `~/.openclaw` Anda yang nyata.
- Set `OPENCLAW_LIVE_USE_REAL_HOME=1` hanya saat Anda memang sengaja memerlukan pengujian live untuk menggunakan direktori home nyata Anda.
- `pnpm test:live` sekarang secara bawaan menggunakan mode yang lebih senyap: mempertahankan output progres `[live] ...`, tetapi menyembunyikan pemberitahuan `~/.profile` tambahan dan membisukan log bootstrap Gateway/obrolan Bonjour. Set `OPENCLAW_LIVE_TEST_QUIET=0` jika Anda ingin log startup lengkap kembali.
- Rotasi kunci API (khusus provider): set `*_API_KEYS` dengan format koma/titik koma atau `*_API_KEY_1`, `*_API_KEY_2` (misalnya `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) atau override per-live melalui `OPENCLAW_LIVE_*_KEY`; pengujian mencoba ulang pada respons rate limit.
- Output progres/heartbeat:
  - Suite live sekarang memancarkan baris progres ke stderr sehingga pemanggilan provider yang panjang tampak aktif bahkan saat penangkapan konsol Vitest senyap.
  - `vitest.live.config.ts` menonaktifkan intersepsi konsol Vitest sehingga baris progres provider/Gateway mengalir segera selama run live.
  - Sesuaikan heartbeat direct-model dengan `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Sesuaikan heartbeat Gateway/probe dengan `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Suite mana yang sebaiknya saya jalankan?

Gunakan tabel keputusan ini:

- Mengedit logika/pengujian: jalankan `pnpm test` (dan `pnpm test:coverage` jika Anda banyak mengubah)
- Menyentuh jaringan Gateway / protokol WS / pairing: tambahkan `pnpm test:e2e`
- Men-debug “bot saya down” / kegagalan khusus provider / pemanggilan tool: jalankan `pnpm test:live` yang dipersempit

## Live: sapuan kapabilitas Node Android

- Pengujian: `src/gateway/android-node.capabilities.live.test.ts`
- Skrip: `pnpm android:test:integration`
- Tujuan: memanggil **setiap perintah yang saat ini diiklankan** oleh Node Android yang terhubung dan menegaskan perilaku kontrak perintah.
- Cakupan:
  - Setup manual/prasyarat (suite ini tidak menginstal/menjalankan/melakukan pairing aplikasi).
  - Validasi `node.invoke` Gateway per perintah untuk Node Android yang dipilih.
- Pra-setup yang diperlukan:
  - Aplikasi Android sudah terhubung + dipairing ke Gateway.
  - Aplikasi tetap berada di foreground.
  - Izin/persetujuan capture diberikan untuk kapabilitas yang Anda harapkan lolos.
- Override target opsional:
  - `OPENCLAW_ANDROID_NODE_ID` atau `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Detail setup Android lengkap: [Aplikasi Android](/id/platforms/android)

## Live: smoke model (kunci profile)

Pengujian live dibagi menjadi dua lapisan agar kita bisa mengisolasi kegagalan:

- “Direct model” memberi tahu kita apakah provider/model dapat menjawab sama sekali dengan kunci yang diberikan.
- “Gateway smoke” memberi tahu kita apakah pipeline Gateway+agen penuh berfungsi untuk model tersebut (sesi, riwayat, tool, kebijakan sandbox, dll.).

### Lapisan 1: penyelesaian model langsung (tanpa Gateway)

- Pengujian: `src/agents/models.profiles.live.test.ts`
- Tujuan:
  - Menginventarisasi model yang ditemukan
  - Menggunakan `getApiKeyForModel` untuk memilih model yang Anda miliki kredensialnya
  - Menjalankan penyelesaian kecil per model (dan regresi terarah bila perlu)
- Cara mengaktifkan:
  - `pnpm test:live` (atau `OPENCLAW_LIVE_TEST=1` jika memanggil Vitest secara langsung)
- Set `OPENCLAW_LIVE_MODELS=modern` (atau `all`, alias untuk modern) untuk benar-benar menjalankan suite ini; jika tidak, suite ini dilewati agar `pnpm test:live` tetap fokus pada smoke Gateway
- Cara memilih model:
  - `OPENCLAW_LIVE_MODELS=modern` untuk menjalankan allowlist modern (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` adalah alias untuk allowlist modern
  - atau `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (allowlist dipisahkan koma)
  - Sapuan modern/all secara bawaan menggunakan batas terkurasi dengan sinyal tinggi; set `OPENCLAW_LIVE_MAX_MODELS=0` untuk sapuan modern yang menyeluruh atau angka positif untuk batas yang lebih kecil.
- Cara memilih provider:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (allowlist dipisahkan koma)
- Dari mana kunci berasal:
  - Secara bawaan: profile store dan fallback env
  - Set `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` untuk memaksakan **profile store** saja
- Mengapa ini ada:
  - Memisahkan “API provider rusak / kunci tidak valid” dari “pipeline agen Gateway rusak”
  - Memuat regresi kecil dan terisolasi (contoh: replay reasoning OpenAI Responses/Codex Responses + alur tool-call)

### Lapisan 2: smoke Gateway + agen dev (apa yang sebenarnya dilakukan "@openclaw")

- Pengujian: `src/gateway/gateway-models.profiles.live.test.ts`
- Tujuan:
  - Menyalakan Gateway in-process
  - Membuat/menambal sesi `agent:dev:*` (override model per run)
  - Mengiterasi model-dengan-kunci dan menegaskan:
    - respons yang “bermakna” (tanpa tool)
    - pemanggilan tool nyata berfungsi (probe `read`)
    - probe tool tambahan opsional (probe `exec+read`)
    - path regresi OpenAI (hanya tool-call → tindak lanjut) tetap berfungsi
- Detail probe (agar Anda bisa menjelaskan kegagalan dengan cepat):
  - probe `read`: pengujian menulis file nonce di workspace lalu meminta agen untuk `read` file itu dan menggemakan nonce kembali.
  - probe `exec+read`: pengujian meminta agen untuk menulis nonce ke file sementara melalui `exec`, lalu `read` file itu kembali.
  - probe image: pengujian melampirkan PNG yang dihasilkan (kucing + kode acak) dan mengharapkan model mengembalikan `cat <CODE>`.
  - Referensi implementasi: `src/gateway/gateway-models.profiles.live.test.ts` dan `src/gateway/live-image-probe.ts`.
- Cara mengaktifkan:
  - `pnpm test:live` (atau `OPENCLAW_LIVE_TEST=1` jika memanggil Vitest secara langsung)
- Cara memilih model:
  - Bawaan: allowlist modern (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` adalah alias untuk allowlist modern
  - Atau set `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (atau daftar dipisahkan koma) untuk mempersempit
  - Sapuan Gateway modern/all secara bawaan menggunakan batas terkurasi dengan sinyal tinggi; set `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` untuk sapuan modern yang menyeluruh atau angka positif untuk batas yang lebih kecil.
- Cara memilih provider (hindari “semua OpenRouter”):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (allowlist dipisahkan koma)
- Probe tool + image selalu aktif dalam pengujian live ini:
  - probe `read` + probe `exec+read` (stress tool)
  - probe image berjalan saat model mengiklankan dukungan input image
  - Alur (tingkat tinggi):
    - Pengujian menghasilkan PNG kecil dengan “CAT” + kode acak (`src/gateway/live-image-probe.ts`)
    - Mengirimkannya melalui `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Gateway mem-parse lampiran ke `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Agen embedded meneruskan pesan pengguna multimodal ke model
    - Penegasan: balasan berisi `cat` + kode tersebut (toleransi OCR: kesalahan kecil diperbolehkan)

Tip: untuk melihat apa yang bisa Anda uji di mesin Anda (dan id `provider/model` yang tepat), jalankan:

```bash
openclaw models list
openclaw models list --json
```

## Live: smoke backend CLI (Claude, Codex, Gemini, atau CLI lokal lain)

- Pengujian: `src/gateway/gateway-cli-backend.live.test.ts`
- Tujuan: memvalidasi pipeline Gateway + agen menggunakan backend CLI lokal, tanpa menyentuh config bawaan Anda.
- Bawaan smoke khusus backend berada bersama definisi `cli-backend.ts` milik extension yang memilikinya.
- Aktifkan:
  - `pnpm test:live` (atau `OPENCLAW_LIVE_TEST=1` jika memanggil Vitest secara langsung)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Bawaan:
  - Provider/model bawaan: `claude-cli/claude-sonnet-4-6`
  - Perilaku perintah/argumen/image berasal dari metadata plugin backend CLI yang memilikinya.
- Override (opsional):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` untuk mengirim lampiran image nyata (path diinjeksi ke prompt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` untuk meneruskan path file image sebagai argumen CLI alih-alih injeksi prompt.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (atau `"list"`) untuk mengontrol bagaimana argumen image diteruskan saat `IMAGE_ARG` diatur.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` untuk mengirim giliran kedua dan memvalidasi alur resume.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` untuk menonaktifkan probe kontinuitas sesi yang sama bawaan Claude Sonnet -> Opus (set ke `1` untuk memaksanya aktif saat model yang dipilih mendukung target switch).

Contoh:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Resep Docker:

```bash
pnpm test:docker:live-cli-backend
```

Resep Docker provider tunggal:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Catatan:

- Runner Docker berada di `scripts/test-live-cli-backend-docker.sh`.
- Runner ini menjalankan smoke CLI-backend live di dalam image Docker repo sebagai pengguna `node` non-root.
- Runner ini menyelesaikan metadata smoke CLI dari extension yang memilikinya, lalu memasang paket CLI Linux yang sesuai (`@anthropic-ai/claude-code`, `@openai/codex`, atau `@google/gemini-cli`) ke prefix writable yang di-cache di `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (bawaan: `~/.cache/openclaw/docker-cli-tools`).
- `pnpm test:docker:live-cli-backend:claude-subscription` memerlukan OAuth langganan portabel Claude Code melalui `~/.claude/.credentials.json` dengan `claudeAiOauth.subscriptionType` atau `CLAUDE_CODE_OAUTH_TOKEN` dari `claude setup-token`. Runner ini terlebih dahulu membuktikan `claude -p` langsung di Docker, lalu menjalankan dua giliran Gateway CLI-backend tanpa mempertahankan env var kunci API Anthropic. Lane subscription ini menonaktifkan probe MCP/tool dan image Claude secara bawaan karena Claude saat ini merutekan penggunaan aplikasi pihak ketiga melalui penagihan penggunaan tambahan alih-alih batas normal paket langganan.
- Smoke CLI-backend live sekarang menjalankan alur end-to-end yang sama untuk Claude, Codex, dan Gemini: giliran teks, giliran klasifikasi image, lalu pemanggilan tool MCP `cron` yang diverifikasi melalui CLI Gateway.
- Smoke bawaan Claude juga menambal sesi dari Sonnet ke Opus dan memverifikasi bahwa sesi yang dilanjutkan masih mengingat catatan sebelumnya.

## Live: smoke bind ACP (`/acp spawn ... --bind here`)

- Pengujian: `src/gateway/gateway-acp-bind.live.test.ts`
- Tujuan: memvalidasi alur conversation-bind ACP nyata dengan agen ACP live:
  - kirim `/acp spawn <agent> --bind here`
  - bind percakapan message-channel sintetis di tempat
  - kirim tindak lanjut normal pada percakapan yang sama
  - verifikasi bahwa tindak lanjut masuk ke transkrip sesi ACP yang di-bind
- Aktifkan:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Bawaan:
  - Agen ACP di Docker: `claude,codex,gemini`
  - Agen ACP untuk `pnpm test:live ...` langsung: `claude`
  - Channel sintetis: konteks percakapan gaya DM Slack
  - Backend ACP: `acpx`
- Override:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Catatan:
  - Lane ini menggunakan surface Gateway `chat.send` dengan field originating-route sintetis khusus admin agar pengujian dapat melampirkan konteks message-channel tanpa berpura-pura mengirimkannya secara eksternal.
  - Saat `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` tidak diatur, pengujian menggunakan registry agen bawaan milik plugin `acpx` embedded untuk agen harness ACP yang dipilih.

Contoh:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Resep Docker:

```bash
pnpm test:docker:live-acp-bind
```

Resep Docker agen tunggal:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Catatan Docker:

- Runner Docker berada di `scripts/test-live-acp-bind-docker.sh`.
- Secara bawaan, runner ini menjalankan smoke bind ACP terhadap semua agen CLI live yang didukung secara berurutan: `claude`, `codex`, lalu `gemini`.
- Gunakan `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`, atau `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` untuk mempersempit matriks.
- Runner ini mengambil sumber `~/.profile`, men-stage materi auth CLI yang sesuai ke container, memasang `acpx` ke prefix npm writable, lalu memasang CLI live yang diminta (`@anthropic-ai/claude-code`, `@openai/codex`, atau `@google/gemini-cli`) jika belum ada.
- Di dalam Docker, runner menetapkan `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` agar acpx mempertahankan env var provider dari profile yang di-source tetap tersedia untuk CLI harness child.

## Live: smoke harness app-server Codex

- Tujuan: memvalidasi harness Codex milik plugin melalui metode `agent`
  Gateway normal:
  - memuat plugin `codex` bawaan
  - memilih `OPENCLAW_AGENT_RUNTIME=codex`
  - mengirim giliran agen Gateway pertama ke `codex/gpt-5.4`
  - mengirim giliran kedua ke sesi OpenClaw yang sama dan memverifikasi bahwa thread app-server
    dapat dilanjutkan
  - menjalankan `/codex status` dan `/codex models` melalui path perintah Gateway
    yang sama
- Pengujian: `src/gateway/gateway-codex-harness.live.test.ts`
- Aktifkan: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Model bawaan: `codex/gpt-5.4`
- Probe image opsional: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- Probe MCP/tool opsional: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- Smoke ini menetapkan `OPENCLAW_AGENT_HARNESS_FALLBACK=none` agar harness Codex yang rusak
  tidak bisa lolos dengan diam-diam fallback ke PI.
- Auth: `OPENAI_API_KEY` dari shell/profile, ditambah salinan opsional
  `~/.codex/auth.json` dan `~/.codex/config.toml`

Resep lokal:

```bash
source ~/.profile
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=codex/gpt-5.4 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Resep Docker:

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

Catatan Docker:

- Runner Docker berada di `scripts/test-live-codex-harness-docker.sh`.
- Runner ini mengambil sumber `~/.profile` yang di-mount, meneruskan `OPENAI_API_KEY`, menyalin file auth CLI Codex
  saat ada, memasang `@openai/codex` ke prefix npm writable yang di-mount,
  men-stage source tree, lalu hanya menjalankan pengujian live Codex-harness.
- Docker mengaktifkan probe image dan MCP/tool secara bawaan. Set
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` atau
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` saat Anda memerlukan run debug yang lebih sempit.
- Docker juga mengekspor `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, sesuai dengan konfigurasi pengujian live
  sehingga fallback `openai-codex/*` atau PI tidak dapat menyembunyikan regresi
  harness Codex.

### Resep live yang direkomendasikan

Allowlist yang sempit dan eksplisit adalah yang tercepat dan paling tidak flaky:

- Model tunggal, langsung (tanpa Gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Model tunggal, smoke Gateway:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Pemanggilan tool di beberapa provider:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Fokus Google (kunci API Gemini + Antigravity):
  - Gemini (kunci API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Catatan:

- `google/...` menggunakan API Gemini (kunci API).
- `google-antigravity/...` menggunakan bridge OAuth Antigravity (endpoint agen gaya Cloud Code Assist).
- `google-gemini-cli/...` menggunakan CLI Gemini lokal di mesin Anda (auth terpisah + keanehan tooling).
- API Gemini vs CLI Gemini:
  - API: OpenClaw memanggil API Gemini hosted milik Google melalui HTTP (auth kunci API / profile); inilah yang dimaksud sebagian besar pengguna dengan “Gemini”.
  - CLI: OpenClaw melakukan shell-out ke biner `gemini` lokal; ia memiliki auth sendiri dan bisa berperilaku berbeda (streaming/dukungan tool/version skew).

## Live: matriks model (apa yang kami cakup)

Tidak ada “daftar model CI” yang tetap (live bersifat opt-in), tetapi ini adalah model-model **yang direkomendasikan** untuk dicakup secara rutin di mesin dev dengan kunci.

### Set smoke modern (tool calling + image)

Ini adalah run “model umum” yang kami harapkan tetap berfungsi:

- OpenAI (non-Codex): `openai/gpt-5.4` (opsional: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (atau `anthropic/claude-sonnet-4-6`)
- Google (API Gemini): `google/gemini-3.1-pro-preview` dan `google/gemini-3-flash-preview` (hindari model Gemini 2.x yang lebih lama)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` dan `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Jalankan smoke Gateway dengan tool + image:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Baseline: tool calling (Read + Exec opsional)

Pilih setidaknya satu per keluarga provider:

- OpenAI: `openai/gpt-5.4` (atau `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (atau `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (atau `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Cakupan tambahan opsional (bagus jika ada):

- xAI: `xai/grok-4` (atau versi terbaru yang tersedia)
- Mistral: `mistral/`… (pilih satu model yang mampu `tools` yang Anda aktifkan)
- Cerebras: `cerebras/`… (jika Anda punya akses)
- LM Studio: `lmstudio/`… (lokal; tool calling bergantung pada mode API)

### Vision: pengiriman image (lampiran → pesan multimodal)

Sertakan setidaknya satu model yang mampu memproses image dalam `OPENCLAW_LIVE_GATEWAY_MODELS` (varian Claude/Gemini/OpenAI yang mendukung vision, dll.) untuk menjalankan probe image.

### Agregator / Gateway alternatif

Jika Anda mengaktifkan kunci, kami juga mendukung pengujian melalui:

- OpenRouter: `openrouter/...` (ratusan model; gunakan `openclaw models scan` untuk menemukan kandidat yang mendukung tool+image)
- OpenCode: `opencode/...` untuk Zen dan `opencode-go/...` untuk Go (auth melalui `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Provider lain yang bisa Anda sertakan dalam matriks live (jika Anda punya kredensial/config):

- Bawaan: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Melalui `models.providers` (endpoint kustom): `minimax` (cloud/API), ditambah proxy yang kompatibel dengan OpenAI/Anthropic (LM Studio, vLLM, LiteLLM, dll.)

Tip: jangan mencoba meng-hardcode “semua model” di dokumen. Daftar yang otoritatif adalah apa pun yang dikembalikan `discoverModels(...)` di mesin Anda + kunci apa pun yang tersedia.

## Kredensial (jangan pernah commit)

Pengujian live menemukan kredensial dengan cara yang sama seperti CLI. Implikasi praktis:

- Jika CLI berfungsi, pengujian live seharusnya menemukan kunci yang sama.
- Jika pengujian live mengatakan “tidak ada kredensial”, debug dengan cara yang sama seperti Anda men-debug `openclaw models list` / pemilihan model.

- Profile auth per agen: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (inilah yang dimaksud dengan “kunci profile” dalam pengujian live)
- Config: `~/.openclaw/openclaw.json` (atau `OPENCLAW_CONFIG_PATH`)
- Direktori state lama: `~/.openclaw/credentials/` (disalin ke home live yang di-stage saat ada, tetapi bukan penyimpanan utama kunci profile)
- Run lokal live secara bawaan menyalin config aktif, file `auth-profiles.json` per agen, `credentials/` lama, dan direktori auth CLI eksternal yang didukung ke home pengujian sementara; home live yang di-stage melewati `workspace/` dan `sandboxes/`, dan override path `agents.*.workspace` / `agentDir` dihapus agar probe tetap tidak menyentuh workspace host nyata Anda.

Jika Anda ingin mengandalkan kunci env (misalnya diekspor di `~/.profile` Anda), jalankan pengujian lokal setelah `source ~/.profile`, atau gunakan runner Docker di bawah (mereka dapat me-mount `~/.profile` ke dalam container).

## Live Deepgram (transkripsi audio)

- Pengujian: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Aktifkan: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## Live rencana coding BytePlus

- Pengujian: `src/agents/byteplus.live.test.ts`
- Aktifkan: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Override model opsional: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## Live media alur kerja ComfyUI

- Pengujian: `extensions/comfy/comfy.live.test.ts`
- Aktifkan: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Cakupan:
  - Menjalankan path image, video, dan `music_generate` comfy bawaan
  - Melewati tiap kapabilitas kecuali `models.providers.comfy.<capability>` dikonfigurasi
  - Berguna setelah mengubah pengiriman alur kerja comfy, polling, unduhan, atau registrasi plugin

## Live pembuatan image

- Pengujian: `src/image-generation/runtime.live.test.ts`
- Perintah: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- Cakupan:
  - Menginventarisasi setiap plugin provider pembuatan image yang terdaftar
  - Memuat env var provider yang belum ada dari shell login Anda (`~/.profile`) sebelum probing
  - Secara bawaan menggunakan kunci API live/env sebelum profile auth yang tersimpan, sehingga kunci pengujian usang di `auth-profiles.json` tidak menutupi kredensial shell nyata
  - Melewati provider tanpa auth/profile/model yang dapat digunakan
  - Menjalankan varian pembuatan image stok melalui kapabilitas runtime bersama:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Provider bawaan saat ini yang dicakup:
  - `openai`
  - `google`
- Penyempitan opsional:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Perilaku auth opsional:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` untuk memaksakan auth profile store dan mengabaikan override khusus env

## Live pembuatan musik

- Pengujian: `extensions/music-generation-providers.live.test.ts`
- Aktifkan: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- Cakupan:
  - Menjalankan path provider pembuatan musik bawaan bersama
  - Saat ini mencakup Google dan MiniMax
  - Memuat env var provider dari shell login Anda (`~/.profile`) sebelum probing
  - Secara bawaan menggunakan kunci API live/env sebelum profile auth yang tersimpan, sehingga kunci pengujian usang di `auth-profiles.json` tidak menutupi kredensial shell nyata
  - Melewati provider tanpa auth/profile/model yang dapat digunakan
  - Menjalankan kedua mode runtime yang dideklarasikan saat tersedia:
    - `generate` dengan input hanya prompt
    - `edit` saat provider mendeklarasikan `capabilities.edit.enabled`
  - Cakupan lane bersama saat ini:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: file live Comfy terpisah, bukan sapuan bersama ini
- Penyempitan opsional:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Perilaku auth opsional:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` untuk memaksakan auth profile store dan mengabaikan override khusus env

## Live pembuatan video

- Pengujian: `extensions/video-generation-providers.live.test.ts`
- Aktifkan: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- Cakupan:
  - Menjalankan path provider pembuatan video bawaan bersama
  - Secara bawaan menggunakan path smoke yang aman untuk rilis: provider non-FAL, satu permintaan text-to-video per provider, prompt lobster satu detik, dan batas operasi per provider dari `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (`180000` secara bawaan)
  - Melewati FAL secara bawaan karena latensi antrean di sisi provider dapat mendominasi waktu rilis; teruskan `--video-providers fal` atau `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"` untuk menjalankannya secara eksplisit
  - Memuat env var provider dari shell login Anda (`~/.profile`) sebelum probing
  - Secara bawaan menggunakan kunci API live/env sebelum profile auth yang tersimpan, sehingga kunci pengujian usang di `auth-profiles.json` tidak menutupi kredensial shell nyata
  - Melewati provider tanpa auth/profile/model yang dapat digunakan
  - Secara bawaan hanya menjalankan `generate`
  - Set `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` untuk juga menjalankan mode transform yang dideklarasikan saat tersedia:
    - `imageToVideo` saat provider mendeklarasikan `capabilities.imageToVideo.enabled` dan provider/model yang dipilih menerima input image lokal berbasis buffer dalam sapuan bersama
    - `videoToVideo` saat provider mendeklarasikan `capabilities.videoToVideo.enabled` dan provider/model yang dipilih menerima input video lokal berbasis buffer dalam sapuan bersama
  - Provider `imageToVideo` yang saat ini dideklarasikan tetapi dilewati dalam sapuan bersama:
    - `vydra` karena `veo3` bawaan hanya teks dan `kling` bawaan memerlukan URL image jarak jauh
  - Cakupan Vydra khusus provider:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - file itu menjalankan `veo3` text-to-video plus lane `kling` yang secara bawaan menggunakan fixture URL image jarak jauh
  - Cakupan live `videoToVideo` saat ini:
    - `runway` hanya saat model yang dipilih adalah `runway/gen4_aleph`
  - Provider `videoToVideo` yang saat ini dideklarasikan tetapi dilewati dalam sapuan bersama:
    - `alibaba`, `qwen`, `xai` karena path tersebut saat ini memerlukan URL referensi jarak jauh `http(s)` / MP4
    - `google` karena lane Gemini/Veo bersama saat ini menggunakan input lokal berbasis buffer dan path itu tidak diterima dalam sapuan bersama
    - `openai` karena lane bersama saat ini tidak memiliki jaminan akses inpaint/remix video khusus org
- Penyempitan opsional:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""` untuk menyertakan setiap provider dalam sapuan bawaan, termasuk FAL
  - `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000` untuk mengurangi batas operasi tiap provider demi run smoke yang agresif
- Perilaku auth opsional:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` untuk memaksakan auth profile store dan mengabaikan override khusus env

## Harness live media

- Perintah: `pnpm test:live:media`
- Tujuan:
  - Menjalankan suite live image, musik, dan video bersama melalui satu entrypoint native repo
  - Memuat otomatis env var provider yang belum ada dari `~/.profile`
  - Secara bawaan mempersempit otomatis tiap suite ke provider yang saat ini memiliki auth yang dapat digunakan
  - Menggunakan ulang `scripts/test-live.mjs`, sehingga perilaku heartbeat dan mode senyap tetap konsisten
- Contoh:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Runner Docker (opsional, pemeriksaan "berfungsi di Linux")

Runner Docker ini terbagi menjadi dua kelompok:

- Runner live-model: `test:docker:live-models` dan `test:docker:live-gateway` hanya menjalankan file live profile-key yang cocok di dalam image Docker repo (`src/agents/models.profiles.live.test.ts` dan `src/gateway/gateway-models.profiles.live.test.ts`), dengan me-mount direktori config dan workspace lokal Anda (serta mengambil sumber `~/.profile` jika di-mount). Entrypoint lokal yang cocok adalah `test:live:models-profiles` dan `test:live:gateway-profiles`.
- Runner live Docker secara bawaan menggunakan batas smoke yang lebih kecil agar sapuan Docker penuh tetap praktis:
  `test:docker:live-models` secara bawaan menggunakan `OPENCLAW_LIVE_MAX_MODELS=12`, dan
  `test:docker:live-gateway` secara bawaan menggunakan `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`, dan
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Override env var tersebut saat Anda
  memang ingin pemindaian menyeluruh yang lebih besar.
- `test:docker:all` membangun image Docker live satu kali melalui `test:docker:live-build`, lalu menggunakannya kembali untuk dua lane Docker live tersebut.
- Runner smoke container: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels`, dan `test:docker:plugins` menyalakan satu atau lebih container nyata dan memverifikasi path integrasi tingkat lebih tinggi.

Runner Docker live-model juga melakukan bind-mount hanya pada home auth CLI yang diperlukan (atau semuanya yang didukung saat run tidak dipersempit), lalu menyalinnya ke home container sebelum run sehingga OAuth CLI eksternal dapat menyegarkan token tanpa memodifikasi penyimpanan auth host:

- Model langsung: `pnpm test:docker:live-models` (skrip: `scripts/test-live-models-docker.sh`)
- Smoke bind ACP: `pnpm test:docker:live-acp-bind` (skrip: `scripts/test-live-acp-bind-docker.sh`)
- Smoke backend CLI: `pnpm test:docker:live-cli-backend` (skrip: `scripts/test-live-cli-backend-docker.sh`)
- Smoke harness app-server Codex: `pnpm test:docker:live-codex-harness` (skrip: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + agen dev: `pnpm test:docker:live-gateway` (skrip: `scripts/test-live-gateway-models-docker.sh`)
- Smoke live Open WebUI: `pnpm test:docker:openwebui` (skrip: `scripts/e2e/openwebui-docker.sh`)
- Wizard onboarding (TTY, scaffolding penuh): `pnpm test:docker:onboard` (skrip: `scripts/e2e/onboard-docker.sh`)
- Jaringan Gateway (dua container, auth WS + health): `pnpm test:docker:gateway-network` (skrip: `scripts/e2e/gateway-network-docker.sh`)
- Bridge channel MCP (Gateway ber-seed + bridge stdio + smoke notification-frame Claude mentah): `pnpm test:docker:mcp-channels` (skrip: `scripts/e2e/mcp-channels-docker.sh`)
- Plugin (smoke instalasi + alias `/plugin` + semantik restart bundle Claude): `pnpm test:docker:plugins` (skrip: `scripts/e2e/plugins-docker.sh`)

Runner Docker live-model juga melakukan bind-mount checkout saat ini sebagai read-only dan
men-stage-nya ke workdir sementara di dalam container. Ini menjaga image runtime
tetap ramping sambil tetap menjalankan Vitest terhadap source/config lokal Anda yang tepat.
Langkah staging melewati cache lokal besar dan output build aplikasi seperti
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__`, dan direktori output `.build` lokal aplikasi atau
Gradle sehingga run live Docker tidak menghabiskan beberapa menit untuk menyalin
artefak khusus mesin.
Runner ini juga menetapkan `OPENCLAW_SKIP_CHANNELS=1` agar probe live Gateway tidak memulai
worker channel nyata Telegram/Discord/dll. di dalam container.
`test:docker:live-models` tetap menjalankan `pnpm test:live`, jadi teruskan juga
`OPENCLAW_LIVE_GATEWAY_*` saat Anda perlu mempersempit atau mengecualikan cakupan live
Gateway dari lane Docker tersebut.
`test:docker:openwebui` adalah smoke kompatibilitas tingkat lebih tinggi: ia memulai
container Gateway OpenClaw dengan endpoint HTTP yang kompatibel OpenAI diaktifkan,
memulai container Open WebUI yang dipin terhadap Gateway tersebut, sign in melalui
Open WebUI, memverifikasi `/api/models` mengekspos `openclaw/default`, lalu mengirim
permintaan chat nyata melalui proxy `/api/chat/completions` Open WebUI.
Run pertama dapat terasa jauh lebih lambat karena Docker mungkin perlu menarik
image Open WebUI dan Open WebUI mungkin perlu menyelesaikan setup cold-start-nya sendiri.
Lane ini mengharapkan kunci model live yang dapat digunakan, dan `OPENCLAW_PROFILE_FILE`
(`~/.profile` secara bawaan) adalah cara utama untuk menyediakannya pada run dalam Docker.
Run yang berhasil mencetak payload JSON kecil seperti `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` sengaja dibuat deterministik dan tidak memerlukan
akun Telegram, Discord, atau iMessage nyata. Ia menyalakan container Gateway
ber-seed, memulai container kedua yang menjalankan `openclaw mcp serve`, lalu
memverifikasi penemuan percakapan yang dirutekan, pembacaan transkrip, metadata lampiran,
perilaku antrean event live, routing pengiriman outbound, dan notifikasi channel +
izin gaya Claude melalui bridge MCP stdio nyata. Pemeriksaan notifikasi tersebut
menginspeksi frame MCP stdio mentah secara langsung sehingga smoke memvalidasi apa yang
sebenarnya dipancarkan bridge, bukan hanya apa yang kebetulan diekspos oleh SDK klien tertentu.

Smoke thread bahasa alami ACP manual (bukan CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Pertahankan skrip ini untuk alur kerja regresi/debug. Ini mungkin diperlukan lagi untuk validasi routing thread ACP, jadi jangan dihapus.

Env var yang berguna:

- `OPENCLAW_CONFIG_DIR=...` (bawaan: `~/.openclaw`) di-mount ke `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (bawaan: `~/.openclaw/workspace`) di-mount ke `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (bawaan: `~/.profile`) di-mount ke `/home/node/.profile` dan di-source sebelum menjalankan pengujian
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (bawaan: `~/.cache/openclaw/docker-cli-tools`) di-mount ke `/home/node/.npm-global` untuk instalasi CLI yang di-cache di dalam Docker
- Direktori/file auth CLI eksternal di bawah `$HOME` di-mount sebagai read-only di bawah `/host-auth...`, lalu disalin ke `/home/node/...` sebelum pengujian dimulai
  - Direktori bawaan: `.minimax`
  - File bawaan: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Run provider yang dipersempit hanya me-mount direktori/file yang diperlukan yang diinferensikan dari `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Override secara manual dengan `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none`, atau daftar dipisahkan koma seperti `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` untuk mempersempit run
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` untuk memfilter provider di dalam container
- `OPENCLAW_SKIP_DOCKER_BUILD=1` untuk menggunakan ulang image `openclaw:local-live` yang ada untuk rerun yang tidak memerlukan rebuild
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` untuk memastikan kredensial berasal dari profile store (bukan env)
- `OPENCLAW_OPENWEBUI_MODEL=...` untuk memilih model yang diekspos oleh Gateway untuk smoke Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` untuk override prompt nonce-check yang digunakan oleh smoke Open WebUI
- `OPENWEBUI_IMAGE=...` untuk override tag image Open WebUI yang dipin

## Sanitasi dokumen

Jalankan pemeriksaan dokumen setelah mengedit dokumen: `pnpm check:docs`.
Jalankan validasi anchor Mintlify penuh saat Anda juga membutuhkan pemeriksaan heading dalam halaman: `pnpm docs:check-links:anchors`.

## Regresi offline (aman untuk CI)

Ini adalah regresi “pipeline nyata” tanpa provider nyata:

- Pemanggilan tool Gateway (mock OpenAI, Gateway nyata + loop agen): `src/gateway/gateway.test.ts` (kasus: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Wizard Gateway (WS `wizard.start`/`wizard.next`, menulis config + auth ditegakkan): `src/gateway/gateway.test.ts` (kasus: "runs wizard over ws and writes auth token config")

## Evals keandalan agen (Skills)

Kami sudah memiliki beberapa pengujian aman-CI yang berperilaku seperti “evals keandalan agen”:

- Mock tool-calling melalui Gateway nyata + loop agen (`src/gateway/gateway.test.ts`).
- Alur wizard end-to-end yang memvalidasi wiring sesi dan efek config (`src/gateway/gateway.test.ts`).

Yang masih kurang untuk Skills (lihat [Skills](/id/tools/skills)):

- **Pengambilan keputusan:** saat Skills dicantumkan di prompt, apakah agen memilih skill yang tepat (atau menghindari yang tidak relevan)?
- **Kepatuhan:** apakah agen membaca `SKILL.md` sebelum digunakan dan mengikuti langkah/argumen yang diwajibkan?
- **Kontrak alur kerja:** skenario multi-giliran yang menegaskan urutan tool, carryover riwayat sesi, dan batas sandbox.

Evals mendatang sebaiknya tetap deterministik terlebih dahulu:

- Runner skenario yang menggunakan mock provider untuk menegaskan pemanggilan tool + urutan, pembacaan file skill, dan wiring sesi.
- Suite kecil skenario yang berfokus pada skill (gunakan vs hindari, gating, injeksi prompt).
- Evals live opsional (opt-in, digerbangkan env) hanya setelah suite aman-CI tersedia.

## Pengujian kontrak (bentuk plugin dan channel)

Pengujian kontrak memverifikasi bahwa setiap plugin dan channel terdaftar sesuai dengan
kontrak interface-nya. Mereka mengiterasi seluruh plugin yang ditemukan dan menjalankan suite
penegasan bentuk dan perilaku. Lane unit bawaan `pnpm test` dengan sengaja
melewati file seam dan smoke bersama ini; jalankan perintah kontrak secara eksplisit
saat Anda menyentuh surface channel atau provider bersama.

### Perintah

- Semua kontrak: `pnpm test:contracts`
- Hanya kontrak channel: `pnpm test:contracts:channels`
- Hanya kontrak provider: `pnpm test:contracts:plugins`

### Kontrak channel

Berada di `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Bentuk plugin dasar (id, nama, kapabilitas)
- **setup** - Kontrak wizard setup
- **session-binding** - Perilaku binding sesi
- **outbound-payload** - Struktur payload pesan
- **inbound** - Penanganan pesan masuk
- **actions** - Handler aksi channel
- **threading** - Penanganan ID thread
- **directory** - API direktori/roster
- **group-policy** - Penegakan kebijakan grup

### Kontrak status provider

Berada di `src/plugins/contracts/*.contract.test.ts`.

- **status** - Probe status channel
- **registry** - Bentuk registry plugin

### Kontrak provider

Berada di `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Kontrak alur auth
- **auth-choice** - Pilihan/seleksi auth
- **catalog** - API katalog model
- **discovery** - Penemuan plugin
- **loader** - Pemuatan plugin
- **runtime** - Runtime provider
- **shape** - Bentuk/interface plugin
- **wizard** - Wizard setup

### Kapan dijalankan

- Setelah mengubah ekspor atau subpath plugin-sdk
- Setelah menambahkan atau memodifikasi plugin channel atau provider
- Setelah merombak registrasi atau penemuan plugin

Pengujian kontrak berjalan di CI dan tidak memerlukan kunci API nyata.

## Menambahkan regresi (panduan)

Saat Anda memperbaiki masalah provider/model yang ditemukan dalam live:

- Tambahkan regresi aman-CI jika memungkinkan (mock/stub provider, atau tangkap transformasi bentuk permintaan yang tepat)
- Jika secara inheren hanya live (rate limit, kebijakan auth), buat pengujian live tetap sempit dan opt-in melalui env var
- Utamakan menargetkan lapisan terkecil yang menangkap bug:
  - bug konversi/replay permintaan provider → pengujian model langsung
  - bug pipeline sesi/riwayat/tool Gateway → smoke live Gateway atau pengujian mock Gateway aman-CI
- Guardrail traversal SecretRef:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` menurunkan satu target sampel per kelas SecretRef dari metadata registry (`listSecretTargetRegistryEntries()`), lalu menegaskan bahwa id exec segmen traversal ditolak.
  - Jika Anda menambahkan keluarga target SecretRef `includeInPlan` baru di `src/secrets/target-registry-data.ts`, perbarui `classifyTargetClass` dalam pengujian tersebut. Pengujian ini sengaja gagal pada id target yang tidak terklasifikasi sehingga kelas baru tidak dapat dilewati secara diam-diam.
