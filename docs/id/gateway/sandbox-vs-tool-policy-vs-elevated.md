---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'Mengapa sebuah alat diblokir: runtime sandbox, kebijakan izinkan/tolak alat, dan gerbang exec elevated'
title: Sandbox vs Kebijakan Alat vs Elevated
x-i18n:
    generated_at: "2026-04-06T03:07:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 331f5b2f0d5effa1320125d9f29948e16d0deaffa59eb1e4f25a63481cbe22d6
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 15
---

# Sandbox vs Kebijakan Alat vs Elevated

OpenClaw memiliki tiga kontrol yang saling terkait (tetapi berbeda):

1. **Sandbox** (`agents.defaults.sandbox.*` / `agents.list[].sandbox.*`) menentukan **di mana alat dijalankan** (Docker vs host).
2. **Kebijakan alat** (`tools.*`, `tools.sandbox.tools.*`, `agents.list[].tools.*`) menentukan **alat mana yang tersedia/diizinkan**.
3. **Elevated** (`tools.elevated.*`, `agents.list[].tools.elevated.*`) adalah **jalur keluar khusus exec** untuk berjalan di luar sandbox saat Anda berada dalam sandbox (`gateway` secara default, atau `node` saat target exec dikonfigurasi ke `node`).

## Debug cepat

Gunakan inspector untuk melihat apa yang _sebenarnya_ dilakukan OpenClaw:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

Perintah ini menampilkan:

- mode/cakupan/akses workspace sandbox yang efektif
- apakah sesi saat ini berada dalam sandbox (main vs non-main)
- allow/deny alat sandbox yang efektif (dan apakah berasal dari agent/global/default)
- gerbang elevated dan path kunci perbaikan

## Sandbox: tempat alat dijalankan

Sandboxing dikendalikan oleh `agents.defaults.sandbox.mode`:

- `"off"`: semuanya berjalan di host.
- `"non-main"`: hanya sesi non-main yang berada dalam sandbox (sering menjadi “kejutan” umum untuk grup/channel).
- `"all"`: semuanya berada dalam sandbox.

Lihat [Sandboxing](/id/gateway/sandboxing) untuk matriks lengkapnya (cakupan, mount workspace, image).

### Bind mount (pemeriksaan keamanan cepat)

- `docker.binds` _menembus_ filesystem sandbox: apa pun yang Anda mount akan terlihat di dalam container dengan mode yang Anda tetapkan (`:ro` atau `:rw`).
- Default-nya adalah read-write jika Anda menghilangkan mode; utamakan `:ro` untuk source/secret.
- `scope: "shared"` mengabaikan bind per-agent (hanya bind global yang berlaku).
- OpenClaw memvalidasi sumber bind dua kali: pertama pada path sumber yang telah dinormalisasi, lalu sekali lagi setelah menyelesaikannya melalui ancestor terdalam yang ada. Escape melalui parent symlink tidak melewati pemeriksaan blocked-path atau allowed-root.
- Path leaf yang belum ada tetap diperiksa dengan aman. Jika `/workspace/alias-out/new-file` diselesaikan melalui parent yang disymlink ke path yang diblokir atau ke luar allowed roots yang dikonfigurasi, bind akan ditolak.
- Melakukan bind pada `/var/run/docker.sock` secara efektif memberikan kontrol host ke sandbox; lakukan ini hanya dengan sengaja.
- Akses workspace (`workspaceAccess: "ro"`/`"rw"`) independen dari mode bind.

## Kebijakan alat: alat mana yang ada/dapat dipanggil

Dua lapisan penting:

