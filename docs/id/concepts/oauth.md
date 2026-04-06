---
read_when:
    - Anda ingin memahami OAuth OpenClaw secara menyeluruh
    - Anda mengalami masalah token tidak valid / logout
    - Anda menginginkan alur autentikasi Claude CLI atau OAuth
    - Anda menginginkan beberapa akun atau perutean profil
summary: 'OAuth di OpenClaw: pertukaran token, penyimpanan, dan pola multi-akun'
title: OAuth
x-i18n:
    generated_at: "2026-04-06T03:06:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 402e20dfeb6ae87a90cba5824a56a7ba3b964f3716508ea5cc48a47e5affdd73
    source_path: concepts/oauth.md
    workflow: 15
---

# OAuth

OpenClaw mendukung “subscription auth” melalui OAuth untuk provider yang menawarkannya
(terutama **OpenAI Codex (ChatGPT OAuth)**). Untuk Anthropic, pembagian praktisnya
sekarang adalah:

- **Kunci API Anthropic**: penagihan API Anthropic biasa
- **Anthropic subscription auth di dalam OpenClaw**: Anthropic memberi tahu pengguna OpenClaw
  pada **4 April 2026 pukul 12.00 PM PT / 8.00 PM BST** bahwa ini sekarang
  memerlukan **Extra Usage**

OpenAI Codex OAuth secara eksplisit didukung untuk digunakan di alat eksternal seperti
OpenClaw. Halaman ini menjelaskan:

Untuk Anthropic di lingkungan produksi, autentikasi dengan kunci API adalah jalur yang lebih aman dan direkomendasikan.

- cara kerja **pertukaran token** OAuth (PKCE)
- tempat **penyimpanan** token (dan alasannya)
- cara menangani **beberapa akun** (profil + override per sesi)

OpenClaw juga mendukung **plugin provider** yang menyertakan alur OAuth atau kunci API
mereka sendiri. Jalankan melalui:

```bash
openclaw models auth login --provider <id>
```

## Token sink (mengapa ini ada)

Provider OAuth umumnya menerbitkan **refresh token baru** selama alur login/refresh. Beberapa provider (atau klien OAuth) dapat membatalkan refresh token yang lebih lama saat token baru diterbitkan untuk pengguna/aplikasi yang sama.

Gejala praktis:

- Anda login melalui OpenClaw _dan_ melalui Claude Code / Codex CLI → salah satunya nanti secara acak menjadi “logout”

Untuk mengurangi itu, OpenClaw memperlakukan `auth-profiles.json` sebagai **token sink**:

- runtime membaca kredensial dari **satu tempat**
- kita dapat menyimpan beberapa profil dan merutekannya secara deterministik
- saat kredensial digunakan ulang dari CLI eksternal seperti Codex CLI, OpenClaw
  mencerminkannya dengan provenance dan membaca ulang sumber eksternal tersebut alih-alih
  memutar refresh token itu sendiri

## Penyimpanan (tempat token disimpan)

Secret disimpan **per-agent**:

- Profil auth (OAuth + kunci API + ref tingkat nilai opsional): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- File kompatibilitas legacy: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (`api_key` statis akan dibersihkan saat ditemukan)

File legacy khusus impor (masih didukung, tetapi bukan penyimpanan utama):

- `~/.openclaw/credentials/oauth.json` (diimpor ke `auth-profiles.json` saat pertama kali digunakan)

