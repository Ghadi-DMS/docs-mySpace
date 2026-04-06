---
read_when:
    - Anda sedang membangun plugin OpenClaw
    - Anda perlu mengirimkan skema konfigurasi plugin atau men-debug error validasi plugin
summary: Manifest plugin + persyaratan skema JSON (validasi konfigurasi ketat)
title: Manifest Plugin
x-i18n:
    generated_at: "2026-04-06T03:08:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: f6f915a761cdb5df77eba5d2ccd438c65445bd2ab41b0539d1200e63e8cf2c3a
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifest plugin (openclaw.plugin.json)

Halaman ini hanya untuk **manifest plugin OpenClaw native**.

Untuk layout bundle yang kompatibel, lihat [Bundle plugin](/id/plugins/bundles).

Format bundle yang kompatibel menggunakan file manifest yang berbeda:

- Bundle Codex: `.codex-plugin/plugin.json`
- Bundle Claude: `.claude-plugin/plugin.json` atau layout komponen Claude default
  tanpa manifest
- Bundle Cursor: `.cursor-plugin/plugin.json`

OpenClaw juga mendeteksi layout bundle tersebut secara otomatis, tetapi layout itu tidak divalidasi
terhadap skema `openclaw.plugin.json` yang dijelaskan di sini.

Untuk bundle yang kompatibel, OpenClaw saat ini membaca metadata bundle ditambah
root Skills yang dideklarasikan, root perintah Claude, default `settings.json` bundle Claude,
default LSP bundle Claude, dan paket hook yang didukung ketika layout tersebut cocok
dengan ekspektasi runtime OpenClaw.

Setiap plugin OpenClaw native **harus** mengirimkan file `openclaw.plugin.json` di
**root plugin**. OpenClaw menggunakan manifest ini untuk memvalidasi konfigurasi
**tanpa menjalankan kode plugin**. Manifest yang hilang atau tidak valid diperlakukan sebagai
error plugin dan memblokir validasi konfigurasi.

