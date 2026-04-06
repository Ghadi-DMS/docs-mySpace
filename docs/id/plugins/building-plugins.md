---
read_when:
    - Anda ingin membuat plugin OpenClaw baru
    - Anda memerlukan panduan cepat untuk pengembangan plugin
    - Anda menambahkan channel, provider, tool, atau kapabilitas lain baru ke OpenClaw
sidebarTitle: Getting Started
summary: Buat plugin OpenClaw pertama Anda dalam hitungan menit
title: Membangun Plugin
x-i18n:
    generated_at: "2026-04-06T03:08:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9be344cb300ecbcba08e593a95bcc93ab16c14b28a0ff0c29b26b79d8249146c
    source_path: plugins/building-plugins.md
    workflow: 15
---

# Membangun Plugin

Plugin memperluas OpenClaw dengan kapabilitas baru: channel, provider model,
speech, transkripsi realtime, suara realtime, pemahaman media, pembuatan gambar,
pembuatan video, web fetch, web search, tool agent, atau kombinasi apa pun.

Anda tidak perlu menambahkan plugin Anda ke repositori OpenClaw. Publikasikan ke
[ClawHub](/id/tools/clawhub) atau npm dan pengguna menginstalnya dengan
`openclaw plugins install <package-name>`. OpenClaw mencoba ClawHub terlebih dahulu dan
secara otomatis fallback ke npm.

## Prasyarat

- Node >= 22 dan package manager (npm atau pnpm)
- Familiar dengan TypeScript (ESM)
- Untuk plugin di dalam repo: repositori sudah di-clone dan `pnpm install` sudah dijalankan

## Plugin jenis apa?

<CardGroup cols={3}>
  <Card title="Plugin channel" icon="messages-square" href="/id/plugins/sdk-channel-plugins">
    Hubungkan OpenClaw ke platform perpesanan (Discord, IRC, dll.)
  </Card>
  <Card title="Plugin provider" icon="cpu" href="/id/plugins/sdk-provider-plugins">
    Tambahkan provider model (LLM, proxy, atau endpoint kustom)
  </Card>
  <Card title="Plugin tool / hook" icon="wrench">
    Daftarkan tool agent, event hook, atau layanan — lanjutkan di bawah
  </Card>
</CardGroup>

Jika plugin channel bersifat opsional dan mungkin belum terpasang ketika onboarding/penyiapan
dijalankan, gunakan `createOptionalChannelSetupSurface(...)` dari
`openclaw/plugin-sdk/channel-setup`. Ini menghasilkan adapter penyiapan + pasangan wizard
yang mengumumkan kebutuhan instalasi dan gagal secara tertutup pada penulisan config nyata
sampai plugin terpasang.

## Mulai cepat: plugin tool

Panduan ini membuat plugin minimal yang mendaftarkan tool agent. Plugin channel
dan provider memiliki panduan khusus yang ditautkan di atas.

