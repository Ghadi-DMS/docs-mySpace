---
read_when:
    - Anda ingin promosi memori berjalan secara otomatis
    - Anda ingin memahami apa yang dilakukan setiap fase Dreaming
    - Anda ingin menyetel konsolidasi tanpa mengotori `MEMORY.md`
summary: Konsolidasi memori latar belakang dengan fase ringan, dalam, dan REM serta Buku Harian Mimpi
title: Dreaming (eksperimental)
x-i18n:
    generated_at: "2026-04-15T09:14:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5882a5068f2eabe54ca9893184e5385330a432b921870c38626399ce11c31e25
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming (eksperimental)

Dreaming adalah sistem konsolidasi memori latar belakang di `memory-core`.
Sistem ini membantu OpenClaw memindahkan sinyal jangka pendek yang kuat ke memori yang tahan lama sambil
menjaga prosesnya tetap dapat dijelaskan dan ditinjau.

Dreaming bersifat **opsional** dan dinonaktifkan secara default.

## Apa yang ditulis oleh dreaming

Dreaming menyimpan dua jenis output:

- **Status mesin** di `memory/.dreams/` (penyimpanan recall, sinyal fase, checkpoint ingestion, lock).
- **Output yang dapat dibaca manusia** di `DREAMS.md` (atau `dreams.md` yang sudah ada) dan file laporan fase opsional di bawah `memory/dreaming/<phase>/YYYY-MM-DD.md`.

Promosi jangka panjang tetap hanya menulis ke `MEMORY.md`.

## Model fase

Dreaming menggunakan tiga fase kooperatif:

| Fase | Tujuan                                    | Penulisan tahan lama |
| ----- | ----------------------------------------- | -------------------- |
| Light | Mengurutkan dan menyiapkan materi jangka pendek terbaru | Tidak                |
| Deep  | Menilai dan mempromosikan kandidat tahan lama | Ya (`MEMORY.md`)     |
| REM   | Merefleksikan tema dan gagasan yang berulang | Tidak                |

Fase-fase ini adalah detail implementasi internal, bukan "mode"
terpisah yang dikonfigurasi pengguna.

### Fase Light

Fase Light mengingest sinyal memori harian terbaru dan jejak recall, menghapus duplikasi,
dan menyiapkan baris kandidat.

- Membaca dari status recall jangka pendek, file memori harian terbaru, dan transkrip sesi yang telah disunting bila tersedia.
- Menulis blok `## Light Sleep` yang dikelola ketika penyimpanan mencakup output inline.
- Mencatat sinyal penguatan untuk peringkat deep di tahap selanjutnya.
- Tidak pernah menulis ke `MEMORY.md`.

### Fase Deep

Fase Deep menentukan apa yang menjadi memori jangka panjang.

- Memberi peringkat kandidat menggunakan penilaian berbobot dan gerbang ambang.
- Mengharuskan `minScore`, `minRecallCount`, dan `minUniqueQueries` terpenuhi.
- Menghidrasi ulang cuplikan dari file harian aktif sebelum menulis, sehingga cuplikan lama/yang dihapus dilewati.
- Menambahkan entri yang dipromosikan ke `MEMORY.md`.
- Menulis ringkasan `## Deep Sleep` ke `DREAMS.md` dan secara opsional menulis `memory/dreaming/deep/YYYY-MM-DD.md`.

### Fase REM

Fase REM mengekstrak pola dan sinyal reflektif.

- Membangun ringkasan tema dan refleksi dari jejak jangka pendek terbaru.
- Menulis blok `## REM Sleep` yang dikelola ketika penyimpanan mencakup output inline.
- Mencatat sinyal penguatan REM yang digunakan oleh peringkat deep.
- Tidak pernah menulis ke `MEMORY.md`.

## Ingestion transkrip sesi

Dreaming dapat mengingest transkrip sesi yang telah disunting ke dalam korpus dreaming. Ketika
transkrip tersedia, transkrip tersebut dimasukkan ke fase Light bersama
sinyal memori harian dan jejak recall. Konten pribadi dan sensitif disunting
sebelum ingestion.

## Dream Diary

Dreaming juga menyimpan **Dream Diary** naratif di `DREAMS.md`.
Setelah setiap fase memiliki cukup materi, `memory-core` menjalankan giliran subagen latar belakang
best-effort (menggunakan model runtime default) dan menambahkan entri diary singkat.

Diary ini ditujukan untuk dibaca manusia di UI Dreams, bukan sebagai sumber promosi.
Artefak diary/laporan yang dihasilkan Dreaming dikecualikan dari
promosi jangka pendek. Hanya cuplikan memori yang berlandaskan bukti yang memenuhi syarat untuk dipromosikan ke
`MEMORY.md`.

Ada juga jalur backfill historis yang berlandaskan bukti untuk pekerjaan peninjauan dan pemulihan:

