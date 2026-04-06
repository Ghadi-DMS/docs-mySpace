---
read_when:
    - Menggunakan atau memodifikasi tool exec
    - Men-debug perilaku stdin atau TTY
summary: Penggunaan tool exec, mode stdin, dan dukungan TTY
title: Tool Exec
x-i18n:
    generated_at: "2026-04-06T03:12:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28388971c627292dba9bf65ae38d7af8cde49a33bb3b5fc8b20da4f0e350bedd
    source_path: tools/exec.md
    workflow: 15
---

# Tool exec

Jalankan perintah shell di workspace. Mendukung eksekusi foreground + background melalui `process`.
Jika `process` tidak diizinkan, `exec` berjalan sinkron dan mengabaikan `yieldMs`/`background`.
Sesi background dibatasi per agent; `process` hanya melihat sesi dari agent yang sama.

## Parameter

- `command` (wajib)
- `workdir` (default ke cwd)
- `env` (override key/value)
- `yieldMs` (default 10000): otomatis jadi background setelah jeda
- `background` (bool): langsung ke background
- `timeout` (detik, default 1800): hentikan saat kedaluwarsa
- `pty` (bool): jalankan dalam pseudo-terminal saat tersedia (CLI khusus TTY, coding agent, UI terminal)
- `host` (`auto | sandbox | gateway | node`): tempat eksekusi
- `security` (`deny | allowlist | full`): mode penegakan untuk `gateway`/`node`
- `ask` (`off | on-miss | always`): prompt persetujuan untuk `gateway`/`node`
- `node` (string): id/nama node untuk `host=node`
- `elevated` (bool): minta mode elevated (keluar dari sandbox ke path host yang dikonfigurasi); `security=full` hanya dipaksakan saat elevated diselesaikan ke `full`

Catatan:

- `host` default ke `auto`: sandbox saat runtime sandbox aktif untuk sesi tersebut, jika tidak maka gateway.
- `auto` adalah strategi perutean default, bukan wildcard. `host=node` per panggilan diizinkan dari `auto`; `host=gateway` per panggilan hanya diizinkan saat tidak ada runtime sandbox yang aktif.
- Tanpa config tambahan, `host=auto` tetap “langsung berfungsi”: tanpa sandbox berarti diselesaikan ke `gateway`; sandbox yang aktif berarti tetap di sandbox.
- `elevated` keluar dari sandbox ke path host yang dikonfigurasi: defaultnya `gateway`, atau `node` saat `tools.exec.host=node` (atau default sesi adalah `host=node`). Ini hanya tersedia saat akses elevated diaktifkan untuk sesi/provider saat ini.
- Persetujuan `gateway`/`node` dikendalikan oleh `~/.openclaw/exec-approvals.json`.
- `node` memerlukan node yang dipasangkan (companion app atau host node headless).
- Jika tersedia beberapa node, setel `exec.node` atau `tools.exec.node` untuk memilih satu.
- `exec host=node` adalah satu-satunya jalur eksekusi shell untuk node; wrapper lama `nodes.run` telah dihapus.
- Pada host non-Windows, exec menggunakan `SHELL` saat disetel; jika `SHELL` adalah `fish`, exec lebih memilih `bash` (atau `sh`)
  dari `PATH` untuk menghindari skrip yang tidak kompatibel dengan fish, lalu fallback ke `SHELL` jika keduanya tidak ada.
- Pada host Windows, exec lebih memilih penemuan PowerShell 7 (`pwsh`) (Program Files, ProgramW6432, lalu PATH),
  lalu fallback ke Windows PowerShell 5.1.
- Eksekusi host (`gateway`/`node`) menolak `env.PATH` dan override loader (`LD_*`/`DYLD_*`) untuk
  mencegah pembajakan biner atau injeksi kode.
- OpenClaw menyetel `OPENCLAW_SHELL=exec` di environment perintah yang di-spawn (termasuk eksekusi PTY dan sandbox) agar aturan shell/profile dapat mendeteksi konteks tool exec.
- Penting: sandboxing **nonaktif secara default**. Jika sandboxing nonaktif, `host=auto`
  implisit diselesaikan ke `gateway`. `host=sandbox` eksplisit tetap gagal secara tertutup alih-alih diam-diam
  berjalan di host gateway. Aktifkan sandboxing atau gunakan `host=gateway` dengan persetujuan.
- Pemeriksaan preflight skrip (untuk kesalahan sintaks shell Python/Node umum) hanya memeriksa file di dalam
  batas `workdir` efektif. Jika path skrip diselesaikan di luar `workdir`, preflight dilewati untuk
  file itu.
- Untuk pekerjaan berjalan lama yang mulai sekarang, mulai sekali dan andalkan
  automatic completion wake saat diaktifkan dan perintah menghasilkan output atau gagal.
  Gunakan `process` untuk log, status, input, atau intervensi; jangan meniru
  penjadwalan dengan loop sleep, loop timeout, atau polling berulang.