Lihat panduan lengkap sistem plugin: [Plugins](/id/tools/plugin).
Untuk model kapabilitas native dan panduan kompatibilitas eksternal saat ini:
[Model kapabilitas](/id/plugins/architecture#public-capability-model).

## Fungsi file ini

`openclaw.plugin.json` adalah metadata yang dibaca OpenClaw sebelum memuat
kode plugin Anda.

Gunakan untuk:

- identitas plugin
- validasi konfigurasi
- metadata auth dan onboarding yang harus tersedia tanpa menjalankan runtime plugin
- metadata alias dan pengaktifan otomatis yang harus diselesaikan sebelum runtime plugin dimuat
- metadata kepemilikan keluarga model bentuk singkat yang harus mengaktifkan plugin
  secara otomatis sebelum runtime dimuat
- snapshot kepemilikan kapabilitas statis yang digunakan untuk wiring kompatibilitas bawaan dan
  cakupan kontrak
- metadata konfigurasi spesifik channel yang harus digabungkan ke dalam permukaan katalog dan validasi
  tanpa memuat runtime
- petunjuk UI konfigurasi

Jangan gunakan untuk:

- mendaftarkan perilaku runtime
- mendeklarasikan entrypoint kode
- metadata instalasi npm

Hal-hal tersebut milik kode plugin Anda dan `package.json`.

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
  "description": "Plugin provider OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "API key OpenRouter",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "API key OpenRouter",
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

| Field                               | Wajib | Tipe                             | Artinya                                                                                                                                                                                                        |
| ----------------------------------- | ----- | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                | Ya    | `string`                         | ID plugin kanonis. Ini adalah id yang digunakan di `plugins.entries.<id>`.                                                                                                                                     |
| `configSchema`                      | Ya    | `object`                         | JSON Schema inline untuk konfigurasi plugin ini.                                                                                                                                                                |
| `enabledByDefault`                  | Tidak | `true`                           | Menandai plugin bawaan sebagai aktif secara default. Hilangkan field ini, atau set ke nilai apa pun selain `true`, agar plugin tetap nonaktif secara default.                                                |
| `legacyPluginIds`                   | Tidak | `string[]`                       | ID lama yang dinormalkan ke id plugin kanonis ini.                                                                                                                                                              |
| `autoEnableWhenConfiguredProviders` | Tidak | `string[]`                       | ID provider yang harus mengaktifkan plugin ini secara otomatis saat auth, konfigurasi, atau referensi model menyebutkannya.                                                                                   |
| `kind`                              | Tidak | `"memory"` \| `"context-engine"` | Mendeklarasikan jenis plugin eksklusif yang digunakan oleh `plugins.slots.*`.                                                                                                                                   |
| `channels`                          | Tidak | `string[]`                       | ID channel yang dimiliki plugin ini. Digunakan untuk discovery dan validasi konfigurasi.                                                                                                                        |
| `providers`                         | Tidak | `string[]`                       | ID provider yang dimiliki plugin ini.                                                                                                                                                                           |
| `modelSupport`                      | Tidak | `object`                         | Metadata bentuk singkat keluarga model milik manifest yang digunakan untuk memuat plugin secara otomatis sebelum runtime.                                                                                      |
| `providerAuthEnvVars`               | Tidak | `Record<string, string[]>`       | Metadata env auth provider yang ringan yang dapat diperiksa OpenClaw tanpa memuat kode plugin.                                                                                                                  |
| `providerAuthChoices`               | Tidak | `object[]`                       | Metadata pilihan auth yang ringan untuk picker onboarding, resolusi provider pilihan, dan wiring flag CLI sederhana.                                                                                           |
| `contracts`                         | Tidak | `object`                         | Snapshot kapabilitas bawaan statis untuk speech, realtime transcription, realtime voice, media-understanding, image-generation, music-generation, video-generation, web-fetch, web search, dan kepemilikan tool. |
| `channelConfigs`                    | Tidak | `Record<string, object>`         | Metadata konfigurasi channel milik manifest yang digabungkan ke dalam permukaan discovery dan validasi sebelum runtime dimuat.                                                                                 |
| `skills`                            | Tidak | `string[]`                       | Direktori Skills yang akan dimuat, relatif terhadap root plugin.                                                                                                                                                |
| `name`                              | Tidak | `string`                         | Nama plugin yang dapat dibaca manusia.                                                                                                                                                                          |
| `description`                       | Tidak | `string`                         | Ringkasan singkat yang ditampilkan di permukaan plugin.                                                                                                                                                         |
| `version`                           | Tidak | `string`                         | Versi plugin yang bersifat informatif.                                                                                                                                                                          |
| `uiHints`                           | Tidak | `Record<string, object>`         | Label UI, placeholder, dan petunjuk sensitivitas untuk field konfigurasi.                                                                                                                                       |

## Referensi `providerAuthChoices`

Setiap entri `providerAuthChoices` menjelaskan satu pilihan onboarding atau auth.
OpenClaw membacanya sebelum runtime provider dimuat.

| Field                 | Wajib | Tipe                                            | Artinya                                                                                             |
| --------------------- | ----- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `provider`            | Ya    | `string`                                        | ID provider tempat pilihan ini berada.                                                              |
| `method`              | Ya    | `string`                                        | ID metode auth yang akan didispatch.                                                                |
| `choiceId`            | Ya    | `string`                                        | ID pilihan auth stabil yang digunakan oleh alur onboarding dan CLI.                                 |
| `choiceLabel`         | Tidak | `string`                                        | Label untuk pengguna. Jika dihilangkan, OpenClaw kembali ke `choiceId`.                             |
| `choiceHint`          | Tidak | `string`                                        | Teks bantuan singkat untuk picker.                                                                  |
| `assistantPriority`   | Tidak | `number`                                        | Nilai yang lebih rendah diurutkan lebih awal dalam picker interaktif yang digerakkan asisten.      |
| `assistantVisibility` | Tidak | `"visible"` \| `"manual-only"`                  | Menyembunyikan pilihan dari picker asisten sambil tetap mengizinkan pemilihan CLI manual.          |
| `deprecatedChoiceIds` | Tidak | `string[]`                                      | ID pilihan lama yang harus mengarahkan pengguna ke pilihan pengganti ini.                           |
| `groupId`             | Tidak | `string`                                        | ID grup opsional untuk mengelompokkan pilihan terkait.                                              |
| `groupLabel`          | Tidak | `string`                                        | Label untuk pengguna untuk grup tersebut.                                                           |
| `groupHint`           | Tidak | `string`                                        | Teks bantuan singkat untuk grup.                                                                    |
| `optionKey`           | Tidak | `string`                                        | Kunci opsi internal untuk alur auth satu-flag sederhana.                                            |
| `cliFlag`             | Tidak | `string`                                        | Nama flag CLI, seperti `--openrouter-api-key`.                                                      |
| `cliOption`           | Tidak | `string`                                        | Bentuk opsi CLI lengkap, seperti `--openrouter-api-key <key>`.                                      |
| `cliDescription`      | Tidak | `string`                                        | Deskripsi yang digunakan di bantuan CLI.                                                            |
| `onboardingScopes`    | Tidak | `Array<"text-inference" \| "image-generation">` | Permukaan onboarding tempat pilihan ini harus muncul. Jika dihilangkan, default-nya `["text-inference"]`. |

## Referensi `uiHints`

`uiHints` adalah map dari nama field konfigurasi ke petunjuk rendering kecil.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "help": "Digunakan untuk request OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Setiap petunjuk field dapat mencakup:

| Field         | Tipe       | Artinya                                  |
| ------------- | ---------- | ---------------------------------------- |
| `label`       | `string`   | Label field untuk pengguna.              |
| `help`        | `string`   | Teks bantuan singkat.                    |
| `tags`        | `string[]` | Tag UI opsional.                         |
| `advanced`    | `boolean`  | Menandai field sebagai lanjutan.         |
| `sensitive`   | `boolean`  | Menandai field sebagai rahasia/sensitif. |
| `placeholder` | `string`   | Teks placeholder untuk input formulir.   |

## Referensi `contracts`

Gunakan `contracts` hanya untuk metadata kepemilikan kapabilitas statis yang dapat
dibaca OpenClaw tanpa mengimpor runtime plugin.

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
| `realtimeTranscriptionProviders` | `string[]` | ID provider realtime-transcription yang dimiliki plugin ini. |
| `realtimeVoiceProviders`         | `string[]` | ID provider realtime-voice yang dimiliki plugin ini.        |
| `mediaUnderstandingProviders`    | `string[]` | ID provider media-understanding yang dimiliki plugin ini.   |
| `imageGenerationProviders`       | `string[]` | ID provider image-generation yang dimiliki plugin ini.      |
| `videoGenerationProviders`       | `string[]` | ID provider video-generation yang dimiliki plugin ini.      |
| `webFetchProviders`              | `string[]` | ID provider web-fetch yang dimiliki plugin ini.             |
| `webSearchProviders`             | `string[]` | ID provider web-search yang dimiliki plugin ini.            |
| `tools`                          | `string[]` | Nama tool agen yang dimiliki plugin ini untuk pemeriksaan kontrak bawaan. |

## Referensi `channelConfigs`

Gunakan `channelConfigs` ketika plugin channel memerlukan metadata konfigurasi ringan sebelum
runtime dimuat.

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
| `uiHints`     | `Record<string, object>` | Label UI/placeholders/petunjuk sensitif opsional untuk bagian konfigurasi channel tersebut. |
| `label`       | `string`                 | Label channel yang digabungkan ke permukaan picker dan inspect saat metadata runtime belum siap. |
| `description` | `string`                 | Deskripsi channel singkat untuk permukaan inspect dan katalog.                              |
| `preferOver`  | `string[]`               | ID plugin lama atau berprioritas lebih rendah yang harus dikalahkan channel ini di permukaan seleksi. |

## Referensi `modelSupport`

Gunakan `modelSupport` ketika OpenClaw harus menyimpulkan plugin provider Anda dari
id model bentuk singkat seperti `gpt-5.4` atau `claude-sonnet-4.6` sebelum runtime plugin
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

- referensi `provider/model` eksplisit menggunakan metadata manifest `providers` yang memiliki
- `modelPatterns` mengalahkan `modelPrefixes`
- jika satu plugin non-bawaan dan satu plugin bawaan sama-sama cocok, plugin non-bawaan
  menang
- ambiguitas yang tersisa diabaikan sampai pengguna atau konfigurasi menentukan provider

Field:

| Field           | Tipe       | Artinya                                                                      |
| --------------- | ---------- | ---------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Prefiks yang dicocokkan dengan `startsWith` terhadap id model bentuk singkat. |
| `modelPatterns` | `string[]` | Sumber regex yang dicocokkan terhadap id model bentuk singkat setelah sufiks profil dihapus. |

Kunci kapabilitas tingkat atas lama sudah deprecated. Gunakan `openclaw doctor --fix` untuk
memindahkan `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders`, dan `webSearchProviders` ke bawah `contracts`; pemuatan
manifest normal tidak lagi memperlakukan field tingkat atas tersebut sebagai
kepemilikan kapabilitas.

## Manifest versus package.json

Kedua file memiliki fungsi yang berbeda:

| File                   | Gunakan untuk                                                                                                                       |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Discovery, validasi konfigurasi, metadata pilihan auth, dan petunjuk UI yang harus ada sebelum kode plugin berjalan               |
| `package.json`         | Metadata npm, instalasi dependensi, dan blok `openclaw` yang digunakan untuk entrypoint, gating instalasi, penyiapan, atau metadata katalog |

Jika Anda ragu metadata tertentu harus ditempatkan di mana, gunakan aturan ini:

- jika OpenClaw harus mengetahuinya sebelum memuat kode plugin, taruh di `openclaw.plugin.json`
- jika itu tentang packaging, file entri, atau perilaku instalasi npm, taruh di `package.json`

### Field `package.json` yang memengaruhi discovery

Beberapa metadata plugin pra-runtime memang sengaja ditempatkan di `package.json` di bawah blok
`openclaw` alih-alih `openclaw.plugin.json`.

Contoh penting:

| Field                                                             | Artinya                                                                                                                                 |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Mendeklarasikan entrypoint plugin native.                                                                                               |
| `openclaw.setupEntry`                                             | EntryPoint ringan khusus penyiapan yang digunakan selama onboarding dan startup channel tertunda.                                       |
| `openclaw.channel`                                                | Metadata katalog channel ringan seperti label, path dokumen, alias, dan teks pilihan.                                                  |
| `openclaw.channel.configuredState`                                | Metadata checker status terkonfigurasi ringan yang dapat menjawab "apakah penyiapan hanya-env sudah ada?" tanpa memuat runtime channel penuh. |
| `openclaw.channel.persistedAuthState`                             | Metadata checker auth tersimpan ringan yang dapat menjawab "apakah ada yang sudah login?" tanpa memuat runtime channel penuh.          |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Petunjuk instalasi/pembaruan untuk plugin bawaan dan plugin yang dipublikasikan secara eksternal.                                      |
| `openclaw.install.defaultChoice`                                  | Jalur instalasi yang diprioritaskan ketika beberapa sumber instalasi tersedia.                                                          |
| `openclaw.install.minHostVersion`                                 | Versi host OpenClaw minimum yang didukung, menggunakan batas bawah semver seperti `>=2026.3.22`.                                       |
| `openclaw.install.allowInvalidConfigRecovery`                     | Mengizinkan jalur pemulihan reinstall plugin bawaan yang sempit ketika konfigurasi tidak valid.                                        |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Memungkinkan permukaan channel khusus penyiapan dimuat sebelum plugin channel penuh saat startup.                                      |

`openclaw.install.minHostVersion` ditegakkan selama instalasi dan pemuatan registry
manifest. Nilai yang tidak valid ditolak; nilai yang valid tetapi lebih baru akan melewati
plugin pada host yang lebih lama.

`openclaw.install.allowInvalidConfigRecovery` sengaja dibuat sempit. Field ini
tidak membuat konfigurasi rusak sembarang menjadi dapat diinstal. Saat ini field itu hanya mengizinkan
alur instalasi untuk memulihkan kegagalan upgrade plugin bawaan lama tertentu, seperti
path plugin bawaan yang hilang atau entri `channels.<id>` lama untuk plugin bawaan yang sama.
Error konfigurasi yang tidak terkait tetap memblokir instalasi dan mengarahkan operator
ke `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` adalah metadata package untuk modul checker
kecil:

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

Gunakan saat alur setup, doctor, atau configured-state memerlukan probe auth ya/tidak yang ringan
sebelum plugin channel penuh dimuat. Export target harus berupa fungsi kecil
yang hanya membaca status tersimpan; jangan arahkan melalui barrel runtime
channel penuh.

`openclaw.channel.configuredState` mengikuti bentuk yang sama untuk pemeriksaan status
terkonfigurasi ringan yang hanya-env:

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

Gunakan ketika sebuah channel dapat menjawab status terkonfigurasi dari env atau input kecil
non-runtime lainnya. Jika pemeriksaan memerlukan resolusi konfigurasi penuh atau runtime
channel yang sebenarnya, simpan logika itu di hook plugin `config.hasConfiguredState`.

## Persyaratan JSON Schema

- **Setiap plugin harus mengirimkan JSON Schema**, bahkan jika tidak menerima konfigurasi apa pun.
- Skema kosong dapat diterima (misalnya, `{ "type": "object", "additionalProperties": false }`).
- Skema divalidasi pada waktu baca/tulis konfigurasi, bukan pada runtime.

## Perilaku validasi

- Kunci `channels.*` yang tidak dikenal adalah **error**, kecuali id channel tersebut dideklarasikan oleh
  manifest plugin.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny`, dan `plugins.slots.*`
  harus mereferensikan id plugin yang **dapat ditemukan**. ID yang tidak dikenal adalah **error**.
- Jika plugin terinstal tetapi memiliki manifest atau skema yang rusak atau hilang,
  validasi gagal dan Doctor melaporkan error plugin.
- Jika konfigurasi plugin ada tetapi plugin **dinonaktifkan**, konfigurasi itu tetap disimpan dan
  **peringatan** ditampilkan di Doctor + log.

Lihat [Referensi konfigurasi](/id/gateway/configuration) untuk skema `plugins.*` lengkap.

## Catatan

- Manifest **wajib untuk plugin OpenClaw native**, termasuk pemuatan filesystem lokal.
- Runtime tetap memuat modul plugin secara terpisah; manifest hanya untuk
  discovery + validasi.
- Manifest native diparse dengan JSON5, sehingga komentar, trailing comma, dan
  key tanpa tanda kutip diterima selama nilai akhirnya tetap berupa objek.
- Hanya field manifest yang terdokumentasi yang dibaca oleh pemuat manifest. Hindari menambahkan
  key tingkat atas kustom di sini.
- `providerAuthEnvVars` adalah jalur metadata ringan untuk probe auth, validasi
  penanda env, dan permukaan auth provider serupa yang tidak boleh menyalakan runtime plugin
  hanya untuk memeriksa nama env.
- `providerAuthChoices` adalah jalur metadata ringan untuk picker pilihan auth,
  resolusi `--auth-choice`, pemetaan provider pilihan, dan pendaftaran flag CLI
  onboarding sederhana sebelum runtime provider dimuat. Untuk metadata wizard runtime
  yang memerlukan kode provider, lihat
  [Hook runtime provider](/id/plugins/architecture#provider-runtime-hooks).
- Jenis plugin eksklusif dipilih melalui `plugins.slots.*`.
  - `kind: "memory"` dipilih oleh `plugins.slots.memory`.
  - `kind: "context-engine"` dipilih oleh `plugins.slots.contextEngine`
    (default: `legacy` bawaan).
- `channels`, `providers`, dan `skills` dapat dihilangkan ketika sebuah
  plugin tidak membutuhkannya.
- Jika plugin Anda bergantung pada modul native, dokumentasikan langkah build dan setiap
  persyaratan allowlist package manager (misalnya, pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Terkait

- [Membangun Plugins](/id/plugins/building-plugins) — memulai dengan plugin
- [Arsitektur Plugin](/id/plugins/architecture) — arsitektur internal
- [Ikhtisar SDK](/id/plugins/sdk-overview) — referensi Plugin SDK
