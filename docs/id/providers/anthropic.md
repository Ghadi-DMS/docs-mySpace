---
read_when:
    - Anda ingin menggunakan model Anthropic di OpenClaw
summary: Gunakan Anthropic Claude melalui API key di OpenClaw
title: Anthropic
x-i18n:
    generated_at: "2026-04-06T03:09:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: bbc6c4938674aedf20ff944bc04e742c9a7e77a5ff10ae4f95b5718504c57c2d
    source_path: providers/anthropic.md
    workflow: 15
---

# Anthropic (Claude)

Anthropic membangun keluarga model **Claude** dan menyediakan akses melalui API.
Di OpenClaw, penyiapan Anthropic baru harus menggunakan API key. Profil token
Anthropic lama yang sudah ada konfigurasinya tetap dihormati saat runtime.

<Warning>
Untuk Anthropic di OpenClaw, pembagian penagihannya adalah:

- **API key Anthropic**: penagihan API Anthropic normal.
- **Autentikasi langganan Claude di dalam OpenClaw**: Anthropic memberi tahu pengguna OpenClaw pada
  **4 April 2026 pukul 12:00 PM PT / 8:00 PM BST** bahwa ini dihitung sebagai
  penggunaan harness pihak ketiga dan memerlukan **Extra Usage** (pay-as-you-go,
  ditagih terpisah dari langganan).

Reproduksi lokal kami sesuai dengan pembagian itu:

- `claude -p` langsung mungkin masih berfungsi
- `claude -p --append-system-prompt ...` dapat memicu guard Extra Usage ketika
  prompt mengidentifikasi OpenClaw
- prompt sistem mirip OpenClaw yang sama **tidak** mereproduksi pemblokiran pada
  jalur SDK Anthropic + `ANTHROPIC_API_KEY`

Jadi aturan praktisnya adalah: **API key Anthropic, atau langganan Claude dengan
Extra Usage**. Jika Anda menginginkan jalur produksi yang paling jelas, gunakan API key
Anthropic.

Docs publik Anthropic saat ini:

- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Claude Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview)

- [Using Claude Code with your Pro or Max plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [Using Claude Code with your Team or Enterprise plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)

Jika Anda menginginkan jalur penagihan yang paling jelas, gunakan API key Anthropic sebagai gantinya.
OpenClaw juga mendukung opsi gaya langganan lainnya, termasuk [OpenAI
Codex](/id/providers/openai), [Qwen Cloud Coding Plan](/id/providers/qwen),
[MiniMax Coding Plan](/id/providers/minimax), dan [Z.AI / GLM Coding
Plan](/id/providers/glm).
</Warning>

## Opsi A: API key Anthropic

**Terbaik untuk:** akses API standar dan penagihan berbasis penggunaan.
Buat API key Anda di Anthropic Console.

### Penyiapan CLI

```bash
openclaw onboard
# choose: Anthropic API key

# or non-interactive
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### Cuplikan config Anthropic

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Default thinking (Claude 4.6)

- Model Anthropic Claude 4.6 secara default menggunakan thinking `adaptive` di OpenClaw ketika tidak ada tingkat thinking eksplisit yang ditetapkan.
- Anda dapat mengoverride per pesan (`/think:<level>`) atau di param model:
  `agents.defaults.models["anthropic/<model>"].params.thinking`.
- Docs Anthropic terkait:
  - [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
  - [Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## Fast mode (API Anthropic)

Toggle `/fast` bersama milik OpenClaw juga mendukung trafik Anthropic publik langsung, termasuk permintaan yang diautentikasi dengan API key dan OAuth yang dikirim ke `api.anthropic.com`.

- `/fast on` dipetakan ke `service_tier: "auto"`
- `/fast off` dipetakan ke `service_tier: "standard_only"`
- Default config:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-sonnet-4-6": {
          params: { fastMode: true },
        },
      },
    },
  },
}
```

Batasan penting:

- OpenClaw hanya menyuntikkan service tier Anthropic untuk permintaan langsung ke `api.anthropic.com`. Jika Anda merutekan `anthropic/*` melalui proxy atau gateway, `/fast` membiarkan `service_tier` tidak berubah.
- Param model Anthropic `serviceTier` atau `service_tier` yang eksplisit mengoverride default `/fast` saat keduanya ditetapkan.
- Anthropic melaporkan tier efektif pada respons di bawah `usage.service_tier`. Pada akun tanpa kapasitas Priority Tier, `service_tier: "auto"` mungkin tetap di-resolve menjadi `standard`.

## Prompt caching (API Anthropic)

OpenClaw mendukung fitur prompt caching Anthropic. Ini **khusus API**; autentikasi token Anthropic lama tidak menghormati pengaturan cache.

### Konfigurasi

Gunakan parameter `cacheRetention` dalam config model Anda:

| Nilai  | Durasi Cache | Deskripsi                   |
| ------ | ------------ | --------------------------- |
| `none` | Tanpa cache  | Nonaktifkan prompt caching  |
| `short` | 5 menit     | Default untuk auth API Key  |
| `long` | 1 jam        | Cache yang diperpanjang     |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

### Default

Saat menggunakan autentikasi API Key Anthropic, OpenClaw secara otomatis menerapkan `cacheRetention: "short"` (cache 5 menit) untuk semua model Anthropic. Anda dapat mengoverride ini dengan secara eksplisit menetapkan `cacheRetention` di config Anda.

### Override cacheRetention per agen

Gunakan param tingkat model sebagai baseline Anda, lalu override agen tertentu melalui `agents.list[].params`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" }, // baseline untuk sebagian besar agen
        },
      },
    },
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } }, // override hanya untuk agen ini
    ],
  },
}
```

Urutan penggabungan config untuk param terkait cache:

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params` (id yang cocok, override berdasarkan key)