- Untuk pekerjaan yang harus terjadi nanti atau terjadwal, gunakan cron alih-alih
  pola sleep/delay `exec`.

## Config

- `tools.exec.notifyOnExit` (default: true): saat true, sesi exec yang dibackground akan memasukkan event sistem ke antrean dan meminta heartbeat saat selesai.
- `tools.exec.approvalRunningNoticeMs` (default: 10000): keluarkan satu pemberitahuan “running” saat exec yang dibatasi persetujuan berjalan lebih lama dari ini (0 menonaktifkan).
- `tools.exec.host` (default: `auto`; diselesaikan ke `sandbox` saat runtime sandbox aktif, `gateway` jika tidak)
- `tools.exec.security` (default: `deny` untuk sandbox, `full` untuk gateway + node saat tidak disetel)
- `tools.exec.ask` (default: `off`)
- Exec host tanpa persetujuan adalah default untuk gateway + node. Jika Anda ingin perilaku persetujuan/allowlist, perketat `tools.exec.*` dan host `~/.openclaw/exec-approvals.json`; lihat [Exec approvals](/id/tools/exec-approvals#no-approval-yolo-mode).
- YOLO berasal dari default kebijakan host (`security=full`, `ask=off`), bukan dari `host=auto`. Jika Anda ingin memaksa perutean gateway atau node, setel `tools.exec.host` atau gunakan `/exec host=...`.
- Dalam mode `security=full` plus `ask=off`, exec host mengikuti kebijakan yang dikonfigurasi secara langsung; tidak ada prefilter heuristik tambahan untuk obfuscation perintah.
- `tools.exec.node` (default: tidak disetel)
- `tools.exec.strictInlineEval` (default: false): saat true, bentuk eval interpreter inline seperti `python -c`, `node -e`, `ruby -e`, `perl -e`, `php -r`, `lua -e`, dan `osascript -e` selalu memerlukan persetujuan eksplisit. `allow-always` tetap dapat menyimpan invokasi interpreter/skrip yang aman, tetapi bentuk inline-eval tetap meminta prompt setiap kali.
- `tools.exec.pathPrepend`: daftar direktori yang akan ditambahkan di depan `PATH` untuk run exec (hanya gateway + sandbox).
- `tools.exec.safeBins`: biner aman khusus stdin yang dapat berjalan tanpa entri allowlist eksplisit. Untuk detail perilaku, lihat [Safe bins](/id/tools/exec-approvals#safe-bins-stdin-only).
- `tools.exec.safeBinTrustedDirs`: direktori tepercaya tambahan yang eksplisit untuk pemeriksaan path `safeBins`. Entri `PATH` tidak pernah otomatis dipercaya. Default bawaan adalah `/bin` dan `/usr/bin`.
- `tools.exec.safeBinProfiles`: kebijakan argv kustom opsional per safe bin (`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`).

Contoh:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### Penanganan PATH

- `host=gateway`: menggabungkan `PATH` login-shell Anda ke environment exec. Override `env.PATH`
  ditolak untuk eksekusi host. Daemon itu sendiri tetap berjalan dengan `PATH` minimal:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
- `host=sandbox`: menjalankan `sh -lc` (login shell) di dalam container, sehingga `/etc/profile` dapat mereset `PATH`.
  OpenClaw menambahkan `env.PATH` di depan setelah profile sourcing melalui variabel env internal (tanpa interpolasi shell);
  `tools.exec.pathPrepend` juga berlaku di sini.
- `host=node`: hanya override env yang tidak diblokir yang Anda berikan yang dikirim ke node. Override `env.PATH`
  ditolak untuk eksekusi host dan diabaikan oleh host node. Jika Anda memerlukan entri PATH tambahan pada node,
  konfigurasikan environment layanan host node (systemd/launchd) atau instal tool di lokasi standar.

Binding node per agent (gunakan indeks daftar agent di config):

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

Control UI: tab Nodes menyertakan panel kecil “Exec node binding” untuk setelan yang sama.

## Override sesi (`/exec`)

Gunakan `/exec` untuk menetapkan default **per sesi** bagi `host`, `security`, `ask`, dan `node`.
Kirim `/exec` tanpa argumen untuk menampilkan nilai saat ini.

Contoh:

```
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

## Model otorisasi

`/exec` hanya dihormati untuk **pengirim yang diotorisasi** (allowlist/pairing channel plus `commands.useAccessGroups`).
Perintah ini hanya memperbarui **state sesi** dan tidak menulis config. Untuk menonaktifkan exec secara keras, tolak melalui kebijakan tool
(`tools.deny: ["exec"]` atau per-agent). Persetujuan host tetap berlaku kecuali Anda secara eksplisit menyetel
`security=full` dan `ask=off`.

## Persetujuan exec (companion app / host node)

Agent yang disandbox dapat memerlukan persetujuan per permintaan sebelum `exec` berjalan pada host gateway atau node.
Lihat [Exec approvals](/id/tools/exec-approvals) untuk kebijakan, allowlist, dan alur UI.

Saat persetujuan diperlukan, tool exec langsung mengembalikan
`status: "approval-pending"` dan id persetujuan. Setelah disetujui (atau ditolak / timeout),
Gateway mengeluarkan event sistem (`Exec finished` / `Exec denied`). Jika perintah masih
berjalan setelah `tools.exec.approvalRunningNoticeMs`, satu pemberitahuan `Exec running` akan dikeluarkan.
Pada channel dengan kartu/tombol persetujuan bawaan, agent sebaiknya mengandalkan
UI bawaan tersebut terlebih dahulu dan hanya menyertakan perintah manual `/approve` saat
hasil tool secara eksplisit menyatakan bahwa persetujuan chat tidak tersedia atau persetujuan manual adalah
satu-satunya jalur.

## Allowlist + safe bins

Penegakan allowlist manual hanya mencocokkan **path biner yang diselesaikan** (tanpa kecocokan nama dasar). Saat
`security=allowlist`, perintah shell hanya diizinkan otomatis jika setiap segmen pipeline
ada di allowlist atau merupakan safe bin. Chaining (`;`, `&&`, `||`) dan redirection ditolak dalam
mode allowlist kecuali setiap segmen tingkat atas memenuhi allowlist (termasuk safe bin).
Redirection tetap tidak didukung.
Kepercayaan `allow-always` yang tahan lama tidak melewati aturan itu: perintah berantai tetap memerlukan setiap
segmen tingkat atas untuk cocok.

`autoAllowSkills` adalah jalur kemudahan terpisah dalam persetujuan exec. Ini tidak sama dengan
entri allowlist path manual. Untuk trust eksplisit yang ketat, biarkan `autoAllowSkills` nonaktif.

Gunakan dua kontrol tersebut untuk pekerjaan yang berbeda:

- `tools.exec.safeBins`: filter stream kecil khusus stdin.
- `tools.exec.safeBinTrustedDirs`: direktori tepercaya tambahan yang eksplisit untuk path executable safe-bin.
- `tools.exec.safeBinProfiles`: kebijakan argv eksplisit untuk safe bin kustom.
- allowlist: trust eksplisit untuk path executable.

Jangan perlakukan `safeBins` sebagai allowlist generik, dan jangan tambahkan biner interpreter/runtime (misalnya `python3`, `node`, `ruby`, `bash`). Jika Anda memerlukannya, gunakan entri allowlist eksplisit dan tetap aktifkan prompt persetujuan.
`openclaw security audit` memperingatkan saat entri interpreter/runtime `safeBins` tidak memiliki profil eksplisit, dan `openclaw doctor --fix` dapat membuatkan entri `safeBinProfiles` kustom yang belum ada.
`openclaw security audit` dan `openclaw doctor` juga memperingatkan saat Anda secara eksplisit menambahkan kembali bin dengan perilaku luas seperti `jq` ke `safeBins`.
Jika Anda secara eksplisit meng-allowlist interpreter, aktifkan `tools.exec.strictInlineEval` agar bentuk eval kode inline tetap memerlukan persetujuan baru.

Untuk detail kebijakan lengkap dan contoh, lihat [Exec approvals](/id/tools/exec-approvals#safe-bins-stdin-only) dan [Safe bins versus allowlist](/id/tools/exec-approvals#safe-bins-versus-allowlist).

## Contoh

Foreground:

```json
{ "tool": "exec", "command": "ls -la" }
```

Background + poll:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

Polling adalah untuk status sesuai permintaan, bukan loop menunggu. Jika automatic completion wake
diaktifkan, perintah dapat membangunkan sesi saat menghasilkan output atau gagal.

Kirim tombol (gaya tmux):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Kirim (hanya kirim CR):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Tempel (secara default diberi bracket):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` adalah subtool dari `exec` untuk edit multi-file terstruktur.
Tool ini aktif secara default untuk model OpenAI dan OpenAI Codex. Gunakan config hanya
jika Anda ingin menonaktifkannya atau membatasinya ke model tertentu:

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.4"] },
    },
  },
}
```

Catatan:

- Hanya tersedia untuk model OpenAI/OpenAI Codex.
- Kebijakan tool tetap berlaku; `allow: ["write"]` secara implisit mengizinkan `apply_patch`.
- Config berada di bawah `tools.exec.applyPatch`.
- `tools.exec.applyPatch.enabled` default-nya `true`; setel ke `false` untuk menonaktifkan tool bagi model OpenAI.
- `tools.exec.applyPatch.workspaceOnly` default-nya `true` (terbatas di dalam workspace). Setel ke `false` hanya jika Anda memang ingin `apply_patch` menulis/menghapus di luar direktori workspace.

## Terkait

- [Exec Approvals](/id/tools/exec-approvals) — gerbang persetujuan untuk perintah shell
- [Sandboxing](/id/gateway/sandboxing) — menjalankan perintah dalam lingkungan sandbox
- [Background Process](/id/gateway/background-process) — exec berjalan lama dan tool process
- [Security](/id/gateway/security) — kebijakan tool dan akses elevated
