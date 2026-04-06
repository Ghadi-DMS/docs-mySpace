---
read_when:
    - Anda memerlukan perilaku terperinci untuk `openclaw onboard`
    - Anda sedang men-debug hasil onboarding atau mengintegrasikan klien onboarding
sidebarTitle: CLI reference
summary: Referensi lengkap untuk alur setup CLI, setup auth/model, output, dan detail internal
title: Referensi Setup CLI
x-i18n:
    generated_at: "2026-04-06T03:11:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: 92f379b34a2b48c68335dae4f759117c770f018ec51b275f4f40421c6b3abb23
    source_path: start/wizard-cli-reference.md
    workflow: 15
---

# Referensi Setup CLI

Halaman ini adalah referensi lengkap untuk `openclaw onboard`.
Untuk panduan singkat, lihat [Onboarding (CLI)](/id/start/wizard).

## Apa yang dilakukan wizard

Mode lokal (default) memandu Anda melalui:

- Setup model dan auth (OAuth subscription OpenAI Code, Anthropic Claude CLI atau API key, serta opsi MiniMax, GLM, Ollama, Moonshot, StepFun, dan AI Gateway)
- Lokasi workspace dan file bootstrap
- Pengaturan gateway (port, bind, auth, Tailscale)
- Channel dan provider (Telegram, WhatsApp, Discord, Google Chat, Mattermost, Signal, BlueBubbles, dan plugin channel bawaan lainnya)
- Instalasi daemon (LaunchAgent, systemd user unit, atau native Windows Scheduled Task dengan fallback Startup-folder)
- Pemeriksaan kesehatan
- Setup Skills

Mode remote mengonfigurasi mesin ini untuk terhubung ke gateway di tempat lain.
Mode ini tidak menginstal atau memodifikasi apa pun di host remote.

## Detail alur lokal