- `memory rem-harness --path ... --grounded` mempratinjau output diary berlandaskan bukti dari catatan historis `YYYY-MM-DD.md`.
- `memory rem-backfill --path ...` menulis entri diary berlandaskan bukti yang dapat dibalik ke `DREAMS.md`.
- `memory rem-backfill --path ... --stage-short-term` menyiapkan kandidat tahan lama berlandaskan bukti ke penyimpanan bukti jangka pendek yang sama yang sudah digunakan oleh fase Deep normal.
- `memory rem-backfill --rollback` dan `--rollback-short-term` menghapus artefak backfill yang telah disiapkan tersebut tanpa menyentuh entri diary biasa atau recall jangka pendek aktif.

Control UI menampilkan alur backfill/reset diary yang sama sehingga Anda dapat memeriksa
hasilnya di scene Dreams sebelum memutuskan apakah kandidat berlandaskan bukti tersebut
layak dipromosikan. Scene juga menampilkan jalur berlandaskan bukti yang berbeda sehingga Anda dapat melihat
entri jangka pendek yang disiapkan mana yang berasal dari pemutaran ulang historis, item yang dipromosikan
mana yang dipimpin oleh data berlandaskan bukti, serta menghapus hanya entri tersiap berlandaskan bukti
tanpa menyentuh status jangka pendek aktif biasa.

## Sinyal peringkat Deep

Peringkat deep menggunakan enam sinyal dasar berbobot ditambah penguatan fase:

| Sinyal              | Bobot | Deskripsi                                         |
| ------------------- | ----- | ------------------------------------------------- |
| Frekuensi           | 0.24  | Berapa banyak sinyal jangka pendek yang dikumpulkan entri |
| Relevansi           | 0.30  | Kualitas pengambilan rata-rata untuk entri        |
| Keragaman kueri     | 0.15  | Konteks kueri/hari berbeda yang memunculkannya    |
| Kekinian            | 0.15  | Skor kesegaran yang meluruh terhadap waktu        |
| Konsolidasi         | 0.10  | Kekuatan kemunculan berulang lintas hari          |
| Kekayaan konseptual | 0.06  | Kepadatan tag konsep dari cuplikan/path           |

Hit fase Light dan REM menambahkan sedikit peningkatan yang meluruh terhadap waktu dari
`memory/.dreams/phase-signals.json`.

## Penjadwalan

Saat diaktifkan, `memory-core` secara otomatis mengelola satu job cron untuk satu penyapuan dreaming penuh. Setiap penyapuan menjalankan fase secara berurutan: light -> REM -> deep.

Perilaku cadence default:

| Pengaturan           | Default     |
| -------------------- | ----------- |
| `dreaming.frequency` | `0 3 * * *` |

## Mulai cepat

Aktifkan dreaming:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Aktifkan dreaming dengan cadence penyapuan kustom:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "timezone": "America/Los_Angeles",
            "frequency": "0 */6 * * *"
          }
        }
      }
    }
  }
}
```

## Perintah slash

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## Alur kerja CLI

Gunakan promosi CLI untuk pratinjau atau penerapan manual:

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

`memory promote` manual menggunakan ambang fase Deep secara default kecuali ditimpa
dengan flag CLI.

Jelaskan mengapa kandidat tertentu akan atau tidak akan dipromosikan:

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

Pratinjau refleksi REM, kebenaran kandidat, dan output promosi deep tanpa
menulis apa pun:

```bash
openclaw memory rem-harness
openclaw memory rem-harness --json
```

## Nilai default utama

Semua pengaturan berada di bawah `plugins.entries.memory-core.config.dreaming`.

| Kunci       | Default     |
| ----------- | ----------- |
| `enabled`   | `false`     |
| `frequency` | `0 3 * * *` |

Kebijakan fase, ambang, dan perilaku penyimpanan adalah detail implementasi
internal (bukan konfigurasi yang ditujukan untuk pengguna).

Lihat [Referensi konfigurasi Memory](/id/reference/memory-config#dreaming-experimental)
untuk daftar kunci lengkap.

## UI Dreams

Saat diaktifkan, tab **Dreams** di Gateway menampilkan:

- status dreaming aktif saat ini
- status tingkat fase dan keberadaan managed sweep
- jumlah jangka pendek, berlandaskan bukti, sinyal, dan yang dipromosikan hari ini
- waktu jalankan terjadwal berikutnya
- jalur Scene berlandaskan bukti yang berbeda untuk entri replay historis yang disiapkan
- pembaca Dream Diary yang dapat diperluas yang didukung oleh `doctor.memory.dreamDiary`

## Terkait

- [Memory](/id/concepts/memory)
- [Pencarian Memory](/id/concepts/memory-search)
- [CLI memory](/cli/memory)
- [Referensi konfigurasi Memory](/id/reference/memory-config)
