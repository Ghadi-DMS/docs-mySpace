---
read_when:
    - Anda sedang membangun plugin OpenClaw
    - Anda perlu mengirimkan skema konfigurasi plugin atau men-debug kesalahan validasi plugin
summary: Persyaratan manifest plugin + skema JSON (validasi konfigurasi ketat)
title: Manifest Plugin
x-i18n:
    generated_at: "2026-04-11T15:15:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 42d454b560a8f6bf714c5d782f34216be1216d83d0a319d08d7349332c91a9e4
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifest plugin (`openclaw.plugin.json`)

Halaman ini hanya untuk **manifest plugin OpenClaw native**.

Untuk tata letak bundle yang kompatibel, lihat [Bundle plugin](/id/plugins/bundles).

Format bundle yang kompatibel menggunakan file manifest yang berbeda:

- Bundle Codex: `.codex-plugin/plugin.json`
- Bundle Claude: `.claude-plugin/plugin.json` atau tata letak komponen Claude
  default tanpa manifest
- Bundle Cursor: `.cursor-plugin/plugin.json`

OpenClaw juga mendeteksi otomatis tata letak bundle tersebut, tetapi tidak divalidasi
terhadap skema `openclaw.plugin.json` yang dijelaskan di sini.

Untuk bundle yang kompatibel, OpenClaw saat ini membaca metadata bundle plus root
skill yang dideklarasikan, root perintah Claude, default Claude bundle `settings.json`,
default LSP bundle Claude, dan paket hook yang didukung saat tata letaknya cocok
dengan ekspektasi runtime OpenClaw.

Setiap plugin OpenClaw native **harus** menyertakan file `openclaw.plugin.json` di
**root plugin**. OpenClaw menggunakan manifest ini untuk memvalidasi konfigurasi
**tanpa mengeksekusi kode plugin**. Manifest yang hilang atau tidak valid dianggap
sebagai kesalahan plugin dan memblokir validasi konfigurasi.

