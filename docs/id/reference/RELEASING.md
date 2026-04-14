---
read_when:
    - Mencari definisi saluran rilis publik
    - Mencari penamaan versi dan ritme rilis
summary: Saluran rilis publik, penamaan versi, dan ritme rilis
title: Kebijakan Rilis
x-i18n:
    generated_at: "2026-04-14T09:14:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3eaf9f1786b8c9fd4f5a9c657b623cb69d1a485958e1a9b8f108511839b63587
    source_path: reference/RELEASING.md
    workflow: 15
---

# Kebijakan Rilis

OpenClaw memiliki tiga jalur rilis publik:

- stable: rilis bertag yang dipublikasikan ke npm `beta` secara default, atau ke npm `latest` jika diminta secara eksplisit
- beta: tag prarilis yang dipublikasikan ke npm `beta`
- dev: head `main` yang terus bergerak

## Penamaan versi

- Versi rilis stable: `YYYY.M.D`
  - Git tag: `vYYYY.M.D`
- Versi rilis koreksi stable: `YYYY.M.D-N`
  - Git tag: `vYYYY.M.D-N`
- Versi prarilis beta: `YYYY.M.D-beta.N`
  - Git tag: `vYYYY.M.D-beta.N`
- Jangan tambahkan nol di depan bulan atau hari
- `latest` berarti rilis npm stable yang saat ini dipromosikan
- `beta` berarti target instalasi beta saat ini
- Rilis stable dan rilis koreksi stable dipublikasikan ke npm `beta` secara default; operator rilis dapat menargetkan `latest` secara eksplisit, atau mempromosikan build beta yang telah diverifikasi nanti
- Setiap rilis OpenClaw mengirimkan paket npm dan aplikasi macOS secara bersamaan

## Ritme rilis

- Rilis bergerak dengan beta terlebih dahulu
- Stable mengikuti hanya setelah beta terbaru tervalidasi
- Prosedur rilis terperinci, persetujuan, kredensial, dan catatan pemulihan
  hanya untuk maintainer

## Pemeriksaan awal rilis

- Jalankan `pnpm build && pnpm ui:build` sebelum `pnpm release:check` agar
  artefak rilis `dist/*` yang diharapkan dan bundle Control UI tersedia untuk langkah
  validasi pack
- Jalankan `pnpm release:check` sebelum setiap rilis bertag
- Pemeriksaan rilis sekarang dijalankan dalam workflow manual terpisah:
  `OpenClaw Release Checks`
- Pemisahan ini disengaja: pertahankan jalur rilis npm yang sebenarnya tetap singkat,
  deterministik, dan berfokus pada artefak, sementara pemeriksaan live yang lebih lambat tetap berada di
  jalurnya sendiri agar tidak menunda atau memblokir publikasi
- Pemeriksaan rilis harus dipicu dari workflow ref `main` agar
  logika workflow dan secret tetap kanonis
- Workflow itu menerima tag rilis yang sudah ada atau commit SHA `main`
  penuh 40 karakter saat ini
- Dalam mode commit-SHA, workflow itu hanya menerima HEAD `origin/main` saat ini; gunakan
  tag rilis untuk commit rilis yang lebih lama
- Validasi awal khusus `OpenClaw NPM Release` juga menerima commit SHA `main`
  penuh 40 karakter saat ini tanpa memerlukan tag yang sudah dipush
- Jalur SHA itu hanya untuk validasi dan tidak dapat dipromosikan menjadi publikasi nyata
- Dalam mode SHA, workflow mensintesis `v<package.json version>` hanya untuk
  pemeriksaan metadata paket; publikasi nyata tetap memerlukan tag rilis yang nyata
- Kedua workflow menjaga jalur publikasi dan promosi nyata tetap berjalan di runner yang dihosting GitHub,
  sementara jalur validasi non-mutasi dapat menggunakan runner Linux
  Blacksmith yang lebih besar
