---
read_when:
    - Anda ingin memahami cara kerja memory
    - Anda ingin mengetahui file memory apa yang harus ditulis
summary: Cara OpenClaw mengingat berbagai hal di seluruh sesi
title: Ikhtisar Memory
x-i18n:
    generated_at: "2026-04-06T03:06:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: d19d4fa9c4b3232b7a97f7a382311d2a375b562040de15e9fe4a0b1990b825e7
    source_path: concepts/memory.md
    workflow: 15
---

# Ikhtisar Memory

OpenClaw mengingat berbagai hal dengan menulis **file Markdown biasa** di
workspace agen Anda. Model hanya "mengingat" apa yang disimpan ke disk -- tidak ada
status tersembunyi.

## Cara kerjanya

Agen Anda memiliki tiga file yang terkait dengan memory:

- **`MEMORY.md`** -- memory jangka panjang. Fakta, preferensi, dan
  keputusan yang tahan lama. Dimuat pada awal setiap sesi DM.
- **`memory/YYYY-MM-DD.md`** -- catatan harian. Konteks berjalan dan pengamatan.
  Catatan hari ini dan kemarin dimuat secara otomatis.
- **`DREAMS.md`** (eksperimental, opsional) -- Dream Diary dan ringkasan sapuan
  dreaming untuk tinjauan manusia.

File-file ini berada di workspace agen (default `~/.openclaw/workspace`).

<Tip>
Jika Anda ingin agen Anda mengingat sesuatu, cukup minta: "Ingat bahwa saya
lebih suka TypeScript." Agen akan menuliskannya ke file yang sesuai.
</Tip>

## Alat memory

Agen memiliki dua alat untuk bekerja dengan memory:

- **`memory_search`** -- menemukan catatan yang relevan menggunakan pencarian semantik, bahkan ketika
  susunan katanya berbeda dari aslinya.
- **`memory_get`** -- membaca file memory tertentu atau rentang baris.

Kedua alat disediakan oleh plugin memory yang aktif (default: `memory-core`).

## Pencarian memory

Saat provider embedding dikonfigurasi, `memory_search` menggunakan **pencarian
hibrida** -- menggabungkan kemiripan vektor (makna semantik) dengan pencocokan kata kunci
(istilah persis seperti ID dan simbol kode). Ini langsung berfungsi setelah Anda memiliki
API key untuk provider yang didukung.

<Info>
OpenClaw mendeteksi otomatis provider embedding Anda dari API key yang tersedia. Jika Anda
mengonfigurasi key OpenAI, Gemini, Voyage, atau Mistral, pencarian memory akan
diaktifkan secara otomatis.
</Info>

Untuk detail tentang cara kerja pencarian, opsi penyetelan, dan penyiapan provider, lihat
[Memory Search](/id/concepts/memory-search).

## Backend memory

<CardGroup cols={3}>
<Card title="Bawaan (default)" icon="database" href="/id/concepts/memory-builtin">
Berbasis SQLite. Langsung berfungsi dengan pencarian kata kunci, kemiripan vektor, dan
pencarian hibrida. Tanpa dependensi tambahan.
</Card>
<Card title="QMD" icon="search" href="/id/concepts/memory-qmd">
Sidecar local-first dengan reranking, perluasan kueri, dan kemampuan untuk mengindeks
direktori di luar workspace.
</Card>
<Card title="Honcho" icon="brain" href="/id/concepts/memory-honcho">
Memory lintas sesi yang AI-native dengan pemodelan pengguna, pencarian semantik, dan
kesadaran multi-agen. Instal plugin.
</Card>
</CardGroup>

## Flush memory otomatis

Sebelum [compaction](/id/concepts/compaction) merangkum percakapan Anda, OpenClaw
menjalankan giliran senyap yang mengingatkan agen untuk menyimpan konteks penting ke file
memory. Ini aktif secara default -- Anda tidak perlu mengonfigurasi apa pun.

<Tip>
Flush memory mencegah hilangnya konteks selama compaction. Jika agen Anda memiliki
fakta penting dalam percakapan yang belum ditulis ke file, fakta tersebut
akan disimpan secara otomatis sebelum peringkasan terjadi.
</Tip>

## Dreaming (eksperimental)

Dreaming adalah proses konsolidasi latar belakang opsional untuk memory. Ini mengumpulkan
sinyal jangka pendek, menilai kandidat, dan hanya mempromosikan item yang memenuhi syarat ke
memory jangka panjang (`MEMORY.md`).

Ini dirancang untuk menjaga memory jangka panjang tetap memiliki sinyal tinggi:

- **Opt-in**: dinonaktifkan secara default.
- **Terjadwal**: saat diaktifkan, `memory-core` mengelola otomatis satu cron job berulang
  untuk satu sapuan dreaming penuh.
- **Berambang batas**: promosi harus lolos ambang skor, frekuensi recall, dan
  keragaman kueri.
- **Dapat ditinjau**: ringkasan fase dan entri diary ditulis ke `DREAMS.md`
  untuk tinjauan manusia.

Untuk perilaku fase, sinyal penilaian, dan detail Dream Diary, lihat
[Dreaming (eksperimental)](/concepts/dreaming).

## CLI

```bash
openclaw memory status          # Periksa status indeks dan provider
openclaw memory search "query"  # Cari dari command line
openclaw memory index --force   # Bangun ulang indeks
```

## Bacaan lebih lanjut

- [Builtin Memory Engine](/id/concepts/memory-builtin) -- backend SQLite default
- [QMD Memory Engine](/id/concepts/memory-qmd) -- sidecar local-first tingkat lanjut
- [Honcho Memory](/id/concepts/memory-honcho) -- memory lintas sesi AI-native
- [Memory Search](/id/concepts/memory-search) -- pipeline pencarian, provider, dan
  penyetelan
- [Dreaming (eksperimental)](/concepts/dreaming) -- promosi latar belakang
  dari recall jangka pendek ke memory jangka panjang
- [Referensi konfigurasi Memory](/id/reference/memory-config) -- semua opsi konfigurasi
- [Compaction](/id/concepts/compaction) -- cara compaction berinteraksi dengan memory
