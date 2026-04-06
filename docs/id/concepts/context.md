---
read_when:
    - Anda ingin memahami arti “konteks” dalam OpenClaw
    - Anda sedang men-debug mengapa model “mengetahui” sesuatu (atau melupakannya)
    - Anda ingin mengurangi overhead konteks (/context, /status, /compact)
summary: 'Konteks: apa yang dilihat model, bagaimana konteks dibangun, dan cara memeriksanya'
title: Konteks
x-i18n:
    generated_at: "2026-04-06T03:06:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: fe7dfe52cb1a64df229c8622feed1804df6c483a6243e0d2f309f6ff5c9fe521
    source_path: concepts/context.md
    workflow: 15
---

# Konteks

“Konteks” adalah **segala hal yang dikirim OpenClaw ke model untuk satu proses run**. Konteks dibatasi oleh **jendela konteks** model (batas token).

Model mental untuk pemula:

- **System prompt** (dibuat oleh OpenClaw): aturan, alat, daftar Skills, waktu/runtime, dan file workspace yang disuntikkan.
- **Riwayat percakapan**: pesan Anda + pesan asisten untuk sesi ini.
- **Pemanggilan/hasil alat + lampiran**: output perintah, pembacaan file, gambar/audio, dan sebagainya.

Konteks _tidak sama_ dengan “memori”: memori dapat disimpan di disk dan dimuat ulang nanti; konteks adalah apa yang berada di dalam jendela model saat ini.

## Mulai cepat (periksa konteks)

- `/status` → tampilan cepat “seberapa penuh jendela saya?” + pengaturan sesi.
- `/context list` → apa yang disuntikkan + ukuran perkiraan (per file + total).
- `/context detail` → perincian yang lebih dalam: per file, ukuran schema alat, ukuran entri skill, dan ukuran system prompt.
- `/usage tokens` → tambahkan footer penggunaan per balasan ke balasan normal.
- `/compact` → ringkas riwayat yang lebih lama menjadi entri ringkas untuk membebaskan ruang jendela.

Lihat juga: [Perintah slash](/id/tools/slash-commands), [Penggunaan token & biaya](/id/reference/token-use), [Pemadatan](/id/concepts/compaction).

## Contoh output

Nilai berbeda-beda menurut model, provider, kebijakan alat, dan isi workspace Anda.

### `/context list`

```
🧠 Perincian konteks
Workspace: <workspaceDir>
Bootstrap maks/file: 20,000 chars
Sandbox: mode=non-main sandboxed=false
System prompt (run): 38,412 chars (~9,603 tok) (Project Context 23,901 chars (~5,976 tok))

File workspace yang disuntikkan:
- AGENTS.md: OK | raw 1,742 chars (~436 tok) | injected 1,742 chars (~436 tok)
- SOUL.md: OK | raw 912 chars (~228 tok) | injected 912 chars (~228 tok)
- TOOLS.md: TRUNCATED | raw 54,210 chars (~13,553 tok) | injected 20,962 chars (~5,241 tok)
- IDENTITY.md: OK | raw 211 chars (~53 tok) | injected 211 chars (~53 tok)
- USER.md: OK | raw 388 chars (~97 tok) | injected 388 chars (~97 tok)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 chars (~0 tok) | injected 0 chars (~0 tok)

Daftar Skills (teks system prompt): 2,184 chars (~546 tok) (12 skills)
Alat: read, edit, write, exec, process, browser, message, sessions_send, …
Daftar alat (teks system prompt): 1,032 chars (~258 tok)
Schema alat (JSON): 31,988 chars (~7,997 tok) (dihitung sebagai konteks; tidak ditampilkan sebagai teks)
Alat: (sama seperti di atas)

Token sesi (cached): 14,250 total / ctx=32,000
```

### `/context detail`

```
🧠 Perincian konteks (terperinci)
…
Skill teratas (ukuran entri prompt):
- frontend-design: 412 chars (~103 tok)
- oracle: 401 chars (~101 tok)
… (+10 skill lainnya)

Alat teratas (ukuran schema):
- browser: 9,812 chars (~2,453 tok)
- exec: 6,240 chars (~1,560 tok)
… (+N alat lainnya)
```

## Apa yang dihitung terhadap jendela konteks

Segala hal yang diterima model akan dihitung, termasuk:

- System prompt (semua bagian).
- Riwayat percakapan.
- Pemanggilan alat + hasil alat.
- Lampiran/transkrip (gambar/audio/file).
- Ringkasan pemadatan dan artefak pemangkasan.
- “Wrapper” provider atau header tersembunyi (tidak terlihat, tetap dihitung).

## Cara OpenClaw membangun system prompt

System prompt **dimiliki oleh OpenClaw** dan dibangun ulang di setiap run. Isinya mencakup:

- Daftar alat + deskripsi singkat.
- Daftar Skills (hanya metadata; lihat di bawah).
- Lokasi workspace.
- Waktu (UTC + waktu pengguna yang dikonversi jika dikonfigurasi).
- Metadata runtime (host/OS/model/thinking).
- File bootstrap workspace yang disuntikkan di bawah **Project Context**.