<Steps>
  <Step title="Buat package dan manifest">
    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "my-plugin",
      "name": "My Plugin",
      "description": "Adds a custom tool to OpenClaw",
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    Setiap plugin memerlukan manifest, bahkan tanpa config. Lihat
    [Manifest](/id/plugins/manifest) untuk skema lengkapnya. Snippet publikasi ClawHub kanonis
    berada di `docs/snippets/plugin-publish/`.

  </Step>

  <Step title="Tulis entry point">

    ```typescript
    // index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { Type } from "@sinclair/typebox";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Do a thing",
          parameters: Type.Object({ input: Type.String() }),
          async execute(_id, params) {
            return { content: [{ type: "text", text: `Got: ${params.input}` }] };
          },
        });
      },
    });
    ```

    `definePluginEntry` digunakan untuk plugin non-channel. Untuk channel, gunakan
    `defineChannelPluginEntry` — lihat [Plugin Channel](/id/plugins/sdk-channel-plugins).
    Untuk opsi entry point lengkap, lihat [Entry Points](/id/plugins/sdk-entrypoints).

  </Step>

  <Step title="Uji dan publikasikan">

    **Plugin eksternal:** validasikan dan publikasikan dengan ClawHub, lalu instal:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```

    OpenClaw juga memeriksa ClawHub sebelum npm untuk spesifikasi package polos seperti
    `@myorg/openclaw-my-plugin`.

    **Plugin di dalam repo:** tempatkan di bawah pohon workspace plugin bawaan — akan ditemukan secara otomatis.

    ```bash
    pnpm test -- <bundled-plugin-root>/my-plugin/
    ```

  </Step>
</Steps>

## Kapabilitas plugin

Satu plugin dapat mendaftarkan sejumlah kapabilitas melalui objek `api`:

| Kapabilitas           | Metode pendaftaran                              | Panduan terperinci                                                             |
| --------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------ |
| Inferensi teks (LLM)  | `api.registerProvider(...)`                     | [Plugin Provider](/id/plugins/sdk-provider-plugins)                               |
| Channel / perpesanan  | `api.registerChannel(...)`                      | [Plugin Channel](/id/plugins/sdk-channel-plugins)                                 |
| Speech (TTS/STT)      | `api.registerSpeechProvider(...)`               | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Transkripsi realtime  | `api.registerRealtimeTranscriptionProvider(...)` | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Suara realtime        | `api.registerRealtimeVoiceProvider(...)`        | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Pemahaman media       | `api.registerMediaUnderstandingProvider(...)`   | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Pembuatan gambar      | `api.registerImageGenerationProvider(...)`      | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Pembuatan musik       | `api.registerMusicGenerationProvider(...)`      | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Pembuatan video       | `api.registerVideoGenerationProvider(...)`      | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Web fetch             | `api.registerWebFetchProvider(...)`             | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Web search            | `api.registerWebSearchProvider(...)`            | [Plugin Provider](/id/plugins/sdk-provider-plugins#step-5-add-extra-capabilities) |
| Tool agent            | `api.registerTool(...)`                         | Di bawah                                                                       |
| Perintah kustom       | `api.registerCommand(...)`                      | [Entry Points](/id/plugins/sdk-entrypoints)                                       |
| Event hook            | `api.registerHook(...)`                         | [Entry Points](/id/plugins/sdk-entrypoints)                                       |
| Rute HTTP             | `api.registerHttpRoute(...)`                    | [Internals](/id/plugins/architecture#gateway-http-routes)                         |
| Subperintah CLI       | `api.registerCli(...)`                          | [Entry Points](/id/plugins/sdk-entrypoints)                                       |

Untuk API pendaftaran lengkap, lihat [Ikhtisar SDK](/id/plugins/sdk-overview#registration-api).

Jika plugin Anda mendaftarkan metode RPC gateway kustom, simpan metode tersebut pada
prefix khusus plugin. Namespace admin inti (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) tetap dicadangkan dan selalu diselesaikan ke
`operator.admin`, meskipun plugin meminta cakupan yang lebih sempit.

Semantik guard hook yang perlu diingat:

- `before_tool_call`: `{ block: true }` bersifat terminal dan menghentikan handler prioritas lebih rendah.
- `before_tool_call`: `{ block: false }` diperlakukan sebagai tidak ada keputusan.
- `before_tool_call`: `{ requireApproval: true }` menjeda eksekusi agent dan meminta persetujuan pengguna melalui overlay persetujuan exec, tombol Telegram, interaksi Discord, atau perintah `/approve` di channel mana pun.
- `before_install`: `{ block: true }` bersifat terminal dan menghentikan handler prioritas lebih rendah.
- `before_install`: `{ block: false }` diperlakukan sebagai tidak ada keputusan.
- `message_sending`: `{ cancel: true }` bersifat terminal dan menghentikan handler prioritas lebih rendah.
- `message_sending`: `{ cancel: false }` diperlakukan sebagai tidak ada keputusan.

Perintah `/approve` menangani persetujuan exec dan plugin dengan fallback terbatas: ketika id persetujuan exec tidak ditemukan, OpenClaw mencoba ulang id yang sama melalui persetujuan plugin. Penerusan persetujuan plugin dapat dikonfigurasi secara independen melalui `approvals.plugin` di config.

Jika plumbing persetujuan kustom perlu mendeteksi kasus fallback terbatas yang sama,
gunakan `isApprovalNotFoundError` dari `openclaw/plugin-sdk/error-runtime`
alih-alih mencocokkan string kedaluwarsa persetujuan secara manual.

Lihat [Semantik keputusan hook Ikhtisar SDK](/id/plugins/sdk-overview#hook-decision-semantics) untuk detailnya.

## Mendaftarkan tool agent

Tool adalah fungsi bertipe yang dapat dipanggil LLM. Tool dapat bersifat wajib (selalu
tersedia) atau opsional (ikut serta oleh pengguna):

```typescript
register(api) {
  // Tool wajib — selalu tersedia
  api.registerTool({
    name: "my_tool",
    description: "Do a thing",
    parameters: Type.Object({ input: Type.String() }),
    async execute(_id, params) {
      return { content: [{ type: "text", text: params.input }] };
    },
  });

  // Tool opsional — pengguna harus menambahkannya ke allowlist
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      async execute(_id, params) {
        return { content: [{ type: "text", text: params.pipeline }] };
      },
    },
    { optional: true },
  );
}
```

Pengguna mengaktifkan tool opsional di config:

```json5
{
  tools: { allow: ["workflow_tool"] },
}
```

- Nama tool tidak boleh bentrok dengan tool inti (konflik akan dilewati)
- Gunakan `optional: true` untuk tool dengan efek samping atau kebutuhan biner tambahan
- Pengguna dapat mengaktifkan semua tool dari suatu plugin dengan menambahkan id plugin ke `tools.allow`

## Konvensi impor

Selalu impor dari path `openclaw/plugin-sdk/<subpath>` yang terfokus:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";

// Salah: root monolitik (deprecated, akan dihapus)
import { ... } from "openclaw/plugin-sdk";
```

