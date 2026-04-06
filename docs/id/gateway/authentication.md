---
read_when:
    - Men-debug autentikasi model atau masa berlaku OAuth
    - Mendokumentasikan autentikasi atau penyimpanan kredensial
summary: 'Autentikasi model: OAuth, API key, dan setup-token Anthropic lama'
title: Autentikasi
x-i18n:
    generated_at: "2026-04-06T03:07:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: f59ede3fcd7e692ad4132287782a850526acf35474b5bfcea29e0e23610636c2
    source_path: gateway/authentication.md
    workflow: 15
---

# Autentikasi (Provider Model)

<Note>
Halaman ini membahas autentikasi **provider model** (API key, OAuth, dan setup-token Anthropic lama). Untuk autentikasi **koneksi gateway** (token, kata sandi, trusted-proxy), lihat [Configuration](/id/gateway/configuration) dan [Trusted Proxy Auth](/id/gateway/trusted-proxy-auth).
</Note>

OpenClaw mendukung OAuth dan API key untuk provider model. Untuk host gateway
yang selalu aktif, API key biasanya merupakan opsi yang paling dapat diprediksi. Alur
langganan/OAuth juga didukung jika sesuai dengan model akun provider Anda.

Lihat [/concepts/oauth](/id/concepts/oauth) untuk alur OAuth lengkap dan tata letak
penyimpanan.
Untuk autentikasi berbasis SecretRef (provider `env`/`file`/`exec`), lihat [Secrets Management](/id/gateway/secrets).
Untuk aturan kelayakan kredensial/kode alasan yang digunakan oleh `models status --probe`, lihat
[Semantik Kredensial Autentikasi](/id/auth-credential-semantics).

## Penyiapan yang direkomendasikan (API key, provider apa pun)

Jika Anda menjalankan gateway yang berumur panjang, mulai dengan API key untuk
provider pilihan Anda.
Khusus untuk Anthropic, autentikasi API key adalah jalur yang aman. Autentikasi gaya
langganan Anthropic di dalam OpenClaw adalah jalur setup-token lama dan
harus diperlakukan sebagai jalur **Extra Usage**, bukan jalur batas paket.

1. Buat API key di konsol provider Anda.
2. Tempatkan di **host gateway** (mesin yang menjalankan `openclaw gateway`).

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Jika Gateway berjalan di bawah systemd/launchd, sebaiknya letakkan key di
   `~/.openclaw/.env` agar daemon dapat membacanya:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

Lalu mulai ulang daemon (atau mulai ulang proses Gateway Anda) dan periksa kembali:

```bash
openclaw models status
openclaw doctor
```

Jika Anda lebih memilih untuk tidak mengelola env vars sendiri, onboarding dapat menyimpan
API key untuk penggunaan daemon: `openclaw onboard`.

Lihat [Help](/id/help) untuk detail tentang pewarisan env (`env.shellEnv`,
`~/.openclaw/.env`, systemd/launchd).

## Anthropic: kompatibilitas token lama

Autentikasi setup-token Anthropic masih tersedia di OpenClaw sebagai
jalur lama/manual. Docs publik Claude Code milik Anthropic masih membahas
penggunaan terminal Claude Code langsung di bawah paket Claude, tetapi Anthropic secara terpisah memberi tahu
pengguna OpenClaw bahwa jalur login Claude **OpenClaw** dihitung sebagai penggunaan harness pihak ketiga
dan memerlukan **Extra Usage** yang ditagih terpisah dari
langganan.

Untuk jalur penyiapan yang paling jelas, gunakan API key Anthropic. Jika Anda harus tetap menggunakan
jalur Anthropic gaya langganan di OpenClaw, gunakan jalur setup-token lama
dengan harapan bahwa Anthropic memperlakukannya sebagai **Extra Usage**.

Entri token manual (provider apa pun; menulis `auth-profiles.json` + memperbarui config):

```bash
openclaw models auth paste-token --provider openrouter
```

Referensi profil auth juga didukung untuk kredensial statis:

- kredensial `api_key` dapat menggunakan `keyRef: { source, provider, id }`
- kredensial `token` dapat menggunakan `tokenRef: { source, provider, id }`
- profil mode OAuth tidak mendukung kredensial SecretRef; jika `auth.profiles.<id>.mode` disetel ke `"oauth"`, input `keyRef`/`tokenRef` berbasis SecretRef untuk profil tersebut akan ditolak.

Pemeriksaan yang ramah otomatisasi (keluar `1` saat kedaluwarsa/tidak ada, `2` saat akan kedaluwarsa):

```bash
openclaw models status --check
```

Probe auth langsung:

```bash
openclaw models status --probe
```

Catatan:

