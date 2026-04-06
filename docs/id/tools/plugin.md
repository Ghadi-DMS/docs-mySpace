---
read_when:
    - Menginstal atau mengonfigurasi plugin
    - Memahami aturan penemuan dan pemuatan plugin
    - Bekerja dengan bundle plugin yang kompatibel dengan Codex/Claude
sidebarTitle: Install and Configure
summary: Menginstal, mengonfigurasi, dan mengelola plugin OpenClaw
title: Plugin
x-i18n:
    generated_at: "2026-04-06T03:12:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e2472a3023f3c1c6ee05b0cdc228f6b713cc226a08695b327de8a3ad6973c83
    source_path: tools/plugin.md
    workflow: 15
---

# Plugin

Plugin memperluas OpenClaw dengan kapabilitas baru: channel, provider model,
tool, skill, speech, realtime transcription, realtime voice,
media-understanding, image generation, video generation, web fetch, web
search, dan lainnya. Beberapa plugin bersifat **core** (disertakan bersama OpenClaw), lainnya
bersifat **external** (dipublikasikan di npm oleh komunitas).

## Mulai cepat

<Steps>
  <Step title="Lihat apa yang dimuat">
    ```bash
    openclaw plugins list
    ```
  </Step>

  <Step title="Instal plugin">
    ```bash
    # Dari npm
    openclaw plugins install @openclaw/voice-call

    # Dari direktori atau arsip lokal
    openclaw plugins install ./my-plugin
    openclaw plugins install ./my-plugin.tgz
    ```

  </Step>

  <Step title="Mulai ulang Gateway">
    ```bash
    openclaw gateway restart
    ```

    Lalu konfigurasikan di bawah `plugins.entries.\<id\>.config` dalam file config Anda.

  </Step>
</Steps>

Jika Anda lebih suka kontrol native chat, aktifkan `commands.plugins: true` lalu gunakan:

```text
/plugin install clawhub:@openclaw/voice-call
/plugin show voice-call
/plugin enable voice-call
```

Path instalasi menggunakan resolver yang sama dengan CLI: path/arsip lokal, spesifikasi
`clawhub:<pkg>` yang eksplisit, atau spesifikasi paket polos (ClawHub terlebih dahulu, lalu fallback ke npm).

Jika config tidak valid, instalasi biasanya gagal tertutup dan mengarahkan Anda ke
`openclaw doctor --fix`. Satu-satunya pengecualian pemulihan adalah jalur reinstall bundled-plugin yang sempit
untuk plugin yang ikut serta dalam
`openclaw.install.allowInvalidConfigRecovery`.

## Jenis plugin

OpenClaw mengenali dua format plugin:

| Format     | Cara kerjanya                                                      | Contoh                                                 |
| ---------- | ------------------------------------------------------------------ | ------------------------------------------------------ |
| **Native** | `openclaw.plugin.json` + modul runtime; dieksekusi dalam proses    | Plugin resmi, paket npm komunitas                      |
| **Bundle** | Tata letak yang kompatibel dengan Codex/Claude/Cursor; dipetakan ke fitur OpenClaw | `.codex-plugin/`, `.claude-plugin/`, `.cursor-plugin/` |

Keduanya muncul di `openclaw plugins list`. Lihat [Plugin Bundles](/id/plugins/bundles) untuk detail bundle.

Jika Anda menulis plugin native, mulai dari [Building Plugins](/id/plugins/building-plugins)
dan [Plugin SDK Overview](/id/plugins/sdk-overview).

## Plugin resmi

### Dapat diinstal (npm)

| Plugin          | Paket                  | Dokumen                              |
| --------------- | ---------------------- | ------------------------------------ |
| Matrix          | `@openclaw/matrix`     | [Matrix](/id/channels/matrix)           |
| Microsoft Teams | `@openclaw/msteams`    | [Microsoft Teams](/id/channels/msteams) |
| Nostr           | `@openclaw/nostr`      | [Nostr](/id/channels/nostr)             |
| Voice Call      | `@openclaw/voice-call` | [Voice Call](/id/plugins/voice-call)    |
| Zalo            | `@openclaw/zalo`       | [Zalo](/id/channels/zalo)               |
| Zalo Personal   | `@openclaw/zalouser`   | [Zalo Personal](/id/plugins/zalouser)   |