- Workflow itu menjalankan
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  menggunakan secret workflow `OPENAI_API_KEY` dan `ANTHROPIC_API_KEY`
- Validasi awal rilis npm tidak lagi menunggu jalur pemeriksaan rilis terpisah
- Jalankan `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (atau tag beta/koreksi yang sesuai) sebelum persetujuan
- Setelah npm publish, jalankan
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (atau versi beta/koreksi yang sesuai) untuk memverifikasi jalur instalasi registry
  yang dipublikasikan dalam temp prefix baru
- Otomatisasi rilis maintainer sekarang menggunakan preflight-then-promote:
  - publikasi npm nyata harus lulus dengan npm `preflight_run_id` yang berhasil
  - rilis npm stable secara default menuju `beta`
  - publikasi npm stable dapat menargetkan `latest` secara eksplisit melalui input workflow
  - promosi npm stable dari `beta` ke `latest` masih tersedia sebagai mode manual eksplisit pada workflow tepercaya `OpenClaw NPM Release`
  - publikasi stable langsung juga dapat menjalankan mode sinkronisasi dist-tag eksplisit yang
    mengarahkan `latest` dan `beta` ke versi stable yang sudah dipublikasikan
  - mode dist-tag tersebut tetap memerlukan `NPM_TOKEN` yang valid di environment `npm-release` karena pengelolaan npm `dist-tag` terpisah dari trusted publishing
  - `macOS Release` publik hanya untuk validasi
  - publikasi mac privat yang nyata harus lulus dengan `preflight_run_id` dan `validate_run_id`
    mac privat yang berhasil
  - jalur publikasi nyata mempromosikan artefak yang telah disiapkan alih-alih membangunnya
    lagi
- Untuk rilis koreksi stable seperti `YYYY.M.D-N`, verifier pascapublikasi
  juga memeriksa jalur upgrade temp-prefix yang sama dari `YYYY.M.D` ke `YYYY.M.D-N`
  agar koreksi rilis tidak diam-diam meninggalkan instalasi global yang lebih lama pada
  payload stable dasar
- Validasi awal rilis npm gagal secara tertutup kecuali tarball mencakup
  `dist/control-ui/index.html` dan payload `dist/control-ui/assets/` yang tidak kosong
  sehingga kita tidak mengirim dashboard browser kosong lagi
- `pnpm test:install:smoke` juga menegakkan anggaran `unpackedSize` npm pack pada
  tarball pembaruan kandidat, sehingga installer e2e menangkap pembengkakan pack yang tidak disengaja
  sebelum jalur publikasi rilis
- Jika pekerjaan rilis menyentuh perencanaan CI, manifest timing extension, atau
  matriks pengujian extension, regenerasikan dan tinjau keluaran matriks workflow
  `checks-node-extensions` milik planner dari `.github/workflows/ci.yml`
  sebelum persetujuan agar catatan rilis tidak menjelaskan tata letak CI yang usang
- Kesiapan rilis macOS stable juga mencakup permukaan updater:
  - rilis GitHub harus berakhir dengan `.zip`, `.dmg`, dan `.dSYM.zip` yang sudah dipaketkan
  - `appcast.xml` di `main` harus mengarah ke zip stable baru setelah publish
  - aplikasi yang dipaketkan harus tetap memiliki bundle id non-debug, Sparkle feed
    URL yang tidak kosong, dan `CFBundleVersion` pada atau di atas batas build Sparkle kanonis
    untuk versi rilis tersebut

## Input workflow NPM

`OpenClaw NPM Release` menerima input yang dikendalikan operator berikut:

- `tag`: tag rilis yang wajib seperti `v2026.4.2`, `v2026.4.2-1`, atau
  `v2026.4.2-beta.1`; ketika `preflight_only=true`, ini juga dapat berupa
  commit SHA `main` penuh 40 karakter saat ini untuk validasi awal khusus validasi
- `preflight_only`: `true` untuk validasi/build/package saja, `false` untuk
  jalur publish nyata
- `preflight_run_id`: wajib pada jalur publish nyata agar workflow menggunakan kembali
  tarball yang telah disiapkan dari run preflight yang berhasil
- `npm_dist_tag`: tag target npm untuk jalur publish; default ke `beta`
- `promote_beta_to_latest`: `true` untuk melewati publish dan memindahkan build `beta` stable
  yang sudah dipublikasikan ke `latest`
- `sync_stable_dist_tags`: `true` untuk melewati publish dan mengarahkan `latest` dan
  `beta` ke versi stable yang sudah dipublikasikan

`OpenClaw Release Checks` menerima input yang dikendalikan operator berikut:

- `ref`: tag rilis yang sudah ada atau commit
  SHA `main` penuh 40 karakter saat ini untuk divalidasi

Aturan:

- Tag stable dan koreksi dapat dipublikasikan ke `beta` atau `latest`
- Tag prarilis beta hanya dapat dipublikasikan ke `beta`
- Input commit SHA penuh hanya diizinkan saat `preflight_only=true`
- Mode commit-SHA pemeriksaan rilis juga mensyaratkan HEAD `origin/main` saat ini
- Jalur publish nyata harus menggunakan `npm_dist_tag` yang sama dengan yang digunakan saat preflight;
  workflow memverifikasi metadata tersebut sebelum publish dilanjutkan
- Mode promosi harus menggunakan tag stable atau koreksi, `preflight_only=false`,
  `preflight_run_id` kosong, dan `npm_dist_tag=beta`
- Mode sinkronisasi dist-tag harus menggunakan tag stable atau koreksi,
  `preflight_only=false`, `preflight_run_id` kosong, `npm_dist_tag=latest`,
  dan `promote_beta_to_latest=false`
- Mode promosi dan sinkronisasi dist-tag juga memerlukan `NPM_TOKEN` yang valid karena
  `npm dist-tag add` tetap memerlukan autentikasi npm biasa; trusted publishing hanya mencakup
  jalur publish paket

## Urutan rilis npm stable

Saat memotong rilis npm stable:

1. Jalankan `OpenClaw NPM Release` dengan `preflight_only=true`
   - Sebelum tag ada, Anda dapat menggunakan commit SHA `main` penuh saat ini untuk
     dry run hanya validasi dari workflow preflight
2. Pilih `npm_dist_tag=beta` untuk alur beta-first normal, atau `latest` hanya
   ketika Anda memang ingin melakukan publish stable langsung
3. Jalankan `OpenClaw Release Checks` secara terpisah dengan tag yang sama atau
   commit SHA `main` penuh saat ini ketika Anda ingin cakupan cache prompt live
   - Ini dipisahkan dengan sengaja agar cakupan live tetap tersedia tanpa
     menghubungkan kembali pemeriksaan yang berjalan lama atau tidak stabil ke workflow publish
4. Simpan `preflight_run_id` yang berhasil
5. Jalankan `OpenClaw NPM Release` lagi dengan `preflight_only=false`, `tag`
   yang sama, `npm_dist_tag` yang sama, dan `preflight_run_id` yang disimpan
6. Jika rilis mendarat di `beta`, jalankan `OpenClaw NPM Release` nanti dengan
   `tag` stable yang sama, `promote_beta_to_latest=true`, `preflight_only=false`,
   `preflight_run_id` kosong, dan `npm_dist_tag=beta` ketika Anda ingin memindahkan build
   yang telah dipublikasikan itu ke `latest`
7. Jika rilis sengaja dipublikasikan langsung ke `latest` dan `beta`
   harus mengikuti build stable yang sama, jalankan `OpenClaw NPM Release` dengan
   `tag` stable yang sama, `sync_stable_dist_tags=true`, `promote_beta_to_latest=false`,
   `preflight_only=false`, `preflight_run_id` kosong, dan `npm_dist_tag=latest`

Mode promosi dan sinkronisasi dist-tag tetap memerlukan persetujuan environment `npm-release`
dan `NPM_TOKEN` yang valid yang dapat diakses oleh workflow tersebut.

Ini menjaga jalur publish langsung dan jalur promosi beta-first tetap
terdokumentasi dan terlihat oleh operator.

## Referensi publik

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Maintainer menggunakan dokumentasi rilis privat di
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
untuk runbook yang sebenarnya.
