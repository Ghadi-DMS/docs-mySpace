---
read_when:
    - Memperluas qa-lab atau qa-channel
    - Menambahkan skenario QA yang didukung repo
    - Membangun otomatisasi QA dengan realisme lebih tinggi di sekitar dashboard Gateway
summary: Bentuk otomatisasi QA privat untuk qa-lab, qa-channel, skenario seed, dan laporan protokol
title: Otomatisasi QA E2E
x-i18n:
    generated_at: "2026-04-06T03:06:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: df35f353d5ab0e0432e6a828c82772f9a88edb41c20ec5037315b7ba310b28e6
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Otomatisasi QA E2E

Stack QA privat dimaksudkan untuk menguji OpenClaw dengan cara yang lebih realistis dan
berbentuk channel dibandingkan yang dapat dilakukan oleh satu unit test.

Komponen saat ini:

- `extensions/qa-channel`: channel pesan sintetis dengan permukaan DM, channel, thread,
  reaksi, edit, dan hapus.
- `extensions/qa-lab`: UI debugger dan bus QA untuk mengamati transkrip,
  menyuntikkan pesan masuk, dan mengekspor laporan Markdown.
- `qa/`: aset seed yang didukung repo untuk tugas kickoff dan skenario QA
  dasar.

Tujuan jangka panjangnya adalah situs QA dua panel:

- Kiri: dashboard Gateway (UI Control) dengan agen.
- Kanan: QA Lab, menampilkan transkrip bergaya Slack dan rencana skenario.

Ini memungkinkan operator atau loop otomatisasi memberi agen misi QA, mengamati
perilaku channel nyata, dan mencatat apa yang berhasil, gagal, atau tetap terblokir.

## Seed yang didukung repo

Aset seed berada di `qa/`:

- `qa/QA_KICKOFF_TASK.md`
- `qa/seed-scenarios.json`

Aset ini sengaja berada di git agar rencana QA terlihat baik oleh manusia maupun
agen. Daftar dasar harus tetap cukup luas untuk mencakup:

- obrolan DM dan channel
- perilaku thread
- siklus hidup aksi pesan
- callback cron
- recall memory
- perpindahan model
- handoff subagen
- pembacaan repo dan pembacaan docs
- satu tugas build kecil seperti Lobster Invaders

## Pelaporan

`qa-lab` mengekspor laporan protokol Markdown dari timeline bus yang diamati.
Laporan tersebut harus menjawab:

- Apa yang berhasil
- Apa yang gagal
- Apa yang tetap terblokir
- Skenario tindak lanjut apa yang layak ditambahkan

## Docs terkait

- [Testing](/id/help/testing)
- [QA Channel](/channels/qa-channel)
- [Dashboard](/web/dashboard)