- **Profil alat**: `tools.profile` dan `agents.list[].tools.profile` (allowlist dasar)
- **Profil alat provider**: `tools.byProvider[provider].profile` dan `agents.list[].tools.byProvider[provider].profile`
- **Kebijakan alat global/per-agent**: `tools.allow`/`tools.deny` dan `agents.list[].tools.allow`/`agents.list[].tools.deny`
- **Kebijakan alat provider**: `tools.byProvider[provider].allow/deny` dan `agents.list[].tools.byProvider[provider].allow/deny`
- **Kebijakan alat sandbox** (hanya berlaku saat dalam sandbox): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` dan `agents.list[].tools.sandbox.tools.*`

Aturan praktis:

- `deny` selalu menang.
- Jika `allow` tidak kosong, semua yang lain dianggap diblokir.
- Kebijakan alat adalah penghentian keras: `/exec` tidak dapat mengesampingkan alat `exec` yang ditolak.
- `/exec` hanya mengubah default sesi untuk pengirim yang berwenang; perintah ini tidak memberikan akses alat.
  Kunci alat provider menerima `provider` (misalnya `google-antigravity`) atau `provider/model` (misalnya `openai/gpt-5.4`).

### Grup alat (singkatan)

Kebijakan alat (global, agent, sandbox) mendukung entri `group:*` yang diperluas menjadi beberapa alat:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

Grup yang tersedia:

- `group:runtime`: `exec`, `process`, `code_execution` (`bash` diterima sebagai
  alias untuk `exec`)
- `group:fs`: `read`, `write`, `edit`, `apply_patch`
- `group:sessions`: `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`
- `group:memory`: `memory_search`, `memory_get`
- `group:web`: `web_search`, `x_search`, `web_fetch`
- `group:ui`: `browser`, `canvas`
- `group:automation`: `cron`, `gateway`
- `group:messaging`: `message`
- `group:nodes`: `nodes`
- `group:agents`: `agents_list`
- `group:media`: `image`, `image_generate`, `video_generate`, `tts`
- `group:openclaw`: semua alat OpenClaw bawaan (tidak termasuk plugin provider)

## Elevated: "jalankan di host" khusus exec

Elevated **tidak** memberikan alat tambahan; fitur ini hanya memengaruhi `exec`.

- Jika Anda berada dalam sandbox, `/elevated on` (atau `exec` dengan `elevated: true`) akan berjalan di luar sandbox (persetujuan mungkin tetap berlaku).
- Gunakan `/elevated full` untuk melewati persetujuan exec untuk sesi tersebut.
- Jika Anda sudah berjalan secara langsung, elevated pada dasarnya tidak berpengaruh (tetap berada di balik gerbang).
- Elevated **tidak** dibatasi per skill dan **tidak** mengesampingkan allow/deny alat.
- Elevated tidak memberikan override lintas host sembarang dari `host=auto`; fitur ini mengikuti aturan target exec normal dan hanya mempertahankan `node` saat target yang dikonfigurasi/sesi memang sudah `node`.
- `/exec` terpisah dari elevated. Perintah ini hanya menyesuaikan default exec per sesi untuk pengirim yang berwenang.

Gerbang:

- Pengaktifan: `tools.elevated.enabled` (dan secara opsional `agents.list[].tools.elevated.enabled`)
- Allowlist pengirim: `tools.elevated.allowFrom.<provider>` (dan secara opsional `agents.list[].tools.elevated.allowFrom.<provider>`)

Lihat [Elevated Mode](/id/tools/elevated).

## Perbaikan umum "sandbox jail"

### "Alat X diblokir oleh kebijakan alat sandbox"

Kunci perbaikan (pilih salah satu):

- Nonaktifkan sandbox: `agents.defaults.sandbox.mode=off` (atau per-agent `agents.list[].sandbox.mode=off`)
- Izinkan alat tersebut di dalam sandbox:
  - hapus dari `tools.sandbox.tools.deny` (atau per-agent `agents.list[].tools.sandbox.tools.deny`)
  - atau tambahkan ke `tools.sandbox.tools.allow` (atau allow per-agent)

### "Saya kira ini main, kenapa berada dalam sandbox?"

Dalam mode `"non-main"`, kunci grup/channel _bukan_ main. Gunakan kunci sesi main (ditampilkan oleh `sandbox explain`) atau ubah mode menjadi `"off"`.

## Lihat juga

- [Sandboxing](/id/gateway/sandboxing) -- referensi sandbox lengkap (mode, cakupan, backend, image)
- [Multi-Agent Sandbox & Tools](/id/tools/multi-agent-sandbox-tools) -- override per-agent dan prioritas
- [Elevated Mode](/id/tools/elevated)