- Baris probe dapat berasal dari profil auth, kredensial env, atau `models.json`.
- Jika `auth.order.<provider>` eksplisit menghilangkan profil yang tersimpan, laporan probe
  menampilkan `excluded_by_auth_order` untuk profil tersebut alih-alih mencobanya.
- Jika auth ada tetapi OpenClaw tidak dapat menyelesaikan kandidat model yang dapat diprobe untuk
  provider tersebut, probe melaporkan `status: no_model`.
- Cooldown batas laju dapat bersifat khusus model. Profil yang sedang cooldown untuk satu
  model masih dapat digunakan untuk model serumpun pada provider yang sama.

Skrip opsional ops (systemd/Termux) didokumentasikan di sini:
[Skrip pemantauan auth](/id/help/scripts#auth-monitoring-scripts)

## Catatan Anthropic

Backend Anthropic `claude-cli` telah dihapus.

- Gunakan API key Anthropic untuk trafik Anthropic di OpenClaw.
- Setup-token Anthropic tetap menjadi jalur lama/manual dan harus digunakan dengan
  ekspektasi penagihan Extra Usage yang Anthropic sampaikan kepada pengguna OpenClaw.
- `openclaw doctor` sekarang mendeteksi status Anthropic Claude CLI lama yang telah dihapus. Jika
  byte kredensial yang tersimpan masih ada, doctor mengonversinya kembali menjadi
  profil token/OAuth Anthropic. Jika tidak, doctor menghapus config Claude CLI
  yang usang dan mengarahkan Anda ke pemulihan API key atau setup-token.

## Memeriksa status auth model

```bash
openclaw models status
openclaw doctor
```

## Perilaku rotasi API key (gateway)

Beberapa provider mendukung percobaan ulang permintaan dengan key alternatif ketika panggilan API
mencapai batas laju provider.

- Urutan prioritas:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (satu override)
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- Provider Google juga menyertakan `GOOGLE_API_KEY` sebagai fallback tambahan.
- Daftar key yang sama dihapus duplikasinya sebelum digunakan.
- OpenClaw mencoba ulang dengan key berikutnya hanya untuk error batas laju (misalnya
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent
requests`, `ThrottlingException`, `concurrency limit reached`, atau
  `workers_ai ... quota limit exceeded`).
- Error selain batas laju tidak dicoba ulang dengan key alternatif.
- Jika semua key gagal, error terakhir dari upaya terakhir akan dikembalikan.

## Mengontrol kredensial mana yang digunakan

### Per sesi (perintah chat)

Gunakan `/model <alias-or-id>@<profileId>` untuk menyematkan kredensial provider tertentu pada sesi saat ini (contoh id profil: `anthropic:default`, `anthropic:work`).

Gunakan `/model` (atau `/model list`) untuk pemilih ringkas; gunakan `/model status` untuk tampilan lengkap (kandidat + profil auth berikutnya, serta detail endpoint provider saat dikonfigurasi).

### Per agen (override CLI)

Setel override urutan profil auth eksplisit untuk sebuah agen (disimpan di `auth-profiles.json` milik agen tersebut):

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Gunakan `--agent <id>` untuk menargetkan agen tertentu; hilangkan untuk menggunakan agen default yang dikonfigurasi.
Saat Anda men-debug masalah urutan, `openclaw models status --probe` menampilkan profil
tersimpan yang dihilangkan sebagai `excluded_by_auth_order` alih-alih melewatinya secara diam-diam.
Saat Anda men-debug masalah cooldown, ingat bahwa cooldown batas laju dapat terikat
ke satu id model, bukan seluruh profil provider.

## Pemecahan masalah

### "Kredensial tidak ditemukan"

Jika profil Anthropic tidak ada, konfigurasikan API key Anthropic di
**host gateway** atau siapkan jalur setup-token Anthropic lama, lalu periksa kembali:

```bash
openclaw models status
```

### Token akan kedaluwarsa/sudah kedaluwarsa

Jalankan `openclaw models status` untuk mengonfirmasi profil mana yang akan kedaluwarsa. Jika profil token Anthropic
lama tidak ada atau kedaluwarsa, segarkan penyiapan tersebut melalui
setup-token atau migrasikan ke API key Anthropic.

Jika mesin masih memiliki status Anthropic Claude CLI lama yang telah dihapus dari build
sebelumnya, jalankan:

```bash
openclaw doctor --yes
```

Doctor mengonversi `anthropic:claude-cli` kembali ke token/OAuth Anthropic ketika
byte kredensial yang tersimpan masih ada. Jika tidak, doctor menghapus referensi
profil/config/model Claude CLI yang usang dan meninggalkan panduan langkah berikutnya.