Semua hal di atas juga menghormati `$OPENCLAW_STATE_DIR` (override direktori state). Referensi lengkap: [/gateway/configuration](/id/gateway/configuration-reference#auth-storage)

Untuk ref secret statis dan perilaku aktivasi snapshot runtime, lihat [Manajemen Secret](/id/gateway/secrets).

## Kompatibilitas token legacy Anthropic

<Warning>
Dokumentasi publik Claude Code milik Anthropic mengatakan penggunaan langsung Claude Code tetap berada dalam
batas subscription Claude. Secara terpisah, Anthropic memberi tahu pengguna OpenClaw pada
**4 April 2026 pukul 12.00 PM PT / 8.00 PM BST** bahwa **OpenClaw dihitung sebagai
harness pihak ketiga**. Profil token Anthropic yang ada tetap secara teknis
dapat digunakan di OpenClaw, tetapi Anthropic mengatakan jalur OpenClaw kini memerlukan **Extra
Usage** (bayar sesuai pemakaian yang ditagih terpisah dari subscription) untuk lalu lintas tersebut.

Untuk dokumentasi paket direct-Claude-Code Anthropic saat ini, lihat [Menggunakan Claude Code
dengan paket Pro atau Max Anda](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
dan [Menggunakan Claude Code dengan paket Team atau Enterprise
Anda](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/).

Jika Anda menginginkan opsi bergaya subscription lain di OpenClaw, lihat [OpenAI
Codex](/id/providers/openai), [Qwen Cloud Coding
Plan](/id/providers/qwen), [MiniMax Coding Plan](/id/providers/minimax),
dan [Z.AI / GLM Coding Plan](/id/providers/glm).
</Warning>

OpenClaw sekarang kembali mengekspos Anthropic setup-token sebagai jalur legacy/manual.
Pemberitahuan penagihan khusus OpenClaw dari Anthropic tetap berlaku untuk jalur tersebut, jadi
gunakan dengan pemahaman bahwa Anthropic memerlukan **Extra Usage** untuk
lalu lintas login Claude yang digerakkan oleh OpenClaw.

## Migrasi Anthropic Claude CLI

Anthropic tidak lagi memiliki jalur migrasi Claude CLI lokal yang didukung di
OpenClaw. Gunakan kunci API Anthropic untuk lalu lintas Anthropic, atau pertahankan autentikasi berbasis
token legacy hanya jika sudah dikonfigurasi dan dengan pemahaman
bahwa Anthropic memperlakukan jalur OpenClaw tersebut sebagai **Extra Usage**.

## Pertukaran OAuth (cara login bekerja)

Alur login interaktif OpenClaw diimplementasikan di `@mariozechner/pi-ai` dan dihubungkan ke wizard/perintah.

### Anthropic setup-token

Bentuk alur:

1. mulai Anthropic setup-token atau paste-token dari OpenClaw
2. OpenClaw menyimpan kredensial Anthropic yang dihasilkan dalam profil auth
3. pemilihan model tetap pada `anthropic/...`
4. profil auth Anthropic yang ada tetap tersedia untuk rollback/kontrol urutan

### OpenAI Codex (ChatGPT OAuth)

OpenAI Codex OAuth secara eksplisit didukung untuk digunakan di luar Codex CLI, termasuk alur kerja OpenClaw.

Bentuk alur (PKCE):

1. hasilkan verifier/challenge PKCE + `state` acak
2. buka `https://auth.openai.com/oauth/authorize?...`
3. coba tangkap callback di `http://127.0.0.1:1455/auth/callback`
4. jika callback tidak bisa bind (atau Anda bekerja jarak jauh/headless), tempel URL/code hasil redirect
5. tukarkan di `https://auth.openai.com/oauth/token`
6. ekstrak `accountId` dari access token dan simpan `{ access, refresh, expires, accountId }`

Jalur wizard adalah `openclaw onboard` → pilihan auth `openai-codex`.

## Refresh + kedaluwarsa

Profil menyimpan stempel waktu `expires`.

Saat runtime:

- jika `expires` masih di masa depan → gunakan access token yang tersimpan
- jika sudah kedaluwarsa → refresh (di bawah file lock) dan timpa kredensial yang tersimpan
- pengecualian: kredensial CLI eksternal yang digunakan ulang tetap dikelola secara eksternal; OpenClaw
  membaca ulang penyimpanan auth CLI dan tidak pernah menggunakan refresh token hasil salinan itu sendiri

Alur refresh bersifat otomatis; umumnya Anda tidak perlu mengelola token secara manual.

## Beberapa akun (profil) + perutean

Dua pola:

### 1) Yang disarankan: agent terpisah

Jika Anda ingin “pribadi” dan “kantor” tidak pernah saling berinteraksi, gunakan agent terisolasi (sesi + kredensial + workspace terpisah):

```bash
openclaw agents add work
openclaw agents add personal
```

Lalu konfigurasikan auth per-agent (wizard) dan arahkan chat ke agent yang tepat.

### 2) Lanjutan: beberapa profil dalam satu agent

`auth-profiles.json` mendukung beberapa ID profil untuk provider yang sama.

Pilih profil yang digunakan:

- secara global melalui urutan config (`auth.order`)
- per sesi melalui `/model ...@<profileId>`

Contoh (override sesi):

- `/model Opus@anthropic:work`

Cara melihat ID profil yang tersedia:

- `openclaw channels list --json` (menampilkan `auth[]`)

Dokumentasi terkait:

- [/concepts/model-failover](/id/concepts/model-failover) (aturan rotasi + cooldown)
- [/tools/slash-commands](/id/tools/slash-commands) (permukaan perintah)

## Terkait

- [Autentikasi](/id/gateway/authentication) — ikhtisar auth provider model
- [Secrets](/id/gateway/secrets) — penyimpanan kredensial dan SecretRef
- [Referensi Konfigurasi](/id/gateway/configuration-reference#auth-storage) — kunci config auth
