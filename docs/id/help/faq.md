---
read_when:
    - Menjawab pertanyaan umum tentang penyiapan, instalasi, onboarding, atau dukungan runtime
    - Men-triase masalah yang dilaporkan pengguna sebelum debugging lebih mendalam
summary: Pertanyaan yang sering diajukan tentang penyiapan, konfigurasi, dan penggunaan OpenClaw
title: FAQ
x-i18n:
    generated_at: "2026-04-06T03:13:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4d6d09621c6033d580cbcf1ff46f81587d69404d6f64c8d8fd8c3f09185bb920
    source_path: help/faq.md
    workflow: 15
---

# FAQ

Jawaban cepat ditambah pemecahan masalah yang lebih mendalam untuk penyiapan dunia nyata (pengembangan lokal, VPS, multi-agent, OAuth/API key, failover model). Untuk diagnostik runtime, lihat [Troubleshooting](/id/gateway/troubleshooting). Untuk referensi konfigurasi lengkap, lihat [Configuration](/id/gateway/configuration).

## 60 detik pertama jika ada yang rusak

1. **Status cepat (pemeriksaan pertama)**

   ```bash
   openclaw status
   ```

   Ringkasan lokal cepat: OS + pembaruan, keterjangkauan gateway/layanan, agen/sesi, konfigurasi provider + masalah runtime (saat gateway dapat dijangkau).

2. **Laporan yang aman untuk dibagikan**

   ```bash
   openclaw status --all
   ```

   Diagnosis hanya-baca dengan ekor log (token disamarkan).

3. **Status daemon + port**

   ```bash
   openclaw gateway status
   ```

   Menampilkan runtime supervisor vs keterjangkauan RPC, URL target probe, dan konfigurasi mana yang kemungkinan digunakan oleh layanan.

4. **Probe mendalam**

   ```bash
   openclaw status --deep
   ```

   Menjalankan probe kesehatan gateway langsung, termasuk probe channel saat didukung
   (memerlukan gateway yang dapat dijangkau). Lihat [Health](/id/gateway/health).

5. **Ikuti log terbaru**

   ```bash
   openclaw logs --follow
   ```

   Jika RPC tidak aktif, gunakan fallback ke:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   Log file terpisah dari log layanan; lihat [Logging](/id/logging) dan [Troubleshooting](/id/gateway/troubleshooting).

6. **Jalankan doctor (perbaikan)**

   ```bash
   openclaw doctor
   ```

   Memperbaiki/memigrasikan konfigurasi/state + menjalankan pemeriksaan kesehatan. Lihat [Doctor](/id/gateway/doctor).

7. **Snapshot gateway**

   ```bash
   openclaw health --json
   openclaw health --verbose   # shows the target URL + config path on errors
   ```

   Meminta snapshot lengkap dari gateway yang sedang berjalan (khusus-WS). Lihat [Health](/id/gateway/health).

## Mulai cepat dan penyiapan pertama kali