<Steps>
  <Step title="Deteksi config yang ada">
    - Jika `~/.openclaw/openclaw.json` ada, pilih Keep, Modify, atau Reset.
    - Menjalankan ulang wizard tidak menghapus apa pun kecuali Anda secara eksplisit memilih Reset (atau memberikan `--reset`).
    - `--reset` pada CLI default ke `config+creds+sessions`; gunakan `--reset-scope full` untuk juga menghapus workspace.
    - Jika config tidak valid atau berisi kunci legacy, wizard berhenti dan meminta Anda menjalankan `openclaw doctor` sebelum melanjutkan.
    - Reset menggunakan `trash` dan menawarkan cakupan:
      - Hanya config
      - Config + kredensial + sesi
      - Reset penuh (juga menghapus workspace)
  </Step>
  <Step title="Model dan auth">
    - Matriks opsi lengkap ada di [Opsi auth dan model](#opsi-auth-dan-model).
  </Step>
  <Step title="Workspace">
    - Default `~/.openclaw/workspace` (dapat dikonfigurasi).
    - Mengisi file workspace yang diperlukan untuk ritual bootstrap saat pertama kali dijalankan.
    - Tata letak workspace: [Workspace agen](/id/concepts/agent-workspace).
  </Step>
  <Step title="Gateway">
    - Meminta port, bind, mode auth, dan eksposur Tailscale.
    - Disarankan: tetap aktifkan auth token bahkan untuk loopback agar klien WS lokal harus melakukan autentikasi.
    - Dalam mode token, setup interaktif menawarkan:
      - **Buat/simpan token plaintext** (default)
      - **Gunakan SecretRef** (opt-in)
    - Dalam mode password, setup interaktif juga mendukung penyimpanan plaintext atau SecretRef.
    - Path SecretRef token non-interaktif: `--gateway-token-ref-env <ENV_VAR>`.
      - Memerlukan env var yang tidak kosong dalam lingkungan proses onboarding.
      - Tidak dapat digabungkan dengan `--gateway-token`.
    - Nonaktifkan auth hanya jika Anda sepenuhnya memercayai setiap proses lokal.
    - Bind non-loopback tetap memerlukan auth.
  </Step>
  <Step title="Channel">
    - [WhatsApp](/id/channels/whatsapp): login QR opsional
    - [Telegram](/id/channels/telegram): bot token
    - [Discord](/id/channels/discord): bot token
    - [Google Chat](/id/channels/googlechat): JSON service account + webhook audience
    - [Mattermost](/id/channels/mattermost): bot token + base URL
    - [Signal](/id/channels/signal): instalasi `signal-cli` opsional + config akun
    - [BlueBubbles](/id/channels/bluebubbles): direkomendasikan untuk iMessage; URL server + password + webhook
    - [iMessage](/id/channels/imessage): path `imsg` CLI legacy + akses DB
    - Keamanan DM: default-nya adalah pairing. DM pertama mengirim kode; setujui melalui
      `openclaw pairing approve <channel> <code>` atau gunakan allowlist.
  </Step>
  <Step title="Instalasi daemon">
    - macOS: LaunchAgent
      - Memerlukan sesi pengguna yang sedang login; untuk headless, gunakan LaunchDaemon kustom (tidak disertakan).
    - Linux dan Windows melalui WSL2: systemd user unit
      - Wizard mencoba `loginctl enable-linger <user>` agar gateway tetap aktif setelah logout.
      - Dapat meminta sudo (menulis ke `/var/lib/systemd/linger`); pertama-tama mencoba tanpa sudo.
    - Native Windows: Scheduled Task terlebih dahulu
      - Jika pembuatan task ditolak, OpenClaw fallback ke item login Startup-folder per pengguna dan langsung menyalakan gateway.
      - Scheduled Task tetap lebih disukai karena memberikan status supervisor yang lebih baik.
    - Pemilihan runtime: Node (disarankan; wajib untuk WhatsApp dan Telegram). Bun tidak disarankan.
  </Step>
  <Step title="Pemeriksaan kesehatan">
    - Menyalakan gateway (jika perlu) dan menjalankan `openclaw health`.
    - `openclaw status --deep` menambahkan probe kesehatan gateway live ke output status, termasuk probe channel bila didukung.
  </Step>
  <Step title="Skills">
    - Membaca Skills yang tersedia dan memeriksa persyaratan.
    - Memungkinkan Anda memilih manajer node: npm, pnpm, atau bun.
    - Menginstal dependency opsional (sebagian menggunakan Homebrew di macOS).
  </Step>
  <Step title="Selesai">
    - Ringkasan dan langkah selanjutnya, termasuk opsi app iOS, Android, dan macOS.
  </Step>
</Steps>

<Note>
Jika tidak ada GUI yang terdeteksi, wizard mencetak instruksi port-forward SSH untuk Control UI alih-alih membuka browser.
Jika aset Control UI tidak ada, wizard mencoba membangunnya; fallback-nya adalah `pnpm ui:build` (secara otomatis menginstal dependency UI).
</Note>

## Detail mode remote

Mode remote mengonfigurasi mesin ini untuk terhubung ke gateway di tempat lain.

<Info>
Mode remote tidak menginstal atau memodifikasi apa pun di host remote.
</Info>

Yang Anda tetapkan:

- URL gateway remote (`ws://...`)
- Token jika auth gateway remote diperlukan (disarankan)

<Note>
- Jika gateway hanya loopback, gunakan tunneling SSH atau tailnet.
- Petunjuk penemuan:
  - macOS: Bonjour (`dns-sd`)
  - Linux: Avahi (`avahi-browse`)
</Note>

## Opsi auth dan model

<AccordionGroup>
  <Accordion title="API key Anthropic">
    Menggunakan `ANTHROPIC_API_KEY` jika ada atau meminta key, lalu menyimpannya untuk penggunaan daemon.
  </Accordion>
  <Accordion title="Subscription OpenAI Code (penggunaan ulang Codex CLI)">
    Jika `~/.codex/auth.json` ada, wizard dapat menggunakannya kembali.
    Kredensial Codex CLI yang digunakan ulang tetap dikelola oleh Codex CLI; saat kedaluwarsa OpenClaw
    membaca ulang sumber itu terlebih dahulu dan, ketika provider dapat me-refresh-nya, menulis
    kredensial yang diperbarui kembali ke penyimpanan Codex alih-alih mengambil alih kepemilikannya
    sendiri.
  </Accordion>
  <Accordion title="Subscription OpenAI Code (OAuth)">
    Alur browser; tempel `code#state`.

    Menetapkan `agents.defaults.model` ke `openai-codex/gpt-5.4` saat model belum ditetapkan atau `openai/*`.

  </Accordion>
  <Accordion title="API key OpenAI">
    Menggunakan `OPENAI_API_KEY` jika ada atau meminta key, lalu menyimpan kredensial di profil auth.

    Menetapkan `agents.defaults.model` ke `openai/gpt-5.4` saat model belum ditetapkan, `openai/*`, atau `openai-codex/*`.

  </Accordion>
  <Accordion title="API key xAI (Grok)">
    Meminta `XAI_API_KEY` dan mengonfigurasi xAI sebagai provider model.
  </Accordion>
  <Accordion title="OpenCode">
    Meminta `OPENCODE_API_KEY` (atau `OPENCODE_ZEN_API_KEY`) dan memungkinkan Anda memilih katalog Zen atau Go.
    URL setup: [opencode.ai/auth](https://opencode.ai/auth).
  </Accordion>
  <Accordion title="API key (generik)">
    Menyimpan key untuk Anda.
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    Meminta `AI_GATEWAY_API_KEY`.
    Detail lebih lanjut: [Vercel AI Gateway](/id/providers/vercel-ai-gateway).
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    Meminta account ID, gateway ID, dan `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    Detail lebih lanjut: [Cloudflare AI Gateway](/id/providers/cloudflare-ai-gateway).
  </Accordion>
  <Accordion title="MiniMax">
    Config ditulis secara otomatis. Default hosted adalah `MiniMax-M2.7`; setup API key menggunakan
    `minimax/...`, dan setup OAuth menggunakan `minimax-portal/...`.
    Detail lebih lanjut: [MiniMax](/id/providers/minimax).
  </Accordion>
  <Accordion title="StepFun">
    Config ditulis secara otomatis untuk endpoint standar StepFun atau Step Plan pada endpoint China atau global.
    Standar saat ini mencakup `step-3.5-flash`, dan Step Plan juga mencakup `step-3.5-flash-2603`.
    Detail lebih lanjut: [StepFun](/id/providers/stepfun).
  </Accordion>
  <Accordion title="Synthetic (kompatibel dengan Anthropic)">
    Meminta `SYNTHETIC_API_KEY`.
    Detail lebih lanjut: [Synthetic](/id/providers/synthetic).
  </Accordion>
  <Accordion title="Ollama (Cloud dan open model lokal)">
    Meminta base URL (default `http://127.0.0.1:11434`), lalu menawarkan mode Cloud + Local atau Local.
    Menemukan model yang tersedia dan menyarankan default.
    Detail lebih lanjut: [Ollama](/id/providers/ollama).
  </Accordion>
  <Accordion title="Moonshot dan Kimi Coding">
    Config Moonshot (Kimi K2) dan Kimi Coding ditulis secara otomatis.
    Detail lebih lanjut: [Moonshot AI (Kimi + Kimi Coding)](/id/providers/moonshot).
  </Accordion>
  <Accordion title="Provider kustom">
    Berfungsi dengan endpoint yang kompatibel dengan OpenAI dan kompatibel dengan Anthropic.

    Onboarding interaktif mendukung pilihan penyimpanan API key yang sama seperti alur API key provider lainnya:
    - **Tempel API key sekarang** (plaintext)
    - **Gunakan secret reference** (env ref atau provider ref yang dikonfigurasi, dengan validasi preflight)

    Flag non-interaktif:
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key` (opsional; fallback ke `CUSTOM_API_KEY`)
    - `--custom-provider-id` (opsional)
    - `--custom-compatibility <openai|anthropic>` (opsional; default `openai`)

  </Accordion>
  <Accordion title="Lewati">
    Membiarkan auth tidak dikonfigurasi.
  </Accordion>
</AccordionGroup>

Perilaku model:

- Pilih model default dari opsi yang terdeteksi, atau masukkan provider dan model secara manual.
- Saat onboarding dimulai dari pilihan auth provider, pemilih model mengutamakan
  provider tersebut secara otomatis. Untuk Volcengine dan BytePlus, preferensi yang sama
  juga cocok dengan varian coding-plan mereka (`volcengine-plan/*`,
  `byteplus-plan/*`).
- Jika filter provider yang diprioritaskan itu kosong, pemilih akan fallback ke
  katalog penuh alih-alih tidak menampilkan model sama sekali.
- Wizard menjalankan pemeriksaan model dan memperingatkan jika model yang dikonfigurasi tidak dikenal atau auth tidak ada.

Path kredensial dan profil:

- Profil auth (API key + OAuth): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Impor OAuth legacy: `~/.openclaw/credentials/oauth.json`

Mode penyimpanan kredensial:

- Perilaku onboarding default menyimpan API key sebagai nilai plaintext di profil auth.
- `--secret-input-mode ref` mengaktifkan mode reference alih-alih penyimpanan key plaintext.
  Dalam setup interaktif, Anda dapat memilih:
  - env variable ref (misalnya `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)
  - provider ref yang dikonfigurasi (`file` atau `exec`) dengan alias + id provider
- Mode reference interaktif menjalankan validasi preflight cepat sebelum menyimpan.
  - Env refs: memvalidasi nama variabel + nilai tidak kosong dalam lingkungan onboarding saat ini.
  - Provider refs: memvalidasi config provider dan menyelesaikan id yang diminta.
  - Jika preflight gagal, onboarding menampilkan error dan memungkinkan Anda mencoba lagi.
- Dalam mode non-interaktif, `--secret-input-mode ref` hanya didukung berbasis env.
  - Tetapkan env var provider dalam lingkungan proses onboarding.
  - Flag key inline (misalnya `--openai-api-key`) mengharuskan env var tersebut ditetapkan; jika tidak onboarding akan gagal cepat.
  - Untuk provider kustom, mode `ref` non-interaktif menyimpan `models.providers.<id>.apiKey` sebagai `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`.
  - Dalam kasus provider kustom tersebut, `--custom-api-key` mengharuskan `CUSTOM_API_KEY` ditetapkan; jika tidak onboarding akan gagal cepat.
- Kredensial auth gateway mendukung pilihan plaintext dan SecretRef dalam setup interaktif:
  - Mode token: **Buat/simpan token plaintext** (default) atau **Gunakan SecretRef**.
  - Mode password: plaintext atau SecretRef.
- Path SecretRef token non-interaktif: `--gateway-token-ref-env <ENV_VAR>`.
- Setup plaintext yang ada tetap berfungsi tanpa perubahan.

<Note>
Tip headless dan server: selesaikan OAuth di mesin yang memiliki browser, lalu salin
`auth-profiles.json` milik agen tersebut (misalnya
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, atau path
`$OPENCLAW_STATE_DIR/...` yang sesuai) ke host gateway. `credentials/oauth.json`
hanya merupakan sumber impor legacy.
</Note>

## Output dan detail internal

Field yang umum di `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (jika Minimax dipilih)
- `tools.profile` (onboarding lokal default ke `"coding"` saat belum ditetapkan; nilai eksplisit yang sudah ada dipertahankan)
- `gateway.*` (mode, bind, auth, Tailscale)
- `session.dmScope` (onboarding lokal default-nya `per-channel-peer` saat belum ditetapkan; nilai eksplisit yang sudah ada dipertahankan)
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Allowlist channel (Slack, Discord, Matrix, Microsoft Teams) saat Anda melakukan opt-in selama prompt (nama diselesaikan ke ID bila memungkinkan)
- `skills.install.nodeManager`
  - Flag `setup --node-manager` menerima `npm`, `pnpm`, atau `bun`.
  - Config manual tetap dapat menetapkan `skills.install.nodeManager: "yarn"` nanti.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`openclaw agents add` menulis `agents.list[]` dan `bindings` opsional.

Kredensial WhatsApp berada di `~/.openclaw/credentials/whatsapp/<accountId>/`.
Sesi disimpan di `~/.openclaw/agents/<agentId>/sessions/`.

<Note>
Sebagian channel dikirim sebagai plugin. Saat dipilih selama setup, wizard
meminta untuk menginstal plugin tersebut (npm atau path lokal) sebelum konfigurasi channel.
</Note>

RPC wizard gateway:

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

Klien (app macOS dan Control UI) dapat merender langkah-langkah tanpa mengimplementasikan ulang logika onboarding.

Perilaku setup Signal:

- Mengunduh release asset yang sesuai
- Menyimpannya di `~/.openclaw/tools/signal-cli/<version>/`
- Menulis `channels.signal.cliPath` di config
- Build JVM memerlukan Java 21
- Build native digunakan bila tersedia
- Windows menggunakan WSL2 dan mengikuti alur `signal-cli` Linux di dalam WSL

## Dokumentasi terkait

- Pusat onboarding: [Onboarding (CLI)](/id/start/wizard)
- Otomasi dan skrip: [Otomasi CLI](/id/start/wizard-cli-automation)
- Referensi perintah: [`openclaw onboard`](/cli/onboard)
