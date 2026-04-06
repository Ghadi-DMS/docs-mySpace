---
read_when:
    - Mencari langkah atau flag onboarding tertentu
    - Mengotomatiskan onboarding dengan mode non-interaktif
    - Men-debug perilaku onboarding
sidebarTitle: Onboarding Reference
summary: 'Referensi lengkap untuk onboarding CLI: setiap langkah, flag, dan field konfigurasi'
title: Referensi Onboarding
x-i18n:
    generated_at: "2026-04-06T03:11:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: e02a4da4a39ba335199095723f5d3b423671eb12efc2d9e4f9e48c1e8ee18419
    source_path: reference/wizard.md
    workflow: 15
---

# Referensi Onboarding

Ini adalah referensi lengkap untuk `openclaw onboard`.
Untuk ikhtisar tingkat tinggi, lihat [Onboarding (CLI)](/id/start/wizard).

## Detail alur (mode lokal)

<Steps>
  <Step title="Deteksi konfigurasi yang ada">
    - Jika `~/.openclaw/openclaw.json` ada, pilih **Keep / Modify / Reset**.
    - Menjalankan ulang onboarding **tidak** menghapus apa pun kecuali Anda secara eksplisit memilih **Reset**
      (atau memberikan `--reset`).
    - `--reset` pada CLI default ke `config+creds+sessions`; gunakan `--reset-scope full`
      untuk juga menghapus workspace.
    - Jika konfigurasi tidak valid atau berisi key lama, wizard berhenti dan meminta
      Anda menjalankan `openclaw doctor` sebelum melanjutkan.
    - Reset menggunakan `trash` (bukan `rm`) dan menawarkan cakupan:
      - Hanya konfigurasi
      - Konfigurasi + kredensial + sesi
      - Reset penuh (juga menghapus workspace)
  </Step>
  <Step title="Model/Auth">
    - **API key Anthropic**: menggunakan `ANTHROPIC_API_KEY` jika ada atau meminta key, lalu menyimpannya untuk penggunaan daemon.
    - **API key Anthropic**: pilihan asisten Anthropic yang diutamakan dalam onboarding/konfigurasi.
    - **Anthropic setup-token (lama/manual)**: tersedia lagi dalam onboarding/konfigurasi, tetapi Anthropic memberi tahu pengguna OpenClaw bahwa jalur login Claude OpenClaw dihitung sebagai penggunaan harness pihak ketiga dan memerlukan **Extra Usage** pada akun Claude.
    - **Langganan OpenAI Code (Codex) (Codex CLI)**: jika `~/.codex/auth.json` ada, onboarding dapat menggunakannya kembali. Kredensial Codex CLI yang digunakan kembali tetap dikelola oleh Codex CLI; saat kedaluwarsa OpenClaw akan membaca ulang sumber itu terlebih dahulu dan, ketika provider dapat me-refresh-nya, menulis kembali kredensial yang diperbarui ke penyimpanan Codex alih-alih mengambil alih kepemilikannya.
    - **Langganan OpenAI Code (Codex) (OAuth)**: alur browser; tempelkan `code#state`.
      - Mengatur `agents.defaults.model` ke `openai-codex/gpt-5.4` saat model belum diatur atau `openai/*`.
    - **API key OpenAI**: menggunakan `OPENAI_API_KEY` jika ada atau meminta key, lalu menyimpannya dalam profil auth.
      - Mengatur `agents.defaults.model` ke `openai/gpt-5.4` saat model belum diatur, `openai/*`, atau `openai-codex/*`.
    - **API key xAI (Grok)**: meminta `XAI_API_KEY` dan mengonfigurasi xAI sebagai provider model.
    - **OpenCode**: meminta `OPENCODE_API_KEY` (atau `OPENCODE_ZEN_API_KEY`, dapatkan di https://opencode.ai/auth) dan memungkinkan Anda memilih katalog Zen atau Go.
    - **Ollama**: meminta base URL Ollama, menawarkan mode **Cloud + Local** atau **Local**, menemukan model yang tersedia, dan otomatis menarik model lokal yang dipilih saat diperlukan.
    - Detail lebih lanjut: [Ollama](/id/providers/ollama)
    - **API key**: menyimpan key untuk Anda.
    - **Vercel AI Gateway (proxy multi-model)**: meminta `AI_GATEWAY_API_KEY`.
    - Detail lebih lanjut: [Vercel AI Gateway](/id/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: meminta Account ID, Gateway ID, dan `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    - Detail lebih lanjut: [Cloudflare AI Gateway](/id/providers/cloudflare-ai-gateway)
    - **MiniMax**: konfigurasi ditulis otomatis; default terhosting adalah `MiniMax-M2.7`.
      Penyiapan API key menggunakan `minimax/...`, dan penyiapan OAuth menggunakan
      `minimax-portal/...`.
    - Detail lebih lanjut: [MiniMax](/id/providers/minimax)
    - **StepFun**: konfigurasi ditulis otomatis untuk StepFun standar atau Step Plan pada endpoint China atau global.
    - Standar saat ini mencakup `step-3.5-flash`, dan Step Plan juga mencakup `step-3.5-flash-2603`.
    - Detail lebih lanjut: [StepFun](/id/providers/stepfun)
    - **Synthetic (kompatibel dengan Anthropic)**: meminta `SYNTHETIC_API_KEY`.
    - Detail lebih lanjut: [Synthetic](/id/providers/synthetic)
    - **Moonshot (Kimi K2)**: konfigurasi ditulis otomatis.
    - **Kimi Coding**: konfigurasi ditulis otomatis.
    - Detail lebih lanjut: [Moonshot AI (Kimi + Kimi Coding)](/id/providers/moonshot)
    - **Lewati**: belum ada auth yang dikonfigurasi.
    - Pilih model default dari opsi yang terdeteksi (atau masukkan provider/model secara manual). Untuk kualitas terbaik dan risiko injeksi prompt yang lebih rendah, pilih model generasi terbaru terkuat yang tersedia dalam stack provider Anda.
    - Onboarding menjalankan pemeriksaan model dan memperingatkan jika model yang dikonfigurasi tidak dikenal atau auth tidak ada.
    - Mode penyimpanan API key default ke nilai profil-auth plaintext. Gunakan `--secret-input-mode ref` untuk menyimpan ref yang didukung env sebagai gantinya (misalnya `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`).
    - Profil auth berada di `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (API key + OAuth). `~/.openclaw/credentials/oauth.json` adalah jalur impor lama saja.
    - Detail lebih lanjut: [/concepts/oauth](/id/concepts/oauth)
    <Note>
    Tip headless/server: selesaikan OAuth di mesin dengan browser, lalu salin
    `auth-profiles.json` agen tersebut (misalnya
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, atau path
    `$OPENCLAW_STATE_DIR/...` yang sesuai) ke host gateway. `credentials/oauth.json`
    hanya merupakan sumber impor lama.
    </Note>
  </Step>
  <Step title="Workspace">
    - Default `~/.openclaw/workspace` (dapat dikonfigurasi).
    - Menyiapkan file workspace yang diperlukan untuk ritual bootstrap agen.
    - Panduan layout workspace lengkap + cadangan: [Workspace agen](/id/concepts/agent-workspace)
  </Step>
  <Step title="Gateway">
    - Port, bind, mode auth, eksposur Tailscale.
    - Rekomendasi auth: tetap gunakan **Token** bahkan untuk loopback agar klien WS lokal tetap harus melakukan autentikasi.
    - Dalam mode token, penyiapan interaktif menawarkan:
      - **Hasilkan/simpan token plaintext** (default)
      - **Gunakan SecretRef** (opsional)
      - Quickstart menggunakan kembali SecretRef `gateway.auth.token` yang ada di provider `env`, `file`, dan `exec` untuk probe onboarding/bootstrap dashboard.
      - Jika SecretRef itu dikonfigurasi tetapi tidak dapat diselesaikan, onboarding gagal lebih awal dengan pesan perbaikan yang jelas alih-alih secara diam-diam menurunkan auth runtime.
    - Dalam mode kata sandi, penyiapan interaktif juga mendukung penyimpanan plaintext atau SecretRef.
    - Jalur SecretRef token non-interaktif: `--gateway-token-ref-env <ENV_VAR>`.
      - Memerlukan env var yang tidak kosong dalam environment proses onboarding.
      - Tidak dapat digabungkan dengan `--gateway-token`.
    - Nonaktifkan auth hanya jika Anda sepenuhnya mempercayai setiap proses lokal.
    - Bind non-loopback tetap memerlukan auth.
  </Step>
  <Step title="Channels">
    - [WhatsApp](/id/channels/whatsapp): login QR opsional.
    - [Telegram](/id/channels/telegram): token bot.
    - [Discord](/id/channels/discord): token bot.
    - [Google Chat](/id/channels/googlechat): JSON service account + audience webhook.
    - [Mattermost](/id/channels/mattermost) (plugin): token bot + base URL.
    - [Signal](/id/channels/signal): instalasi `signal-cli` opsional + konfigurasi akun.
    - [BlueBubbles](/id/channels/bluebubbles): **direkomendasikan untuk iMessage**; URL server + kata sandi + webhook.
    - [iMessage](/id/channels/imessage): jalur CLI `imsg` lama + akses DB.
    - Keamanan DM: default-nya adalah pairing. DM pertama mengirim kode; setujui melalui `openclaw pairing approve <channel> <code>` atau gunakan allowlist.
  </Step>
  <Step title="Pencarian web">
    - Pilih provider yang didukung seperti Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG, atau Tavily (atau lewati).
    - Provider berbasis API dapat menggunakan env var atau konfigurasi yang ada untuk penyiapan cepat; provider tanpa key menggunakan prasyarat spesifik providernya.
    - Lewati dengan `--skip-search`.
    - Konfigurasikan nanti: `openclaw configure --section web`.
  </Step>
  <Step title="Instalasi daemon">
    - macOS: LaunchAgent
      - Memerlukan sesi pengguna yang login; untuk headless, gunakan LaunchDaemon kustom (tidak disertakan).
    - Linux (dan Windows melalui WSL2): unit pengguna systemd
      - Onboarding mencoba mengaktifkan lingering melalui `loginctl enable-linger <user>` agar Gateway tetap berjalan setelah logout.
      - Mungkin meminta sudo (menulis ke `/var/lib/systemd/linger`); ia mencoba tanpa sudo terlebih dahulu.
    - **Pemilihan runtime:** Node (disarankan; wajib untuk WhatsApp/Telegram). Bun **tidak disarankan**.
    - Jika auth token memerlukan token dan `gateway.auth.token` dikelola SecretRef, instalasi daemon akan memvalidasinya tetapi tidak menyimpan nilai token plaintext yang telah diselesaikan ke metadata environment layanan supervisor.
    - Jika auth token memerlukan token dan SecretRef token yang dikonfigurasi tidak terselesaikan, instalasi daemon diblokir dengan panduan yang dapat ditindaklanjuti.
    - Jika `gateway.auth.token` dan `gateway.auth.password` keduanya dikonfigurasi dan `gateway.auth.mode` tidak diatur, instalasi daemon diblokir sampai mode diatur secara eksplisit.
  </Step>
  <Step title="Pemeriksaan kesehatan">
    - Memulai Gateway (jika diperlukan) dan menjalankan `openclaw health`.
    - Tip: `openclaw status --deep` menambahkan probe kesehatan gateway live ke output status, termasuk probe channel saat didukung (memerlukan gateway yang dapat dijangkau).
  </Step>
  <Step title="Skills (direkomendasikan)">
    - Membaca Skills yang tersedia dan memeriksa persyaratan.
    - Memungkinkan Anda memilih pengelola node: **npm / pnpm** (bun tidak disarankan).
    - Menginstal dependensi opsional (beberapa menggunakan Homebrew di macOS).
  </Step>
  <Step title="Selesai">
    - Ringkasan + langkah berikutnya, termasuk aplikasi iOS/Android/macOS untuk fitur tambahan.
  </Step>
</Steps>

<Note>
Jika tidak ada GUI yang terdeteksi, onboarding mencetak instruksi port-forward SSH untuk Control UI alih-alih membuka browser.
Jika aset Control UI tidak ada, onboarding mencoba membangunnya; fallback adalah `pnpm ui:build` (menginstal dependensi UI secara otomatis).
</Note>

## Mode non-interaktif

Gunakan `--non-interactive` untuk mengotomatiskan atau membuat skrip onboarding:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

Tambahkan `--json` untuk ringkasan yang dapat dibaca mesin.

Gateway token SecretRef dalam mode non-interaktif:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` dan `--gateway-token-ref-env` saling eksklusif.

<Note>
`--json` **tidak** menyiratkan mode non-interaktif. Gunakan `--non-interactive` (dan `--workspace`) untuk skrip.
</Note>

Contoh perintah spesifik provider ada di [Otomatisasi CLI](/id/start/wizard-cli-automation#provider-specific-examples).
Gunakan halaman referensi ini untuk semantik flag dan urutan langkah.

### Tambahkan agen (non-interaktif)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.4 \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

## RPC wizard Gateway

Gateway mengekspos alur onboarding melalui RPC (`wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status`).
Klien (aplikasi macOS, Control UI) dapat merender langkah tanpa mengimplementasikan ulang logika onboarding.

## Penyiapan Signal (signal-cli)

Onboarding dapat menginstal `signal-cli` dari rilis GitHub:

- Mengunduh aset rilis yang sesuai.
- Menyimpannya di bawah `~/.openclaw/tools/signal-cli/<version>/`.
- Menulis `channels.signal.cliPath` ke konfigurasi Anda.

Catatan:

- Build JVM memerlukan **Java 21**.
- Build native digunakan jika tersedia.
- Windows menggunakan WSL2; instalasi signal-cli mengikuti alur Linux di dalam WSL.

## Yang ditulis wizard

Field umum di `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (jika Minimax dipilih)
- `tools.profile` (onboarding lokal default ke `"coding"` saat belum diatur; nilai eksplisit yang sudah ada dipertahankan)
- `gateway.*` (mode, bind, auth, tailscale)
- `session.dmScope` (detail perilaku: [Referensi Penyiapan CLI](/id/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Allowlist channel (Slack/Discord/Matrix/Microsoft Teams) saat Anda memilihnya selama prompt (nama diselesaikan ke ID jika memungkinkan).
- `skills.install.nodeManager`
  - `setup --node-manager` menerima `npm`, `pnpm`, atau `bun`.
  - Konfigurasi manual masih dapat menggunakan `yarn` dengan mengatur `skills.install.nodeManager` secara langsung.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`openclaw agents add` menulis `agents.list[]` dan `bindings` opsional.

Kredensial WhatsApp berada di bawah `~/.openclaw/credentials/whatsapp/<accountId>/`.
Sesi disimpan di bawah `~/.openclaw/agents/<agentId>/sessions/`.

Beberapa channel dikirimkan sebagai plugin. Saat Anda memilih salah satunya selama penyiapan, onboarding
akan meminta untuk menginstalnya (npm atau path lokal) sebelum dapat dikonfigurasi.

## Dokumentasi terkait

- Ikhtisar onboarding: [Onboarding (CLI)](/id/start/wizard)
- Onboarding aplikasi macOS: [Onboarding](/id/start/onboarding)
- Referensi konfigurasi: [Konfigurasi Gateway](/id/gateway/configuration)
- Provider: [WhatsApp](/id/channels/whatsapp), [Telegram](/id/channels/telegram), [Discord](/id/channels/discord), [Google Chat](/id/channels/googlechat), [Signal](/id/channels/signal), [BlueBubbles](/id/channels/bluebubbles) (iMessage), [iMessage](/id/channels/imessage) (lama)
- Skills: [Skills](/id/tools/skills), [Konfigurasi Skills](/id/tools/skills-config)