### Core (disertakan bersama OpenClaw)

<AccordionGroup>
  <Accordion title="Provider model (diaktifkan secara default)">
    `anthropic`, `byteplus`, `cloudflare-ai-gateway`, `github-copilot`, `google`,
    `huggingface`, `kilocode`, `kimi-coding`, `minimax`, `mistral`, `qwen`,
    `moonshot`, `nvidia`, `openai`, `opencode`, `opencode-go`, `openrouter`,
    `qianfan`, `synthetic`, `together`, `venice`,
    `vercel-ai-gateway`, `volcengine`, `xiaomi`, `zai`
  </Accordion>

  <Accordion title="Plugin memory">
    - `memory-core` — pencarian memory bawaan (default melalui `plugins.slots.memory`)
    - `memory-lancedb` — memory jangka panjang install-on-demand dengan auto-recall/capture (atur `plugins.slots.memory = "memory-lancedb"`)
  </Accordion>

  <Accordion title="Provider speech (diaktifkan secara default)">
    `elevenlabs`, `microsoft`
  </Accordion>

  <Accordion title="Lainnya">
    - `browser` — plugin browser bawaan untuk browser tool, CLI `openclaw browser`, metode gateway `browser.request`, runtime browser, dan layanan kontrol browser default (diaktifkan secara default; nonaktifkan sebelum menggantinya)
    - `copilot-proxy` — bridge VS Code Copilot Proxy (nonaktif secara default)
  </Accordion>
</AccordionGroup>

Mencari plugin pihak ketiga? Lihat [Community Plugins](/id/plugins/community).

## Konfigurasi

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-extension"] },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

| Field            | Deskripsi                                                |
| ---------------- | -------------------------------------------------------- |
| `enabled`        | Tombol master (default: `true`)                          |
| `allow`          | Allowlist plugin (opsional)                              |
| `deny`           | Denylist plugin (opsional; deny menang)                  |
| `load.paths`     | File/direktori plugin tambahan                           |
| `slots`          | Selector slot eksklusif (misalnya `memory`, `contextEngine`) |
| `entries.\<id\>` | Tombol per plugin + config                               |

Perubahan config **memerlukan restart gateway**. Jika Gateway berjalan dengan config
watch + restart dalam proses diaktifkan (jalur `openclaw gateway` default), restart
tersebut biasanya dilakukan secara otomatis sesaat setelah penulisan config selesai.

<Accordion title="Status plugin: dinonaktifkan vs hilang vs tidak valid">
  - **Dinonaktifkan**: plugin ada tetapi aturan enablement mematikannya. Config tetap dipertahankan.
  - **Hilang**: config mereferensikan ID plugin yang tidak ditemukan oleh discovery.
  - **Tidak valid**: plugin ada tetapi config-nya tidak cocok dengan skema yang dideklarasikan.
</Accordion>

## Discovery dan prioritas

OpenClaw memindai plugin dalam urutan ini (kecocokan pertama menang):