Lihat panduan lengkap sistem plugin: [Plugin](/id/tools/plugin).
Untuk model kapabilitas native dan panduan kompatibilitas eksternal saat ini:
[Model kapabilitas](/id/plugins/architecture#public-capability-model).

## Fungsi file ini

`openclaw.plugin.json` adalah metadata yang dibaca OpenClaw sebelum memuat kode
plugin Anda.

Gunakan ini untuk:

- identitas plugin
- validasi konfigurasi
- metadata autentikasi dan onboarding yang harus tersedia tanpa menyalakan runtime
  plugin
- petunjuk aktivasi ringan yang dapat diperiksa oleh surface control-plane sebelum
  runtime dimuat
- deskriptor penyiapan ringan yang dapat diperiksa oleh surface setup/onboarding
  sebelum runtime dimuat
- metadata alias dan auto-enable yang harus di-resolve sebelum runtime plugin dimuat
- metadata kepemilikan shorthand model-family yang harus mengaktifkan plugin secara
  otomatis sebelum runtime dimuat
- snapshot kepemilikan kapabilitas statis yang digunakan untuk wiring kompatibilitas
  bundel dan cakupan kontrak
- metadata konfigurasi khusus channel yang harus digabungkan ke dalam katalog dan
  surface validasi tanpa memuat runtime
- petunjuk UI konfigurasi

Jangan gunakan ini untuk:

- mendaftarkan perilaku runtime
- mendeklarasikan entrypoint kode
- metadata instalasi npm

Hal-hal tersebut termasuk dalam kode plugin Anda dan `package.json`.

## Contoh minimal

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Contoh lengkap

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter provider plugin",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API key",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API key",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Referensi field tingkat atas

| Field                               | Wajib    | Tipe                             | Artinya                                                                                                                                                                                                      |
| ----------------------------------- | -------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                | Ya       | `string`                         | ID plugin kanonis. Ini adalah ID yang digunakan dalam `plugins.entries.<id>`.                                                                                                                               |
| `configSchema`                      | Ya       | `object`                         | JSON Schema inline untuk konfigurasi plugin ini.                                                                                                                                                             |
| `enabledByDefault`                  | Tidak    | `true`                           | Menandai plugin bundle sebagai aktif secara default. Hilangkan field ini, atau tetapkan nilai apa pun selain `true`, untuk membiarkan plugin nonaktif secara default.                                     |
| `legacyPluginIds`                   | Tidak    | `string[]`                       | ID lama yang dinormalisasi ke ID plugin kanonis ini.                                                                                                                                                         |
| `autoEnableWhenConfiguredProviders` | Tidak    | `string[]`                       | ID provider yang harus mengaktifkan plugin ini secara otomatis saat autentikasi, konfigurasi, atau referensi model menyebutkannya.                                                                          |
| `kind`                              | Tidak    | `"memory"` \| `"context-engine"` | Mendeklarasikan jenis plugin eksklusif yang digunakan oleh `plugins.slots.*`.                                                                                                                               |
| `channels`                          | Tidak    | `string[]`                       | ID channel yang dimiliki oleh plugin ini. Digunakan untuk discovery dan validasi konfigurasi.                                                                                                               |
| `providers`                         | Tidak    | `string[]`                       | ID provider yang dimiliki oleh plugin ini.                                                                                                                                                                   |
| `modelSupport`                      | Tidak    | `object`                         | Metadata shorthand model-family milik manifest yang digunakan untuk memuat otomatis plugin sebelum runtime.                                                                                                 |
| `cliBackends`                       | Tidak    | `string[]`                       | ID backend inferensi CLI yang dimiliki oleh plugin ini. Digunakan untuk auto-activation saat startup dari referensi konfigurasi eksplisit.                                                                 |
| `commandAliases`                    | Tidak    | `object[]`                       | Nama perintah yang dimiliki oleh plugin ini yang harus menghasilkan konfigurasi sadar-plugin dan diagnostik CLI sebelum runtime dimuat.                                                                     |
| `providerAuthEnvVars`               | Tidak    | `Record<string, string[]>`       | Metadata env autentikasi provider ringan yang dapat diperiksa OpenClaw tanpa memuat kode plugin.                                                                                                            |
| `providerAuthAliases`               | Tidak    | `Record<string, string>`         | ID provider yang harus menggunakan kembali ID provider lain untuk pencarian autentikasi, misalnya provider coding yang berbagi API key provider dasar dan profil autentikasi.                              |
| `channelEnvVars`                    | Tidak    | `Record<string, string[]>`       | Metadata env channel ringan yang dapat diperiksa OpenClaw tanpa memuat kode plugin. Gunakan ini untuk surface penyiapan channel atau autentikasi berbasis env yang harus terlihat oleh helper startup/config generik. |
| `providerAuthChoices`               | Tidak    | `object[]`                       | Metadata pilihan autentikasi ringan untuk pemilih onboarding, resolusi preferred-provider, dan wiring flag CLI sederhana.                                                                                  |
| `activation`                        | Tidak    | `object`                         | Petunjuk aktivasi ringan untuk pemuatan yang dipicu provider, perintah, channel, route, dan kapabilitas. Hanya metadata; runtime plugin tetap memiliki perilaku yang sebenarnya.                          |
| `setup`                             | Tidak    | `object`                         | Deskriptor setup/onboarding ringan yang dapat diperiksa oleh surface discovery dan setup tanpa memuat runtime plugin.                                                                                      |
| `contracts`                         | Tidak    | `object`                         | Snapshot kapabilitas bundle statis untuk kepemilikan speech, transkripsi realtime, suara realtime, media-understanding, image-generation, music-generation, video-generation, web-fetch, pencarian web, dan tool. |
| `channelConfigs`                    | Tidak    | `Record<string, object>`         | Metadata konfigurasi channel milik manifest yang digabungkan ke surface discovery dan validasi sebelum runtime dimuat.                                                                                      |
| `skills`                            | Tidak    | `string[]`                       | Direktori Skills yang akan dimuat, relatif terhadap root plugin.                                                                                                                                             |
| `name`                              | Tidak    | `string`                         | Nama plugin yang dapat dibaca manusia.                                                                                                                                                                       |
| `description`                       | Tidak    | `string`                         | Ringkasan singkat yang ditampilkan di surface plugin.                                                                                                                                                        |
| `version`                           | Tidak    | `string`                         | Versi plugin untuk tujuan informasional.                                                                                                                                                                     |
| `uiHints`                           | Tidak    | `Record<string, object>`         | Label UI, placeholder, dan petunjuk sensitivitas untuk field konfigurasi.                                                                                                                                    |

## Referensi `providerAuthChoices`

Setiap entri `providerAuthChoices` menjelaskan satu pilihan onboarding atau
autentikasi. OpenClaw membaca ini sebelum runtime provider dimuat.

| Field                 | Wajib    | Tipe                                            | Artinya                                                                                                  |
| --------------------- | -------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `provider`            | Ya       | `string`                                        | ID provider tempat pilihan ini berada.                                                                   |
| `method`              | Ya       | `string`                                        | ID metode autentikasi yang akan digunakan.                                                               |
| `choiceId`            | Ya       | `string`                                        | ID auth-choice stabil yang digunakan oleh alur onboarding dan CLI.                                       |
| `choiceLabel`         | Tidak    | `string`                                        | Label yang ditampilkan ke pengguna. Jika dihilangkan, OpenClaw menggunakan `choiceId` sebagai fallback. |
| `choiceHint`          | Tidak    | `string`                                        | Teks bantuan singkat untuk pemilih.                                                                      |
| `assistantPriority`   | Tidak    | `number`                                        | Nilai yang lebih rendah diurutkan lebih awal dalam pemilih interaktif berbasis asisten.                 |
| `assistantVisibility` | Tidak    | `"visible"` \| `"manual-only"`                  | Menyembunyikan pilihan dari pemilih asisten sambil tetap mengizinkan pemilihan manual lewat CLI.        |
| `deprecatedChoiceIds` | Tidak    | `string[]`                                      | ID pilihan lama yang harus mengarahkan pengguna ke pilihan pengganti ini.                                |
| `groupId`             | Tidak    | `string`                                        | ID grup opsional untuk mengelompokkan pilihan yang terkait.                                              |
| `groupLabel`          | Tidak    | `string`                                        | Label yang ditampilkan ke pengguna untuk grup tersebut.                                                  |
| `groupHint`           | Tidak    | `string`                                        | Teks bantuan singkat untuk grup tersebut.                                                                |
| `optionKey`           | Tidak    | `string`                                        | Kunci opsi internal untuk alur autentikasi sederhana dengan satu flag.                                   |
| `cliFlag`             | Tidak    | `string`                                        | Nama flag CLI, seperti `--openrouter-api-key`.                                                           |
| `cliOption`           | Tidak    | `string`                                        | Bentuk opsi CLI lengkap, seperti `--openrouter-api-key <key>`.                                           |
| `cliDescription`      | Tidak    | `string`                                        | Deskripsi yang digunakan dalam bantuan CLI.                                                              |
| `onboardingScopes`    | Tidak    | `Array<"text-inference" \| "image-generation">` | Surface onboarding tempat pilihan ini harus muncul. Jika dihilangkan, default-nya adalah `["text-inference"]`. |

## Referensi `commandAliases`

Gunakan `commandAliases` saat sebuah plugin memiliki nama perintah runtime yang
mungkin keliru dimasukkan pengguna ke `plugins.allow` atau coba dijalankan
sebagai perintah CLI root. OpenClaw menggunakan metadata ini untuk diagnostik
tanpa mengimpor kode runtime plugin.

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| Field        | Wajib    | Tipe              | Artinya                                                                  |
| ------------ | -------- | ----------------- | ------------------------------------------------------------------------ |
| `name`       | Ya       | `string`          | Nama perintah yang dimiliki plugin ini.                                  |
| `kind`       | Tidak    | `"runtime-slash"` | Menandai alias sebagai perintah slash chat, bukan perintah CLI root.     |
| `cliCommand` | Tidak    | `string`          | Perintah CLI root terkait yang dapat disarankan untuk operasi CLI, jika ada. |

## Referensi `activation`

Gunakan `activation` saat plugin dapat mendeklarasikan secara ringan event
control-plane mana yang seharusnya mengaktifkannya nanti.

Blok ini hanya metadata. Ini tidak mendaftarkan perilaku runtime, dan tidak
menggantikan `register(...)`, `setupEntry`, atau entrypoint runtime/plugin lainnya.

```json
{
  "activation": {
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| Field            | Wajib    | Tipe                                                 | Artinya                                                          |
| ---------------- | -------- | ---------------------------------------------------- | ---------------------------------------------------------------- |
| `onProviders`    | Tidak    | `string[]`                                           | ID provider yang harus mengaktifkan plugin ini saat diminta.     |
| `onCommands`     | Tidak    | `string[]`                                           | ID perintah yang harus mengaktifkan plugin ini.                  |
| `onChannels`     | Tidak    | `string[]`                                           | ID channel yang harus mengaktifkan plugin ini.                   |
| `onRoutes`       | Tidak    | `string[]`                                           | Jenis route yang harus mengaktifkan plugin ini.                  |
| `onCapabilities` | Tidak    | `Array<"provider" \| "channel" \| "tool" \| "hook">` | Petunjuk kapabilitas umum yang digunakan oleh perencanaan aktivasi control-plane. |

## Referensi `setup`

Gunakan `setup` saat surface setup dan onboarding membutuhkan metadata milik
plugin yang ringan sebelum runtime dimuat.

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

`cliBackends` tingkat atas tetap valid dan terus mendeskripsikan backend
inferensi CLI. `setup.cliBackends` adalah surface deskriptor khusus setup untuk
alur control-plane/setup yang harus tetap hanya berupa metadata.

### Referensi `setup.providers`

| Field         | Wajib    | Tipe       | Artinya                                                                                 |
| ------------- | -------- | ---------- | --------------------------------------------------------------------------------------- |
| `id`          | Ya       | `string`   | ID provider yang diekspos selama setup atau onboarding.                                 |
| `authMethods` | Tidak    | `string[]` | ID metode setup/autentikasi yang didukung provider ini tanpa memuat runtime penuh.      |
| `envVars`     | Tidak    | `string[]` | Env var yang dapat diperiksa oleh surface setup/status generik sebelum runtime plugin dimuat. |

### Field `setup`

| Field              | Wajib    | Tipe       | Artinya                                                                    |
| ------------------ | -------- | ---------- | -------------------------------------------------------------------------- |
| `providers`        | Tidak    | `object[]` | Deskriptor setup provider yang diekspos selama setup dan onboarding.       |
| `cliBackends`      | Tidak    | `string[]` | ID backend saat setup yang tersedia tanpa aktivasi runtime penuh.          |
| `configMigrations` | Tidak    | `string[]` | ID migrasi konfigurasi yang dimiliki oleh surface setup plugin ini.        |
| `requiresRuntime`  | Tidak    | `boolean`  | Apakah setup masih memerlukan eksekusi runtime plugin setelah lookup deskriptor. |

## Referensi `uiHints`

`uiHints` adalah peta dari nama field konfigurasi ke petunjuk rendering kecil.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "help": "Digunakan untuk permintaan OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Setiap petunjuk field dapat mencakup:

| Field         | Tipe       | Artinya                                  |
| ------------- | ---------- | ---------------------------------------- |
| `label`       | `string`   | Label field yang ditampilkan ke pengguna. |
| `help`        | `string`   | Teks bantuan singkat.                    |
| `tags`        | `string[]` | Tag UI opsional.                         |
| `advanced`    | `boolean`  | Menandai field sebagai lanjutan.         |
| `sensitive`   | `boolean`  | Menandai field sebagai rahasia atau sensitif. |
| `placeholder` | `string`   | Teks placeholder untuk input formulir.   |

## Referensi `contracts`

Gunakan `contracts` hanya untuk metadata kepemilikan kapabilitas statis yang
dapat dibaca OpenClaw tanpa mengimpor runtime plugin.

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Setiap daftar bersifat opsional:

| Field                            | Tipe       | Artinya                                                     |
| -------------------------------- | ---------- | ----------------------------------------------------------- |
| `speechProviders`                | `string[]` | ID provider speech yang dimiliki plugin ini.                |
| `realtimeTranscriptionProviders` | `string[]` | ID provider transkripsi realtime yang dimiliki plugin ini.  |
| `realtimeVoiceProviders`         | `string[]` | ID provider suara realtime yang dimiliki plugin ini.        |
| `mediaUnderstandingProviders`    | `string[]` | ID provider media-understanding yang dimiliki plugin ini.   |
| `imageGenerationProviders`       | `string[]` | ID provider image-generation yang dimiliki plugin ini.      |
| `videoGenerationProviders`       | `string[]` | ID provider video-generation yang dimiliki plugin ini.      |
| `webFetchProviders`              | `string[]` | ID provider web-fetch yang dimiliki plugin ini.             |
| `webSearchProviders`             | `string[]` | ID provider pencarian web yang dimiliki plugin ini.         |
| `tools`                          | `string[]` | Nama tool agen yang dimiliki plugin ini untuk pemeriksaan kontrak bundle. |

## Referensi `channelConfigs`

Gunakan `channelConfigs` saat plugin channel membutuhkan metadata konfigurasi
ringan sebelum runtime dimuat.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "URL homeserver",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Koneksi homeserver Matrix",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Setiap entri channel dapat mencakup:

| Field         | Tipe                     | Artinya                                                                                     |
| ------------- | ------------------------ | ------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | JSON Schema untuk `channels.<id>`. Wajib untuk setiap entri konfigurasi channel yang dideklarasikan. |
| `uiHints`     | `Record<string, object>` | Label UI/placeholder/petunjuk sensitif opsional untuk bagian konfigurasi channel tersebut.  |
| `label`       | `string`                 | Label channel yang digabungkan ke surface pemilih dan inspect saat metadata runtime belum siap. |
| `description` | `string`                 | Deskripsi singkat channel untuk surface inspect dan katalog.                                |
| `preferOver`  | `string[]`               | ID plugin lama atau berprioritas lebih rendah yang harus dikalahkan channel ini di surface pemilihan. |

## Referensi `modelSupport`

Gunakan `modelSupport` saat OpenClaw harus menyimpulkan plugin provider Anda dari
ID model shorthand seperti `gpt-5.4` atau `claude-sonnet-4.6` sebelum runtime plugin
dimuat.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw menerapkan prioritas berikut:

- referensi `provider/model` eksplisit menggunakan metadata manifest `providers` yang memilikinya
- `modelPatterns` mengalahkan `modelPrefixes`
- jika satu plugin non-bundled dan satu plugin bundled sama-sama cocok, plugin non-bundled
  yang menang
- ambiguitas yang tersisa diabaikan sampai pengguna atau konfigurasi menentukan provider

Field:

| Field           | Tipe       | Artinya                                                                        |
| --------------- | ---------- | ------------------------------------------------------------------------------ |
| `modelPrefixes` | `string[]` | Prefiks yang dicocokkan dengan `startsWith` terhadap ID model shorthand.       |
| `modelPatterns` | `string[]` | Sumber regex yang dicocokkan terhadap ID model shorthand setelah penghapusan sufiks profil. |

Kunci kapabilitas tingkat atas lama sudah deprecated. Gunakan `openclaw doctor --fix` untuk
memindahkan `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders`, dan `webSearchProviders` ke bawah `contracts`; pemuatan
manifest normal tidak lagi memperlakukan field tingkat atas tersebut sebagai
kepemilikan kapabilitas.

## Manifest versus package.json

Kedua file ini memiliki fungsi yang berbeda:

| File                   | Gunakan untuk                                                                                                                     |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Discovery, validasi konfigurasi, metadata pilihan autentikasi, dan petunjuk UI yang harus ada sebelum kode plugin dijalankan    |
| `package.json`         | Metadata npm, instalasi dependensi, dan blok `openclaw` yang digunakan untuk entrypoint, gating instalasi, setup, atau metadata katalog |

Jika Anda ragu metadata tertentu seharusnya ditempatkan di mana, gunakan aturan ini:

- jika OpenClaw harus mengetahuinya sebelum memuat kode plugin, letakkan di `openclaw.plugin.json`
- jika berkaitan dengan packaging, file entry, atau perilaku instalasi npm, letakkan di `package.json`

### Field `package.json` yang memengaruhi discovery

Beberapa metadata plugin pra-runtime memang sengaja ditempatkan di `package.json` di bawah blok
`openclaw`, bukan di `openclaw.plugin.json`.

Contoh penting:

| Field                                                             | Artinya                                                                                                                                      |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Mendeklarasikan entrypoint plugin native.                                                                                                    |
| `openclaw.setupEntry`                                             | Entrypoint ringan khusus setup yang digunakan selama onboarding dan startup channel yang ditunda.                                            |
| `openclaw.channel`                                                | Metadata katalog channel ringan seperti label, path docs, alias, dan teks pemilihan.                                                        |
| `openclaw.channel.configuredState`                                | Metadata pemeriksa status configured ringan yang dapat menjawab "apakah setup hanya-env sudah ada?" tanpa memuat runtime channel penuh.     |
| `openclaw.channel.persistedAuthState`                             | Metadata pemeriksa auth tersimpan ringan yang dapat menjawab "apakah sesuatu sudah login?" tanpa memuat runtime channel penuh.              |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Petunjuk instalasi/pembaruan untuk plugin bundled dan plugin yang dipublikasikan secara eksternal.                                          |
| `openclaw.install.defaultChoice`                                  | Jalur instalasi yang dipilih saat beberapa sumber instalasi tersedia.                                                                        |
| `openclaw.install.minHostVersion`                                 | Versi host OpenClaw minimum yang didukung, menggunakan batas bawah semver seperti `>=2026.3.22`.                                            |
| `openclaw.install.allowInvalidConfigRecovery`                     | Mengizinkan jalur pemulihan reinstalasi plugin bundled yang sempit saat konfigurasi tidak valid.                                            |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Memungkinkan surface channel khusus setup dimuat sebelum plugin channel penuh selama startup.                                                |

`openclaw.install.minHostVersion` diterapkan selama instalasi dan pemuatan registri
manifest. Nilai yang tidak valid ditolak; nilai yang valid tetapi lebih baru akan
melewati plugin pada host yang lebih lama.

`openclaw.install.allowInvalidConfigRecovery` sengaja dibatasi. Ini tidak membuat
konfigurasi rusak sembarang menjadi dapat diinstal. Saat ini, ini hanya
mengizinkan alur instalasi memulihkan kegagalan upgrade plugin bundled usang tertentu,
seperti path plugin bundled yang hilang atau entri `channels.<id>` usang untuk plugin
bundled yang sama. Kesalahan konfigurasi lain yang tidak terkait tetap memblokir
instalasi dan mengarahkan operator ke `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` adalah metadata paket untuk modul pemeriksa kecil:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Gunakan ini saat alur setup, doctor, atau configured-state membutuhkan probe auth
ya/tidak yang ringan sebelum plugin channel penuh dimuat. Export target harus berupa
fungsi kecil yang hanya membaca state tersimpan; jangan arahkan melalui barrel runtime
channel penuh.

`openclaw.channel.configuredState` mengikuti bentuk yang sama untuk pemeriksaan configured
khusus env yang ringan:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

Gunakan ini saat sebuah channel dapat menjawab status configured dari env atau input
kecil non-runtime lainnya. Jika pemeriksaan membutuhkan resolusi konfigurasi penuh atau runtime
channel yang sebenarnya, simpan logika itu di hook plugin `config.hasConfiguredState`.

## Persyaratan JSON Schema

- **Setiap plugin harus menyertakan JSON Schema**, bahkan jika tidak menerima konfigurasi.
- Skema kosong dapat diterima (misalnya, `{ "type": "object", "additionalProperties": false }`).
- Skema divalidasi pada saat baca/tulis konfigurasi, bukan saat runtime.

## Perilaku validasi

- Kunci `channels.*` yang tidak dikenal adalah **kesalahan**, kecuali ID channel tersebut dideklarasikan oleh
  manifest plugin.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny`, dan `plugins.slots.*`
  harus merujuk ke ID plugin yang **dapat didiscovery**. ID yang tidak dikenal adalah **kesalahan**.
- Jika plugin terinstal tetapi memiliki manifest atau skema yang rusak atau hilang,
  validasi gagal dan Doctor melaporkan kesalahan plugin.
- Jika konfigurasi plugin ada tetapi plugin tersebut **dinonaktifkan**, konfigurasinya tetap disimpan dan
  **peringatan** ditampilkan di Doctor + log.

Lihat [Referensi konfigurasi](/id/gateway/configuration) untuk skema `plugins.*` lengkap.

## Catatan

- Manifest **wajib untuk plugin OpenClaw native**, termasuk pemuatan dari filesystem lokal.
- Runtime tetap memuat modul plugin secara terpisah; manifest hanya untuk
  discovery + validasi.
- Manifest native diparse dengan JSON5, jadi komentar, trailing comma, dan
  key tanpa tanda kutip diterima selama nilai akhirnya tetap berupa object.
- Hanya field manifest yang terdokumentasi yang dibaca oleh manifest loader. Hindari menambahkan
  key tingkat atas kustom di sini.
- `providerAuthEnvVars` adalah jalur metadata ringan untuk probe autentikasi, validasi
  penanda env, dan surface autentikasi provider serupa yang seharusnya tidak menyalakan runtime
  plugin hanya untuk memeriksa nama env.
- `providerAuthAliases` memungkinkan varian provider menggunakan kembali env var autentikasi,
  profil autentikasi, autentikasi berbasis konfigurasi, dan pilihan onboarding API key
  provider lain tanpa meng-hardcode hubungan tersebut di core.
- `channelEnvVars` adalah jalur metadata ringan untuk fallback shell-env, prompt setup,
  dan surface channel serupa yang seharusnya tidak menyalakan runtime plugin
  hanya untuk memeriksa nama env.
- `providerAuthChoices` adalah jalur metadata ringan untuk pemilih pilihan autentikasi,
  resolusi `--auth-choice`, pemetaan preferred-provider, dan pendaftaran flag CLI onboarding
  sederhana sebelum runtime provider dimuat. Untuk metadata wizard runtime
  yang membutuhkan kode provider, lihat
  [Hook runtime provider](/id/plugins/architecture#provider-runtime-hooks).
- Jenis plugin eksklusif dipilih melalui `plugins.slots.*`.
  - `kind: "memory"` dipilih oleh `plugins.slots.memory`.
  - `kind: "context-engine"` dipilih oleh `plugins.slots.contextEngine`
    (default: `legacy` bawaan).
- `channels`, `providers`, `cliBackends`, dan `skills` dapat dihilangkan saat
  plugin tidak membutuhkannya.
- Jika plugin Anda bergantung pada modul native, dokumentasikan langkah build dan semua
  persyaratan allowlist package manager (misalnya, pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Terkait

- [Membangun Plugin](/id/plugins/building-plugins) — memulai dengan plugin
- [Arsitektur Plugin](/id/plugins/architecture) — arsitektur internal
- [Ikhtisar SDK](/id/plugins/sdk-overview) — referensi SDK Plugin