Ini memungkinkan satu agen mempertahankan cache berumur panjang sementara agen lain pada model yang sama menonaktifkan caching untuk menghindari biaya penulisan pada trafik bursty/penggunaan ulang rendah.

### Catatan Bedrock Claude

- Model Anthropic Claude di Bedrock (`amazon-bedrock/*anthropic.claude*`) menerima pass-through `cacheRetention` saat dikonfigurasi.
- Model Bedrock non-Anthropic dipaksa ke `cacheRetention: "none"` saat runtime.
- Smart default API key Anthropic juga mengisi `cacheRetention: "short"` untuk referensi model Claude-on-Bedrock ketika tidak ada nilai eksplisit yang ditetapkan.

## Jendela konteks 1M (beta Anthropic)

Jendela konteks 1M Anthropic dibatasi beta. Di OpenClaw, aktifkan per model
dengan `params.context1m: true` untuk model Opus/Sonnet yang didukung.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { context1m: true },
        },
      },
    },
  },
}
```

OpenClaw memetakan ini ke `anthropic-beta: context-1m-2025-08-07` pada permintaan
Anthropic.

Ini hanya aktif ketika `params.context1m` secara eksplisit ditetapkan ke `true` untuk
model tersebut.

Persyaratan: Anthropic harus mengizinkan penggunaan konteks panjang pada kredensial itu
(biasanya penagihan API key, atau jalur login-Claude OpenClaw / autentikasi token lama
dengan Extra Usage diaktifkan). Jika tidak, Anthropic mengembalikan:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

Catatan: Anthropic saat ini menolak permintaan beta `context-1m-*` ketika menggunakan
autentikasi token Anthropic lama (`sk-ant-oat-*`). Jika Anda mengonfigurasi
`context1m: true` dengan mode auth lama itu, OpenClaw mencatat peringatan dan
kembali ke jendela konteks standar dengan melewati header beta context1m
sambil tetap mempertahankan beta OAuth yang diperlukan.

## Dihapus: backend Claude CLI

Backend `claude-cli` Anthropic bawaan telah dihapus.

- Pemberitahuan Anthropic pada 4 April 2026 menyatakan bahwa trafik login-Claude yang didorong OpenClaw adalah
  penggunaan harness pihak ketiga dan memerlukan **Extra Usage**.
- Reproduksi lokal kami juga menunjukkan bahwa
  `claude -p --append-system-prompt ...` langsung dapat terkena guard yang sama ketika
  prompt tambahan mengidentifikasi OpenClaw.
- Prompt sistem mirip OpenClaw yang sama tidak terkena guard itu pada
  jalur SDK Anthropic + `ANTHROPIC_API_KEY`.
- Gunakan API key Anthropic untuk trafik Anthropic di OpenClaw.

## Catatan

- Docs publik Claude Code Anthropic masih mendokumentasikan penggunaan CLI langsung seperti
  `claude -p`, tetapi pemberitahuan terpisah Anthropic kepada pengguna OpenClaw menyatakan bahwa
  jalur login-Claude **OpenClaw** adalah penggunaan harness pihak ketiga dan memerlukan
  **Extra Usage** (pay-as-you-go yang ditagih terpisah dari langganan).
  Reproduksi lokal kami juga menunjukkan bahwa
  `claude -p --append-system-prompt ...` langsung dapat terkena guard yang sama ketika
  prompt tambahan mengidentifikasi OpenClaw, sementara bentuk prompt yang sama tidak
  mereproduksi hal itu pada jalur SDK Anthropic + `ANTHROPIC_API_KEY`. Untuk produksi, kami
  merekomendasikan API key Anthropic sebagai gantinya.
- Setup-token Anthropic kembali tersedia di OpenClaw sebagai jalur lama/manual. Pemberitahuan penagihan Anthropic yang khusus untuk OpenClaw tetap berlaku, jadi gunakan dengan ekspektasi bahwa Anthropic memerlukan **Extra Usage** untuk jalur ini.
- Detail auth + aturan penggunaan ulang ada di [/concepts/oauth](/id/concepts/oauth).

## Pemecahan masalah

**Error 401 / token tiba-tiba tidak valid**

- Autentikasi token Anthropic lama dapat kedaluwarsa atau dicabut.
- Untuk penyiapan baru, migrasikan ke API key Anthropic.

**Tidak ada API key ditemukan untuk provider "anthropic"**

- Auth bersifat **per agen**. Agen baru tidak mewarisi key agen utama.
- Jalankan ulang onboarding untuk agen tersebut, atau konfigurasikan API key pada host gateway, lalu verifikasi dengan `openclaw models status`.

**Tidak ada kredensial ditemukan untuk profil `anthropic:default`**

- Jalankan `openclaw models status` untuk melihat profil auth mana yang aktif.
- Jalankan ulang onboarding, atau konfigurasikan API key untuk path profil tersebut.

**Tidak ada profil auth yang tersedia (semuanya cooldown/tidak tersedia)**

- Periksa `openclaw models status --json` untuk `auth.unusableProfiles`.
- Cooldown batas laju Anthropic dapat berscope model, sehingga model Anthropic serumpun
  mungkin masih dapat digunakan meskipun model saat ini sedang cooldown.
- Tambahkan profil Anthropic lain atau tunggu cooldown selesai.

Selengkapnya: [/gateway/troubleshooting](/id/gateway/troubleshooting) dan [/help/faq](/id/help/faq).