<Steps>
  <Step title="Path config">
    `plugins.load.paths` — path file atau direktori yang eksplisit.
  </Step>

  <Step title="Extension workspace">
    `\<workspace\>/.openclaw/<plugin-root>/*.ts` dan `\<workspace\>/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="Extension global">
    `~/.openclaw/<plugin-root>/*.ts` dan `~/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="Plugin bundled">
    Disertakan bersama OpenClaw. Banyak yang diaktifkan secara default (provider model, speech).
    Yang lain memerlukan pengaktifan eksplisit.
  </Step>
</Steps>

### Aturan enablement

- `plugins.enabled: false` menonaktifkan semua plugin
- `plugins.deny` selalu menang atas allow
- `plugins.entries.\<id\>.enabled: false` menonaktifkan plugin tersebut
- Plugin yang berasal dari workspace **nonaktif secara default** (harus diaktifkan secara eksplisit)
- Bundled plugin mengikuti set default-on bawaan kecuali di-override
- Slot eksklusif dapat memaksa plugin yang dipilih untuk slot tersebut menjadi aktif

## Slot plugin (kategori eksklusif)

Beberapa kategori bersifat eksklusif (hanya satu yang aktif pada satu waktu):

```json5
{
  plugins: {
    slots: {
      memory: "memory-core", // atau "none" untuk menonaktifkan
      contextEngine: "legacy", // atau ID plugin
    },
  },
}
```

| Slot            | Yang dikendalikan      | Default             |
| --------------- | ---------------------- | ------------------- |
| `memory`        | Plugin memory aktif    | `memory-core`       |
| `contextEngine` | Engine konteks aktif   | `legacy` (bawaan)   |

## Referensi CLI

```bash
openclaw plugins list                       # inventaris ringkas
openclaw plugins list --enabled            # hanya plugin yang dimuat
openclaw plugins list --verbose            # baris detail per plugin
openclaw plugins list --json               # inventaris yang dapat dibaca mesin
openclaw plugins inspect <id>              # detail mendalam
openclaw plugins inspect <id> --json       # dapat dibaca mesin
openclaw plugins inspect --all             # tabel seluruh armada
openclaw plugins info <id>                 # alias inspect
openclaw plugins doctor                    # diagnostik

openclaw plugins install <package>         # instal (ClawHub terlebih dahulu, lalu npm)
openclaw plugins install clawhub:<pkg>     # instal hanya dari ClawHub
openclaw plugins install <spec> --force    # timpa instalasi yang ada
openclaw plugins install <path>            # instal dari path lokal
openclaw plugins install -l <path>         # link (tanpa copy) untuk pengembangan
openclaw plugins install <plugin> --marketplace <source>
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <spec> --pin      # catat spesifikasi npm terselesaikan yang tepat
openclaw plugins install <spec> --dangerously-force-unsafe-install
openclaw plugins update <id>             # perbarui satu plugin
openclaw plugins update <id> --dangerously-force-unsafe-install
openclaw plugins update --all            # perbarui semua
openclaw plugins uninstall <id>          # hapus catatan config/instalasi
openclaw plugins uninstall <id> --keep-files
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json

openclaw plugins enable <id>
openclaw plugins disable <id>
```

Bundled plugin disertakan bersama OpenClaw. Banyak yang diaktifkan secara default (misalnya
bundled provider model, bundled provider speech, dan bundled browser
plugin). Bundled plugin lain tetap memerlukan `openclaw plugins enable <id>`.

`--force` menimpa plugin atau paket hook yang terinstal di tempat.
Opsi ini tidak didukung dengan `--link`, yang menggunakan kembali path sumber alih-alih
menyalin ke target instalasi yang dikelola.

`--pin` hanya untuk npm. Opsi ini tidak didukung dengan `--marketplace`, karena
instalasi marketplace menyimpan metadata sumber marketplace alih-alih spesifikasi npm.

`--dangerously-force-unsafe-install` adalah override darurat untuk false
positive dari pemindai kode berbahaya bawaan. Opsi ini memungkinkan alur instalasi
dan pembaruan plugin berlanjut melewati temuan bawaan `critical`, tetapi tetap
tidak melewati blok kebijakan `before_install` plugin atau pemblokiran karena kegagalan pemindaian.

Flag CLI ini hanya berlaku untuk alur instalasi/pembaruan plugin. Instalasi dependensi skill berbasis Gateway
menggunakan override permintaan `dangerouslyForceUnsafeInstall` yang sesuai, sedangkan `openclaw skills install`
tetap menjadi alur unduh/instalasi skill ClawHub yang terpisah.

Bundle yang kompatibel berpartisipasi dalam alur list/inspect/enable/disable plugin yang sama.
Dukungan runtime saat ini mencakup bundle skills, Claude command-skills,
default `settings.json` Claude, default Claude `.lsp.json` dan
`lspServers` yang dideklarasikan manifest, Cursor command-skills, dan direktori hook Codex yang kompatibel.

`openclaw plugins inspect <id>` juga melaporkan kapabilitas bundle yang terdeteksi serta
entri server MCP dan LSP yang didukung atau tidak didukung untuk plugin berbasis bundle.

Sumber marketplace dapat berupa nama known-marketplace Claude dari
`~/.claude/plugins/known_marketplaces.json`, root marketplace lokal atau path
`marketplace.json`, singkatan GitHub seperti `owner/repo`, URL repo GitHub,
atau URL git. Untuk marketplace remote, entri plugin harus tetap berada di dalam
repo marketplace yang dikloning dan hanya menggunakan sumber path relatif.

Lihat [referensi CLI `openclaw plugins`](/cli/plugins) untuk detail lengkap.

## Ikhtisar API plugin

Plugin native mengekspor objek entry yang mengekspos `register(api)`. Plugin
lama mungkin masih menggunakan `activate(api)` sebagai alias lama, tetapi plugin baru sebaiknya
menggunakan `register`.

```typescript
export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
    api.registerChannel({
      /* ... */
    });
  },
});
```

OpenClaw memuat objek entry dan memanggil `register(api)` selama aktivasi
plugin. Loader masih melakukan fallback ke `activate(api)` untuk plugin lama,
tetapi bundled plugin dan plugin eksternal baru sebaiknya memperlakukan `register` sebagai
kontrak publik.

Metode registrasi yang umum:

| Metode                                  | Yang didaftarkan           |
| --------------------------------------- | -------------------------- |
| `registerProvider`                      | Provider model (LLM)       |
| `registerChannel`                       | Channel chat               |
| `registerTool`                          | Tool agen                  |
| `registerHook` / `on(...)`              | Hook siklus hidup          |
| `registerSpeechProvider`                | Text-to-speech / STT       |
| `registerRealtimeTranscriptionProvider` | Streaming STT              |
| `registerRealtimeVoiceProvider`         | Suara realtime duplex      |
| `registerMediaUnderstandingProvider`    | Analisis gambar/audio      |
| `registerImageGenerationProvider`       | Image generation           |
| `registerMusicGenerationProvider`       | Music generation           |
| `registerVideoGenerationProvider`       | Video generation           |
| `registerWebFetchProvider`              | Provider web fetch / scrape |
| `registerWebSearchProvider`             | Pencarian web              |
| `registerHttpRoute`                     | Endpoint HTTP              |
| `registerCommand` / `registerCli`       | Perintah CLI               |
| `registerContextEngine`                 | Engine konteks             |
| `registerService`                       | Layanan latar belakang     |

Perilaku guard hook untuk hook siklus hidup bertipe:

- `before_tool_call`: `{ block: true }` bersifat terminal; handler prioritas lebih rendah dilewati.
- `before_tool_call`: `{ block: false }` adalah no-op dan tidak menghapus block sebelumnya.
- `before_install`: `{ block: true }` bersifat terminal; handler prioritas lebih rendah dilewati.
- `before_install`: `{ block: false }` adalah no-op dan tidak menghapus block sebelumnya.
- `message_sending`: `{ cancel: true }` bersifat terminal; handler prioritas lebih rendah dilewati.
- `message_sending`: `{ cancel: false }` adalah no-op dan tidak menghapus cancel sebelumnya.

Untuk perilaku hook bertipe lengkap, lihat [SDK Overview](/id/plugins/sdk-overview#hook-decision-semantics).

## Terkait

- [Building Plugins](/id/plugins/building-plugins) — membuat plugin Anda sendiri
- [Plugin Bundles](/id/plugins/bundles) — kompatibilitas bundle Codex/Claude/Cursor
- [Plugin Manifest](/id/plugins/manifest) — skema manifest
- [Registering Tools](/id/plugins/building-plugins#registering-agent-tools) — menambahkan tool agen dalam plugin
- [Plugin Internals](/id/plugins/architecture) — model kapabilitas dan pipeline pemuatan
- [Community Plugins](/id/plugins/community) — daftar pihak ketiga