<AccordionGroup>
  <Accordion title="Saya terjebak, cara tercepat untuk keluar dari kebuntuan">
    Gunakan agen AI lokal yang dapat **melihat mesin Anda**. Itu jauh lebih efektif daripada bertanya
    di Discord, karena sebagian besar kasus "saya terjebak" adalah **masalah konfigurasi atau lingkungan lokal**
    yang tidak dapat diperiksa oleh orang yang membantu dari jarak jauh.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    Alat ini dapat membaca repo, menjalankan perintah, memeriksa log, dan membantu memperbaiki
    penyiapan tingkat mesin Anda (PATH, layanan, izin, file auth). Berikan mereka **checkout source lengkap**
    melalui instalasi hackable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Ini menginstal OpenClaw **dari checkout git**, sehingga agen dapat membaca kode + dokumentasi dan
    menalar versi persis yang sedang Anda jalankan. Anda selalu dapat kembali ke stable nanti
    dengan menjalankan ulang installer tanpa `--install-method git`.

    Tip: minta agen untuk **merencanakan dan mengawasi** perbaikannya (langkah demi langkah), lalu jalankan hanya
    perintah yang diperlukan. Itu menjaga perubahan tetap kecil dan lebih mudah diaudit.

    Jika Anda menemukan bug atau perbaikan nyata, silakan buat issue GitHub atau kirim PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    Mulailah dengan perintah berikut (bagikan output saat meminta bantuan):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    Fungsinya:

    - `openclaw status`: snapshot cepat kesehatan gateway/agen + konfigurasi dasar.
    - `openclaw models status`: memeriksa auth provider + ketersediaan model.
    - `openclaw doctor`: memvalidasi dan memperbaiki masalah konfigurasi/state umum.

    Pemeriksaan CLI lain yang berguna: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    Loop debug cepat: [60 detik pertama jika ada yang rusak](#60-detik-pertama-jika-ada-yang-rusak).
    Dokumentasi instalasi: [Install](/id/install), [Installer flags](/id/install/installer), [Updating](/id/install/updating).

  </Accordion>

  <Accordion title="Heartbeat terus melewati. Apa arti alasan skip-nya?">
    Alasan skip heartbeat yang umum:

    - `quiet-hours`: di luar jendela jam aktif yang dikonfigurasi
    - `empty-heartbeat-file`: `HEARTBEAT.md` ada tetapi hanya berisi kerangka kosong/header saja
    - `no-tasks-due`: mode tugas `HEARTBEAT.md` aktif tetapi belum ada interval tugas yang jatuh tempo
    - `alerts-disabled`: semua visibilitas heartbeat dinonaktifkan (`showOk`, `showAlerts`, dan `useIndicator` semuanya mati)

    Dalam mode tugas, stempel waktu jatuh tempo hanya dimajukan setelah heartbeat sungguhan
    selesai berjalan. Run yang dilewati tidak menandai tugas sebagai selesai.

    Dokumentasi: [Heartbeat](/id/gateway/heartbeat), [Automation & Tasks](/id/automation).

  </Accordion>

  <Accordion title="Cara yang direkomendasikan untuk menginstal dan menyiapkan OpenClaw">
    Repo ini merekomendasikan menjalankan dari source dan menggunakan onboarding:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    Wizard juga dapat membangun aset UI secara otomatis. Setelah onboarding, Anda biasanya menjalankan Gateway di port **18789**.

    Dari source (kontributor/pengembang):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # auto-installs UI deps on first run
    openclaw onboard
    ```

    Jika Anda belum memiliki instalasi global, jalankan lewat `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="Bagaimana cara membuka dashboard setelah onboarding?">
    Wizard membuka browser Anda dengan URL dashboard yang bersih (tanpa token) tepat setelah onboarding dan juga mencetak tautannya di ringkasan. Biarkan tab itu tetap terbuka; jika tidak terbuka otomatis, salin/tempel URL yang dicetak di mesin yang sama.
  </Accordion>

  <Accordion title="Bagaimana cara mengautentikasi dashboard di localhost vs remote?">
    **Localhost (mesin yang sama):**

    - Buka `http://127.0.0.1:18789/`.
    - Jika meminta auth shared-secret, tempel token atau kata sandi yang dikonfigurasi ke pengaturan Control UI.
    - Sumber token: `gateway.auth.token` (atau `OPENCLAW_GATEWAY_TOKEN`).
    - Sumber kata sandi: `gateway.auth.password` (atau `OPENCLAW_GATEWAY_PASSWORD`).
    - Jika belum ada shared secret yang dikonfigurasi, buat token dengan `openclaw doctor --generate-gateway-token`.

    **Bukan di localhost:**

    - **Tailscale Serve** (disarankan): tetap bind loopback, jalankan `openclaw gateway --tailscale serve`, buka `https://<magicdns>/`. Jika `gateway.auth.allowTailscale` bernilai `true`, header identitas memenuhi auth Control UI/WebSocket (tanpa menempel shared secret, dengan asumsi gateway host tepercaya); HTTP API tetap memerlukan auth shared-secret kecuali Anda sengaja menggunakan private-ingress `none` atau auth HTTP trusted-proxy.
      Upaya auth Serve bersamaan yang buruk dari klien yang sama diserialkan sebelum pembatas auth-gagal mencatatnya, jadi percobaan buruk kedua bisa langsung menampilkan `retry later`.
    - **Bind tailnet**: jalankan `openclaw gateway --bind tailnet --token "<token>"` (atau konfigurasikan auth kata sandi), buka `http://<tailscale-ip>:18789/`, lalu tempel shared secret yang sesuai di pengaturan dashboard.
    - **Reverse proxy sadar identitas**: biarkan Gateway berada di belakang trusted proxy non-loopback, konfigurasikan `gateway.auth.mode: "trusted-proxy"`, lalu buka URL proxy.
    - **SSH tunnel**: `ssh -N -L 18789:127.0.0.1:18789 user@host` lalu buka `http://127.0.0.1:18789/`. Auth shared-secret tetap berlaku melalui tunnel; tempel token atau kata sandi yang dikonfigurasi jika diminta.

    Lihat [Dashboard](/web/dashboard) dan [Web surfaces](/web) untuk detail mode bind dan auth.

  </Accordion>

  <Accordion title="Mengapa ada dua konfigurasi persetujuan exec untuk persetujuan obrolan?">
    Mereka mengendalikan lapisan yang berbeda:

    - `approvals.exec`: meneruskan prompt persetujuan ke tujuan obrolan
    - `channels.<channel>.execApprovals`: membuat channel tersebut bertindak sebagai klien persetujuan native untuk persetujuan exec

    Kebijakan exec host tetap merupakan gerbang persetujuan yang sebenarnya. Konfigurasi obrolan hanya mengendalikan ke mana
    prompt persetujuan muncul dan bagaimana orang dapat menjawabnya.

    Dalam sebagian besar penyiapan Anda **tidak** memerlukan keduanya:

    - Jika obrolan sudah mendukung perintah dan balasan, `/approve` di obrolan yang sama bekerja melalui jalur bersama.
    - Jika channel native yang didukung dapat menyimpulkan penyetuju dengan aman, OpenClaw sekarang otomatis mengaktifkan persetujuan native DM-first ketika `channels.<channel>.execApprovals.enabled` tidak disetel atau `"auto"`.
    - Saat kartu/tombol persetujuan native tersedia, UI native itu adalah jalur utama; agen hanya boleh menyertakan perintah manual `/approve` jika hasil alat mengatakan persetujuan obrolan tidak tersedia atau persetujuan manual adalah satu-satunya jalur.
    - Gunakan `approvals.exec` hanya ketika prompt juga harus diteruskan ke obrolan lain atau ruang operasional eksplisit.
    - Gunakan `channels.<channel>.execApprovals.target: "channel"` atau `"both"` hanya ketika Anda memang secara eksplisit ingin prompt persetujuan diposting kembali ke room/topik asal.
    - Persetujuan plugin terpisah lagi: mereka menggunakan `/approve` di obrolan yang sama secara default, penerusan `approvals.plugin` opsional, dan hanya beberapa channel native yang tetap mempertahankan penanganan native persetujuan plugin di atasnya.

    Versi singkat: penerusan adalah untuk perutean, konfigurasi klien native adalah untuk UX khusus channel yang lebih kaya.
    Lihat [Exec Approvals](/id/tools/exec-approvals).

  </Accordion>

  <Accordion title="Runtime apa yang saya butuhkan?">
    Node **>= 22** diperlukan. `pnpm` direkomendasikan. Bun **tidak direkomendasikan** untuk Gateway.
  </Accordion>

  <Accordion title="Apakah ini berjalan di Raspberry Pi?">
    Ya. Gateway ringan - dokumentasi menyebutkan **512MB-1GB RAM**, **1 core**, dan sekitar **500MB**
    disk sudah cukup untuk penggunaan pribadi, dan mencatat bahwa **Raspberry Pi 4 dapat menjalankannya**.

    Jika Anda ingin ruang ekstra (log, media, layanan lain), **2GB direkomendasikan**, tetapi itu
    bukan minimum mutlak.

    Tip: Pi/VPS kecil dapat meng-host Gateway, dan Anda dapat memasangkan **nodes** di laptop/ponsel Anda untuk
    layar/kamera/canvas lokal atau eksekusi perintah. Lihat [Nodes](/id/nodes).

  </Accordion>

  <Accordion title="Ada tips untuk instalasi Raspberry Pi?">
    Singkatnya: bisa, tetapi bersiap menghadapi beberapa kekasaran.

    - Gunakan OS **64-bit** dan pertahankan Node >= 22.
    - Pilih **instalasi hackable (git)** agar Anda bisa melihat log dan memperbarui dengan cepat.
    - Mulai tanpa channel/Skills, lalu tambahkan satu per satu.
    - Jika Anda menemui masalah biner yang aneh, biasanya itu adalah masalah **kompatibilitas ARM**.

    Dokumentasi: [Linux](/id/platforms/linux), [Install](/id/install).

  </Accordion>

  <Accordion title="Macet di wake up my friend / onboarding tidak mau hatch. Sekarang apa?">
    Layar itu bergantung pada Gateway yang dapat dijangkau dan terautentikasi. TUI juga mengirim
    "Wake up, my friend!" secara otomatis pada hatch pertama. Jika Anda melihat baris itu tanpa **balasan**
    dan token tetap 0, agen tidak pernah berjalan.

    1. Mulai ulang Gateway:

    ```bash
    openclaw gateway restart
    ```

    2. Periksa status + auth:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. Jika masih macet, jalankan:

    ```bash
    openclaw doctor
    ```

    Jika Gateway berada di remote, pastikan koneksi tunnel/Tailscale aktif dan UI
    diarahkan ke Gateway yang benar. Lihat [Remote access](/id/gateway/remote).

  </Accordion>

  <Accordion title="Bisakah saya memigrasikan penyiapan saya ke mesin baru (Mac mini) tanpa mengulang onboarding?">
    Ya. Salin **direktori state** dan **workspace**, lalu jalankan Doctor sekali. Ini
    menjaga bot Anda "tetap persis sama" (memori, riwayat sesi, auth, dan
    state channel) selama Anda menyalin **kedua** lokasi:

    1. Instal OpenClaw di mesin baru.
    2. Salin `$OPENCLAW_STATE_DIR` (default: `~/.openclaw`) dari mesin lama.
    3. Salin workspace Anda (default: `~/.openclaw/workspace`).
    4. Jalankan `openclaw doctor` dan mulai ulang layanan Gateway.

    Ini mempertahankan konfigurasi, profil auth, kredensial WhatsApp, sesi, dan memori. Jika Anda berada dalam
    mode remote, ingat bahwa gateway host memiliki penyimpanan sesi dan workspace.

    **Penting:** jika Anda hanya commit/push workspace ke GitHub, Anda sedang mencadangkan
    **memori + file bootstrap**, tetapi **bukan** riwayat sesi atau auth. Itu berada
    di bawah `~/.openclaw/` (misalnya `~/.openclaw/agents/<agentId>/sessions/`).

    Terkait: [Migrating](/id/install/migrating), [Tempat data berada di disk](#where-things-live-on-disk),
    [Agent workspace](/id/concepts/agent-workspace), [Doctor](/id/gateway/doctor),
    [Remote mode](/id/gateway/remote).

  </Accordion>

  <Accordion title="Di mana saya melihat apa yang baru di versi terbaru?">
    Periksa changelog GitHub:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Entri terbaru ada di bagian atas. Jika bagian teratas ditandai **Unreleased**, bagian bertanggal berikutnya
    adalah versi terbaru yang sudah dirilis. Entri dikelompokkan berdasarkan **Highlights**, **Changes**, dan
    **Fixes** (ditambah bagian dokumentasi/lainnya bila diperlukan).

  </Accordion>

  <Accordion title="Tidak dapat mengakses docs.openclaw.ai (kesalahan SSL)">
    Beberapa koneksi Comcast/Xfinity salah memblokir `docs.openclaw.ai` melalui Xfinity
    Advanced Security. Nonaktifkan atau masukkan `docs.openclaw.ai` ke allowlist, lalu coba lagi.
    Tolong bantu kami membuka blokir ini dengan melapor di sini: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Jika Anda masih tidak dapat menjangkau situs tersebut, dokumentasi dicerminkan di GitHub:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Perbedaan antara stable dan beta">
    **Stable** dan **beta** adalah **npm dist-tag**, bukan jalur kode yang terpisah:

    - `latest` = stable
    - `beta` = build awal untuk pengujian

    Biasanya, rilis stable masuk ke **beta** terlebih dahulu, lalu langkah
    promosi eksplisit memindahkan versi yang sama ke `latest`. Maintainer juga dapat
    memublikasikan langsung ke `latest` bila diperlukan. Itulah sebabnya beta dan stable dapat
    mengarah ke **versi yang sama** setelah promosi.

    Lihat apa yang berubah:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Untuk one-liner instalasi dan perbedaan antara beta dan dev, lihat accordion di bawah.

  </Accordion>

  <Accordion title="Bagaimana cara menginstal versi beta dan apa perbedaan antara beta dan dev?">
    **Beta** adalah npm dist-tag `beta` (dapat sama dengan `latest` setelah promosi).
    **Dev** adalah head bergerak dari `main` (git); saat dipublikasikan, ia menggunakan npm dist-tag `dev`.

    One-liner (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Installer Windows (PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    Detail lebih lanjut: [Development channels](/id/install/development-channels) dan [Installer flags](/id/install/installer).

  </Accordion>

  <Accordion title="Bagaimana cara mencoba bit terbaru?">
    Dua opsi:

    1. **Channel dev (checkout git):**

    ```bash
    openclaw update --channel dev
    ```

    Ini beralih ke branch `main` dan memperbarui dari source.

    2. **Instalasi hackable (dari situs installer):**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Itu memberi Anda repo lokal yang bisa Anda edit, lalu diperbarui melalui git.

    Jika Anda lebih suka clone bersih secara manual, gunakan:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Dokumentasi: [Update](/cli/update), [Development channels](/id/install/development-channels),
    [Install](/id/install).

  </Accordion>

  <Accordion title="Berapa lama biasanya instalasi dan onboarding berlangsung?">
    Panduan kasar:

    - **Instalasi:** 2-5 menit
    - **Onboarding:** 5-15 menit tergantung berapa banyak channel/model yang Anda konfigurasikan

    Jika macet, gunakan [Installer stuck](#quick-start-and-first-run-setup)
    dan loop debug cepat di [Saya terjebak](#quick-start-and-first-run-setup).

  </Accordion>

  <Accordion title="Installer macet? Bagaimana cara mendapatkan lebih banyak umpan balik?">
    Jalankan ulang installer dengan **output verbose**:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    Instalasi beta dengan verbose:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    Untuk instalasi hackable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    Ekuivalen Windows (PowerShell):

    ```powershell
    # install.ps1 has no dedicated -Verbose flag yet.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    Opsi lainnya: [Installer flags](/id/install/installer).

  </Accordion>

  <Accordion title="Instalasi Windows mengatakan git tidak ditemukan atau openclaw tidak dikenali">
    Dua masalah Windows yang umum:

    **1) kesalahan npm spawn git / git tidak ditemukan**

    - Instal **Git for Windows** dan pastikan `git` ada di PATH Anda.
    - Tutup dan buka kembali PowerShell, lalu jalankan ulang installer.

    **2) openclaw tidak dikenali setelah instalasi**

    - Folder bin global npm Anda tidak ada di PATH.
    - Periksa path-nya:

      ```powershell
      npm config get prefix
      ```

    - Tambahkan direktori tersebut ke PATH pengguna Anda (tidak perlu sufiks `\bin` di Windows; di sebagian besar sistem itu adalah `%AppData%\npm`).
    - Tutup dan buka kembali PowerShell setelah memperbarui PATH.

    Jika Anda ingin penyiapan Windows yang paling mulus, gunakan **WSL2** alih-alih Windows native.
    Dokumentasi: [Windows](/id/platforms/windows).

  </Accordion>

  <Accordion title="Output exec Windows menampilkan teks Tionghoa yang rusak - apa yang harus saya lakukan?">
    Ini biasanya ketidakcocokan code page konsol pada shell Windows native.

    Gejala:

    - output `system.run`/`exec` menampilkan teks Tionghoa sebagai mojibake
    - perintah yang sama terlihat baik-baik saja di profil terminal lain

    Solusi cepat di PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Lalu mulai ulang Gateway dan coba lagi perintah Anda:

    ```powershell
    openclaw gateway restart
    ```

    Jika Anda masih dapat mereproduksinya di OpenClaw terbaru, lacak/laporkan di:

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="Dokumentasi tidak menjawab pertanyaan saya - bagaimana cara mendapatkan jawaban yang lebih baik?">
    Gunakan **instalasi hackable (git)** agar Anda memiliki source dan dokumentasi lengkap secara lokal, lalu tanyakan
    kepada bot Anda (atau Claude/Codex) _dari folder itu_ agar ia dapat membaca repo dan menjawab dengan tepat.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Detail lebih lanjut: [Install](/id/install) dan [Installer flags](/id/install/installer).

  </Accordion>

  <Accordion title="Bagaimana cara menginstal OpenClaw di Linux?">
    Jawaban singkat: ikuti panduan Linux, lalu jalankan onboarding.

    - Jalur cepat Linux + instalasi layanan: [Linux](/id/platforms/linux).
    - Panduan lengkap: [Getting Started](/id/start/getting-started).
    - Installer + pembaruan: [Install & updates](/id/install/updating).

  </Accordion>

  <Accordion title="Bagaimana cara menginstal OpenClaw di VPS?">
    Semua VPS Linux bisa digunakan. Instal di server, lalu gunakan SSH/Tailscale untuk menjangkau Gateway.

    Panduan: [exe.dev](/id/install/exe-dev), [Hetzner](/id/install/hetzner), [Fly.io](/id/install/fly).
    Akses remote: [Gateway remote](/id/gateway/remote).

  </Accordion>

  <Accordion title="Di mana panduan instalasi cloud/VPS?">
    Kami menyediakan **pusat hosting** dengan provider umum. Pilih satu dan ikuti panduannya:

    - [VPS hosting](/id/vps) (semua provider di satu tempat)
    - [Fly.io](/id/install/fly)
    - [Hetzner](/id/install/hetzner)
    - [exe.dev](/id/install/exe-dev)

    Cara kerjanya di cloud: **Gateway berjalan di server**, dan Anda mengaksesnya
    dari laptop/ponsel melalui Control UI (atau Tailscale/SSH). State + workspace Anda
    hidup di server, jadi perlakukan host sebagai sumber kebenaran dan buat cadangannya.

    Anda dapat memasangkan **nodes** (Mac/iOS/Android/headless) ke Gateway cloud itu untuk mengakses
    layar/kamera/canvas lokal atau menjalankan perintah di laptop Anda sambil tetap menyimpan
    Gateway di cloud.

    Pusat: [Platforms](/id/platforms). Akses remote: [Gateway remote](/id/gateway/remote).
    Nodes: [Nodes](/id/nodes), [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="Bisakah saya meminta OpenClaw memperbarui dirinya sendiri?">
    Jawaban singkat: **mungkin, tetapi tidak direkomendasikan**. Alur pembaruan dapat memulai ulang
    Gateway (yang memutus sesi aktif), mungkin memerlukan checkout git yang bersih, dan
    dapat meminta konfirmasi. Lebih aman: jalankan pembaruan dari shell sebagai operator.

    Gunakan CLI:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Jika Anda harus mengotomatisasi dari agen:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Dokumentasi: [Update](/cli/update), [Updating](/id/install/updating).

  </Accordion>

  <Accordion title="Apa sebenarnya yang dilakukan onboarding?">
    `openclaw onboard` adalah jalur penyiapan yang direkomendasikan. Dalam **mode lokal** ia memandu Anda melalui:

    - **Penyiapan model/auth** (OAuth provider, API key, Anthropic setup-token lama, plus opsi model lokal seperti LM Studio)
    - Lokasi **workspace** + file bootstrap
    - **Pengaturan gateway** (bind/port/auth/tailscale)
    - **Channels** (WhatsApp, Telegram, Discord, Mattermost, Signal, iMessage, plus plugin channel bawaan seperti QQ Bot)
    - **Instalasi daemon** (LaunchAgent di macOS; systemd user unit di Linux/WSL2)
    - **Pemeriksaan kesehatan** dan pemilihan **Skills**

    Ini juga memperingatkan jika model yang Anda konfigurasikan tidak dikenal atau auth-nya hilang.

  </Accordion>

  <Accordion title="Apakah saya perlu langganan Claude atau OpenAI untuk menjalankannya?">
    Tidak. Anda dapat menjalankan OpenClaw dengan **API key** (Anthropic/OpenAI/lainnya) atau dengan
    **model lokal saja** sehingga data Anda tetap berada di perangkat Anda. Langganan (Claude
    Pro/Max atau OpenAI Codex) adalah cara opsional untuk mengautentikasi provider tersebut.

    Untuk Anthropic di OpenClaw, pembagian praktisnya adalah:

    - **Anthropic API key**: penagihan API Anthropic normal
    - **Auth langganan Claude di OpenClaw**: Anthropic memberi tahu pengguna OpenClaw pada
      **4 April 2026 pukul 12:00 PM PT / 8:00 PM BST** bahwa ini memerlukan
      **Extra Usage** yang ditagih terpisah dari langganan

    Reproduksi lokal kami juga menunjukkan bahwa `claude -p --append-system-prompt ...` dapat
    terkena pembatas Extra Usage yang sama ketika prompt tambahan mengidentifikasi
    OpenClaw, sementara string prompt yang sama **tidak** mereproduksi pemblokiran itu pada
    jalur Anthropic SDK + API-key. OpenAI Codex OAuth didukung secara eksplisit
    untuk alat eksternal seperti OpenClaw.

    OpenClaw juga mendukung opsi host bergaya langganan lain termasuk
    **Qwen Cloud Coding Plan**, **MiniMax Coding Plan**, dan
    **Z.AI / GLM Coding Plan**.

    Dokumentasi: [Anthropic](/id/providers/anthropic), [OpenAI](/id/providers/openai),
    [Qwen Cloud](/id/providers/qwen),
    [MiniMax](/id/providers/minimax), [GLM Models](/id/providers/glm),
    [Local models](/id/gateway/local-models), [Models](/id/concepts/models).

  </Accordion>

  <Accordion title="Bisakah saya menggunakan langganan Claude Max tanpa API key?">
    Ya, tetapi anggap itu sebagai **auth langganan Claude dengan Extra Usage**.

    Langganan Claude Pro/Max tidak mencakup API key. Di OpenClaw, itu
    berarti pemberitahuan penagihan khusus OpenClaw dari Anthropic berlaku: lalu lintas
    langganan memerlukan **Extra Usage**. Jika Anda ingin lalu lintas Anthropic tanpa
    jalur Extra Usage itu, gunakan Anthropic API key sebagai gantinya.

  </Accordion>

  <Accordion title="Apakah Anda mendukung auth langganan Claude (Claude Pro atau Max)?">
    Ya, tetapi interpretasi yang didukung sekarang adalah:

    - Anthropic di OpenClaw dengan langganan berarti **Extra Usage**
    - Anthropic di OpenClaw tanpa jalur itu berarti **API key**

    Anthropic setup-token masih tersedia sebagai jalur OpenClaw lama/manual,
    dan pemberitahuan penagihan khusus OpenClaw dari Anthropic masih berlaku di sana. Kami
    juga mereproduksi pembatas penagihan yang sama secara lokal dengan penggunaan langsung
    `claude -p --append-system-prompt ...` ketika prompt tambahan
    mengidentifikasi OpenClaw, sementara string prompt yang sama **tidak** mereproduksi itu pada
    jalur Anthropic SDK + API-key.

    Untuk workload produksi atau multi-pengguna, auth Anthropic API key adalah
    pilihan yang lebih aman dan direkomendasikan. Jika Anda menginginkan opsi host bergaya langganan lain
    di OpenClaw, lihat [OpenAI](/id/providers/openai), [Qwen / Model
    Cloud](/id/providers/qwen), [MiniMax](/id/providers/minimax), dan
    [GLM Models](/id/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="Mengapa saya melihat HTTP 429 rate_limit_error dari Anthropic?">
Itu berarti **kuota/batas laju Anthropic** Anda habis untuk jendela saat ini. Jika Anda
menggunakan **Claude CLI**, tunggu sampai jendelanya di-reset atau tingkatkan paket Anda. Jika Anda
menggunakan **Anthropic API key**, periksa Anthropic Console
untuk penggunaan/penagihan dan tingkatkan batas bila perlu.

    Jika pesannya secara spesifik adalah:
    `Extra usage is required for long context requests`, permintaan tersebut mencoba menggunakan
    beta konteks 1M Anthropic (`context1m: true`). Itu hanya berfungsi ketika
    kredensial Anda memenuhi syarat untuk penagihan konteks panjang (penagihan API key atau
    jalur login Claude OpenClaw dengan Extra Usage diaktifkan).

    Tip: setel **model fallback** agar OpenClaw dapat terus membalas saat sebuah provider dibatasi laju.
    Lihat [Models](/cli/models), [OAuth](/id/concepts/oauth), dan
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/id/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="Apakah AWS Bedrock didukung?">
    Ya. OpenClaw memiliki provider bawaan **Amazon Bedrock (Converse)**. Dengan marker env AWS yang ada, OpenClaw dapat menemukan otomatis katalog Bedrock streaming/teks dan menggabungkannya sebagai provider implisit `amazon-bedrock`; jika tidak, Anda dapat mengaktifkan secara eksplisit `plugins.entries.amazon-bedrock.config.discovery.enabled` atau menambahkan entri provider manual. Lihat [Amazon Bedrock](/id/providers/bedrock) dan [Model providers](/id/providers/models). Jika Anda lebih suka alur managed key, proksi kompatibel OpenAI di depan Bedrock juga tetap merupakan opsi yang valid.
  </Accordion>

  <Accordion title="Bagaimana cara kerja auth Codex?">
    OpenClaw mendukung **OpenAI Code (Codex)** melalui OAuth (sign-in ChatGPT). Onboarding dapat menjalankan alur OAuth dan akan menyetel model default ke `openai-codex/gpt-5.4` bila sesuai. Lihat [Model providers](/id/concepts/model-providers) dan [Onboarding (CLI)](/id/start/wizard).
  </Accordion>

  <Accordion title="Apakah Anda mendukung auth langganan OpenAI (Codex OAuth)?">
    Ya. OpenClaw sepenuhnya mendukung **OpenAI Code (Codex) subscription OAuth**.
    OpenAI secara eksplisit mengizinkan penggunaan subscription OAuth dalam alat/alur kerja eksternal
    seperti OpenClaw. Onboarding dapat menjalankan alur OAuth untuk Anda.

    Lihat [OAuth](/id/concepts/oauth), [Model providers](/id/concepts/model-providers), dan [Onboarding (CLI)](/id/start/wizard).

  </Accordion>

  <Accordion title="Bagaimana cara menyiapkan Gemini CLI OAuth?">
    Gemini CLI menggunakan **alur auth plugin**, bukan client id atau secret di `openclaw.json`.

    Gunakan provider Gemini API sebagai gantinya:

    1. Aktifkan plugin: `openclaw plugins enable google`
    2. Jalankan `openclaw onboard --auth-choice gemini-api-key`
    3. Setel model Google seperti `google/gemini-3.1-pro-preview`

  </Accordion>

  <Accordion title="Apakah model lokal oke untuk obrolan santai?">
    Biasanya tidak. OpenClaw membutuhkan konteks besar + keamanan yang kuat; kartu kecil akan memangkas dan membocorkan. Jika harus, jalankan build model **terbesar** yang dapat Anda jalankan secara lokal (LM Studio) dan lihat [/gateway/local-models](/id/gateway/local-models). Model yang lebih kecil/terkuantisasi meningkatkan risiko prompt injection - lihat [Security](/id/gateway/security).
  </Accordion>

  <Accordion title="Bagaimana cara menjaga lalu lintas model host tetap di wilayah tertentu?">
    Pilih endpoint yang dipatok ke wilayah tertentu. OpenRouter mengekspos opsi yang di-host di AS untuk MiniMax, Kimi, dan GLM; pilih varian yang di-host di AS untuk menjaga data tetap di wilayah tersebut. Anda tetap dapat mencantumkan Anthropic/OpenAI bersama ini dengan menggunakan `models.mode: "merge"` agar fallback tetap tersedia sambil tetap menghormati provider regional yang Anda pilih.
  </Accordion>

  <Accordion title="Apakah saya harus membeli Mac Mini untuk menginstal ini?">
    Tidak. OpenClaw berjalan di macOS atau Linux (Windows melalui WSL2). Mac mini bersifat opsional - beberapa orang
    membelinya sebagai host yang selalu aktif, tetapi VPS kecil, server rumahan, atau perangkat kelas Raspberry Pi juga bisa.

    Anda hanya membutuhkan Mac **untuk alat khusus macOS**. Untuk iMessage, gunakan [BlueBubbles](/id/channels/bluebubbles) (disarankan) - server BlueBubbles berjalan di Mac apa pun, dan Gateway dapat berjalan di Linux atau di tempat lain. Jika Anda ingin alat khusus macOS lainnya, jalankan Gateway di Mac atau pasangkan node macOS.

    Dokumentasi: [BlueBubbles](/id/channels/bluebubbles), [Nodes](/id/nodes), [Mac remote mode](/id/platforms/mac/remote).

  </Accordion>

  <Accordion title="Apakah saya memerlukan Mac mini untuk dukungan iMessage?">
    Anda membutuhkan **perangkat macOS apa pun** yang masuk ke Messages. Itu **tidak** harus Mac mini -
    Mac apa pun bisa. **Gunakan [BlueBubbles](/id/channels/bluebubbles)** (disarankan) untuk iMessage - server BlueBubbles berjalan di macOS, sementara Gateway dapat berjalan di Linux atau di tempat lain.

    Penyiapan umum:

    - Jalankan Gateway di Linux/VPS, dan jalankan server BlueBubbles di Mac apa pun yang masuk ke Messages.
    - Jalankan semuanya di Mac jika Anda ingin penyiapan satu mesin yang paling sederhana.

    Dokumentasi: [BlueBubbles](/id/channels/bluebubbles), [Nodes](/id/nodes),
    [Mac remote mode](/id/platforms/mac/remote).

  </Accordion>

  <Accordion title="Jika saya membeli Mac mini untuk menjalankan OpenClaw, bisakah saya menghubungkannya ke MacBook Pro saya?">
    Ya. **Mac mini dapat menjalankan Gateway**, dan MacBook Pro Anda dapat terhubung sebagai
    **node** (perangkat pendamping). Nodes tidak menjalankan Gateway - mereka menyediakan
    kapabilitas tambahan seperti layar/kamera/canvas dan `system.run` pada perangkat tersebut.

    Pola umum:

    - Gateway di Mac mini (selalu aktif).
    - MacBook Pro menjalankan aplikasi macOS atau host node dan dipasangkan ke Gateway.
    - Gunakan `openclaw nodes status` / `openclaw nodes list` untuk melihatnya.

    Dokumentasi: [Nodes](/id/nodes), [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="Bisakah saya menggunakan Bun?">
    Bun **tidak direkomendasikan**. Kami melihat bug runtime, terutama dengan WhatsApp dan Telegram.
    Gunakan **Node** untuk gateway yang stabil.

    Jika Anda tetap ingin bereksperimen dengan Bun, lakukan di gateway non-produksi
    tanpa WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram: apa yang dimasukkan ke allowFrom?">
    `channels.telegram.allowFrom` adalah **Telegram user ID pengirim manusia** (numerik). Itu bukan username bot.

    Onboarding menerima input `@username` dan menyelesaikannya menjadi ID numerik, tetapi otorisasi OpenClaw hanya menggunakan ID numerik.

    Lebih aman (tanpa bot pihak ketiga):

    - Kirim DM ke bot Anda, lalu jalankan `openclaw logs --follow` dan baca `from.id`.

    Bot API resmi:

    - Kirim DM ke bot Anda, lalu panggil `https://api.telegram.org/bot<bot_token>/getUpdates` dan baca `message.from.id`.

    Pihak ketiga (kurang privat):

    - Kirim DM ke `@userinfobot` atau `@getidsbot`.

    Lihat [/channels/telegram](/id/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="Bisakah beberapa orang menggunakan satu nomor WhatsApp dengan instans OpenClaw yang berbeda?">
    Ya, melalui **multi-agent routing**. Ikat setiap **DM** WhatsApp pengirim (peer `kind: "direct"`, E.164 pengirim seperti `+15551234567`) ke `agentId` yang berbeda, sehingga setiap orang mendapatkan workspace dan penyimpanan sesi mereka sendiri. Balasan tetap datang dari **akun WhatsApp yang sama**, dan kontrol akses DM (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) bersifat global per akun WhatsApp. Lihat [Multi-Agent Routing](/id/concepts/multi-agent) dan [WhatsApp](/id/channels/whatsapp).
  </Accordion>

  <Accordion title='Bisakah saya menjalankan agen "fast chat" dan agen "Opus for coding"?'>
    Ya. Gunakan multi-agent routing: beri setiap agen model defaultnya sendiri, lalu ikat rute masuk (akun provider atau peer tertentu) ke masing-masing agen. Contoh konfigurasi ada di [Multi-Agent Routing](/id/concepts/multi-agent). Lihat juga [Models](/id/concepts/models) dan [Configuration](/id/gateway/configuration).
  </Accordion>

  <Accordion title="Apakah Homebrew bekerja di Linux?">
    Ya. Homebrew mendukung Linux (Linuxbrew). Penyiapan cepat:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Jika Anda menjalankan OpenClaw melalui systemd, pastikan PATH layanan menyertakan `/home/linuxbrew/.linuxbrew/bin` (atau prefiks brew Anda) agar alat yang diinstal `brew` dapat ditemukan dalam shell non-login.
    Build terbaru juga menambahkan direktori bin pengguna umum pada layanan Linux systemd (misalnya `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`) dan menghormati `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR`, dan `FNM_DIR` saat disetel.

  </Accordion>

  <Accordion title="Perbedaan antara instalasi git hackable dan npm install">
    - **Instalasi hackable (git):** checkout source lengkap, dapat diedit, terbaik untuk kontributor.
      Anda menjalankan build secara lokal dan bisa mem-patch kode/dokumentasi.
    - **npm install:** instalasi CLI global, tanpa repo, terbaik untuk "tinggal jalankan."
      Pembaruan datang dari npm dist-tag.

    Dokumentasi: [Getting started](/id/start/getting-started), [Updating](/id/install/updating).

  </Accordion>

  <Accordion title="Bisakah saya beralih antara instalasi npm dan git nanti?">
    Ya. Instal varian lainnya, lalu jalankan Doctor agar layanan gateway menunjuk ke entrypoint baru.
    Ini **tidak menghapus data Anda** - ini hanya mengubah instalasi kode OpenClaw. State Anda
    (`~/.openclaw`) dan workspace (`~/.openclaw/workspace`) tetap tidak tersentuh.

    Dari npm ke git:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    openclaw doctor
    openclaw gateway restart
    ```

    Dari git ke npm:

    ```bash
    npm install -g openclaw@latest
    openclaw doctor
    openclaw gateway restart
    ```

    Doctor mendeteksi ketidakcocokan entrypoint layanan gateway dan menawarkan untuk menulis ulang konfigurasi layanan agar sesuai dengan instalasi saat ini (gunakan `--repair` dalam otomatisasi).

    Tip cadangan: lihat [Strategi cadangan](#where-things-live-on-disk).

  </Accordion>

  <Accordion title="Sebaiknya saya menjalankan Gateway di laptop atau VPS?">
    Jawaban singkat: **jika Anda menginginkan keandalan 24/7, gunakan VPS**. Jika Anda ingin
    gesekan paling rendah dan Anda tidak masalah dengan sleep/restart, jalankan secara lokal.

    **Laptop (Gateway lokal)**

    - **Kelebihan:** tidak ada biaya server, akses langsung ke file lokal, jendela browser langsung.
    - **Kekurangan:** sleep/jaringan putus = diskoneksi, pembaruan/reboot OS mengganggu, harus tetap aktif.

    **VPS / cloud**

    - **Kelebihan:** selalu aktif, jaringan stabil, tidak ada masalah sleep laptop, lebih mudah dijaga tetap berjalan.
    - **Kekurangan:** sering berjalan headless (gunakan tangkapan layar), hanya akses file remote, Anda harus SSH untuk pembaruan.

    **Catatan khusus OpenClaw:** WhatsApp/Telegram/Slack/Mattermost/Discord semuanya bekerja baik dari VPS. Satu-satunya trade-off nyata adalah **browser headless** vs jendela yang terlihat. Lihat [Browser](/id/tools/browser).

    **Default yang direkomendasikan:** VPS jika Anda pernah mengalami gateway terputus sebelumnya. Lokal sangat baik saat Anda aktif menggunakan Mac dan menginginkan akses file lokal atau otomasi UI dengan browser yang terlihat.

  </Accordion>

  <Accordion title="Seberapa penting menjalankan OpenClaw di mesin khusus?">
    Tidak wajib, tetapi **direkomendasikan untuk keandalan dan isolasi**.

    - **Host khusus (VPS/Mac mini/Pi):** selalu aktif, lebih sedikit gangguan sleep/reboot, izin lebih bersih, lebih mudah dijaga tetap berjalan.
    - **Laptop/desktop bersama:** sepenuhnya baik untuk pengujian dan penggunaan aktif, tetapi perkirakan jeda saat mesin sleep atau melakukan pembaruan.

    Jika Anda menginginkan yang terbaik dari keduanya, biarkan Gateway di host khusus dan pasangkan laptop Anda sebagai **node** untuk alat layar/kamera/exec lokal. Lihat [Nodes](/id/nodes).
    Untuk panduan keamanan, baca [Security](/id/gateway/security).

  </Accordion>

  <Accordion title="Apa persyaratan minimum VPS dan OS yang direkomendasikan?">
    OpenClaw ringan. Untuk Gateway dasar + satu channel obrolan:

    - **Minimum mutlak:** 1 vCPU, 1GB RAM, ~500MB disk.
    - **Direkomendasikan:** 1-2 vCPU, 2GB RAM atau lebih untuk ruang ekstra (log, media, beberapa channel). Alat Node dan otomasi browser bisa boros sumber daya.

    OS: gunakan **Ubuntu LTS** (atau Debian/Ubuntu modern apa pun). Jalur instalasi Linux paling teruji di sana.

    Dokumentasi: [Linux](/id/platforms/linux), [VPS hosting](/id/vps).

  </Accordion>

  <Accordion title="Bisakah saya menjalankan OpenClaw di VM dan apa persyaratannya?">
    Ya. Perlakukan VM sama seperti VPS: harus selalu aktif, dapat dijangkau, dan memiliki cukup
    RAM untuk Gateway dan channel apa pun yang Anda aktifkan.

    Panduan dasar:

    - **Minimum mutlak:** 1 vCPU, 1GB RAM.
    - **Direkomendasikan:** 2GB RAM atau lebih jika Anda menjalankan beberapa channel, otomasi browser, atau alat media.
    - **OS:** Ubuntu LTS atau Debian/Ubuntu modern lainnya.

    Jika Anda menggunakan Windows, **WSL2 adalah penyiapan bergaya VM termudah** dan memiliki kompatibilitas
    alat terbaik. Lihat [Windows](/id/platforms/windows), [VPS hosting](/id/vps).
    Jika Anda menjalankan macOS di VM, lihat [macOS VM](/id/install/macos-vm).

  </Accordion>
</AccordionGroup>

## Apa itu OpenClaw?

<AccordionGroup>
  <Accordion title="Apa itu OpenClaw, dalam satu paragraf?">
    OpenClaw adalah asisten AI pribadi yang Anda jalankan di perangkat Anda sendiri. Ia membalas di permukaan perpesanan yang sudah Anda gunakan (WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat, dan plugin channel bawaan seperti QQ Bot) dan juga dapat melakukan suara + Canvas langsung pada platform yang didukung. **Gateway** adalah control plane yang selalu aktif; asistennya adalah produknya.
  </Accordion>

  <Accordion title="Proposisi nilai">
    OpenClaw bukan "sekadar wrapper Claude." Ini adalah **control plane local-first** yang memungkinkan Anda menjalankan
    asisten yang mumpuni di **perangkat keras Anda sendiri**, dapat dijangkau dari aplikasi obrolan yang sudah Anda gunakan, dengan
    sesi yang stateful, memori, dan alat - tanpa menyerahkan kontrol alur kerja Anda ke
    SaaS yang di-host.

    Sorotan:

    - **Perangkat Anda, data Anda:** jalankan Gateway di mana pun Anda mau (Mac, Linux, VPS) dan simpan
      workspace + riwayat sesi secara lokal.
    - **Channel nyata, bukan sandbox web:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/dll,
      ditambah suara seluler dan Canvas pada platform yang didukung.
    - **Agnostik-model:** gunakan Anthropic, OpenAI, MiniMax, OpenRouter, dll., dengan perutean
      dan failover per agen.
    - **Opsi lokal saja:** jalankan model lokal sehingga **semua data dapat tetap di perangkat Anda** jika Anda mau.
    - **Multi-agent routing:** agen terpisah per channel, akun, atau tugas, masing-masing dengan
      workspace dan default sendiri.
    - **Open source dan hackable:** periksa, perluas, dan self-host tanpa vendor lock-in.

    Dokumentasi: [Gateway](/id/gateway), [Channels](/id/channels), [Multi-agent](/id/concepts/multi-agent),
    [Memory](/id/concepts/memory).

  </Accordion>

  <Accordion title="Saya baru saja menyiapkannya - apa yang sebaiknya saya lakukan dulu?">
    Proyek pertama yang bagus:

    - Bangun situs web (WordPress, Shopify, atau situs statis sederhana).
    - Buat prototipe aplikasi seluler (kerangka, layar, rencana API).
    - Rapikan file dan folder (pembersihan, penamaan, penandaan).
    - Hubungkan Gmail dan otomatisasi ringkasan atau tindak lanjut.

    Ini dapat menangani tugas besar, tetapi bekerja paling baik ketika Anda membaginya menjadi beberapa fase dan
    menggunakan sub agen untuk pekerjaan paralel.

  </Accordion>

  <Accordion title="Apa lima kasus penggunaan sehari-hari teratas untuk OpenClaw?">
    Kemenangan sehari-hari biasanya terlihat seperti:

    - **Briefing pribadi:** ringkasan kotak masuk, kalender, dan berita yang Anda pedulikan.
    - **Riset dan drafting:** riset cepat, ringkasan, dan draf pertama untuk email atau dokumen.
    - **Pengingat dan tindak lanjut:** dorongan dan daftar periksa yang digerakkan cron atau heartbeat.
    - **Otomasi browser:** mengisi formulir, mengumpulkan data, dan mengulang tugas web.
    - **Koordinasi lintas perangkat:** kirim tugas dari ponsel Anda, biarkan Gateway menjalankannya di server, dan dapatkan hasilnya kembali di obrolan.

  </Accordion>

  <Accordion title="Bisakah OpenClaw membantu lead gen, outreach, iklan, dan blog untuk SaaS?">
    Ya untuk **riset, kualifikasi, dan drafting**. Ia dapat memindai situs, membangun shortlist,
    merangkum prospek, dan menulis draf outreach atau salinan iklan.

    Untuk **outreach atau penayangan iklan**, pertahankan manusia dalam loop. Hindari spam, patuhi hukum lokal dan
    kebijakan platform, dan tinjau apa pun sebelum dikirim. Pola paling aman adalah membiarkan
    OpenClaw membuat draf dan Anda yang menyetujuinya.

    Dokumentasi: [Security](/id/gateway/security).

  </Accordion>

  <Accordion title="Apa kelebihannya dibanding Claude Code untuk pengembangan web?">
    OpenClaw adalah **asisten pribadi** dan lapisan koordinasi, bukan pengganti IDE. Gunakan
    Claude Code atau Codex untuk loop coding langsung tercepat di dalam repo. Gunakan OpenClaw ketika Anda
    menginginkan memori yang tahan lama, akses lintas perangkat, dan orkestrasi alat.

    Kelebihan:

    - **Memori + workspace persisten** antar sesi
    - **Akses multi-platform** (WhatsApp, Telegram, TUI, WebChat)
    - **Orkestrasi alat** (browser, file, penjadwalan, hooks)
    - **Gateway selalu aktif** (jalankan di VPS, berinteraksi dari mana saja)
    - **Nodes** untuk browser/layar/kamera/exec lokal

    Showcase: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills dan otomasi

<AccordionGroup>
  <Accordion title="Bagaimana cara menyesuaikan Skills tanpa membuat repo tetap kotor?">
    Gunakan override terkelola alih-alih mengedit salinan repo. Letakkan perubahan Anda di `~/.openclaw/skills/<name>/SKILL.md` (atau tambahkan folder melalui `skills.load.extraDirs` di `~/.openclaw/openclaw.json`). Urutan prioritas adalah `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → bawaan → `skills.load.extraDirs`, sehingga override terkelola tetap menang atas Skills bawaan tanpa menyentuh git. Jika Anda memerlukan skill diinstal secara global tetapi hanya terlihat oleh beberapa agen, simpan salinan bersama di `~/.openclaw/skills` dan kontrol visibilitas dengan `agents.defaults.skills` dan `agents.list[].skills`. Hanya edit yang layak upstream yang sebaiknya hidup di repo dan dikirim sebagai PR.
  </Accordion>

  <Accordion title="Bisakah saya memuat Skills dari folder kustom?">
    Ya. Tambahkan direktori ekstra melalui `skills.load.extraDirs` di `~/.openclaw/openclaw.json` (prioritas terendah). Prioritas default adalah `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → bawaan → `skills.load.extraDirs`. `clawhub` menginstal ke `./skills` secara default, yang diperlakukan OpenClaw sebagai `<workspace>/skills` pada sesi berikutnya. Jika skill hanya boleh terlihat oleh agen tertentu, pasangkan dengan `agents.defaults.skills` atau `agents.list[].skills`.
  </Accordion>

  <Accordion title="Bagaimana saya bisa menggunakan model yang berbeda untuk tugas yang berbeda?">
    Saat ini pola yang didukung adalah:

    - **Cron jobs**: job terisolasi dapat menyetel override `model` per job.
    - **Sub-agents**: arahkan tugas ke agen terpisah dengan model default yang berbeda.
    - **Peralihan on-demand**: gunakan `/model` untuk mengganti model sesi saat ini kapan saja.

    Lihat [Cron jobs](/id/automation/cron-jobs), [Multi-Agent Routing](/id/concepts/multi-agent), dan [Slash commands](/id/tools/slash-commands).

  </Accordion>

  <Accordion title="Bot membeku saat melakukan pekerjaan berat. Bagaimana cara memindahkannya?">
    Gunakan **sub-agents** untuk tugas yang panjang atau paralel. Sub-agents berjalan di sesi mereka sendiri,
    mengembalikan ringkasan, dan menjaga obrolan utama Anda tetap responsif.

    Minta bot Anda untuk "spawn a sub-agent for this task" atau gunakan `/subagents`.
    Gunakan `/status` di obrolan untuk melihat apa yang sedang dilakukan Gateway saat ini (dan apakah sedang sibuk).

    Tip token: tugas panjang dan sub-agents sama-sama mengonsumsi token. Jika biaya menjadi perhatian, setel
    model yang lebih murah untuk sub-agents melalui `agents.defaults.subagents.model`.

    Dokumentasi: [Sub-agents](/id/tools/subagents), [Background Tasks](/id/automation/tasks).

  </Accordion>

  <Accordion title="Bagaimana cara kerja sesi subagent yang terikat thread di Discord?">
    Gunakan binding thread. Anda dapat mengikat thread Discord ke target subagent atau sesi agar pesan tindak lanjut di thread itu tetap berada pada sesi yang terikat tersebut.

    Alur dasar:

    - Spawn dengan `sessions_spawn` menggunakan `thread: true` (dan opsional `mode: "session"` untuk tindak lanjut persisten).
    - Atau ikat secara manual dengan `/focus <target>`.
    - Gunakan `/agents` untuk memeriksa state binding.
    - Gunakan `/session idle <duration|off>` dan `/session max-age <duration|off>` untuk mengontrol auto-unfocus.
    - Gunakan `/unfocus` untuk melepaskan thread.

    Konfigurasi yang diperlukan:

    - Default global: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - Override Discord: `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - Auto-bind saat spawn: setel `channels.discord.threadBindings.spawnSubagentSessions: true`.

    Dokumentasi: [Sub-agents](/id/tools/subagents), [Discord](/id/channels/discord), [Configuration Reference](/id/gateway/configuration-reference), [Slash commands](/id/tools/slash-commands).

  </Accordion>

  <Accordion title="Subagent selesai, tetapi pembaruan selesai dikirim ke tempat yang salah atau tidak pernah diposting. Apa yang harus saya periksa?">
    Periksa rute peminta yang telah diselesaikan terlebih dahulu:

    - Pengiriman subagent mode-selesai memprioritaskan thread atau rute percakapan apa pun yang terikat ketika ada.
    - Jika asal penyelesaian hanya membawa channel, OpenClaw fallback ke rute tersimpan sesi peminta (`lastChannel` / `lastTo` / `lastAccountId`) sehingga pengiriman langsung masih bisa berhasil.
    - Jika tidak ada rute terikat maupun rute tersimpan yang dapat digunakan, pengiriman langsung dapat gagal dan hasilnya fallback ke pengiriman sesi yang masuk antrean alih-alih diposting langsung ke obrolan.
    - Target yang tidak valid atau stale masih dapat memaksa fallback antrean atau kegagalan pengiriman akhir.
    - Jika balasan asisten terakhir yang terlihat dari child adalah token diam persis `NO_REPLY` / `no_reply`, atau tepat `ANNOUNCE_SKIP`, OpenClaw sengaja menekan pengumuman alih-alih memposting progres lama yang stale.
    - Jika child timeout setelah hanya melakukan pemanggilan alat, pengumuman dapat meringkasnya menjadi ringkasan progres parsial singkat alih-alih memutar ulang output alat mentah.

    Debug:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    Dokumentasi: [Sub-agents](/id/tools/subagents), [Background Tasks](/id/automation/tasks), [Session Tools](/id/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron atau pengingat tidak berjalan. Apa yang harus saya periksa?">
    Cron berjalan di dalam proses Gateway. Jika Gateway tidak berjalan terus-menerus,
    job terjadwal tidak akan berjalan.

    Daftar periksa:

    - Konfirmasi cron diaktifkan (`cron.enabled`) dan `OPENCLAW_SKIP_CRON` tidak disetel.
    - Periksa bahwa Gateway berjalan 24/7 (tanpa sleep/restart).
    - Verifikasi pengaturan zona waktu untuk job (`--tz` vs zona waktu host).

    Debug:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Dokumentasi: [Cron jobs](/id/automation/cron-jobs), [Automation & Tasks](/id/automation).

  </Accordion>

  <Accordion title="Cron berjalan, tetapi tidak ada yang dikirim ke channel. Mengapa?">
    Periksa mode pengiriman terlebih dahulu:

    - `--no-deliver` / `delivery.mode: "none"` berarti tidak ada pesan eksternal yang diharapkan.
    - Target pengumuman yang hilang atau tidak valid (`channel` / `to`) berarti runner melewati pengiriman outbound.
    - Kegagalan auth channel (`unauthorized`, `Forbidden`) berarti runner mencoba mengirim tetapi kredensial memblokirnya.
    - Hasil terisolasi yang diam (`NO_REPLY` / `no_reply` saja) diperlakukan sebagai sengaja tidak dapat dikirim, sehingga runner juga menekan pengiriman fallback terantre.

    Untuk cron jobs terisolasi, runner memiliki pengiriman akhir. Agen diharapkan
    mengembalikan ringkasan teks biasa agar dikirim oleh runner. `--no-deliver` menjaga
    hasil itu tetap internal; itu tidak membiarkan agen mengirim langsung dengan
    message tool sebagai gantinya.

    Debug:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Dokumentasi: [Cron jobs](/id/automation/cron-jobs), [Background Tasks](/id/automation/tasks).

  </Accordion>

  <Accordion title="Mengapa run cron terisolasi beralih model atau mencoba ulang sekali?">
    Itu biasanya adalah jalur peralihan model langsung, bukan penjadwalan ganda.

    Cron terisolasi dapat mempertahankan handoff model runtime dan mencoba ulang ketika run aktif
    melempar `LiveSessionModelSwitchError`. Percobaan ulang mempertahankan
    provider/model yang dialihkan, dan jika peralihan itu membawa override profil auth baru, cron
    juga mempertahankannya sebelum mencoba ulang.

    Aturan pemilihan terkait:

    - Override model hook Gmail menang terlebih dahulu bila berlaku.
    - Lalu `model` per-job.
    - Lalu override model sesi-cron yang tersimpan.
    - Lalu pemilihan model agen/default normal.

    Loop percobaan ulang dibatasi. Setelah percobaan awal ditambah 2 percobaan ulang peralihan,
    cron dibatalkan alih-alih berulang tanpa akhir.

    Debug:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Dokumentasi: [Cron jobs](/id/automation/cron-jobs), [cron CLI](/cli/cron).

  </Accordion>

  <Accordion title="Bagaimana cara menginstal Skills di Linux?">
    Gunakan perintah native `openclaw skills` atau letakkan skills ke workspace Anda. UI Skills macOS tidak tersedia di Linux.
    Telusuri skills di [https://clawhub.ai](https://clawhub.ai).

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    `openclaw skills install` native menulis ke direktori `skills/`
    workspace aktif. Instal CLI `clawhub` terpisah hanya jika Anda ingin memublikasikan atau
    menyinkronkan skill Anda sendiri. Untuk instalasi bersama antar agen, letakkan skill di bawah
    `~/.openclaw/skills` dan gunakan `agents.defaults.skills` atau
    `agents.list[].skills` jika Anda ingin membatasi agen mana yang dapat melihatnya.

  </Accordion>

  <Accordion title="Bisakah OpenClaw menjalankan tugas sesuai jadwal atau terus-menerus di latar belakang?">
    Ya. Gunakan penjadwal Gateway:

    - **Cron jobs** untuk tugas terjadwal atau berulang (persisten antar restart).
    - **Heartbeat** untuk pemeriksaan periodik "sesi utama".
    - **Job terisolasi** untuk agen otonom yang memposting ringkasan atau mengirim ke obrolan.

    Dokumentasi: [Cron jobs](/id/automation/cron-jobs), [Automation & Tasks](/id/automation),
    [Heartbeat](/id/gateway/heartbeat).

  </Accordion>

  <Accordion title="Bisakah saya menjalankan skill Apple khusus macOS dari Linux?">
    Tidak secara langsung. Skills macOS dibatasi oleh `metadata.openclaw.os` plus biner yang diperlukan, dan skills hanya muncul di system prompt ketika memenuhi syarat pada **gateway host**. Di Linux, skill khusus `darwin` (seperti `apple-notes`, `apple-reminders`, `things-mac`) tidak akan dimuat kecuali Anda mengubah gating.

    Anda memiliki tiga pola yang didukung:

    **Opsi A - jalankan Gateway di Mac (paling sederhana).**
    Jalankan Gateway di tempat biner macOS tersedia, lalu hubungkan dari Linux dalam [mode remote](#gateway-ports-already-running-and-remote-mode) atau melalui Tailscale. Skills dimuat secara normal karena gateway host adalah macOS.

    **Opsi B - gunakan node macOS (tanpa SSH).**
    Jalankan Gateway di Linux, pasangkan node macOS (aplikasi menubar), dan setel **Node Run Commands** ke "Always Ask" atau "Always Allow" di Mac. OpenClaw dapat memperlakukan skill khusus macOS sebagai memenuhi syarat ketika biner yang diperlukan ada di node. Agen menjalankan skills tersebut melalui alat `nodes`. Jika Anda memilih "Always Ask", menyetujui "Always Allow" di prompt akan menambahkan perintah itu ke allowlist.

    **Opsi C - proksikan biner macOS melalui SSH (lanjutan).**
    Biarkan Gateway di Linux, tetapi buat biner CLI yang diperlukan diselesaikan ke wrapper SSH yang berjalan di Mac. Lalu override skill agar mengizinkan Linux sehingga tetap memenuhi syarat.

    1. Buat wrapper SSH untuk biner (contoh: `memo` untuk Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Letakkan wrapper di `PATH` pada host Linux (misalnya `~/bin/memo`).
    3. Override metadata skill (workspace atau `~/.openclaw/skills`) agar mengizinkan Linux:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. Mulai sesi baru agar snapshot skill dimuat ulang.

  </Accordion>

  <Accordion title="Apakah Anda punya integrasi Notion atau HeyGen?">
    Belum bawaan saat ini.

    Opsi:

    - **Skill / plugin kustom:** terbaik untuk akses API yang andal (Notion/HeyGen sama-sama punya API).
    - **Otomasi browser:** bekerja tanpa kode tetapi lebih lambat dan lebih rapuh.

    Jika Anda ingin menjaga konteks per klien (alur kerja agensi), pola sederhananya adalah:

    - Satu halaman Notion per klien (konteks + preferensi + pekerjaan aktif).
    - Minta agen mengambil halaman itu di awal sesi.

    Jika Anda menginginkan integrasi native, buka permintaan fitur atau bangun skill
    yang menargetkan API tersebut.

    Instal skills:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Instalasi native mendarat di direktori `skills/` workspace aktif. Untuk skill bersama antar agen, tempatkan di `~/.openclaw/skills/<name>/SKILL.md`. Jika hanya beberapa agen yang boleh melihat instalasi bersama, konfigurasikan `agents.defaults.skills` atau `agents.list[].skills`. Beberapa skills mengharapkan biner yang diinstal melalui Homebrew; di Linux itu berarti Linuxbrew (lihat entri FAQ Homebrew Linux di atas). Lihat [Skills](/id/tools/skills), [Skills config](/id/tools/skills-config), dan [ClawHub](/id/tools/clawhub).

  </Accordion>

  <Accordion title="Bagaimana cara menggunakan Chrome yang sudah masuk dengan OpenClaw?">
    Gunakan profil browser bawaan `user`, yang terhubung melalui Chrome DevTools MCP:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Jika Anda menginginkan nama kustom, buat profil MCP eksplisit:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Jalur ini bersifat lokal pada host. Jika Gateway berjalan di tempat lain, jalankan host node pada mesin browser atau gunakan CDP remote sebagai gantinya.

    Batas saat ini pada `existing-session` / `user`:

    - aksi berbasis ref, bukan berbasis CSS selector
    - unggahan memerlukan `ref` / `inputRef` dan saat ini hanya mendukung satu file dalam satu waktu
    - `responsebody`, ekspor PDF, intersepsi unduhan, dan aksi batch masih memerlukan browser terkelola atau profil CDP mentah

  </Accordion>
</AccordionGroup>

## Sandboxing dan memori

<AccordionGroup>
  <Accordion title="Apakah ada dokumentasi khusus sandboxing?">
    Ya. Lihat [Sandboxing](/id/gateway/sandboxing). Untuk penyiapan khusus Docker (gateway penuh di Docker atau image sandbox), lihat [Docker](/id/install/docker).
  </Accordion>

  <Accordion title="Docker terasa terbatas - bagaimana cara mengaktifkan fitur lengkap?">
    Image default mengutamakan keamanan dan berjalan sebagai pengguna `node`, sehingga tidak
    menyertakan paket sistem, Homebrew, atau browser bawaan. Untuk penyiapan yang lebih lengkap:

    - Persistenkan `/home/node` dengan `OPENCLAW_HOME_VOLUME` agar cache bertahan.
    - Panggang dependensi sistem ke dalam image dengan `OPENCLAW_DOCKER_APT_PACKAGES`.
    - Instal browser Playwright melalui CLI bawaan:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - Setel `PLAYWRIGHT_BROWSERS_PATH` dan pastikan path itu dipersistenkan.

    Dokumentasi: [Docker](/id/install/docker), [Browser](/id/tools/browser).

  </Accordion>

  <Accordion title="Bisakah saya menjaga DM tetap pribadi tetapi membuat grup publik/disandbox dengan satu agen?">
    Ya - jika lalu lintas privat Anda adalah **DM** dan lalu lintas publik Anda adalah **grup**.

    Gunakan `agents.defaults.sandbox.mode: "non-main"` agar sesi grup/channel (kunci non-utama) berjalan di Docker, sementara sesi DM utama tetap berjalan di host. Lalu batasi alat apa yang tersedia di sesi sandbox melalui `tools.sandbox.tools`.

    Panduan penyiapan + contoh konfigurasi: [Groups: DM pribadi + grup publik](/id/channels/groups#pattern-personal-dms-public-groups-single-agent)

    Referensi konfigurasi utama: [Gateway configuration](/id/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="Bagaimana cara mengikat folder host ke sandbox?">
    Setel `agents.defaults.sandbox.docker.binds` ke `["host:path:mode"]` (misalnya `"/home/user/src:/src:ro"`). Binding global + per-agen digabungkan; binding per-agen diabaikan ketika `scope: "shared"`. Gunakan `:ro` untuk apa pun yang sensitif dan ingat bahwa binding melewati dinding filesystem sandbox.

    OpenClaw memvalidasi sumber bind terhadap path yang dinormalisasi dan path kanonis yang diselesaikan melalui leluhur terdalam yang ada. Ini berarti lolos dari induk symlink tetap gagal-tertutup bahkan ketika segmen path terakhir belum ada, dan pemeriksaan allowed-root tetap berlaku setelah resolusi symlink.

    Lihat [Sandboxing](/id/gateway/sandboxing#custom-bind-mounts) dan [Sandbox vs Tool Policy vs Elevated](/id/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) untuk contoh dan catatan keamanan.

  </Accordion>

  <Accordion title="Bagaimana cara kerja memori?">
    Memori OpenClaw hanyalah file Markdown di workspace agen:

    - Catatan harian di `memory/YYYY-MM-DD.md`
    - Catatan jangka panjang terkurasi di `MEMORY.md` (hanya sesi utama/pribadi)

    OpenClaw juga menjalankan **flush memori pra-kompaksi diam-diam** untuk mengingatkan model
    agar menulis catatan yang tahan lama sebelum auto-compaction. Ini hanya berjalan ketika workspace
    dapat ditulisi (sandbox hanya-baca melewatinya). Lihat [Memory](/id/concepts/memory).

  </Accordion>

  <Accordion title="Memori terus melupakan hal-hal. Bagaimana cara membuatnya menempel?">
    Minta bot untuk **menulis fakta itu ke memori**. Catatan jangka panjang berada di `MEMORY.md`,
    konteks jangka pendek masuk ke `memory/YYYY-MM-DD.md`.

    Ini masih area yang terus kami perbaiki. Mengingatkan model untuk menyimpan memori akan membantu;
    model akan tahu apa yang harus dilakukan. Jika masih terus lupa, verifikasi Gateway menggunakan
    workspace yang sama pada setiap run.

    Dokumentasi: [Memory](/id/concepts/memory), [Agent workspace](/id/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Apakah memori bertahan selamanya? Apa batasnya?">
    File memori hidup di disk dan bertahan sampai Anda menghapusnya. Batasnya adalah
    penyimpanan Anda, bukan model. **Konteks sesi** tetap dibatasi oleh jendela konteks model,
    jadi percakapan yang panjang dapat mengalami compaction atau pemotongan. Itulah sebabnya
    pencarian memori ada - ia hanya menarik bagian yang relevan kembali ke konteks.

    Dokumentasi: [Memory](/id/concepts/memory), [Context](/id/concepts/context).

  </Accordion>

  <Accordion title="Apakah pencarian memori semantik memerlukan OpenAI API key?">
    Hanya jika Anda menggunakan **OpenAI embeddings**. Codex OAuth mencakup chat/completions dan
    **tidak** memberikan akses embeddings, jadi **sign-in dengan Codex (OAuth atau login
    Codex CLI)** tidak membantu untuk pencarian memori semantik. OpenAI embeddings
    tetap memerlukan API key sungguhan (`OPENAI_API_KEY` atau `models.providers.openai.apiKey`).

    Jika Anda tidak menyetel provider secara eksplisit, OpenClaw akan otomatis memilih provider ketika
    ia dapat menyelesaikan API key (profil auth, `models.providers.*.apiKey`, atau env vars).
    Ia memprioritaskan OpenAI jika kunci OpenAI berhasil diselesaikan, jika tidak Gemini jika kunci Gemini
    berhasil diselesaikan, lalu Voyage, lalu Mistral. Jika tidak ada kunci remote yang tersedia, pencarian
    memori tetap nonaktif sampai Anda mengonfigurasinya. Jika Anda memiliki jalur model lokal
    yang dikonfigurasi dan tersedia, OpenClaw
    memprioritaskan `local`. Ollama didukung ketika Anda secara eksplisit menyetel
    `memorySearch.provider = "ollama"`.

    Jika Anda lebih suka tetap lokal, setel `memorySearch.provider = "local"` (dan opsional
    `memorySearch.fallback = "none"`). Jika Anda menginginkan Gemini embeddings, setel
    `memorySearch.provider = "gemini"` dan sediakan `GEMINI_API_KEY` (atau
    `memorySearch.remote.apiKey`). Kami mendukung model embedding **OpenAI, Gemini, Voyage, Mistral, Ollama, atau local**
    - lihat [Memory](/id/concepts/memory) untuk detail penyiapannya.

  </Accordion>
</AccordionGroup>

## Tempat data berada di disk

<AccordionGroup>
  <Accordion title="Apakah semua data yang digunakan dengan OpenClaw disimpan secara lokal?">
    Tidak - **state OpenClaw bersifat lokal**, tetapi **layanan eksternal tetap melihat apa yang Anda kirim ke mereka**.

    - **Lokal secara default:** sesi, file memori, konfigurasi, dan workspace hidup di gateway host
      (`~/.openclaw` + direktori workspace Anda).
    - **Remote karena keharusan:** pesan yang Anda kirim ke provider model (Anthropic/OpenAI/dll.) pergi ke
      API mereka, dan platform obrolan (WhatsApp/Telegram/Slack/dll.) menyimpan data pesan di
      server mereka.
    - **Anda mengendalikan jejaknya:** menggunakan model lokal menjaga prompt di mesin Anda, tetapi lalu lintas
      channel tetap melalui server channel tersebut.

    Terkait: [Agent workspace](/id/concepts/agent-workspace), [Memory](/id/concepts/memory).

  </Accordion>

  <Accordion title="Di mana OpenClaw menyimpan datanya?">
    Semuanya berada di bawah `$OPENCLAW_STATE_DIR` (default: `~/.openclaw`):

    | Path                                                            | Tujuan                                                             |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | Konfigurasi utama (JSON5)                                          |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | Impor OAuth lama (disalin ke profil auth pada penggunaan pertama)  |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | Profil auth (OAuth, API key, dan `keyRef`/`tokenRef` opsional)     |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | Payload secret berbasis file opsional untuk provider `file` SecretRef |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | File kompatibilitas lama (entri `api_key` statis dibersihkan)      |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | State provider (mis. `whatsapp/<accountId>/creds.json`)            |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | State per agen (agentDir + sesi)                                   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | Riwayat & state percakapan (per agen)                              |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | Metadata sesi (per agen)                                           |

    Jalur lama agen tunggal: `~/.openclaw/agent/*` (dimigrasikan oleh `openclaw doctor`).

    **Workspace** Anda (AGENTS.md, file memori, Skills, dll.) terpisah dan dikonfigurasi melalui `agents.defaults.workspace` (default: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="Di mana AGENTS.md / SOUL.md / USER.md / MEMORY.md seharusnya berada?">
    File-file ini berada di **workspace agen**, bukan `~/.openclaw`.

    - **Workspace (per agen)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` (atau fallback lama `memory.md` saat `MEMORY.md` tidak ada),
      `memory/YYYY-MM-DD.md`, `HEARTBEAT.md` opsional.
    - **Direktori state (`~/.openclaw`)**: konfigurasi, state channel/provider, profil auth, sesi, log,
      dan skill bersama (`~/.openclaw/skills`).

    Workspace default adalah `~/.openclaw/workspace`, dapat dikonfigurasi melalui:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Jika bot "lupa" setelah restart, pastikan Gateway menggunakan workspace yang sama
    di setiap peluncuran (dan ingat: mode remote menggunakan workspace milik **gateway host**,
    bukan laptop lokal Anda).

    Tip: jika Anda menginginkan perilaku atau preferensi yang tahan lama, minta bot untuk **menuliskannya ke
    AGENTS.md atau MEMORY.md** alih-alih mengandalkan riwayat obrolan.

    Lihat [Agent workspace](/id/concepts/agent-workspace) dan [Memory](/id/concepts/memory).

  </Accordion>

  <Accordion title="Strategi cadangan yang direkomendasikan">
    Letakkan **workspace agen** Anda di repo git **pribadi** dan cadangkan di tempat
    privat (misalnya GitHub private). Ini menangkap memori + file AGENTS/SOUL/USER
    dan memungkinkan Anda memulihkan "pikiran" asisten nanti.

    **Jangan** commit apa pun di bawah `~/.openclaw` (kredensial, sesi, token, atau payload secret terenkripsi).
    Jika Anda membutuhkan pemulihan penuh, cadangkan workspace dan direktori state
    secara terpisah (lihat pertanyaan migrasi di atas).

    Dokumentasi: [Agent workspace](/id/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Bagaimana cara menghapus OpenClaw sepenuhnya?">
    Lihat panduan khusus: [Uninstall](/id/install/uninstall).
  </Accordion>

  <Accordion title="Bisakah agen bekerja di luar workspace?">
    Ya. Workspace adalah **cwd default** dan jangkar memori, bukan sandbox keras.
    Path relatif diselesaikan di dalam workspace, tetapi path absolut dapat mengakses lokasi
    host lain kecuali sandboxing diaktifkan. Jika Anda memerlukan isolasi, gunakan
    [`agents.defaults.sandbox`](/id/gateway/sandboxing) atau pengaturan sandbox per agen. Jika Anda
    ingin repo menjadi direktori kerja default, arahkan `workspace`
    agen tersebut ke root repo. Repo OpenClaw hanyalah source code; simpan
    workspace terpisah kecuali Anda memang sengaja ingin agen bekerja di dalamnya.

    Contoh (repo sebagai cwd default):

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Mode remote: di mana penyimpanan sesi?">
    State sesi dimiliki oleh **gateway host**. Jika Anda berada dalam mode remote, penyimpanan sesi yang Anda pedulikan berada di mesin remote, bukan laptop lokal Anda. Lihat [Session management](/id/concepts/session).
  </Accordion>
</AccordionGroup>

## Dasar konfigurasi

<AccordionGroup>
  <Accordion title="Format apa konfigurasi itu? Di mana letaknya?">
    OpenClaw membaca konfigurasi **JSON5** opsional dari `$OPENCLAW_CONFIG_PATH` (default: `~/.openclaw/openclaw.json`):

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    Jika file tidak ada, ia menggunakan default yang cukup aman (termasuk workspace default `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='Saya menyetel gateway.bind: "lan" (atau "tailnet") dan sekarang tidak ada yang listen / UI mengatakan unauthorized'>
    Bind non-loopback **memerlukan jalur auth gateway yang valid**. Dalam praktiknya itu berarti:

    - auth shared-secret: token atau kata sandi
    - `gateway.auth.mode: "trusted-proxy"` di belakang reverse proxy sadar identitas non-loopback yang dikonfigurasi dengan benar

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    Catatan:

    - `gateway.remote.token` / `.password` dengan sendirinya **tidak** mengaktifkan auth gateway lokal.
    - Jalur panggilan lokal dapat menggunakan `gateway.remote.*` sebagai fallback hanya ketika `gateway.auth.*` tidak disetel.
    - Untuk auth kata sandi, setel `gateway.auth.mode: "password"` ditambah `gateway.auth.password` (atau `OPENCLAW_GATEWAY_PASSWORD`) sebagai gantinya.
    - Jika `gateway.auth.token` / `gateway.auth.password` dikonfigurasi secara eksplisit melalui SecretRef dan tidak terselesaikan, resolusi gagal-tertutup (tidak ada fallback remote yang menutupi).
    - Penyiapan Control UI shared-secret mengautentikasi melalui `connect.params.auth.token` atau `connect.params.auth.password` (disimpan di pengaturan aplikasi/UI). Mode pembawa identitas seperti Tailscale Serve atau `trusted-proxy` menggunakan header permintaan sebagai gantinya. Hindari menaruh shared secret di URL.
    - Dengan `gateway.auth.mode: "trusted-proxy"`, reverse proxy loopback pada host yang sama tetap **tidak** memenuhi auth trusted-proxy. Trusted proxy harus merupakan sumber non-loopback yang dikonfigurasi.

  </Accordion>

  <Accordion title="Mengapa saya sekarang membutuhkan token di localhost?">
    OpenClaw menerapkan auth gateway secara default, termasuk loopback. Dalam jalur default normal itu berarti auth token: jika tidak ada jalur auth eksplisit yang dikonfigurasi, startup gateway diselesaikan ke mode token dan otomatis membuat satu, menyimpannya ke `gateway.auth.token`, sehingga **klien WS lokal harus mengautentikasi**. Ini memblokir proses lokal lain agar tidak memanggil Gateway.

    Jika Anda lebih suka jalur auth yang berbeda, Anda dapat secara eksplisit memilih mode kata sandi (atau, untuk reverse proxy sadar identitas non-loopback, `trusted-proxy`). Jika Anda **benar-benar** menginginkan loopback terbuka, setel `gateway.auth.mode: "none"` secara eksplisit di konfigurasi Anda. Doctor dapat membuat token untuk Anda kapan saja: `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="Apakah saya harus restart setelah mengubah konfigurasi?">
    Gateway memantau konfigurasi dan mendukung hot-reload:

    - `gateway.reload.mode: "hybrid"` (default): hot-apply perubahan yang aman, restart untuk yang kritis
    - `hot`, `restart`, `off` juga didukung

  </Accordion>

  <Accordion title="Bagaimana cara menonaktifkan tagline CLI yang lucu?">
    Setel `cli.banner.taglineMode` di konfigurasi:

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`: menyembunyikan teks tagline tetapi tetap mempertahankan baris judul/versi banner.
    - `default`: menggunakan `All your chats, one OpenClaw.` setiap saat.
    - `random`: tagline lucu/musiman yang bergilir (perilaku default).
    - Jika Anda tidak menginginkan banner sama sekali, setel env `OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="Bagaimana cara mengaktifkan web search (dan web fetch)?">
    `web_fetch` bekerja tanpa API key. `web_search` bergantung pada provider
    yang Anda pilih:

    - Provider berbasis API seperti Brave, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Perplexity, dan Tavily memerlukan penyiapan API key normal mereka.
    - Ollama Web Search tidak memerlukan key, tetapi menggunakan host Ollama yang Anda konfigurasi dan memerlukan `ollama signin`.
    - DuckDuckGo tidak memerlukan key, tetapi merupakan integrasi tidak resmi berbasis HTML.
    - SearXNG tidak memerlukan key/self-hosted; konfigurasikan `SEARXNG_BASE_URL` atau `plugins.entries.searxng.config.webSearch.baseUrl`.

    **Direkomendasikan:** jalankan `openclaw configure --section web` dan pilih provider.
    Alternatif environment:

    - Brave: `BRAVE_API_KEY`
    - Exa: `EXA_API_KEY`
    - Firecrawl: `FIRECRAWL_API_KEY`
    - Gemini: `GEMINI_API_KEY`
    - Grok: `XAI_API_KEY`
    - Kimi: `KIMI_API_KEY` atau `MOONSHOT_API_KEY`
    - MiniMax Search: `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, atau `MINIMAX_API_KEY`
    - Perplexity: `PERPLEXITY_API_KEY` atau `OPENROUTER_API_KEY`
    - SearXNG: `SEARXNG_BASE_URL`
    - Tavily: `TAVILY_API_KEY`

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
        },
        tools: {
          web: {
            search: {
              enabled: true,
              provider: "brave",
              maxResults: 5,
            },
            fetch: {
              enabled: true,
              provider: "firecrawl", // optional; omit for auto-detect
            },
          },
        },
    }
    ```

    Konfigurasi web-search khusus provider sekarang berada di bawah `plugins.entries.<plugin>.config.webSearch.*`.
    Jalur provider lama `tools.web.search.*` masih dimuat sementara untuk kompatibilitas, tetapi sebaiknya tidak digunakan untuk konfigurasi baru.
    Konfigurasi fallback web-fetch Firecrawl berada di bawah `plugins.entries.firecrawl.config.webFetch.*`.

    Catatan:

    - Jika Anda menggunakan allowlist, tambahkan `web_search`/`web_fetch`/`x_search` atau `group:web`.
    - `web_fetch` aktif secara default (kecuali dinonaktifkan secara eksplisit).
    - Jika `tools.web.fetch.provider` dihilangkan, OpenClaw otomatis mendeteksi provider fallback fetch pertama yang siap dari kredensial yang tersedia. Saat ini provider bawaan adalah Firecrawl.
    - Daemon membaca env vars dari `~/.openclaw/.env` (atau environment layanan).

    Dokumentasi: [Web tools](/id/tools/web).

  </Accordion>

  <Accordion title="config.apply menghapus konfigurasi saya. Bagaimana cara memulihkan dan menghindarinya?">
    `config.apply` menggantikan **seluruh konfigurasi**. Jika Anda mengirim objek parsial, semua
    yang lain akan dihapus.

    Pemulihan:

    - Pulihkan dari cadangan (git atau salinan `~/.openclaw/openclaw.json`).
    - Jika Anda tidak punya cadangan, jalankan ulang `openclaw doctor` dan konfigurasi ulang channel/model.
    - Jika ini tidak terduga, buat bug dan sertakan konfigurasi terakhir yang Anda ketahui atau cadangan apa pun.
    - Agen coding lokal sering kali dapat merekonstruksi konfigurasi yang berfungsi dari log atau riwayat.

    Cara menghindarinya:

    - Gunakan `openclaw config set` untuk perubahan kecil.
    - Gunakan `openclaw configure` untuk pengeditan interaktif.
    - Gunakan `config.schema.lookup` terlebih dahulu ketika Anda tidak yakin tentang path atau bentuk field yang tepat; ia mengembalikan node skema dangkal ditambah ringkasan child langsung untuk penelusuran bertahap.
    - Gunakan `config.patch` untuk pengeditan RPC parsial; simpan `config.apply` hanya untuk penggantian konfigurasi penuh.
    - Jika Anda menggunakan alat `gateway` khusus owner dari run agen, alat itu tetap akan menolak penulisan ke `tools.exec.ask` / `tools.exec.security` (termasuk alias lama `tools.bash.*` yang dinormalisasi ke path exec terlindungi yang sama).

    Dokumentasi: [Config](/cli/config), [Configure](/cli/configure), [Doctor](/id/gateway/doctor).

  </Accordion>

  <Accordion title="Bagaimana cara menjalankan Gateway pusat dengan worker khusus di berbagai perangkat?">
    Pola umumnya adalah **satu Gateway** (mis. Raspberry Pi) plus **nodes** dan **agents**:

    - **Gateway (pusat):** memiliki channel (Signal/WhatsApp), perutean, dan sesi.
    - **Nodes (perangkat):** Mac/iOS/Android terhubung sebagai periferal dan mengekspos alat lokal (`system.run`, `canvas`, `camera`).
    - **Agents (worker):** otak/workspace terpisah untuk peran khusus (mis. "Hetzner ops", "Data pribadi").
    - **Sub-agents:** spawn pekerjaan latar belakang dari agen utama saat Anda menginginkan paralelisme.
    - **TUI:** sambungkan ke Gateway dan ganti agen/sesi.

    Dokumentasi: [Nodes](/id/nodes), [Remote access](/id/gateway/remote), [Multi-Agent Routing](/id/concepts/multi-agent), [Sub-agents](/id/tools/subagents), [TUI](/web/tui).

  </Accordion>

  <Accordion title="Bisakah browser OpenClaw berjalan headless?">
    Ya. Itu opsi konfigurasi:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    Default adalah `false` (headful). Headless lebih mungkin memicu pemeriksaan anti-bot di beberapa situs. Lihat [Browser](/id/tools/browser).

    Headless menggunakan **engine Chromium yang sama** dan bekerja untuk sebagian besar otomasi (formulir, klik, scraping, login). Perbedaan utamanya:

    - Tidak ada jendela browser yang terlihat (gunakan tangkapan layar jika Anda memerlukan visual).
    - Beberapa situs lebih ketat terhadap otomasi dalam mode headless (CAPTCHA, anti-bot).
      Misalnya, X/Twitter sering memblokir sesi headless.

  </Accordion>

  <Accordion title="Bagaimana cara menggunakan Brave untuk kontrol browser?">
    Setel `browser.executablePath` ke biner Brave Anda (atau browser berbasis Chromium lain) dan restart Gateway.
    Lihat contoh konfigurasi lengkap di [Browser](/id/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Gateway remote dan nodes

<AccordionGroup>
  <Accordion title="Bagaimana perintah merambat antara Telegram, gateway, dan nodes?">
    Pesan Telegram ditangani oleh **gateway**. Gateway menjalankan agen dan
    baru kemudian memanggil nodes melalui **Gateway WebSocket** ketika alat node dibutuhkan:

    Telegram → Gateway → Agent → `node.*` → Node → Gateway → Telegram

    Nodes tidak melihat lalu lintas provider masuk; mereka hanya menerima panggilan RPC node.

  </Accordion>

  <Accordion title="Bagaimana agen saya bisa mengakses komputer saya jika Gateway di-host secara remote?">
    Jawaban singkat: **pasangkan komputer Anda sebagai node**. Gateway berjalan di tempat lain, tetapi dapat
    memanggil alat `node.*` (layar, kamera, sistem) di mesin lokal Anda melalui Gateway WebSocket.

    Penyiapan tipikal:

    1. Jalankan Gateway pada host yang selalu aktif (VPS/server rumahan).
    2. Masukkan gateway host + komputer Anda ke tailnet yang sama.
    3. Pastikan Gateway WS dapat dijangkau (bind tailnet atau SSH tunnel).
    4. Buka aplikasi macOS secara lokal dan hubungkan dalam mode **Remote over SSH** (atau tailnet langsung)
       agar ia dapat mendaftar sebagai node.
    5. Setujui node di Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Tidak diperlukan bridge TCP terpisah; nodes terhubung melalui Gateway WebSocket.

    Pengingat keamanan: memasangkan node macOS memungkinkan `system.run` di mesin itu. Hanya
    pasangkan perangkat yang Anda percayai, dan tinjau [Security](/id/gateway/security).

    Dokumentasi: [Nodes](/id/nodes), [Gateway protocol](/id/gateway/protocol), [macOS remote mode](/id/platforms/mac/remote), [Security](/id/gateway/security).

  </Accordion>

  <Accordion title="Tailscale terhubung tetapi saya tidak mendapat balasan. Sekarang apa?">
    Periksa dasar-dasarnya:

    - Gateway berjalan: `openclaw gateway status`
    - Kesehatan Gateway: `openclaw status`
    - Kesehatan channel: `openclaw channels status`

    Lalu verifikasi auth dan perutean:

    - Jika Anda menggunakan Tailscale Serve, pastikan `gateway.auth.allowTailscale` disetel dengan benar.
    - Jika Anda terhubung melalui SSH tunnel, pastikan tunnel lokal aktif dan menunjuk ke port yang benar.
    - Pastikan allowlist Anda (DM atau grup) mencakup akun Anda.

    Dokumentasi: [Tailscale](/id/gateway/tailscale), [Remote access](/id/gateway/remote), [Channels](/id/channels).

  </Accordion>

  <Accordion title="Bisakah dua instans OpenClaw berbicara satu sama lain (lokal + VPS)?">
    Ya. Tidak ada bridge "bot-to-bot" bawaan, tetapi Anda dapat menghubungkannya dengan beberapa
    cara yang andal:

    **Paling sederhana:** gunakan channel obrolan normal yang bisa diakses kedua bot (Telegram/Slack/WhatsApp).
    Minta Bot A mengirim pesan ke Bot B, lalu biarkan Bot B membalas seperti biasa.

    **CLI bridge (generik):** jalankan skrip yang memanggil Gateway lain dengan
    `openclaw agent --message ... --deliver`, menargetkan obrolan tempat bot lain
    mendengarkan. Jika satu bot berada di VPS remote, arahkan CLI Anda ke Gateway remote itu
    melalui SSH/Tailscale (lihat [Remote access](/id/gateway/remote)).

    Pola contoh (jalankan dari mesin yang dapat menjangkau Gateway target):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Tip: tambahkan guardrail agar kedua bot tidak berputar tanpa akhir (khusus-mention, channel
    allowlist, atau aturan "jangan balas pesan bot").

    Dokumentasi: [Remote access](/id/gateway/remote), [Agent CLI](/cli/agent), [Agent send](/id/tools/agent-send).

  </Accordion>

  <Accordion title="Apakah saya memerlukan VPS terpisah untuk beberapa agen?">
    Tidak. Satu Gateway dapat meng-host banyak agen, masing-masing dengan workspace, default model,
    dan peruteannya sendiri. Itu adalah penyiapan normal dan jauh lebih murah serta lebih sederhana daripada menjalankan
    satu VPS per agen.

    Gunakan VPS terpisah hanya ketika Anda memerlukan isolasi keras (batas keamanan) atau konfigurasi yang sangat
    berbeda yang tidak ingin Anda bagikan. Jika tidak, tetap gunakan satu Gateway dan
    gunakan beberapa agen atau sub-agents.

  </Accordion>

  <Accordion title="Apakah ada manfaat menggunakan node di laptop pribadi saya daripada SSH dari VPS?">
    Ya - nodes adalah cara kelas satu untuk menjangkau laptop Anda dari Gateway remote, dan mereka
    membuka lebih dari sekadar akses shell. Gateway berjalan di macOS/Linux (Windows melalui WSL2) dan
    ringan (VPS kecil atau perangkat kelas Raspberry Pi sudah cukup; RAM 4 GB sangat memadai), jadi penyiapan
    umum adalah host yang selalu aktif plus laptop Anda sebagai node.

    - **Tidak memerlukan SSH masuk.** Nodes terhubung keluar ke Gateway WebSocket dan menggunakan pairing perangkat.
    - **Kontrol eksekusi lebih aman.** `system.run` dibatasi oleh allowlist/persetujuan node di laptop itu.
    - **Lebih banyak alat perangkat.** Nodes mengekspos `canvas`, `camera`, dan `screen` selain `system.run`.
    - **Otomasi browser lokal.** Simpan Gateway di VPS, tetapi jalankan Chrome secara lokal melalui host node di laptop, atau lampirkan ke Chrome lokal pada host melalui Chrome MCP.

    SSH baik untuk akses shell sesekali, tetapi nodes lebih sederhana untuk alur kerja agen yang berkelanjutan dan
    otomasi perangkat.

    Dokumentasi: [Nodes](/id/nodes), [Nodes CLI](/cli/nodes), [Browser](/id/tools/browser).

  </Accordion>

  <Accordion title="Apakah nodes menjalankan layanan gateway?">
    Tidak. Hanya **satu gateway** yang seharusnya berjalan per host kecuali Anda sengaja menjalankan profil terisolasi (lihat [Multiple gateways](/id/gateway/multiple-gateways)). Nodes adalah periferal yang terhubung
    ke gateway (node iOS/Android, atau "mode node" macOS di aplikasi menubar). Untuk host node headless
    dan kontrol CLI, lihat [Node host CLI](/cli/node).

    Restart penuh diperlukan untuk perubahan `gateway`, `discovery`, dan `canvasHost`.

  </Accordion>

  <Accordion title="Apakah ada cara API / RPC untuk menerapkan konfigurasi?">
    Ya.

    - `config.schema.lookup`: periksa satu subtree konfigurasi dengan node skema dangkal, petunjuk UI yang cocok, dan ringkasan child langsung sebelum menulis
    - `config.get`: ambil snapshot + hash saat ini
    - `config.patch`: pembaruan parsial yang aman (lebih disukai untuk sebagian besar pengeditan RPC)
    - `config.apply`: validasi + ganti konfigurasi penuh, lalu restart
    - Alat runtime `gateway` khusus owner tetap menolak penulisan ulang `tools.exec.ask` / `tools.exec.security`; alias lama `tools.bash.*` dinormalisasi ke path exec terlindungi yang sama

  </Accordion>

  <Accordion title="Konfigurasi wajar minimal untuk instalasi pertama">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Ini menetapkan workspace Anda dan membatasi siapa yang dapat memicu bot.

  </Accordion>

  <Accordion title="Bagaimana cara menyiapkan Tailscale di VPS dan terhubung dari Mac saya?">
    Langkah minimal:

    1. **Instal + login di VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **Instal + login di Mac Anda**
       - Gunakan aplikasi Tailscale dan masuk ke tailnet yang sama.
    3. **Aktifkan MagicDNS (disarankan)**
       - Di konsol admin Tailscale, aktifkan MagicDNS agar VPS memiliki nama yang stabil.
    4. **Gunakan hostname tailnet**
       - SSH: `ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS: `ws://your-vps.tailnet-xxxx.ts.net:18789`

    Jika Anda menginginkan Control UI tanpa SSH, gunakan Tailscale Serve di VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Ini menjaga gateway tetap terikat ke loopback dan mengekspos HTTPS melalui Tailscale. Lihat [Tailscale](/id/gateway/tailscale).

  </Accordion>

  <Accordion title="Bagaimana cara menghubungkan node Mac ke Gateway remote (Tailscale Serve)?">
    Serve mengekspos **Gateway Control UI + WS**. Nodes terhubung melalui endpoint Gateway WS yang sama.

    Penyiapan yang direkomendasikan:

    1. **Pastikan VPS + Mac berada di tailnet yang sama**.
    2. **Gunakan aplikasi macOS dalam mode Remote** (target SSH bisa berupa hostname tailnet).
       Aplikasi akan menunnel port Gateway dan terhubung sebagai node.
    3. **Setujui node** di gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Dokumentasi: [Gateway protocol](/id/gateway/protocol), [Discovery](/id/gateway/discovery), [macOS remote mode](/id/platforms/mac/remote).

  </Accordion>

  <Accordion title="Sebaiknya saya instal di laptop kedua atau cukup tambahkan node?">
    Jika Anda hanya membutuhkan **alat lokal** (layar/kamera/exec) di laptop kedua, tambahkan saja sebagai
    **node**. Itu menjaga satu Gateway dan menghindari duplikasi konfigurasi. Alat node lokal
    saat ini hanya macOS, tetapi kami berencana memperluasnya ke OS lain.

    Instal Gateway kedua hanya ketika Anda memerlukan **isolasi keras** atau dua bot yang benar-benar terpisah.

    Dokumentasi: [Nodes](/id/nodes), [Nodes CLI](/cli/nodes), [Multiple gateways](/id/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Env vars dan pemuatan .env

<AccordionGroup>
  <Accordion title="Bagaimana OpenClaw memuat environment variables?">
    OpenClaw membaca env vars dari proses induk (shell, launchd/systemd, CI, dll.) dan tambahan memuat:

    - `.env` dari direktori kerja saat ini
    - `.env` fallback global dari `~/.openclaw/.env` (alias `$OPENCLAW_STATE_DIR/.env`)

    Kedua file `.env` tidak menimpa env vars yang sudah ada.

    Anda juga dapat mendefinisikan env vars inline di konfigurasi (hanya diterapkan jika tidak ada dari process env):

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Lihat [/environment](/id/help/environment) untuk prioritas dan sumber lengkap.

  </Accordion>

  <Accordion title="Saya memulai Gateway melalui layanan dan env vars saya hilang. Sekarang apa?">
    Dua perbaikan umum:

    1. Letakkan key yang hilang di `~/.openclaw/.env` agar tetap diambil meski layanan tidak mewarisi env shell Anda.
    2. Aktifkan shell import (kemudahan opsional):

    ```json5
    {
      env: {
        shellEnv: {
          enabled: true,
          timeoutMs: 15000,
        },
      },
    }
    ```

    Ini menjalankan login shell Anda dan hanya mengimpor key yang diharapkan yang belum ada (tidak pernah menimpa). Ekuivalen env var:
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='Saya menyetel COPILOT_GITHUB_TOKEN, tetapi models status menampilkan "Shell env: off." Mengapa?'>
    `openclaw models status` melaporkan apakah **shell env import** diaktifkan. "Shell env: off"
    **tidak** berarti env vars Anda hilang - itu hanya berarti OpenClaw tidak akan memuat
    login shell Anda secara otomatis.

    Jika Gateway berjalan sebagai layanan (launchd/systemd), ia tidak akan mewarisi
    environment shell Anda. Perbaiki dengan salah satu cara berikut:

    1. Letakkan token di `~/.openclaw/.env`:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. Atau aktifkan shell import (`env.shellEnv.enabled: true`).
    3. Atau tambahkan ke blok `env` konfigurasi Anda (hanya berlaku jika belum ada).

    Lalu restart gateway dan periksa lagi:

    ```bash
    openclaw models status
    ```

    Token Copilot dibaca dari `COPILOT_GITHUB_TOKEN` (juga `GH_TOKEN` / `GITHUB_TOKEN`).
    Lihat [/concepts/model-providers](/id/concepts/model-providers) dan [/environment](/id/help/environment).

  </Accordion>
</AccordionGroup>

## Sesi dan beberapa obrolan

<AccordionGroup>
  <Accordion title="Bagaimana cara memulai percakapan baru?">
    Kirim `/new` atau `/reset` sebagai pesan mandiri. Lihat [Session management](/id/concepts/session).
  </Accordion>

  <Accordion title="Apakah sesi di-reset otomatis jika saya tidak pernah mengirim /new?">
    Sesi dapat kedaluwarsa setelah `session.idleMinutes`, tetapi ini **nonaktif secara default** (default **0**).
    Setel ke nilai positif untuk mengaktifkan kedaluwarsa idle. Saat diaktifkan, pesan **berikutnya**
    setelah periode idle memulai id sesi baru untuk kunci obrolan itu.
    Ini tidak menghapus transkrip - hanya memulai sesi baru.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="Apakah ada cara membuat tim instans OpenClaw (satu CEO dan banyak agen)?">
    Ya, melalui **multi-agent routing** dan **sub-agents**. Anda dapat membuat satu agen
    koordinator dan beberapa agen pekerja dengan workspace dan model mereka sendiri.

    Meski begitu, ini sebaiknya dipandang sebagai **eksperimen yang menyenangkan**. Ini berat token dan sering
    kurang efisien dibanding menggunakan satu bot dengan sesi terpisah. Model tipikal yang kami
    bayangkan adalah satu bot yang Anda ajak bicara, dengan sesi berbeda untuk pekerjaan paralel. Bot itu
    juga dapat men-spawn sub-agents saat dibutuhkan.

    Dokumentasi: [Multi-agent routing](/id/concepts/multi-agent), [Sub-agents](/id/tools/subagents), [Agents CLI](/cli/agents).

  </Accordion>

  <Accordion title="Mengapa konteks terpotong di tengah tugas? Bagaimana cara mencegahnya?">
    Konteks sesi dibatasi oleh jendela model. Obrolan panjang, output alat yang besar, atau banyak
    file dapat memicu compaction atau pemotongan.

    Yang membantu:

    - Minta bot merangkum state saat ini dan menuliskannya ke file.
    - Gunakan `/compact` sebelum tugas panjang, dan `/new` saat berganti topik.
    - Simpan konteks penting di workspace dan minta bot membacanya kembali.
    - Gunakan sub-agents untuk pekerjaan panjang atau paralel agar obrolan utama tetap lebih kecil.
    - Pilih model dengan jendela konteks yang lebih besar jika ini sering terjadi.

  </Accordion>

  <Accordion title="Bagaimana cara mereset OpenClaw sepenuhnya tetapi tetap terinstal?">
    Gunakan perintah reset:

    ```bash
    openclaw reset
    ```

    Reset penuh non-interaktif:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Lalu jalankan ulang penyiapan:

    ```bash
    openclaw onboard --install-daemon
    ```

    Catatan:

    - Onboarding juga menawarkan **Reset** jika mendeteksi konfigurasi yang ada. Lihat [Onboarding (CLI)](/id/start/wizard).
    - Jika Anda menggunakan profil (`--profile` / `OPENCLAW_PROFILE`), reset setiap direktori state (defaultnya `~/.openclaw-<profile>`).
    - Reset dev: `openclaw gateway --dev --reset` (khusus dev; menghapus konfigurasi dev + kredensial + sesi + workspace).

  </Accordion>

  <Accordion title='Saya mendapatkan kesalahan "context too large" - bagaimana cara reset atau compact?'>
    Gunakan salah satu berikut:

    - **Compact** (menjaga percakapan tetapi merangkum giliran lama):

      ```
      /compact
      ```

      atau `/compact <instructions>` untuk memandu ringkasan.

    - **Reset** (id sesi baru untuk kunci obrolan yang sama):

      ```
      /new
      /reset
      ```

    Jika terus terjadi:

    - Aktifkan atau sesuaikan **session pruning** (`agents.defaults.contextPruning`) untuk memangkas output alat lama.
    - Gunakan model dengan jendela konteks yang lebih besar.

    Dokumentasi: [Compaction](/id/concepts/compaction), [Session pruning](/id/concepts/session-pruning), [Session management](/id/concepts/session).

  </Accordion>

  <Accordion title='Mengapa saya melihat "LLM request rejected: messages.content.tool_use.input field required"?'>
    Ini adalah kesalahan validasi provider: model mengeluarkan blok `tool_use` tanpa
    `input` yang diwajibkan. Biasanya ini berarti riwayat sesi stale atau rusak (sering setelah thread panjang
    atau perubahan alat/skema).

    Perbaikan: mulai sesi baru dengan `/new` (pesan mandiri).

  </Accordion>

  <Accordion title="Mengapa saya menerima pesan heartbeat setiap 30 menit?">
    Heartbeat berjalan setiap **30m** secara default (**1h** saat menggunakan auth OAuth). Sesuaikan atau nonaktifkan:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // or "0m" to disable
          },
        },
      },
    }
    ```

    Jika `HEARTBEAT.md` ada tetapi pada dasarnya kosong (hanya baris kosong dan
    header markdown seperti `# Heading`), OpenClaw melewati run heartbeat untuk menghemat panggilan API.
    Jika file tidak ada, heartbeat tetap berjalan dan model memutuskan apa yang harus dilakukan.

    Override per agen menggunakan `agents.list[].heartbeat`. Dokumentasi: [Heartbeat](/id/gateway/heartbeat).

  </Accordion>

  <Accordion title='Apakah saya perlu menambahkan "akun bot" ke grup WhatsApp?'>
    Tidak. OpenClaw berjalan di **akun Anda sendiri**, jadi jika Anda berada di grup, OpenClaw bisa melihatnya.
    Secara default, balasan grup diblokir sampai Anda mengizinkan pengirim (`groupPolicy: "allowlist"`).

    Jika Anda ingin hanya **Anda** yang bisa memicu balasan grup:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Bagaimana cara mendapatkan JID grup WhatsApp?">
    Opsi 1 (tercepat): ikuti log dan kirim pesan uji di grup:

    ```bash
    openclaw logs --follow --json
    ```

    Cari `chatId` (atau `from`) yang berakhir dengan `@g.us`, seperti:
    `1234567890-1234567890@g.us`.

    Opsi 2 (jika sudah dikonfigurasi/diallowlist): daftarkan grup dari konfigurasi:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Dokumentasi: [WhatsApp](/id/channels/whatsapp), [Directory](/cli/directory), [Logs](/cli/logs).

  </Accordion>

  <Accordion title="Mengapa OpenClaw tidak membalas di grup?">
    Dua penyebab umum:

    - Penyaringan mention aktif (default). Anda harus @mention bot (atau cocok dengan `mentionPatterns`).
    - Anda mengonfigurasi `channels.whatsapp.groups` tanpa `"*"` dan grup itu tidak ada dalam allowlist.

    Lihat [Groups](/id/channels/groups) dan [Group messages](/id/channels/group-messages).

  </Accordion>

  <Accordion title="Apakah grup/thread berbagi konteks dengan DM?">
    Obrolan langsung runtuh ke sesi utama secara default. Grup/channel memiliki kunci sesi mereka sendiri, dan topik Telegram / thread Discord adalah sesi terpisah. Lihat [Groups](/id/channels/groups) dan [Group messages](/id/channels/group-messages).
  </Accordion>

  <Accordion title="Berapa banyak workspace dan agen yang bisa saya buat?">
    Tidak ada batas keras. Puluhan (bahkan ratusan) baik-baik saja, tetapi perhatikan:

    - **Pertumbuhan disk:** sesi + transkrip hidup di bawah `~/.openclaw/agents/<agentId>/sessions/`.
    - **Biaya token:** lebih banyak agen berarti lebih banyak penggunaan model bersamaan.
    - **Beban operasional:** profil auth, workspace, dan perutean channel per agen.

    Tips:

    - Pertahankan satu workspace **aktif** per agen (`agents.defaults.workspace`).
    - Pangkas sesi lama (hapus JSONL atau entri store) jika disk membesar.
    - Gunakan `openclaw doctor` untuk menemukan workspace liar dan ketidakcocokan profil.

  </Accordion>

  <Accordion title="Bisakah saya menjalankan beberapa bot atau obrolan pada saat yang sama (Slack), dan bagaimana saya harus menyiapkannya?">
    Ya. Gunakan **Multi-Agent Routing** untuk menjalankan beberapa agen terisolasi dan mengarahkan pesan masuk berdasarkan
    channel/akun/peer. Slack didukung sebagai channel dan dapat diikat ke agen tertentu.

    Akses browser sangat kuat tetapi tidak berarti "bisa melakukan apa pun yang bisa dilakukan manusia" - anti-bot, CAPTCHA, dan MFA tetap
    dapat memblokir otomasi. Untuk kontrol browser yang paling andal, gunakan Chrome MCP lokal di host,
    atau gunakan CDP pada mesin yang benar-benar menjalankan browser.

    Penyiapan praktik terbaik:

    - Gateway host yang selalu aktif (VPS/Mac mini).
    - Satu agen per peran (bindings).
    - Channel Slack terikat ke agen-agen tersebut.
    - Browser lokal melalui Chrome MCP atau node bila diperlukan.

    Dokumentasi: [Multi-Agent Routing](/id/concepts/multi-agent), [Slack](/id/channels/slack),
    [Browser](/id/tools/browser), [Nodes](/id/nodes).

  </Accordion>
</AccordionGroup>

## Models: default, pemilihan, alias, peralihan

<AccordionGroup>
  <Accordion title='Apa itu "default model"?'>
    Default model OpenClaw adalah apa pun yang Anda setel sebagai:

    ```
    agents.defaults.model.primary
    ```

    Model direferensikan sebagai `provider/model` (contoh: `openai/gpt-5.4`). Jika Anda menghilangkan provider, OpenClaw terlebih dahulu mencoba alias, lalu kecocokan provider-terkonfigurasi yang unik untuk id model yang persis sama, dan hanya kemudian fallback ke provider default yang dikonfigurasi sebagai jalur kompatibilitas usang. Jika provider itu tidak lagi mengekspos model default yang dikonfigurasi, OpenClaw fallback ke provider/model terkonfigurasi pertama alih-alih menampilkan default provider lama yang sudah dihapus. Anda tetap sebaiknya **secara eksplisit** menyetel `provider/model`.

  </Accordion>

  <Accordion title="Model apa yang Anda rekomendasikan?">
    **Default yang direkomendasikan:** gunakan model generasi terbaru terkuat yang tersedia di stack provider Anda.
    **Untuk agen dengan alat aktif atau input tidak tepercaya:** prioritaskan kekuatan model di atas biaya.
    **Untuk obrolan rutin/berisiko rendah:** gunakan model fallback yang lebih murah dan rute berdasarkan peran agen.

    MiniMax punya dokumentasi sendiri: [MiniMax](/id/providers/minimax) dan
    [Local models](/id/gateway/local-models).

    Aturan praktis: gunakan **model terbaik yang mampu Anda bayar** untuk pekerjaan berisiko tinggi, dan model yang lebih murah
    untuk obrolan rutin atau ringkasan. Anda dapat merutekan model per agen dan menggunakan sub-agents untuk
    memparalelkan tugas panjang (setiap sub-agent mengonsumsi token). Lihat [Models](/id/concepts/models) dan
    [Sub-agents](/id/tools/subagents).

    Peringatan keras: model yang lebih lemah/terlalu terkuantisasi lebih rentan terhadap prompt
    injection dan perilaku tidak aman. Lihat [Security](/id/gateway/security).

    Konteks lebih lanjut: [Models](/id/concepts/models).

  </Accordion>

  <Accordion title="Bagaimana cara mengganti model tanpa menghapus konfigurasi saya?">
    Gunakan **perintah model** atau edit hanya field **model**. Hindari penggantian konfigurasi penuh.

    Opsi aman:

    - `/model` di obrolan (cepat, per sesi)
    - `openclaw models set ...` (hanya memperbarui konfigurasi model)
    - `openclaw configure --section model` (interaktif)
    - edit `agents.defaults.model` di `~/.openclaw/openclaw.json`

    Hindari `config.apply` dengan objek parsial kecuali Anda memang berniat mengganti seluruh konfigurasi.
    Untuk pengeditan RPC, periksa dengan `config.schema.lookup` terlebih dahulu dan pilih `config.patch`. Payload lookup memberi Anda path yang dinormalisasi, dokumen/kendala skema dangkal, dan ringkasan child langsung
    untuk pembaruan parsial.
    Jika Anda memang menimpa konfigurasi, pulihkan dari cadangan atau jalankan ulang `openclaw doctor` untuk memperbaikinya.

    Dokumentasi: [Models](/id/concepts/models), [Configure](/cli/configure), [Config](/cli/config), [Doctor](/id/gateway/doctor).

  </Accordion>

  <Accordion title="Bisakah saya menggunakan model self-hosted (llama.cpp, vLLM, Ollama)?">
    Ya. Ollama adalah jalur termudah untuk model lokal.

    Penyiapan tercepat:

    1. Instal Ollama dari `https://ollama.com/download`
    2. Pull model lokal seperti `ollama pull glm-4.7-flash`
    3. Jika Anda juga menginginkan model cloud, jalankan `ollama signin`
    4. Jalankan `openclaw onboard` dan pilih `Ollama`
    5. Pilih `Local` atau `Cloud + Local`

    Catatan:

    - `Cloud + Local` memberi Anda model cloud ditambah model Ollama lokal Anda
    - model cloud seperti `kimi-k2.5:cloud` tidak memerlukan pull lokal
    - untuk peralihan manual, gunakan `openclaw models list` dan `openclaw models set ollama/<model>`

    Catatan keamanan: model yang lebih kecil atau sangat terkuantisasi lebih rentan terhadap prompt
    injection. Kami sangat merekomendasikan **model besar** untuk bot apa pun yang dapat menggunakan alat.
    Jika Anda tetap menginginkan model kecil, aktifkan sandboxing dan allowlist alat yang ketat.

    Dokumentasi: [Ollama](/id/providers/ollama), [Local models](/id/gateway/local-models),
    [Model providers](/id/concepts/model-providers), [Security](/id/gateway/security),
    [Sandboxing](/id/gateway/sandboxing).

  </Accordion>

  <Accordion title="Model apa yang digunakan OpenClaw, Flawd, dan Krill?">
    - Deployment ini dapat berbeda dan dapat berubah seiring waktu; tidak ada rekomendasi provider yang tetap.
    - Periksa pengaturan runtime saat ini di setiap gateway dengan `openclaw models status`.
    - Untuk agen yang sensitif keamanan/menggunakan alat, gunakan model generasi terbaru terkuat yang tersedia.
  </Accordion>

  <Accordion title="Bagaimana cara mengganti model secara langsung (tanpa restart)?">
    Gunakan perintah `/model` sebagai pesan mandiri:

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    Ini adalah alias bawaan. Alias kustom dapat ditambahkan melalui `agents.defaults.models`.

    Anda dapat mendaftar model yang tersedia dengan `/model`, `/model list`, atau `/model status`.

    `/model` (dan `/model list`) menampilkan pemilih bernomor yang ringkas. Pilih berdasarkan nomor:

    ```
    /model 3
    ```

    Anda juga dapat memaksa profil auth tertentu untuk provider (per sesi):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Tip: `/model status` menampilkan agen mana yang aktif, file `auth-profiles.json` mana yang sedang digunakan, dan profil auth mana yang akan dicoba berikutnya.
    Ini juga menampilkan endpoint provider yang dikonfigurasi (`baseUrl`) dan mode API (`api`) saat tersedia.

    **Bagaimana cara melepaskan pin profil yang saya setel dengan @profile?**

    Jalankan ulang `/model` **tanpa** sufiks `@profile`:

    ```
    /model anthropic/claude-opus-4-6
    ```

    Jika Anda ingin kembali ke default, pilih dari `/model` (atau kirim `/model <default provider/model>`).
    Gunakan `/model status` untuk mengonfirmasi profil auth mana yang aktif.

  </Accordion>

  <Accordion title="Bisakah saya menggunakan GPT 5.2 untuk tugas harian dan Codex 5.3 untuk coding?">
    Ya. Setel satu sebagai default dan ganti sesuai kebutuhan:

    - **Peralihan cepat (per sesi):** `/model gpt-5.4` untuk tugas harian, `/model openai-codex/gpt-5.4` untuk coding dengan Codex OAuth.
    - **Default + peralihan:** setel `agents.defaults.model.primary` ke `openai/gpt-5.4`, lalu ganti ke `openai-codex/gpt-5.4` saat coding (atau sebaliknya).
    - **Sub-agents:** rute tugas coding ke sub-agents dengan default model berbeda.

    Lihat [Models](/id/concepts/models) dan [Slash commands](/id/tools/slash-commands).

  </Accordion>

  <Accordion title="Bagaimana cara mengonfigurasi fast mode untuk GPT 5.4?">
    Gunakan toggle sesi atau default konfigurasi:

    - **Per sesi:** kirim `/fast on` saat sesi menggunakan `openai/gpt-5.4` atau `openai-codex/gpt-5.4`.
    - **Default per model:** setel `agents.defaults.models["openai/gpt-5.4"].params.fastMode` ke `true`.
    - **Codex OAuth juga:** jika Anda juga menggunakan `openai-codex/gpt-5.4`, setel flag yang sama di sana.

    Contoh:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
            "openai-codex/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    Untuk OpenAI, fast mode dipetakan ke `service_tier = "priority"` pada permintaan native Responses yang didukung. Override `/fast` sesi mengalahkan default konfigurasi.

    Lihat [Thinking and fast mode](/id/tools/thinking) dan [OpenAI fast mode](/id/providers/openai#openai-fast-mode).

  </Accordion>

  <Accordion title='Mengapa saya melihat "Model ... is not allowed" lalu tidak ada balasan?'>
    Jika `agents.defaults.models` disetel, itu menjadi **allowlist** untuk `/model` dan semua
    override sesi. Memilih model yang tidak ada dalam daftar itu akan mengembalikan:

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    Kesalahan itu dikembalikan **sebagai ganti** balasan normal. Perbaikan: tambahkan model ke
    `agents.defaults.models`, hapus allowlist, atau pilih model dari `/model list`.

  </Accordion>

  <Accordion title='Mengapa saya melihat "Unknown model: minimax/MiniMax-M2.7"?'>
    Ini berarti **provider belum dikonfigurasi** (tidak ada konfigurasi provider MiniMax atau profil
    auth ditemukan), jadi model tidak dapat diselesaikan.

    Daftar periksa perbaikan:

    1. Upgrade ke rilis OpenClaw saat ini (atau jalankan dari source `main`), lalu restart gateway.
    2. Pastikan MiniMax dikonfigurasi (wizard atau JSON), atau auth MiniMax
       ada di env/profil auth sehingga provider yang cocok dapat disuntikkan
       (`MINIMAX_API_KEY` untuk `minimax`, `MINIMAX_OAUTH_TOKEN` atau MiniMax
       OAuth tersimpan untuk `minimax-portal`).
    3. Gunakan id model yang persis (peka huruf besar/kecil) untuk jalur auth Anda:
       `minimax/MiniMax-M2.7` atau `minimax/MiniMax-M2.7-highspeed` untuk
       penyiapan API-key, atau `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` untuk penyiapan OAuth.
    4. Jalankan:

       ```bash
       openclaw models list
       ```

       dan pilih dari daftar (atau `/model list` di obrolan).

    Lihat [MiniMax](/id/providers/minimax) dan [Models](/id/concepts/models).

  </Accordion>

  <Accordion title="Bisakah saya menggunakan MiniMax sebagai default dan OpenAI untuk tugas kompleks?">
    Ya. Gunakan **MiniMax sebagai default** dan ganti model **per sesi** bila diperlukan.
    Fallback untuk **kesalahan**, bukan "tugas sulit", jadi gunakan `/model` atau agen terpisah.

    **Opsi A: ganti per sesi**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M2.7" },
          models: {
            "minimax/MiniMax-M2.7": { alias: "minimax" },
            "openai/gpt-5.4": { alias: "gpt" },
          },
        },
      },
    }
    ```

    Lalu:

    ```
    /model gpt
    ```

    **Opsi B: agen terpisah**

    - Agen A default: MiniMax
    - Agen B default: OpenAI
    - Rute berdasarkan agen atau gunakan `/agent` untuk berpindah

    Dokumentasi: [Models](/id/concepts/models), [Multi-Agent Routing](/id/concepts/multi-agent), [MiniMax](/id/providers/minimax), [OpenAI](/id/providers/openai).

  </Accordion>

  <Accordion title="Apakah opus / sonnet / gpt adalah pintasan bawaan?">
    Ya. OpenClaw menyertakan beberapa singkatan default (hanya diterapkan ketika model ada di `agents.defaults.models`):

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3-flash-preview`
    - `gemini-flash-lite` → `google/gemini-3.1-flash-lite-preview`

    Jika Anda menyetel alias Anda sendiri dengan nama yang sama, nilai Anda yang menang.

  </Accordion>

  <Accordion title="Bagaimana cara mendefinisikan/menimpa pintasan model (alias)?">
    Alias berasal dari `agents.defaults.models.<modelId>.alias`. Contoh:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
            "anthropic/claude-haiku-4-5": { alias: "haiku" },
          },
        },
      },
    }
    ```

    Lalu `/model sonnet` (atau `/<alias>` bila didukung) akan diselesaikan ke id model tersebut.

  </Accordion>

  <Accordion title="Bagaimana cara menambahkan model dari provider lain seperti OpenRouter atau Z.AI?">
    OpenRouter (bayar per token; banyak model):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (model GLM):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5" },
          models: { "zai/glm-5": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    Jika Anda mereferensikan provider/model tetapi key provider yang diperlukan hilang, Anda akan mendapatkan kesalahan auth runtime (misalnya `No API key found for provider "zai"`).

    **No API key found for provider setelah menambahkan agen baru**

    Ini biasanya berarti **agen baru** memiliki penyimpanan auth kosong. Auth bersifat per agen dan
    disimpan di:

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Opsi perbaikan:

    - Jalankan `openclaw agents add <id>` dan konfigurasi auth saat wizard.
    - Atau salin `auth-profiles.json` dari `agentDir` agen utama ke `agentDir` agen baru.

    **Jangan** gunakan ulang `agentDir` antar agen; itu menyebabkan tabrakan auth/sesi.

  </Accordion>
</AccordionGroup>

## Failover model dan "All models failed"

<AccordionGroup>
  <Accordion title="Bagaimana cara kerja failover?">
    Failover terjadi dalam dua tahap:

    1. **Rotasi profil auth** di dalam provider yang sama.
    2. **Fallback model** ke model berikutnya di `agents.defaults.model.fallbacks`.

    Cooldown berlaku untuk profil yang gagal (exponential backoff), sehingga OpenClaw dapat terus membalas meski provider dibatasi laju atau sementara gagal.

    Bucket rate-limit mencakup lebih dari sekadar respons `429` biasa. OpenClaw
    juga memperlakukan pesan seperti `Too many concurrent requests`,
    `ThrottlingException`, `concurrency limit reached`,
    `workers_ai ... quota limit exceeded`, `resource exhausted`, dan batas
    jendela penggunaan periodik (`weekly/monthly limit reached`) sebagai rate limit
    yang layak failover.

    Beberapa respons yang terlihat seperti penagihan bukan `402`, dan beberapa respons HTTP `402`
    juga tetap berada di bucket transien itu. Jika provider mengembalikan
    teks penagihan eksplisit pada `401` atau `403`, OpenClaw masih dapat menyimpannya di
    jalur penagihan, tetapi pencocok teks khusus provider tetap dibatasi pada
    provider yang memilikinya (misalnya OpenRouter `Key limit exceeded`). Jika pesan `402`
    malah terlihat seperti batas jendela penggunaan yang dapat dicoba ulang atau
    batas pengeluaran organisasi/workspace (`daily limit reached, resets tomorrow`,
    `organization spending limit exceeded`), OpenClaw memperlakukannya sebagai
    `rate_limit`, bukan penonaktifan penagihan jangka panjang.

    Kesalahan luapan konteks berbeda: signature seperti
    `request_too_large`, `input exceeds the maximum number of tokens`,
    `input token count exceeds the maximum number of input tokens`,
    `input is too long for the model`, atau `ollama error: context length
    exceeded` tetap berada pada jalur compaction/retry alih-alih memajukan model
    fallback.

    Teks generic server-error sengaja dibuat lebih sempit daripada "apa pun yang
    mengandung unknown/error". OpenClaw memang memperlakukan bentuk transien
    terbatas-provider seperti Anthropic bare `An unknown error occurred`, OpenRouter bare
    `Provider returned error`, kesalahan stop-reason seperti `Unhandled stop reason:
    error`, payload JSON `api_error` dengan teks server transien
    (`internal server error`, `unknown error, 520`, `upstream error`, `backend
    error`), dan kesalahan provider-sibuk seperti `ModelNotReadyException` sebagai
    sinyal timeout/overloaded yang layak failover ketika konteks provider
    cocok.
    Teks fallback internal generik seperti `LLM request failed with an unknown
    error.` tetap konservatif dan tidak memicu model fallback dengan sendirinya.

  </Accordion>

  <Accordion title='Apa arti "No credentials found for profile anthropic:default"?'>
    Artinya sistem mencoba menggunakan ID profil auth `anthropic:default`, tetapi tidak dapat menemukan kredensial untuknya di penyimpanan auth yang diharapkan.

    **Daftar periksa perbaikan:**

    - **Konfirmasi lokasi profil auth** (jalur baru vs lama)
      - Saat ini: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
      - Lama: `~/.openclaw/agent/*` (dimigrasikan oleh `openclaw doctor`)
    - **Konfirmasi env var Anda dimuat oleh Gateway**
      - Jika Anda menyetel `ANTHROPIC_API_KEY` di shell tetapi menjalankan Gateway melalui systemd/launchd, ia mungkin tidak mewarisinya. Letakkan di `~/.openclaw/.env` atau aktifkan `env.shellEnv`.
    - **Pastikan Anda mengedit agen yang benar**
      - Penyiapan multi-agent berarti bisa ada beberapa file `auth-profiles.json`.
    - **Periksa kewarasan status model/auth**
      - Gunakan `openclaw models status` untuk melihat model yang dikonfigurasi dan apakah provider terautentikasi.

    **Daftar periksa perbaikan untuk "No credentials found for profile anthropic"**

    Ini berarti run dipin ke profil auth Anthropic, tetapi Gateway
    tidak dapat menemukannya di penyimpanan auth.

    - **Gunakan Claude CLI**
      - Jalankan `openclaw models auth login --provider anthropic --method cli --set-default` pada gateway host.
    - **Jika Anda ingin menggunakan API key sebagai gantinya**
      - Letakkan `ANTHROPIC_API_KEY` di `~/.openclaw/.env` pada **gateway host**.
      - Hapus urutan pin apa pun yang memaksa profil yang hilang:

        ```bash
        openclaw models auth order clear --provider anthropic
        ```

    - **Pastikan Anda menjalankan perintah di gateway host**
      - Dalam mode remote, profil auth hidup di mesin gateway, bukan laptop Anda.

  </Accordion>

  <Accordion title="Mengapa ia juga mencoba Google Gemini dan gagal?">
    Jika konfigurasi model Anda menyertakan Google Gemini sebagai fallback (atau Anda beralih ke singkatan Gemini), OpenClaw akan mencobanya selama fallback model. Jika Anda belum mengonfigurasi kredensial Google, Anda akan melihat `No API key found for provider "google"`.

    Perbaikan: sediakan auth Google, atau hapus/hindari model Google di `agents.defaults.model.fallbacks` / alias agar fallback tidak dirutekan ke sana.

    **LLM request rejected: thinking signature required (Google Antigravity)**

    Penyebab: riwayat sesi berisi **thinking blocks tanpa signature** (sering dari
    stream yang dibatalkan/parsial). Google Antigravity memerlukan signature untuk thinking blocks.

    Perbaikan: OpenClaw sekarang menghapus thinking blocks tanpa signature untuk Google Antigravity Claude. Jika masih muncul, mulai **sesi baru** atau setel `/thinking off` untuk agen tersebut.

  </Accordion>
</AccordionGroup>

## Profil auth: apa itu dan bagaimana mengelolanya

Terkait: [/concepts/oauth](/id/concepts/oauth) (alur OAuth, penyimpanan token, pola multi-akun)

<AccordionGroup>
  <Accordion title="Apa itu profil auth?">
    Profil auth adalah catatan kredensial bernama (OAuth atau API key) yang terikat ke provider. Profil hidup di:

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

  </Accordion>

  <Accordion title="Apa id profil yang umum?">
    OpenClaw menggunakan ID berawalan provider seperti:

    - `anthropic:default` (umum ketika tidak ada identitas email)
    - `anthropic:<email>` untuk identitas OAuth
    - ID kustom yang Anda pilih (mis. `anthropic:work`)

  </Accordion>

  <Accordion title="Bisakah saya mengendalikan profil auth mana yang dicoba lebih dulu?">
    Ya. Konfigurasi mendukung metadata opsional untuk profil dan urutan per provider (`auth.order.<provider>`). Ini **tidak** menyimpan secret; ini memetakan ID ke provider/mode dan menetapkan urutan rotasi.

    OpenClaw dapat sementara melewati profil jika berada dalam **cooldown** singkat (rate limit/timeout/kegagalan auth) atau state **disabled** yang lebih lama (penagihan/kredit tidak cukup). Untuk memeriksa ini, jalankan `openclaw models status --json` dan lihat `auth.unusableProfiles`. Penyetelan: `auth.cooldowns.billingBackoffHours*`.

    Cooldown rate-limit bisa terbatas pada model. Profil yang sedang cooldown
    untuk satu model masih dapat digunakan untuk model saudara pada provider yang sama,
    sementara jendela billing/disabled tetap memblokir seluruh profil.

    Anda juga dapat menyetel override urutan **per agen** (disimpan dalam `auth-profiles.json` agen tersebut) melalui CLI:

    ```bash
    # Defaults to the configured default agent (omit --agent)
    openclaw models auth order get --provider anthropic

    # Lock rotation to a single profile (only try this one)
    openclaw models auth order set --provider anthropic anthropic:default

    # Or set an explicit order (fallback within provider)
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # Clear override (fall back to config auth.order / round-robin)
    openclaw models auth order clear --provider anthropic
    ```

    Untuk menargetkan agen tertentu:

    ```bash
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    Untuk memverifikasi apa yang benar-benar akan dicoba, gunakan:

    ```bash
    openclaw models status --probe
    ```

    Jika profil tersimpan dihilangkan dari urutan eksplisit, probe melaporkan
    `excluded_by_auth_order` untuk profil itu alih-alih mencobanya secara diam-diam.

  </Accordion>

  <Accordion title="OAuth vs API key - apa bedanya?">
    OpenClaw mendukung keduanya:

    - **OAuth** sering memanfaatkan akses berbasis langganan (bila berlaku).
    - **API keys** menggunakan penagihan bayar per token.

    Wizard secara eksplisit mendukung Anthropic Claude CLI, OpenAI Codex OAuth, dan API keys.

  </Accordion>
</AccordionGroup>

## Gateway: port, "already running", dan mode remote

<AccordionGroup>
  <Accordion title="Port apa yang digunakan Gateway?">
    `gateway.port` mengendalikan satu port multipleks untuk WebSocket + HTTP (Control UI, hooks, dll.).

    Prioritas:

    ```
    --port > OPENCLAW_GATEWAY_PORT > gateway.port > default 18789
    ```

  </Accordion>

  <Accordion title='Mengapa openclaw gateway status menampilkan "Runtime: running" tetapi "RPC probe: failed"?'>
    Karena "running" adalah tampilan **supervisor** (launchd/systemd/schtasks). Probe RPC adalah CLI yang benar-benar terhubung ke gateway WebSocket dan memanggil `status`.

    Gunakan `openclaw gateway status` dan percayai baris-baris ini:

    - `Probe target:` (URL yang benar-benar digunakan probe)
    - `Listening:` (apa yang benar-benar terikat pada port)
    - `Last gateway error:` (akar masalah umum saat proses hidup tetapi port tidak mendengarkan)

  </Accordion>

  <Accordion title='Mengapa openclaw gateway status menampilkan "Config (cli)" dan "Config (service)" berbeda?'>
    Anda mengedit satu file konfigurasi sementara layanan menjalankan file yang lain (sering kali ketidakcocokan `--profile` / `OPENCLAW_STATE_DIR`).

    Perbaikan:

    ```bash
    openclaw gateway install --force
    ```

    Jalankan itu dari `--profile` / environment yang sama yang Anda inginkan agar digunakan oleh layanan.

  </Accordion>

  <Accordion title='Apa arti "another gateway instance is already listening"?'>
    OpenClaw