Perincian lengkap: [System Prompt](/id/concepts/system-prompt).

## File workspace yang disuntikkan (Project Context)

Secara default, OpenClaw menyuntikkan sekumpulan file workspace tetap (jika ada):

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (hanya saat pertama kali dijalankan)

File besar dipotong per file menggunakan `agents.defaults.bootstrapMaxChars` (default `20000` chars). OpenClaw juga menerapkan batas total injeksi bootstrap di seluruh file dengan `agents.defaults.bootstrapTotalMaxChars` (default `150000` chars). `/context` menampilkan ukuran **raw vs injected** dan apakah pemotongan terjadi.

Saat pemotongan terjadi, runtime dapat menyuntikkan blok peringatan di dalam prompt di bawah Project Context. Konfigurasikan ini dengan `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`; default `once`).

## Skills: disuntikkan vs dimuat sesuai kebutuhan

System prompt menyertakan **daftar Skills** yang ringkas (nama + deskripsi + lokasi). Daftar ini memiliki overhead nyata.

Instruksi skill _tidak_ disertakan secara default. Model diharapkan melakukan `read` pada `SKILL.md` milik skill **hanya saat diperlukan**.

## Tools: ada dua biaya

Alat memengaruhi konteks dengan dua cara:

1. **Teks daftar alat** dalam system prompt (yang Anda lihat sebagai “Tooling”).
2. **Schema alat** (JSON). Schema ini dikirim ke model agar model dapat memanggil alat. Schema ini dihitung terhadap konteks meskipun Anda tidak melihatnya sebagai teks biasa.

`/context detail` menguraikan schema alat terbesar agar Anda dapat melihat apa yang paling mendominasi.

## Perintah, directive, dan "shortcut inline"

Perintah slash ditangani oleh Gateway. Ada beberapa perilaku berbeda:

- **Perintah mandiri**: pesan yang hanya berisi `/...` dijalankan sebagai perintah.
- **Directive**: `/think`, `/verbose`, `/reasoning`, `/elevated`, `/model`, `/queue` dihapus sebelum model melihat pesan.
  - Pesan yang hanya berisi directive akan mempertahankan pengaturan sesi.
  - Directive inline dalam pesan normal bertindak sebagai petunjuk per pesan.
- **Shortcut inline** (hanya untuk pengirim dalam allowlist): token `/...` tertentu di dalam pesan normal dapat langsung dijalankan (contoh: “hey /status”), dan dihapus sebelum model melihat teks yang tersisa.

Detail: [Perintah slash](/id/tools/slash-commands).

## Sesi, pemadatan, dan pemangkasan (apa yang dipertahankan)

Apa yang dipertahankan antar pesan bergantung pada mekanismenya:

- **Riwayat normal** dipertahankan dalam transkrip sesi sampai dipadatkan/dipangkas oleh kebijakan.
- **Pemadatan** mempertahankan ringkasan ke dalam transkrip dan menjaga pesan terbaru tetap utuh.
- **Pemangkasan** menghapus hasil alat lama dari prompt _dalam memori_ untuk sebuah run, tetapi tidak menulis ulang transkrip.

Dokumentasi: [Sesi](/id/concepts/session), [Pemadatan](/id/concepts/compaction), [Pemangkasan sesi](/id/concepts/session-pruning).

Secara default, OpenClaw menggunakan mesin konteks bawaan `legacy` untuk perakitan dan
pemadatan. Jika Anda memasang plugin yang menyediakan `kind: "context-engine"` dan
memilihnya dengan `plugins.slots.contextEngine`, OpenClaw mendelegasikan perakitan konteks,
`/compact`, dan hook siklus hidup konteks subagen terkait ke mesin tersebut.
`ownsCompaction: false` tidak secara otomatis fallback ke mesin legacy;
mesin aktif tetap harus mengimplementasikan `compact()` dengan benar. Lihat
[Context Engine](/id/concepts/context-engine) untuk antarmuka yang dapat dipasang sebagai plugin secara lengkap,
hook siklus hidup, dan konfigurasi.

## Apa yang sebenarnya dilaporkan `/context`

`/context` lebih memilih laporan system prompt **yang dibangun saat run** terbaru jika tersedia:

- `System prompt (run)` = diambil dari run embedded terakhir (mampu menggunakan alat) dan dipertahankan di session store.
- `System prompt (estimate)` = dihitung secara langsung ketika belum ada laporan run.

Dalam kedua kasus, ukuran dan kontributor teratas dilaporkan; perintah ini **tidak** menampilkan seluruh system prompt atau schema alat.

## Terkait

- [Context Engine](/id/concepts/context-engine) — injeksi konteks kustom melalui plugin
- [Pemadatan](/id/concepts/compaction) — meringkas percakapan panjang
- [System Prompt](/id/concepts/system-prompt) — cara system prompt dibangun
- [Loop Agen](/id/concepts/agent-loop) — siklus eksekusi agen secara penuh
