---
read_when:
    - Anda sedang menghubungkan transport QA sintetis ke dalam pengujian lokal atau CI
    - Anda memerlukan permukaan konfigurasi `qa-channel` bawaan
    - Anda sedang mengiterasi otomatisasi QA end-to-end
summary: Plugin channel sintetis kelas Slack untuk skenario QA OpenClaw yang deterministik
title: Channel QA
x-i18n:
    generated_at: "2026-04-06T03:05:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b88cd73df2f61b34ad1eb83c3450f8fe15a51ac69fbb5a9eca0097564d67a06
    source_path: channels/qa-channel.md
    workflow: 15
---

# Channel QA

`qa-channel` adalah transport pesan sintetis bawaan untuk QA OpenClaw otomatis.

Ini bukan channel produksi. Channel ini ada untuk menguji batas plugin channel
yang sama seperti yang digunakan oleh transport nyata sambil menjaga status tetap
deterministik dan sepenuhnya dapat diinspeksi.

## Apa yang dilakukannya saat ini

- Tata bahasa target kelas Slack:
  - `dm:<user>`
  - `channel:<room>`
  - `thread:<room>/<thread>`
- Bus sintetis berbasis HTTP untuk:
  - injeksi pesan masuk
  - penangkapan transkrip keluar
  - pembuatan thread
  - reaksi
  - pengeditan
  - penghapusan
  - tindakan pencarian dan pembacaan
- Runner pemeriksaan mandiri sisi host bawaan yang menulis laporan Markdown

## Konfigurasi

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

Kunci akun yang didukung:

- `baseUrl`
- `botUserId`
- `botDisplayName`
- `pollTimeoutMs`
- `allowFrom`
- `defaultTo`
- `actions.messages`
- `actions.reactions`
- `actions.search`
- `actions.threads`

## Runner

Irisan vertikal saat ini:

```bash
pnpm qa:e2e
```

Ini sekarang dirutekan melalui ekstensi `qa-lab` bawaan. Ekstensi ini memulai
bus QA di dalam repo, menjalankan irisan runtime `qa-channel` bawaan, menjalankan
pemeriksaan mandiri yang deterministik, dan menulis laporan Markdown di bawah
`.artifacts/qa-e2e/`.

UI debugger privat:

```bash
pnpm qa:lab:build
pnpm openclaw qa ui
```

Suite QA lengkap berbasis repo:

```bash
pnpm openclaw qa suite
```

Itu meluncurkan debugger QA privat di URL lokal, terpisah dari bundle Control UI
yang dikirimkan.

## Cakupan

Cakupan saat ini sengaja sempit:

- bus + transport plugin
- tata bahasa perutean thread
- tindakan pesan milik channel
- pelaporan Markdown

Pekerjaan lanjutan akan menambahkan:

- orkestrasi OpenClaw yang didockerisasi
- eksekusi matriks provider/model
- penemuan skenario yang lebih kaya
- orkestrasi native OpenClaw nanti
