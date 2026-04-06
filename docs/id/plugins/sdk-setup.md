---
read_when:
    - Anda sedang menambahkan wizard setup ke sebuah plugin
    - Anda perlu memahami `setup-entry.ts` vs `index.ts`
    - Anda sedang mendefinisikan schema config plugin atau metadata openclaw di `package.json`
sidebarTitle: Setup and Config
summary: Wizard setup, `setup-entry.ts`, schema config, dan metadata `package.json`
title: Setup dan Config Plugin
x-i18n:
    generated_at: "2026-04-06T03:09:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: eac2586516d27bcd94cc4c259fe6274c792b3f9938c7ddd6dbf04a6dbb988dc9
    source_path: plugins/sdk-setup.md
    workflow: 15
---

# Setup dan Config Plugin

Referensi untuk packaging plugin (metadata `package.json`), manifest
(`openclaw.plugin.json`), entri setup, dan schema config.

<Tip>
  **Mencari panduan langkah demi langkah?** Panduan cara pakai membahas packaging dalam konteks:
  [Plugin Channel](/id/plugins/sdk-channel-plugins#step-1-package-and-manifest) dan
  [Plugin Provider](/id/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## Metadata package

`package.json` Anda memerlukan field `openclaw` yang memberi tahu sistem plugin apa yang
disediakan plugin Anda:

**Plugin channel:**

```json
{
  "name": "@myorg/openclaw-my-channel",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "blurb": "Short description of the channel."
    }
  }
}
```

**Plugin provider / baseline publikasi ClawHub:**

```json openclaw-clawhub-package.json
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

Jika Anda memublikasikan plugin secara eksternal di ClawHub, field `compat` dan `build`
tersebut wajib ada. Snippet publikasi kanonis berada di
`docs/snippets/plugin-publish/`.

### Field `openclaw`

| Field        | Tipe       | Deskripsi                                                                                             |
| ------------ | ---------- | ----------------------------------------------------------------------------------------------------- |
| `extensions` | `string[]` | File entry point (relatif terhadap root package)                                                      |
| `setupEntry` | `string`   | Entri ringan khusus setup (opsional)                                                                  |
| `channel`    | `object`   | Metadata katalog channel untuk surface setup, picker, quickstart, dan status                          |
| `providers`  | `string[]` | ID provider yang didaftarkan oleh plugin ini                                                          |
| `install`    | `object`   | Petunjuk instalasi: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `allowInvalidConfigRecovery` |
| `startup`    | `object`   | Flag perilaku startup                                                                                 |

### `openclaw.channel`

`openclaw.channel` adalah metadata package yang ringan untuk penemuan channel dan
surface setup sebelum runtime dimuat.

| Field                                  | Tipe       | Artinya                                                                      |
| -------------------------------------- | ---------- | ---------------------------------------------------------------------------- |
| `id`                                   | `string`   | ID channel kanonis.                                                          |
| `label`                                | `string`   | Label channel utama.                                                         |
| `selectionLabel`                       | `string`   | Label picker/setup saat perlu berbeda dari `label`.                          |
| `detailLabel`                          | `string`   | Label detail sekunder untuk katalog channel dan surface status yang lebih kaya. |
| `docsPath`                             | `string`   | Path docs untuk tautan setup dan pemilihan.                                  |
| `docsLabel`                            | `string`   | Label override yang digunakan untuk tautan docs saat perlu berbeda dari id channel. |
| `blurb`                                | `string`   | Deskripsi singkat onboarding/katalog.                                        |
| `order`                                | `number`   | Urutan sortir di katalog channel.                                            |
| `aliases`                              | `string[]` | Alias pencarian tambahan untuk pemilihan channel.                            |
| `preferOver`                           | `string[]` | ID plugin/channel prioritas lebih rendah yang harus dikalahkan oleh channel ini. |
| `systemImage`                          | `string`   | Nama ikon/system-image opsional untuk katalog UI channel.                    |
| `selectionDocsPrefix`                  | `string`   | Teks awalan sebelum tautan docs di surface pemilihan.                        |
| `selectionDocsOmitLabel`               | `boolean`  | Tampilkan path docs secara langsung alih-alih tautan docs berlabel dalam copy pemilihan. |
| `selectionExtras`                      | `string[]` | String pendek tambahan yang ditambahkan dalam copy pemilihan.                |
| `markdownCapable`                      | `boolean`  | Menandai channel sebagai capable markdown untuk keputusan formatting keluar.  |
| `exposure`                             | `object`   | Kontrol visibilitas channel untuk setup, daftar terkonfigurasi, dan surface docs. |
| `quickstartAllowFrom`                  | `boolean`  | Memasukkan channel ini ke alur setup `allowFrom` quickstart standar.         |
| `forceAccountBinding`                  | `boolean`  | Wajibkan binding akun eksplisit meskipun hanya ada satu akun.                |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | Utamakan pencarian sesi saat menyelesaikan target announce untuk channel ini. |

Contoh:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "selectionLabel": "My Channel (self-hosted)",
      "detailLabel": "My Channel Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-based self-hosted chat integration.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guide:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` mendukung:

- `configured`: sertakan channel dalam surface daftar bergaya configured/status
- `setup`: sertakan channel dalam picker setup/configure interaktif
- `docs`: tandai channel sebagai terlihat publik di surface docs/navigasi

`showConfigured` dan `showInSetup` tetap didukung sebagai alias legacy. Sebaiknya gunakan
`exposure`.

### `openclaw.install`

`openclaw.install` adalah metadata package, bukan metadata manifest.

| Field                        | Tipe                 | Artinya                                                                          |
| ---------------------------- | -------------------- | -------------------------------------------------------------------------------- |
| `npmSpec`                    | `string`             | Spec npm kanonis untuk alur install/update.                                      |
| `localPath`                  | `string`             | Path instalasi lokal pengembangan atau bundled.                                  |
| `defaultChoice`              | `"npm"` \| `"local"` | Sumber instalasi yang dipilih saat keduanya tersedia.                            |
| `minHostVersion`             | `string`             | Versi minimum OpenClaw yang didukung dalam bentuk `>=x.y.z`.                     |
| `allowInvalidConfigRecovery` | `boolean`            | Memungkinkan alur instal ulang bundled-plugin memulihkan kegagalan config usang tertentu. |

Jika `minHostVersion` ditetapkan, instalasi dan pemuatan manifest-registry sama-sama menegakkannya.
Host yang lebih lama akan melewati plugin; string versi yang tidak valid akan ditolak.

`allowInvalidConfigRecovery` bukan bypass umum untuk config yang rusak. Field ini
khusus untuk pemulihan bundled-plugin yang sempit, sehingga instal ulang/setup dapat memperbaiki
sisa upgrade yang diketahui seperti path bundled plugin yang hilang atau entri `channels.<id>`
yang usang untuk plugin yang sama. Jika config rusak karena alasan yang tidak terkait, instalasi
tetap gagal tertutup dan memberi tahu operator untuk menjalankan `openclaw doctor --fix`.

### Penundaan full load

Plugin channel dapat memilih deferred loading dengan:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

Jika diaktifkan, OpenClaw hanya memuat `setupEntry` selama fase startup pra-listen,
bahkan untuk channel yang sudah dikonfigurasi. Entry penuh dimuat setelah
gateway mulai listen.

<Warning>
  Aktifkan deferred loading hanya jika `setupEntry` Anda mendaftarkan semua yang
  dibutuhkan gateway sebelum mulai listen (pendaftaran channel, route HTTP,
  metode gateway). Jika entry penuh memiliki kapabilitas startup yang wajib, biarkan
  perilaku default.
</Warning>

Jika setup/full entry Anda mendaftarkan metode gateway RPC, pertahankan pada
awalan khusus plugin. Namespace admin core yang dicadangkan (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) tetap dimiliki core dan selalu diselesaikan
ke `operator.admin`.

## Manifest plugin

Setiap plugin native harus menyertakan `openclaw.plugin.json` di root package.
OpenClaw menggunakannya untuk memvalidasi config tanpa mengeksekusi kode plugin.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

Untuk plugin channel, tambahkan `kind` dan `channels`:

```json
{
  "id": "my-channel",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Bahkan plugin tanpa config pun harus menyertakan schema. Schema kosong valid:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

Lihat [Manifest Plugin](/id/plugins/manifest) untuk referensi schema lengkap.

## Publikasi ClawHub

Untuk package plugin, gunakan perintah ClawHub yang khusus package:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

Alias publikasi legacy khusus skill hanya untuk Skills. Package plugin harus
selalu menggunakan `clawhub package publish`.

## Setup entry

File `setup-entry.ts` adalah alternatif ringan untuk `index.ts` yang
dimuat OpenClaw saat hanya memerlukan surface setup (onboarding, perbaikan config,
inspeksi channel nonaktif).

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

Ini menghindari pemuatan kode runtime berat (library kripto, pendaftaran CLI,
layanan latar belakang) selama alur setup.

**Saat OpenClaw menggunakan `setupEntry` alih-alih entry penuh:**

- Channel dinonaktifkan tetapi membutuhkan surface setup/onboarding
- Channel diaktifkan tetapi belum dikonfigurasi
- Deferred loading diaktifkan (`deferConfiguredChannelFullLoadUntilAfterListen`)

**Yang harus didaftarkan `setupEntry`:**

- Objek plugin channel (melalui `defineSetupPluginEntry`)
- Semua route HTTP yang diperlukan sebelum gateway listen
- Semua metode gateway yang diperlukan selama startup

Metode gateway startup tersebut tetap sebaiknya menghindari namespace admin core yang
dicadangkan seperti `config.*` atau `update.*`.

**Yang TIDAK seharusnya disertakan `setupEntry`:**

- Pendaftaran CLI
- Layanan latar belakang
- Import runtime berat (kripto, SDK)
- Metode gateway yang hanya diperlukan setelah startup

### Import helper setup yang sempit

Untuk path khusus setup yang hot, pilih seam helper setup yang lebih sempit daripada
payung `plugin-sdk/setup` yang lebih luas saat Anda hanya membutuhkan sebagian dari surface setup:

| Path import                        | Gunakan untuk                                                                            | Export utama                                                                                                                                                                                                                                                                                 |
| ---------------------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime`         | helper runtime pada waktu setup yang tetap tersedia di `setupEntry` / startup channel tertunda | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-adapter-runtime` | adapter setup akun yang sadar lingkungan                                                 | `createEnvPatchedAccountSetupAdapter`                                                                                                                                                                                                                                                        |
| `plugin-sdk/setup-tools`           | helper setup/install CLI/archive/docs                                                    | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                               |

Gunakan seam `plugin-sdk/setup` yang lebih luas saat Anda menginginkan seluruh
toolbox setup bersama, termasuk helper patch config seperti
`moveSingleAccountChannelSectionToDefaultAccount(...)`.

Adapter patch setup tetap aman untuk hot-path saat di-import. Pencarian contract-surface
promosi single-account bundled yang dimuat di dalamnya bersifat lazy, sehingga import
`plugin-sdk/setup-runtime` tidak secara eager memuat penemuan contract-surface bundled
sebelum adapter benar-benar digunakan.

### Promosi single-account yang dimiliki channel

Saat sebuah channel di-upgrade dari config top-level single-account menjadi
`channels.<id>.accounts.*`, perilaku bersama default adalah memindahkan nilai
yang di-scope ke akun yang dipromosikan ke `accounts.default`.

Bundled channel dapat mempersempit atau meng-override promosi tersebut melalui
contract surface setup mereka:

- `singleAccountKeysToMove`: kunci top-level tambahan yang harus dipindahkan ke
  akun yang dipromosikan
- `namedAccountPromotionKeys`: saat named account sudah ada, hanya kunci ini yang
  dipindahkan ke akun yang dipromosikan; kunci kebijakan/pengiriman bersama tetap berada di
  root channel
- `resolveSingleAccountPromotionTarget(...)`: pilih akun yang ada mana yang
  menerima nilai yang dipromosikan

Matrix adalah contoh bundled saat ini. Jika tepat satu akun Matrix bernama sudah ada,
atau jika `defaultAccount` menunjuk ke kunci non-kanonis yang sudah ada seperti
`Ops`, promosi mempertahankan akun tersebut alih-alih membuat entri baru
`accounts.default`.

## Schema config

Config plugin divalidasi terhadap JSON Schema di manifest Anda. Pengguna
mengonfigurasi plugin melalui:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

Plugin Anda menerima config ini sebagai `api.pluginConfig` saat pendaftaran.

Untuk config khusus channel, gunakan bagian config channel sebagai gantinya:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### Membangun schema config channel

Gunakan `buildChannelConfigSchema` dari `openclaw/plugin-sdk/core` untuk mengonversi
schema Zod menjadi wrapper `ChannelConfigSchema` yang divalidasi oleh OpenClaw:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/core";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

## Wizard setup

Plugin channel dapat menyediakan wizard setup interaktif untuk `openclaw onboard`.
Wizard ini adalah objek `ChannelSetupWizard` pada `ChannelPlugin`:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

Tipe `ChannelSetupWizard` mendukung `credentials`, `textInputs`,
`dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize`, dan lainnya.
Lihat package bundled plugin (misalnya plugin Discord `src/channel.setup.ts`) untuk
contoh lengkap.

Untuk prompt allowlist DM yang hanya memerlukan alur standar
`note -> prompt -> parse -> merge -> patch`, pilih helper setup bersama
dari `openclaw/plugin-sdk/setup`: `createPromptParsedAllowFromForAccount(...)`,
`createTopLevelChannelParsedAllowFromPrompt(...)`, dan
`createNestedChannelParsedAllowFromPrompt(...)`.

Untuk blok status setup channel yang hanya berbeda pada label, skor, dan baris
tambahan opsional, pilih `createStandardChannelSetupStatus(...)` dari
`openclaw/plugin-sdk/setup` alih-alih menulis sendiri objek `status` yang sama di
setiap plugin.

Untuk surface setup opsional yang seharusnya hanya muncul dalam konteks tertentu, gunakan
`createOptionalChannelSetupSurface` dari `openclaw/plugin-sdk/channel-setup`:

```typescript
import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

const setupSurface = createOptionalChannelSetupSurface({
  channel: "my-channel",
  label: "My Channel",
  npmSpec: "@myorg/openclaw-my-channel",
  docsPath: "/channels/my-channel",
});
// Returns { setupAdapter, setupWizard }
```

`plugin-sdk/channel-setup` juga mengekspos builder tingkat lebih rendah
`createOptionalChannelSetupAdapter(...)` dan
`createOptionalChannelSetupWizard(...)` saat Anda hanya membutuhkan satu bagian dari
surface install opsional tersebut.

Adapter/wizard opsional yang dihasilkan gagal tertutup pada penulisan config nyata. Keduanya
menggunakan ulang satu pesan wajib-instal di `validateInput`,
`applyAccountConfig`, dan `finalize`, serta menambahkan tautan docs saat `docsPath`
ditetapkan.

Untuk UI setup berbasis binary, pilih helper delegasi bersama alih-alih
menyalin glue binary/status yang sama ke setiap channel:

- `createDetectedBinaryStatus(...)` untuk blok status yang hanya berbeda pada label,
  petunjuk, skor, dan deteksi binary
- `createCliPathTextInput(...)` untuk input teks berbasis path
- `createDelegatedSetupWizardStatusResolvers(...)`,
  `createDelegatedPrepare(...)`, `createDelegatedFinalize(...)`, dan
  `createDelegatedResolveConfigured(...)` saat `setupEntry` perlu meneruskan ke
  wizard penuh yang lebih berat secara lazy
- `createDelegatedTextInputShouldPrompt(...)` saat `setupEntry` hanya perlu
  mendelegasikan keputusan `textInputs[*].shouldPrompt`

## Publikasi dan instalasi

**Plugin eksternal:** publikasikan ke [ClawHub](/id/tools/clawhub) atau npm, lalu instal:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

OpenClaw mencoba ClawHub terlebih dahulu dan otomatis fallback ke npm. Anda juga dapat
memaksa ClawHub secara eksplisit:

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin   # hanya ClawHub
```

Tidak ada override `npm:` yang sesuai. Gunakan spec package npm normal saat Anda
menginginkan path npm setelah fallback ClawHub:

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

**Plugin dalam repo:** tempatkan di bawah tree workspace bundled plugin dan plugin tersebut akan otomatis
ditemukan selama build.

**Pengguna dapat menginstal:**

```bash
openclaw plugins install <package-name>
```

<Info>
  Untuk instalasi yang bersumber dari npm, `openclaw plugins install` menjalankan
  `npm install --ignore-scripts` (tanpa lifecycle scripts). Pertahankan tree dependency plugin
  tetap JS/TS murni dan hindari package yang memerlukan build `postinstall`.
</Info>

## Terkait

- [Entry Point SDK](/id/plugins/sdk-entrypoints) -- `definePluginEntry` dan `defineChannelPluginEntry`
- [Manifest Plugin](/id/plugins/manifest) -- referensi schema manifest lengkap
- [Membangun Plugin](/id/plugins/building-plugins) -- panduan langkah demi langkah untuk memulai