Untuk referensi subpath lengkap, lihat [Ikhtisar SDK](/id/plugins/sdk-overview).

Di dalam plugin Anda, gunakan file barrel lokal (`api.ts`, `runtime-api.ts`) untuk
impor internal — jangan pernah mengimpor plugin Anda sendiri melalui path SDK-nya.

Untuk plugin provider, simpan helper khusus provider di barrel akar package tersebut
kecuali jika seam-nya benar-benar generik. Contoh bawaan saat ini:

- Anthropic: wrapper stream Claude dan helper `service_tier` / beta
- OpenAI: builder provider, helper model default, provider realtime
- OpenRouter: builder provider plus helper onboarding/config

Jika helper hanya berguna di dalam satu package provider bawaan, simpan helper itu pada
seam akar package tersebut alih-alih menaikkannya ke `openclaw/plugin-sdk/*`.

Beberapa seam helper `openclaw/plugin-sdk/<bundled-id>` yang dihasilkan masih ada untuk
pemeliharaan dan kompatibilitas plugin bawaan, misalnya
`plugin-sdk/feishu-setup` atau `plugin-sdk/zalo-setup`. Perlakukan seam tersebut sebagai
permukaan yang dicadangkan, bukan sebagai pola default untuk plugin pihak ketiga baru.

## Checklist sebelum pengiriman

<Check>**package.json** memiliki metadata `openclaw` yang benar</Check>
<Check>Manifest **openclaw.plugin.json** ada dan valid</Check>
<Check>Entry point menggunakan `defineChannelPluginEntry` atau `definePluginEntry`</Check>
<Check>Semua impor menggunakan path `plugin-sdk/<subpath>` yang terfokus</Check>
<Check>Impor internal menggunakan modul lokal, bukan self-import SDK</Check>
<Check>Pengujian lulus (`pnpm test -- <bundled-plugin-root>/my-plugin/`)</Check>
<Check>`pnpm check` lulus (plugin di dalam repo)</Check>

## Pengujian Rilis Beta

1. Pantau tag rilis GitHub di [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) dan berlangganan melalui `Watch` > `Releases`. Tag beta terlihat seperti `v2026.3.N-beta.1`. Anda juga dapat mengaktifkan notifikasi untuk akun X resmi OpenClaw [@openclaw](https://x.com/openclaw) untuk pengumuman rilis.
2. Uji plugin Anda terhadap tag beta segera setelah tag tersebut muncul. Jendela sebelum stabil biasanya hanya beberapa jam.
3. Posting di thread plugin Anda di channel Discord `plugin-forum` setelah pengujian dengan `all good` atau apa yang rusak. Jika Anda belum memiliki thread, buat satu.
4. Jika ada yang rusak, buka atau perbarui issue berjudul `Beta blocker: <plugin-name> - <summary>` dan terapkan label `beta-blocker`. Cantumkan tautan issue di thread Anda.
5. Buka PR ke `main` berjudul `fix(<plugin-id>): beta blocker - <summary>` dan tautkan issue tersebut baik di PR maupun di thread Discord Anda. Kontributor tidak dapat memberi label pada PR, jadi judul adalah sinyal sisi-PR untuk maintainer dan otomasi. Blocker yang memiliki PR akan digabungkan; blocker tanpa PR mungkin tetap dirilis. Maintainer memantau thread ini selama pengujian beta.
6. Diam berarti hijau. Jika Anda melewatkan jendela ini, perbaikan Anda kemungkinan masuk ke siklus berikutnya.

## Langkah berikutnya

<CardGroup cols={2}>
  <Card title="Plugin Channel" icon="messages-square" href="/id/plugins/sdk-channel-plugins">
    Bangun plugin channel perpesanan
  </Card>
  <Card title="Plugin Provider" icon="cpu" href="/id/plugins/sdk-provider-plugins">
    Bangun plugin provider model
  </Card>
  <Card title="Ikhtisar SDK" icon="book-open" href="/id/plugins/sdk-overview">
    Peta impor dan referensi API pendaftaran
  </Card>
  <Card title="Helper Runtime" icon="settings" href="/id/plugins/sdk-runtime">
    TTS, pencarian, subagent via api.runtime
  </Card>
  <Card title="Pengujian" icon="test-tubes" href="/id/plugins/sdk-testing">
    Utilitas dan pola pengujian
  </Card>
  <Card title="Manifest Plugin" icon="file-json" href="/id/plugins/manifest">
    Referensi skema manifest lengkap
  </Card>
</CardGroup>

## Terkait

- [Arsitektur Plugin](/id/plugins/architecture) — pendalaman arsitektur internal
- [Ikhtisar SDK](/id/plugins/sdk-overview) — referensi Plugin SDK
- [Manifest](/id/plugins/manifest) — format manifest plugin
- [Plugin Channel](/id/plugins/sdk-channel-plugins) — membangun plugin channel
- [Plugin Provider](/id/plugins/sdk-provider-plugins) — membangun plugin provider
