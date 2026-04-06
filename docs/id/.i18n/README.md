---
x-i18n:
    generated_at: "2026-04-06T03:05:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6e1cf417b0c04d001bc494fbe03ac2fcb66866f759e21646dbfd1a9c3a968bff
    source_path: .i18n/README.md
    workflow: 15
---

# Aset i18n docs OpenClaw

Folder ini menyimpan konfigurasi terjemahan untuk repo docs sumber.

Halaman lokal yang dihasilkan dan memori terjemahan lokal langsung kini berada di repo publish (`openclaw/docs`, checkout sibling lokal `~/Projects/openclaw-docs`).

## File

- `glossary.<lang>.json` — pemetaan istilah yang diutamakan (digunakan dalam panduan prompt).
- `<lang>.tm.jsonl` — memori terjemahan (cache) yang dikunci oleh alur kerja + model + hash teks. Di repo ini, file TM lokal dibuat sesuai permintaan.

## Format glosarium

`glossary.<lang>.json` adalah array dari entri:

```json
{
  "source": "troubleshooting",
  "target": "故障排除",
  "ignore_case": true,
  "whole_word": false
}
```

Bidang:

- `source`: frasa bahasa Inggris (atau sumber) yang diprioritaskan.
- `target`: keluaran terjemahan yang diutamakan.

## Catatan

- Entri glosarium diteruskan ke model sebagai **panduan prompt** (tanpa penulisan ulang deterministik).
- `scripts/docs-i18n` masih menangani pembuatan terjemahan.
- Repo sumber menyinkronkan docs bahasa Inggris ke repo publish; pembuatan lokal berjalan di sana per lokal saat push, jadwal, dan dispatch rilis.
