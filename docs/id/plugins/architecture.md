---
read_when:
    - Membangun atau men-debug plugin OpenClaw native
    - Memahami model kapabilitas plugin atau batas kepemilikan
    - Mengerjakan pipeline pemuatan plugin atau registri
    - Mengimplementasikan hook runtime provider atau plugin saluran
sidebarTitle: Internals
summary: 'Internal plugin: model kapabilitas, kepemilikan, kontrak, pipeline pemuatan, dan helper runtime'
title: Internal plugin
x-i18n:
    generated_at: "2026-04-11T15:15:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7cac67984d0d729c0905bcf5c18372fb0d9b02bbd3a531580b7e2ef483ef40a6
    source_path: plugins/architecture.md
    workflow: 15
---

# Internal plugin

<Info>
  Ini adalah **referensi arsitektur mendalam**. Untuk panduan praktis, lihat:
  - [Instal dan gunakan plugin](/id/tools/plugin) — panduan pengguna
  - [Memulai](/id/plugins/building-plugins) — tutorial plugin pertama
  - [Plugin Saluran](/id/plugins/sdk-channel-plugins) — bangun saluran perpesanan
  - [Plugin Provider](/id/plugins/sdk-provider-plugins) — bangun provider model
  - [Ikhtisar SDK](/id/plugins/sdk-overview) — peta impor dan API registrasi
</Info>

Halaman ini membahas arsitektur internal sistem plugin OpenClaw.

## Model kapabilitas publik

Kapabilitas adalah model **plugin native** publik di dalam OpenClaw. Setiap
plugin OpenClaw native mendaftar ke satu atau lebih jenis kapabilitas:

| Capability             | Registration method                              | Example plugins                      |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| Inferensi teks         | `api.registerProvider(...)`                      | `openai`, `anthropic`                |
| Backend inferensi CLI  | `api.registerCliBackend(...)`                    | `openai`, `anthropic`                |
| Ucapan                 | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`            |
| Transkripsi realtime   | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| Suara realtime         | `api.registerRealtimeVoiceProvider(...)`         | `openai`                             |
| Pemahaman media        | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                   |
| Pembuatan gambar       | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Pembuatan musik        | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                  |
| Pembuatan video        | `api.registerVideoGenerationProvider(...)`       | `qwen`                               |
| Pengambilan web        | `api.registerWebFetchProvider(...)`              | `firecrawl`                          |
| Pencarian web          | `api.registerWebSearchProvider(...)`             | `google`                             |
| Saluran / perpesanan   | `api.registerChannel(...)`                       | `msteams`, `matrix`                  |

Plugin yang mendaftarkan nol kapabilitas tetapi menyediakan hook, alat, atau
layanan adalah plugin **legacy khusus hook**. Pola tersebut masih sepenuhnya didukung.

### Sikap kompatibilitas eksternal

Model kapabilitas sudah masuk ke core dan digunakan oleh plugin
bundled/native saat ini, tetapi kompatibilitas plugin eksternal masih
memerlukan standar yang lebih ketat daripada "ini diekspor, jadi ini dibekukan."

Panduan saat ini:

- **plugin eksternal yang sudah ada:** pertahankan integrasi berbasis hook tetap berfungsi; perlakukan
  ini sebagai baseline kompatibilitas
- **plugin bundled/native baru:** utamakan registrasi kapabilitas yang eksplisit dibanding
  akses langsung khusus vendor atau desain baru khusus hook
- **plugin eksternal yang mengadopsi registrasi kapabilitas:** diperbolehkan, tetapi perlakukan
  surface helper khusus kapabilitas sebagai sesuatu yang masih berkembang kecuali dokumentasi secara eksplisit menandai sebuah
  kontrak sebagai stabil

Aturan praktis:

- API registrasi kapabilitas adalah arah yang dituju
- hook legacy tetap menjadi jalur paling aman tanpa kerusakan untuk plugin eksternal selama
  transisi
- subpath helper yang diekspor tidak semuanya setara; utamakan kontrak terdokumentasi yang sempit,
  bukan ekspor helper yang insidental

### Bentuk plugin

OpenClaw mengklasifikasikan setiap plugin yang dimuat ke dalam sebuah bentuk berdasarkan
perilaku registrasinya yang sebenarnya (bukan hanya metadata statis):

- **plain-capability** -- mendaftarkan tepat satu jenis kapabilitas (misalnya
  plugin khusus provider seperti `mistral`)
- **hybrid-capability** -- mendaftarkan beberapa jenis kapabilitas (misalnya
  `openai` memiliki inferensi teks, ucapan, pemahaman media, dan
  pembuatan gambar)
- **hook-only** -- hanya mendaftarkan hook (typed atau custom), tanpa kapabilitas,
  alat, perintah, atau layanan
- **non-capability** -- mendaftarkan alat, perintah, layanan, atau rute tetapi tanpa
  kapabilitas

Gunakan `openclaw plugins inspect <id>` untuk melihat bentuk plugin dan rincian
kapabilitasnya. Lihat [referensi CLI](/cli/plugins#inspect) untuk detail.

### Hook legacy

Hook `before_agent_start` tetap didukung sebagai jalur kompatibilitas untuk
plugin khusus hook. Plugin legacy di dunia nyata masih bergantung padanya.

Arah ke depan:

- pertahankan agar tetap berfungsi
- dokumentasikan sebagai legacy
- utamakan `before_model_resolve` untuk pekerjaan override model/provider
- utamakan `before_prompt_build` untuk pekerjaan mutasi prompt
- hapus hanya setelah penggunaan nyata menurun dan cakupan fixture membuktikan keamanan migrasi

### Sinyal kompatibilitas

Saat Anda menjalankan `openclaw doctor` atau `openclaw plugins inspect <id>`, Anda mungkin melihat
salah satu label berikut:

| Signal                     | Meaning                                                      |
| -------------------------- | ------------------------------------------------------------ |
| **konfigurasi valid**      | Konfigurasi dapat diparse dengan baik dan plugin berhasil di-resolve |
| **advisori kompatibilitas** | Plugin menggunakan pola yang didukung tetapi lebih lama (mis. `hook-only`) |
| **peringatan legacy**      | Plugin menggunakan `before_agent_start`, yang sudah deprecated |
| **error keras**            | Konfigurasi tidak valid atau plugin gagal dimuat             |

Baik `hook-only` maupun `before_agent_start` tidak akan merusak plugin Anda saat ini --
`hook-only` bersifat advisori, dan `before_agent_start` hanya memicu peringatan. Sinyal-sinyal ini
juga muncul di `openclaw status --all` dan `openclaw plugins doctor`.

## Ikhtisar arsitektur

Sistem plugin OpenClaw memiliki empat lapisan:

1. **Manifest + discovery**
   OpenClaw menemukan kandidat plugin dari path yang dikonfigurasi, root workspace,
   root ekstensi global, dan ekstensi bundled. Discovery membaca manifest native
   `openclaw.plugin.json` ditambah manifest bundle yang didukung terlebih dahulu.
2. **Enablement + validation**
   Core memutuskan apakah plugin yang ditemukan diaktifkan, dinonaktifkan, diblokir, atau
   dipilih untuk slot eksklusif seperti memori.
3. **Runtime loading**
   Plugin OpenClaw native dimuat in-process melalui jiti dan mendaftarkan
   kapabilitas ke dalam registri pusat. Bundle yang kompatibel dinormalisasi menjadi
   catatan registri tanpa mengimpor kode runtime.
4. **Surface consumption**
   Bagian lain dari OpenClaw membaca registri untuk mengekspos alat, saluran, pengaturan
   provider, hook, rute HTTP, perintah CLI, dan layanan.

Khusus untuk plugin CLI, discovery perintah root dibagi menjadi dua fase:

- metadata saat parse berasal dari `registerCli(..., { descriptors: [...] })`
- modul CLI plugin yang sebenarnya dapat tetap lazy dan mendaftar pada pemanggilan pertama

Hal ini menjaga kode CLI milik plugin tetap berada di dalam plugin sambil tetap memungkinkan OpenClaw
mencadangkan nama perintah root sebelum parsing.

Batas desain yang penting:

- discovery + validasi konfigurasi harus bekerja dari **metadata manifest/schema**
  tanpa mengeksekusi kode plugin
- perilaku runtime native berasal dari jalur `register(api)` modul plugin

Pemisahan itu memungkinkan OpenClaw memvalidasi konfigurasi, menjelaskan plugin yang hilang/dinonaktifkan, dan
membangun petunjuk UI/schema sebelum runtime penuh aktif.

### Plugin saluran dan alat pesan bersama

Plugin saluran tidak perlu mendaftarkan alat kirim/edit/reaksi terpisah untuk
aksi chat normal. OpenClaw mempertahankan satu alat `message` bersama di core, dan
plugin saluran memiliki discovery dan eksekusi khusus saluran di baliknya.

Batas saat ini adalah:

- core memiliki host alat `message` bersama, wiring prompt, pembukuan
  sesi/thread, dan dispatch eksekusi
- plugin saluran memiliki discovery aksi terlingkup, discovery kapabilitas, dan fragmen schema khusus saluran
- plugin saluran memiliki tata bahasa percakapan sesi khusus provider, seperti
  bagaimana id percakapan mengodekan id thread atau mewarisi dari percakapan induk
- plugin saluran mengeksekusi aksi akhir melalui action adapter mereka

Untuk plugin saluran, surface SDK-nya adalah
`ChannelMessageActionAdapter.describeMessageTool(...)`. Panggilan discovery terpadu
itu memungkinkan plugin mengembalikan aksi yang terlihat, kapabilitas, dan kontribusi
schema sekaligus sehingga bagian-bagian tersebut tidak saling drift.

Core meneruskan scope runtime ke langkah discovery tersebut. Field penting meliputi:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- `requesterSenderId` masuk tepercaya

Ini penting untuk plugin yang sensitif terhadap konteks. Sebuah saluran dapat menyembunyikan atau mengekspos
aksi pesan berdasarkan akun aktif, ruang/thread/pesan saat ini, atau
identitas peminta tepercaya tanpa meng-hardcode cabang khusus saluran di
alat `message` core.

Inilah sebabnya perubahan routing embedded-runner masih merupakan pekerjaan plugin: runner bertanggung jawab
untuk meneruskan identitas chat/sesi saat ini ke batas discovery plugin agar
alat `message` bersama mengekspos surface milik saluran yang tepat untuk
giliran saat ini.

Untuk helper eksekusi milik saluran, plugin bundled harus menjaga runtime eksekusi
tetap berada di dalam modul ekstensi mereka sendiri. Core tidak lagi memiliki runtime
aksi pesan Discord, Slack, Telegram, atau WhatsApp di bawah `src/agents/tools`.
Kami tidak memublikasikan subpath `plugin-sdk/*-action-runtime` terpisah, dan plugin
bundled harus mengimpor kode runtime lokal mereka sendiri secara langsung dari
modul milik ekstensi mereka.

Batas yang sama juga berlaku untuk seam SDK bernama provider secara umum: core tidak boleh
mengimpor convenience barrel khusus saluran untuk ekstensi Slack, Discord, Signal,
WhatsApp, atau yang serupa. Jika core membutuhkan suatu perilaku, gunakan salah satu dari dua cara:
konsumsi barrel `api.ts` / `runtime-api.ts` milik plugin bundled itu sendiri atau
naikkan kebutuhan tersebut menjadi kapabilitas generik yang sempit di SDK bersama.

Khusus untuk poll, ada dua jalur eksekusi:

- `outbound.sendPoll` adalah baseline bersama untuk saluran yang cocok dengan model
  poll umum
- `actions.handleAction("poll")` adalah jalur yang diutamakan untuk semantik poll khusus saluran
  atau parameter poll tambahan

Core sekarang menunda parsing poll bersama sampai setelah dispatch poll plugin menolak
aksi tersebut, sehingga handler poll milik plugin dapat menerima field poll
khusus saluran tanpa lebih dahulu diblokir oleh parser poll generik.

Lihat [Load pipeline](#load-pipeline) untuk urutan startup lengkap.

## Model kepemilikan kapabilitas

OpenClaw memperlakukan plugin native sebagai batas kepemilikan untuk sebuah **perusahaan** atau sebuah
**fitur**, bukan sebagai kumpulan acak integrasi yang tidak saling terkait.

Artinya:

- plugin perusahaan biasanya harus memiliki semua
  surface OpenClaw yang menghadap perusahaan tersebut
- plugin fitur biasanya harus memiliki surface fitur penuh yang diperkenalkannya
- saluran harus menggunakan kapabilitas core bersama alih-alih mengimplementasikan ulang
  perilaku provider secara ad hoc

Contoh:

- plugin bundled `openai` memiliki perilaku provider model OpenAI dan perilaku
  OpenAI speech + realtime-voice + media-understanding + image-generation
- plugin bundled `elevenlabs` memiliki perilaku ucapan ElevenLabs
- plugin bundled `microsoft` memiliki perilaku ucapan Microsoft
- plugin bundled `google` memiliki perilaku provider model Google ditambah perilaku
  Google media-understanding + image-generation + web-search
- plugin bundled `firecrawl` memiliki perilaku web-fetch Firecrawl
- plugin bundled `minimax`, `mistral`, `moonshot`, dan `zai` memiliki backend
  media-understanding mereka
- plugin `qwen` bundled memiliki perilaku text-provider Qwen ditambah
  perilaku media-understanding dan video-generation
- plugin `voice-call` adalah plugin fitur: plugin ini memiliki transport panggilan, alat,
  CLI, rute, dan jembatan media-stream Twilio, tetapi menggunakan kapabilitas speech bersama
  ditambah realtime-transcription dan realtime-voice alih-alih
  mengimpor plugin vendor secara langsung

Kondisi akhir yang dituju adalah:

- OpenAI berada dalam satu plugin meskipun mencakup model teks, ucapan, gambar, dan
  video di masa depan
- vendor lain dapat melakukan hal yang sama untuk surface miliknya sendiri
- saluran tidak peduli plugin vendor mana yang memiliki provider; mereka menggunakan
  kontrak kapabilitas bersama yang diekspos oleh core

Inilah pembedaan utamanya:

- **plugin** = batas kepemilikan
- **capability** = kontrak core yang dapat diimplementasikan atau digunakan oleh banyak plugin

Jadi jika OpenClaw menambahkan domain baru seperti video, pertanyaan pertama bukan
"provider mana yang harus meng-hardcode penanganan video?" Pertanyaan pertamanya adalah "apa
kontrak kapabilitas video core?" Setelah kontrak itu ada, plugin vendor
dapat mendaftar padanya dan plugin saluran/fitur dapat menggunakannya.

Jika kapabilitas tersebut belum ada, langkah yang tepat biasanya adalah:

1. definisikan kapabilitas yang belum ada di core
2. ekspos melalui API/runtime plugin dengan cara yang bertipe
3. hubungkan saluran/fitur ke kapabilitas tersebut
4. biarkan plugin vendor mendaftarkan implementasinya

Hal ini menjaga kepemilikan tetap eksplisit sambil menghindari perilaku core yang bergantung pada
satu vendor atau jalur kode khusus plugin yang sekali pakai.

### Pelapisan kapabilitas

Gunakan model mental ini saat memutuskan di mana kode harus ditempatkan:

- **lapisan kapabilitas core**: orkestrasi bersama, kebijakan, fallback, aturan merge
  konfigurasi, semantik pengiriman, dan kontrak bertipe
- **lapisan plugin vendor**: API khusus vendor, autentikasi, katalog model, sintesis
  ucapan, pembuatan gambar, backend video di masa depan, endpoint penggunaan
- **lapisan plugin saluran/fitur**: integrasi Slack/Discord/voice-call/dll.
  yang menggunakan kapabilitas core dan menyajikannya pada suatu surface

Misalnya, TTS mengikuti bentuk ini:

- core memiliki kebijakan TTS pada waktu balasan, urutan fallback, preferensi, dan pengiriman saluran
- `openai`, `elevenlabs`, dan `microsoft` memiliki implementasi sintesis
- `voice-call` menggunakan helper runtime TTS teleponi

Pola yang sama sebaiknya diutamakan untuk kapabilitas di masa depan.

### Contoh plugin perusahaan multi-kapabilitas

Sebuah plugin perusahaan harus terasa kohesif dari luar. Jika OpenClaw memiliki
kontrak bersama untuk model, ucapan, transkripsi realtime, suara realtime, pemahaman
media, pembuatan gambar, pembuatan video, pengambilan web, dan pencarian web,
sebuah vendor dapat memiliki semua surface-nya di satu tempat:

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // auth/model catalog/runtime hooks
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // vendor speech config — implement the SpeechProviderPlugin interface directly
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // credential + fetch logic
      }),
    );
  },
};

export default plugin;
```

Yang penting bukan nama helper yang persis sama. Yang penting adalah bentuknya:

- satu plugin memiliki surface vendor
- core tetap memiliki kontrak kapabilitas
- plugin saluran dan fitur menggunakan helper `api.runtime.*`, bukan kode vendor
- pengujian kontrak dapat menegaskan bahwa plugin mendaftarkan kapabilitas yang
  diklaimnya dimiliki

### Contoh kapabilitas: pemahaman video

OpenClaw sudah memperlakukan pemahaman gambar/audio/video sebagai satu
kapabilitas bersama. Model kepemilikan yang sama berlaku di sana:

1. core mendefinisikan kontrak media-understanding
2. plugin vendor mendaftarkan `describeImage`, `transcribeAudio`, dan
   `describeVideo` sesuai kebutuhan
3. plugin saluran dan fitur menggunakan perilaku core bersama alih-alih
   terhubung langsung ke kode vendor

Ini mencegah asumsi video dari satu provider tertanam di core. Plugin memiliki
surface vendor; core memiliki kontrak kapabilitas dan perilaku fallback.

Pembuatan video sudah menggunakan urutan yang sama: core memiliki
kontrak kapabilitas bertipe dan helper runtime, dan plugin vendor mendaftarkan
implementasi `api.registerVideoGenerationProvider(...)` terhadapnya.

Butuh checklist rollout yang konkret? Lihat
[Capability Cookbook](/id/plugins/architecture).

## Kontrak dan penegakan

Surface API plugin sengaja dibuat bertipe dan dipusatkan di
`OpenClawPluginApi`. Kontrak itu mendefinisikan titik registrasi yang didukung dan
helper runtime yang dapat diandalkan oleh sebuah plugin.

Mengapa ini penting:

- penulis plugin mendapatkan satu standar internal yang stabil
- core dapat menolak kepemilikan ganda seperti dua plugin yang mendaftarkan provider id yang sama
- startup dapat menampilkan diagnostik yang dapat ditindaklanjuti untuk registrasi yang malformed
- pengujian kontrak dapat menegakkan kepemilikan plugin bundled dan mencegah drift senyap

Ada dua lapisan penegakan:

1. **penegakan registrasi runtime**
   Registri plugin memvalidasi registrasi saat plugin dimuat. Contoh:
   provider id duplikat, speech provider id duplikat, dan registrasi
   yang malformed menghasilkan diagnostik plugin alih-alih perilaku yang tidak terdefinisi.
2. **pengujian kontrak**
   Plugin bundled ditangkap dalam registri kontrak selama pengujian berjalan sehingga
   OpenClaw dapat menegaskan kepemilikan secara eksplisit. Saat ini ini digunakan untuk model
   provider, speech provider, web search provider, dan kepemilikan registrasi bundled.

Efek praktisnya adalah OpenClaw mengetahui, sejak awal, plugin mana yang memiliki surface mana.
Hal ini memungkinkan core dan saluran tersusun dengan mulus karena kepemilikan
dideklarasikan, bertipe, dan dapat diuji alih-alih implisit.

### Apa yang termasuk dalam sebuah kontrak

Kontrak plugin yang baik adalah:

- bertipe
- kecil
- spesifik terhadap kapabilitas
- dimiliki oleh core
- dapat digunakan kembali oleh banyak plugin
- dapat digunakan oleh saluran/fitur tanpa pengetahuan vendor

Kontrak plugin yang buruk adalah:

- kebijakan khusus vendor yang tersembunyi di core
- escape hatch plugin sekali pakai yang melewati registri
- kode saluran yang langsung menjangkau implementasi vendor
- objek runtime ad hoc yang bukan bagian dari `OpenClawPluginApi` atau
  `api.runtime`

Jika ragu, naikkan tingkat abstraksinya: definisikan dulu kapabilitasnya, lalu
biarkan plugin terhubung ke sana.

## Model eksekusi

Plugin OpenClaw native berjalan **in-process** dengan Gateway. Mereka tidak
disandbox. Plugin native yang dimuat memiliki batas kepercayaan tingkat proses yang sama dengan
kode core.

Implikasinya:

- plugin native dapat mendaftarkan alat, handler jaringan, hook, dan layanan
- bug pada plugin native dapat membuat gateway crash atau tidak stabil
- plugin native yang berbahaya setara dengan eksekusi kode arbitrer di dalam
  proses OpenClaw

Bundle yang kompatibel lebih aman secara default karena OpenClaw saat ini memperlakukannya
sebagai paket metadata/konten. Dalam rilis saat ini, itu terutama berarti
Skills bundled.

Gunakan allowlist dan path instalasi/pemuatan yang eksplisit untuk plugin non-bundled. Perlakukan
plugin workspace sebagai kode waktu pengembangan, bukan default produksi.

Untuk nama package workspace bundled, pertahankan plugin id tertanam dalam nama npm:
`@openclaw/<id>` secara default, atau sufiks bertipe yang disetujui seperti
`-provider`, `-plugin`, `-speech`, `-sandbox`, atau `-media-understanding` ketika
package memang mengekspos peran plugin yang lebih sempit.

Catatan kepercayaan penting:

- `plugins.allow` mempercayai **id plugin**, bukan asal sumbernya.
- Plugin workspace dengan id yang sama dengan plugin bundled sengaja membayangi
  salinan bundled saat plugin workspace tersebut diaktifkan/masuk allowlist.
- Ini normal dan berguna untuk pengembangan lokal, pengujian patch, dan hotfix.

## Batas ekspor

OpenClaw mengekspor kapabilitas, bukan kemudahan implementasi.

Pertahankan registrasi kapabilitas tetap publik. Pangkas ekspor helper non-kontrak:

- subpath helper khusus plugin bundled
- subpath plumbing runtime yang tidak dimaksudkan sebagai API publik
- helper convenience khusus vendor
- helper setup/onboarding yang merupakan detail implementasi

Beberapa subpath helper plugin bundled masih tetap ada di peta ekspor SDK yang
dihasilkan untuk kompatibilitas dan pemeliharaan plugin bundled. Contoh saat ini mencakup
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, dan beberapa seam `plugin-sdk/matrix*`. Perlakukan itu sebagai
ekspor detail implementasi yang dicadangkan, bukan sebagai pola SDK yang direkomendasikan untuk
plugin pihak ketiga baru.

## Pipeline pemuatan

Saat startup, OpenClaw kira-kira melakukan ini:

1. menemukan root plugin kandidat
2. membaca manifest native atau bundle yang kompatibel serta metadata package
3. menolak kandidat yang tidak aman
4. menormalkan konfigurasi plugin (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. memutuskan enablement untuk setiap kandidat
6. memuat modul native yang diaktifkan melalui jiti
7. memanggil hook native `register(api)` (atau `activate(api)` — alias legacy) dan mengumpulkan registrasi ke dalam registri plugin
8. mengekspos registri ke surface perintah/runtime

<Note>
`activate` adalah alias legacy untuk `register` — loader me-resolve mana pun yang ada (`def.register ?? def.activate`) dan memanggilnya pada titik yang sama. Semua plugin bundled menggunakan `register`; utamakan `register` untuk plugin baru.
</Note>

Gerbang keamanan terjadi **sebelum** eksekusi runtime. Kandidat diblokir
ketika entri keluar dari root plugin, path dapat ditulis oleh semua orang, atau
kepemilikan path terlihat mencurigakan untuk plugin non-bundled.

### Perilaku manifest-first

Manifest adalah sumber kebenaran control-plane. OpenClaw menggunakannya untuk:

- mengidentifikasi plugin
- menemukan saluran/Skills/schema konfigurasi yang dideklarasikan atau kapabilitas bundle
- memvalidasi `plugins.entries.<id>.config`
- menambah label/placeholder Control UI
- menampilkan metadata instalasi/katalog
- mempertahankan descriptor aktivasi dan setup yang murah tanpa memuat runtime plugin

Untuk plugin native, modul runtime adalah bagian data-plane. Modul ini mendaftarkan
perilaku nyata seperti hook, alat, perintah, atau alur provider.

Blok `activation` dan `setup` manifest opsional tetap berada di control plane.
Keduanya adalah descriptor khusus metadata untuk perencanaan aktivasi dan discovery setup;
keduanya tidak menggantikan registrasi runtime, `register(...)`, atau `setupEntry`.

### Apa yang di-cache oleh loader

OpenClaw menyimpan cache in-process jangka pendek untuk:

- hasil discovery
- data registri manifest
- registri plugin yang dimuat

Cache ini mengurangi startup yang meledak-ledak dan overhead perintah berulang. Cache ini aman
untuk dipandang sebagai cache performa berumur pendek, bukan persistensi.

Catatan performa:

- Setel `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` atau
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` untuk menonaktifkan cache ini.
- Atur jendela cache dengan `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` dan
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Model registri

Plugin yang dimuat tidak langsung memutasi global core acak. Mereka mendaftar ke dalam
registri plugin pusat.

Registri melacak:

- catatan plugin (identitas, sumber, asal, status, diagnostik)
- alat
- hook legacy dan hook bertipe
- saluran
- provider
- handler RPC gateway
- rute HTTP
- registrar CLI
- layanan latar belakang
- perintah milik plugin

Fitur core kemudian membaca dari registri itu alih-alih berbicara langsung dengan modul plugin.
Ini menjaga pemuatan tetap satu arah:

- modul plugin -> registrasi ke registri
- runtime core -> konsumsi registri

Pemisahan ini penting untuk kemudahan pemeliharaan. Artinya sebagian besar surface core hanya
memerlukan satu titik integrasi: "baca registri", bukan "buat kasus khusus untuk setiap modul plugin".

## Callback binding percakapan

Plugin yang melakukan binding percakapan dapat bereaksi saat sebuah persetujuan diselesaikan.

Gunakan `api.onConversationBindingResolved(...)` untuk menerima callback setelah sebuah permintaan bind
disetujui atau ditolak:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // A binding now exists for this plugin + conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // The request was denied; clear any local pending state.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Field payload callback:

- `status`: `"approved"` atau `"denied"`
- `decision`: `"allow-once"`, `"allow-always"`, atau `"deny"`
- `binding`: binding yang telah di-resolve untuk permintaan yang disetujui
- `request`: ringkasan permintaan asli, petunjuk detach, sender id, dan
  metadata percakapan

Callback ini hanya untuk notifikasi. Ini tidak mengubah siapa yang diizinkan untuk melakukan bind
percakapan, dan callback ini berjalan setelah penanganan persetujuan core selesai.

## Hook runtime provider

Plugin provider sekarang memiliki dua lapisan:

- metadata manifest: `providerAuthEnvVars` untuk lookup env-auth provider murah
  sebelum runtime dimuat, `providerAuthAliases` untuk varian provider yang berbagi
  auth, `channelEnvVars` untuk lookup env/setup saluran murah sebelum runtime
  dimuat, ditambah `providerAuthChoices` untuk label onboarding/pilihan auth murah dan
  metadata flag CLI sebelum runtime dimuat
- hook saat konfigurasi: `catalog` / `discovery` legacy ditambah `applyConfigDefaults`
- hook runtime: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `resolveExternalAuthProfiles`,
  `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`, `normalizeResolvedModel`,
  `contributeResolvedModelCompat`, `capabilities`,
  `normalizeToolSchemas`, `inspectToolSchemas`,
  `resolveReasoningOutputMode`, `prepareExtraParams`, `createStreamFn`,
  `wrapStreamFn`, `resolveTransportTurnState`,
  `resolveWebSocketSessionPolicy`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `isCacheTtlEligible`,
  `buildMissingAuthMessage`, `suppressBuiltInModel`, `augmentModelCatalog`,
  `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `isModernModelRef`, `prepareRuntimeAuth`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `createEmbeddingProvider`,
  `buildReplayPolicy`,
  `sanitizeReplayHistory`, `validateReplayTurns`, `onModelSelected`

OpenClaw tetap memiliki loop agen generik, failover, penanganan transkrip, dan
kebijakan alat. Hook-hook ini adalah surface ekstensi untuk perilaku khusus provider tanpa
memerlukan seluruh transport inferensi kustom.

Gunakan manifest `providerAuthEnvVars` ketika provider memiliki kredensial berbasis env
yang harus terlihat oleh jalur auth/status/model-picker generik tanpa memuat runtime
plugin. Gunakan manifest `providerAuthAliases` ketika satu provider id harus menggunakan ulang
env vars, profil auth, auth berbasis konfigurasi, dan pilihan onboarding API-key milik provider id lain. Gunakan manifest `providerAuthChoices` ketika surface CLI
onboarding/pilihan auth harus mengetahui choice id provider, label grup, dan wiring
auth satu-flag sederhana tanpa memuat runtime provider. Pertahankan runtime provider
`envVars` untuk petunjuk yang ditujukan ke operator seperti label onboarding atau
variabel setup OAuth client-id/client-secret.

Gunakan manifest `channelEnvVars` ketika sebuah saluran memiliki auth atau setup berbasis env yang
harus terlihat oleh fallback shell-env generik, pemeriksaan config/status, atau prompt setup
tanpa memuat runtime saluran.

### Urutan hook dan penggunaan

Untuk plugin model/provider, OpenClaw memanggil hook dalam urutan kasar berikut.
Kolom "Kapan digunakan" adalah panduan keputusan cepat.

| #   | Hook                              | Apa fungsinya                                                                                                   | Kapan digunakan                                                                                                                             |
| --- | --------------------------------- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Mempublikasikan konfigurasi provider ke `models.providers` selama pembuatan `models.json`                       | Provider memiliki katalog atau default base URL                                                                                             |
| 2   | `applyConfigDefaults`             | Menerapkan default konfigurasi global milik provider selama materialisasi konfigurasi                            | Default bergantung pada mode auth, env, atau semantik keluarga model provider                                                               |
| --  | _(lookup model bawaan)_           | OpenClaw mencoba jalur registri/katalog normal terlebih dahulu                                                  | _(bukan hook plugin)_                                                                                                                       |
| 3   | `normalizeModelId`                | Menormalkan alias model-id legacy atau preview sebelum lookup                                                   | Provider memiliki pembersihan alias sebelum resolusi model kanonis                                                                          |
| 4   | `normalizeTransport`              | Menormalkan `api` / `baseUrl` keluarga provider sebelum perakitan model generik                                 | Provider memiliki pembersihan transport untuk provider id kustom dalam keluarga transport yang sama                                         |
| 5   | `normalizeConfig`                 | Menormalkan `models.providers.<id>` sebelum resolusi runtime/provider                                           | Provider membutuhkan pembersihan konfigurasi yang harus berada bersama plugin; helper keluarga Google bundled juga menjadi backstop untuk entri konfigurasi Google yang didukung |
| 6   | `applyNativeStreamingUsageCompat` | Menerapkan penulisan ulang kompatibilitas native streaming-usage ke provider konfigurasi                        | Provider membutuhkan perbaikan metadata native streaming usage yang didorong endpoint                                                       |
| 7   | `resolveConfigApiKey`             | Me-resolve auth env-marker untuk provider konfigurasi sebelum pemuatan auth runtime                             | Provider memiliki resolusi API-key env-marker milik provider; `amazon-bedrock` juga memiliki resolver env-marker AWS bawaan di sini       |
| 8   | `resolveSyntheticAuth`            | Menampilkan auth lokal/self-hosted atau berbasis konfigurasi tanpa menyimpan plaintext                          | Provider dapat beroperasi dengan penanda kredensial sintetis/lokal                                                                         |
| 9   | `resolveExternalAuthProfiles`     | Menimpa profil auth eksternal milik provider; default `persistence` adalah `runtime-only` untuk kredensial milik CLI/app | Provider menggunakan ulang kredensial auth eksternal tanpa menyimpan refresh token hasil salinan                                           |
| 10  | `shouldDeferSyntheticProfileAuth` | Menurunkan prioritas placeholder profil sintetis yang tersimpan di bawah auth berbasis env/konfigurasi         | Provider menyimpan profil placeholder sintetis yang seharusnya tidak menang dalam urutan prioritas                                        |
| 11  | `resolveDynamicModel`             | Fallback sinkron untuk model id milik provider yang belum ada di registri lokal                                 | Provider menerima model id upstream arbitrer                                                                                                |
| 12  | `prepareDynamicModel`             | Warm-up async, lalu `resolveDynamicModel` dijalankan lagi                                                       | Provider membutuhkan metadata jaringan sebelum me-resolve id yang tidak dikenal                                                             |
| 13  | `normalizeResolvedModel`          | Penulisan ulang akhir sebelum embedded runner menggunakan model yang telah di-resolve                           | Provider membutuhkan penulisan ulang transport tetapi tetap menggunakan transport core                                                      |
| 14  | `contributeResolvedModelCompat`   | Menambahkan flag kompatibilitas untuk model vendor di balik transport kompatibel lain                           | Provider mengenali model miliknya sendiri pada transport proxy tanpa mengambil alih provider                                                |
| 15  | `capabilities`                    | Metadata transkrip/tooling milik provider yang digunakan oleh logika core bersama                               | Provider membutuhkan quirk transkrip/keluarga provider                                                                                      |
| 16  | `normalizeToolSchemas`            | Menormalkan schema alat sebelum embedded runner melihatnya                                                      | Provider membutuhkan pembersihan schema keluarga transport                                                                                  |
| 17  | `inspectToolSchemas`              | Menampilkan diagnostik schema milik provider setelah normalisasi                                                | Provider menginginkan peringatan keyword tanpa mengajarkan aturan khusus provider ke core                                                  |
| 18  | `resolveReasoningOutputMode`      | Memilih kontrak keluaran reasoning native vs bertag                                                             | Provider membutuhkan reasoning bertag/output akhir alih-alih field native                                                                  |
| 19  | `prepareExtraParams`              | Normalisasi parameter permintaan sebelum wrapper opsi stream generik                                            | Provider membutuhkan parameter permintaan default atau pembersihan parameter per-provider                                                   |
| 20  | `createStreamFn`                  | Sepenuhnya mengganti jalur stream normal dengan transport kustom                                                | Provider membutuhkan protokol wire kustom, bukan sekadar wrapper                                                                           |
| 21  | `wrapStreamFn`                    | Wrapper stream setelah wrapper generik diterapkan                                                               | Provider membutuhkan wrapper kompatibilitas header/body/model permintaan tanpa transport kustom                                            |
| 22  | `resolveTransportTurnState`       | Melampirkan header atau metadata transport native per-turn                                                      | Provider ingin transport generik mengirim identitas turn native milik provider                                                              |
| 23  | `resolveWebSocketSessionPolicy`   | Melampirkan header WebSocket native atau kebijakan cool-down sesi                                               | Provider ingin transport WS generik menyetel header sesi atau kebijakan fallback                                                           |
| 24  | `formatApiKey`                    | Formatter profil auth: profil yang disimpan menjadi string `apiKey` runtime                                     | Provider menyimpan metadata auth tambahan dan membutuhkan bentuk token runtime kustom                                                      |
| 25  | `refreshOAuth`                    | Override refresh OAuth untuk endpoint refresh kustom atau kebijakan kegagalan refresh                           | Provider tidak cocok dengan refresher `pi-ai` bersama                                                                                       |
| 26  | `buildAuthDoctorHint`             | Petunjuk perbaikan yang ditambahkan saat refresh OAuth gagal                                                    | Provider membutuhkan panduan perbaikan auth milik provider setelah kegagalan refresh                                                       |
| 27  | `matchesContextOverflowError`     | Matcher overflow context-window milik provider                                                                  | Provider memiliki error overflow mentah yang akan terlewat oleh heuristik generik                                                          |
| 28  | `classifyFailoverReason`          | Klasifikasi alasan failover milik provider                                                                      | Provider dapat memetakan error API/transport mentah ke rate-limit/overload/dll.                                                           |
| 29  | `isCacheTtlEligible`              | Kebijakan prompt-cache untuk provider proxy/backhaul                                                            | Provider membutuhkan pengaturan cache TTL khusus proxy                                                                                      |
| 30  | `buildMissingAuthMessage`         | Pengganti pesan pemulihan missing-auth generik                                                                  | Provider membutuhkan petunjuk pemulihan missing-auth khusus provider                                                                        |
| 31  | `suppressBuiltInModel`            | Penekanan model upstream usang ditambah petunjuk error opsional yang terlihat oleh pengguna                     | Provider perlu menyembunyikan baris upstream usang atau menggantinya dengan petunjuk vendor                                                |
| 32  | `augmentModelCatalog`             | Baris katalog sintetis/akhir yang ditambahkan setelah discovery                                                 | Provider membutuhkan baris forward-compat sintetis di `models list` dan picker                                                             |
| 33  | `isBinaryThinking`                | Toggle reasoning on/off untuk provider binary-thinking                                                          | Provider hanya mengekspos binary thinking nyala/mati                                                                                        |
| 34  | `supportsXHighThinking`           | Dukungan reasoning `xhigh` untuk model tertentu                                                                 | Provider menginginkan `xhigh` hanya pada sebagian model                                                                                     |
| 35  | `resolveDefaultThinkingLevel`     | Level `/think` default untuk keluarga model tertentu                                                            | Provider memiliki kebijakan `/think` default untuk sebuah keluarga model                                                                    |
| 36  | `isModernModelRef`                | Matcher modern-model untuk filter live profile dan pemilihan smoke                                              | Provider memiliki pencocokan preferred-model live/smoke                                                                                     |
| 37  | `prepareRuntimeAuth`              | Menukar kredensial yang dikonfigurasi menjadi token/kunci runtime yang sebenarnya tepat sebelum inferensi       | Provider membutuhkan pertukaran token atau kredensial permintaan berumur pendek                                                             |
| 38  | `resolveUsageAuth`                | Me-resolve kredensial usage/billing untuk `/usage` dan surface status terkait                                  | Provider membutuhkan parsing token usage/kuota kustom atau kredensial usage yang berbeda                                                   |
| 39  | `fetchUsageSnapshot`              | Mengambil dan menormalkan snapshot usage/kuota khusus provider setelah auth di-resolve                         | Provider membutuhkan endpoint usage atau parser payload khusus provider                                                                     |
| 40  | `createEmbeddingProvider`         | Membangun adapter embedding milik provider untuk memory/search                                                 | Perilaku embedding memory menjadi milik plugin provider                                                                                    |
| 41  | `buildReplayPolicy`               | Mengembalikan kebijakan replay yang mengendalikan penanganan transkrip untuk provider                          | Provider membutuhkan kebijakan transkrip kustom (misalnya, menghapus blok thinking)                                                       |
| 42  | `sanitizeReplayHistory`           | Menulis ulang riwayat replay setelah pembersihan transkrip generik                                             | Provider membutuhkan penulisan ulang replay khusus provider di luar helper pemadatan bersama                                               |
| 43  | `validateReplayTurns`             | Validasi atau pembentukan ulang replay-turn akhir sebelum embedded runner                                      | Transport provider membutuhkan validasi turn yang lebih ketat setelah sanitasi generik                                                     |
| 44  | `onModelSelected`                 | Menjalankan efek samping pasca-pemilihan milik provider                                                        | Provider membutuhkan telemetri atau state milik provider saat sebuah model menjadi aktif                                                   |

`normalizeModelId`, `normalizeTransport`, dan `normalizeConfig` pertama-tama memeriksa
plugin provider yang cocok, lalu meneruskan ke plugin provider lain yang mendukung hook
sampai salah satunya benar-benar mengubah model id atau transport/config. Ini menjaga
shim alias/compat provider tetap berfungsi tanpa mengharuskan pemanggil mengetahui plugin bundled mana
yang memiliki penulisan ulang tersebut. Jika tidak ada hook provider yang menulis ulang
entri konfigurasi keluarga Google yang didukung, penormalisasi konfigurasi Google bundled tetap menerapkan
pembersihan kompatibilitas tersebut.

Jika provider membutuhkan protokol wire yang sepenuhnya kustom atau eksekutor permintaan kustom,
itu adalah kelas ekstensi yang berbeda. Hook-hook ini ditujukan untuk perilaku provider
yang tetap berjalan pada loop inferensi normal OpenClaw.

### Contoh provider

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Contoh bawaan

- Anthropic menggunakan `resolveDynamicModel`, `capabilities`, `buildAuthDoctorHint`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `isCacheTtlEligible`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  dan `wrapStreamFn` karena plugin ini memiliki forward-compat Claude 4.6,
  petunjuk keluarga provider, panduan perbaikan auth, integrasi endpoint usage,
  kelayakan prompt-cache, default konfigurasi yang sadar auth, kebijakan
  thinking default/adaptif Claude, dan pembentukan stream khusus Anthropic untuk
  header beta, `/fast` / `serviceTier`, dan `context1m`.
- Helper stream khusus Claude milik Anthropic untuk saat ini tetap berada di
  seam publik `api.ts` / `contract-api.ts` milik plugin bundled itu sendiri. Surface package
  tersebut mengekspor `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, dan builder wrapper
  Anthropic level lebih rendah alih-alih memperluas SDK generik di sekitar aturan
  beta-header milik satu provider.
- OpenAI menggunakan `resolveDynamicModel`, `normalizeResolvedModel`, dan
  `capabilities` ditambah `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking`, dan `isModernModelRef`
  karena plugin ini memiliki forward-compat GPT-5.4, normalisasi langsung OpenAI
  `openai-completions` -> `openai-responses`, petunjuk auth yang sadar Codex,
  penekanan Spark, baris daftar OpenAI sintetis, dan kebijakan thinking /
  live-model GPT-5; keluarga stream `openai-responses-defaults` memiliki
  wrapper OpenAI Responses native bersama untuk header atribusi,
  `/fast`/`serviceTier`, verbositas teks, pencarian web Codex native,
  pembentukan payload reasoning-compat, dan manajemen konteks Responses.
- OpenRouter menggunakan `catalog` ditambah `resolveDynamicModel` dan
  `prepareDynamicModel` karena provider ini bersifat pass-through dan dapat mengekspos
  model id baru sebelum katalog statis OpenClaw diperbarui; plugin ini juga menggunakan
  `capabilities`, `wrapStreamFn`, dan `isCacheTtlEligible` agar
  header permintaan, metadata routing, patch reasoning, dan
  kebijakan prompt-cache khusus provider tetap berada di luar core. Kebijakan replay-nya berasal dari
  keluarga `passthrough-gemini`, sedangkan keluarga stream `openrouter-thinking`
  memiliki injeksi reasoning proxy dan skip untuk model yang tidak didukung / `auto`.
- GitHub Copilot menggunakan `catalog`, `auth`, `resolveDynamicModel`, dan
  `capabilities` ditambah `prepareRuntimeAuth` dan `fetchUsageSnapshot` karena plugin ini
  membutuhkan login perangkat milik provider, perilaku fallback model, quirk transkrip Claude,
  pertukaran token GitHub -> token Copilot, dan endpoint usage milik provider.
- OpenAI Codex menggunakan `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth`, dan `augmentModelCatalog` ditambah
  `prepareExtraParams`, `resolveUsageAuth`, dan `fetchUsageSnapshot` karena plugin ini
  masih berjalan pada transport OpenAI core tetapi memiliki normalisasi
  transport/base URL sendiri, kebijakan fallback refresh OAuth, pilihan transport default,
  baris katalog Codex sintetis, dan integrasi endpoint usage ChatGPT; plugin ini
  berbagi keluarga stream `openai-responses-defaults` yang sama dengan OpenAI langsung.
- Google AI Studio dan Gemini CLI OAuth menggunakan `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn`, dan `isModernModelRef` karena
  keluarga replay `google-gemini` memiliki fallback forward-compat Gemini 3.1,
  validasi replay Gemini native, sanitasi replay bootstrap, mode
  keluaran reasoning bertag, dan pencocokan modern-model, sedangkan
  keluarga stream `google-thinking` memiliki normalisasi payload thinking Gemini;
  Gemini CLI OAuth juga menggunakan `formatApiKey`, `resolveUsageAuth`, dan
  `fetchUsageSnapshot` untuk pemformatan token, parsing token, dan wiring endpoint
  kuota.
- Anthropic Vertex menggunakan `buildReplayPolicy` melalui
  keluarga replay `anthropic-by-model` sehingga pembersihan replay khusus Claude tetap
  terlingkup pada id Claude alih-alih setiap transport `anthropic-messages`.
- Amazon Bedrock menggunakan `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason`, dan `resolveDefaultThinkingLevel` karena plugin ini memiliki
  klasifikasi error throttle/not-ready/context-overflow khusus Bedrock
  untuk trafik Anthropic-on-Bedrock; kebijakan replay-nya tetap berbagi
  guard `anthropic-by-model` khusus Claude yang sama.
- OpenRouter, Kilocode, Opencode, dan Opencode Go menggunakan `buildReplayPolicy`
  melalui keluarga replay `passthrough-gemini` karena mereka mem-proxy model Gemini
  melalui transport yang kompatibel dengan OpenAI dan membutuhkan
  sanitasi thought-signature Gemini tanpa validasi replay Gemini native atau
  penulisan ulang bootstrap.
- MiniMax menggunakan `buildReplayPolicy` melalui
  keluarga replay `hybrid-anthropic-openai` karena satu provider memiliki semantik
  Anthropic-message dan OpenAI-compatible sekaligus; plugin ini mempertahankan
  penghapusan thinking-block khusus Claude di sisi Anthropic sambil menimpa mode keluaran
  reasoning kembali ke native, dan keluarga stream `minimax-fast-mode` memiliki
  penulisan ulang model fast-mode pada jalur stream bersama.
- Moonshot menggunakan `catalog` ditambah `wrapStreamFn` karena plugin ini masih menggunakan
  transport OpenAI bersama tetapi membutuhkan normalisasi payload thinking milik provider; keluarga
  stream `moonshot-thinking` memetakan konfigurasi ditambah state `/think` ke payload
  binary thinking native.
- Kilocode menggunakan `catalog`, `capabilities`, `wrapStreamFn`, dan
  `isCacheTtlEligible` karena plugin ini membutuhkan header permintaan milik provider,
  normalisasi payload reasoning, petunjuk transkrip Gemini, dan pengaturan
  cache-TTL Anthropic; keluarga stream `kilocode-thinking` menjaga injeksi thinking Kilo
  tetap pada jalur stream proxy bersama sambil melewati `kilo/auto` dan
  model id proxy lain yang tidak mendukung payload reasoning eksplisit.
- Z.AI menggunakan `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth`, dan `fetchUsageSnapshot` karena plugin ini memiliki fallback GLM-5,
  default `tool_stream`, UX binary thinking, pencocokan modern-model, dan keduanya:
  auth usage + pengambilan kuota; keluarga stream `tool-stream-default-on` menjaga
  wrapper `tool_stream` yang aktif secara default tetap berada di luar glue tulisan tangan per-provider.
- xAI menggunakan `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel`, dan `isModernModelRef`
  karena plugin ini memiliki normalisasi transport xAI Responses native, penulisan ulang alias
  fast-mode Grok, default `tool_stream`, pembersihan strict-tool / payload reasoning,
  penggunaan ulang auth fallback untuk alat milik plugin, resolusi model Grok
  forward-compat, dan patch kompatibilitas milik provider seperti profil tool-schema xAI,
  keyword schema yang tidak didukung, `web_search` native, dan decoding argumen
  tool-call entitas HTML.
- Mistral, OpenCode Zen, dan OpenCode Go hanya menggunakan `capabilities` untuk menjaga
  quirk transkrip/tooling tetap berada di luar core.
- Provider bundled khusus katalog seperti `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway`, dan `volcengine` hanya menggunakan
  `catalog`.
- Qwen menggunakan `catalog` untuk provider teksnya ditambah registrasi
  media-understanding dan video-generation bersama untuk surface multimodalnya.
- MiniMax dan Xiaomi menggunakan `catalog` ditambah hook usage karena perilaku `/usage`
  mereka dimiliki plugin meskipun inferensi tetap berjalan melalui transport bersama.

## Helper runtime

Plugin dapat mengakses helper core terpilih melalui `api.runtime`. Untuk TTS:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Catatan:

- `textToSpeech` mengembalikan payload keluaran TTS core normal untuk surface file/voice-note.
- Menggunakan konfigurasi core `messages.tts` dan pemilihan provider.
- Mengembalikan buffer audio PCM + sample rate. Plugin harus melakukan resample/encode untuk provider.
- `listVoices` bersifat opsional per provider. Gunakan untuk voice picker milik vendor atau alur setup.
- Daftar suara dapat menyertakan metadata yang lebih kaya seperti locale, gender, dan tag personality untuk picker yang sadar provider.
- OpenAI dan ElevenLabs saat ini mendukung teleponi. Microsoft tidak.

Plugin juga dapat mendaftarkan provider ucapan melalui `api.registerSpeechProvider(...)`.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Catatan:

- Pertahankan kebijakan TTS, fallback, dan pengiriman balasan di core.
- Gunakan provider ucapan untuk perilaku sintesis milik vendor.
- Input legacy Microsoft `edge` dinormalisasi ke provider id `microsoft`.
- Model kepemilikan yang diutamakan berorientasi pada perusahaan: satu plugin vendor dapat memiliki
  provider teks, ucapan, gambar, dan media di masa depan saat OpenClaw menambahkan
  kontrak kapabilitas tersebut.

Untuk pemahaman gambar/audio/video, plugin mendaftarkan satu provider
media-understanding bertipe alih-alih bag key/value generik:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Catatan:

- Pertahankan orkestrasi, fallback, konfigurasi, dan wiring saluran di core.
- Pertahankan perilaku vendor di plugin provider.
- Ekspansi aditif harus tetap bertipe: metode opsional baru, field hasil opsional
  baru, kapabilitas opsional baru.
- Pembuatan video sudah mengikuti pola yang sama:
  - core memiliki kontrak kapabilitas dan helper runtime
  - plugin vendor mendaftarkan `api.registerVideoGenerationProvider(...)`
  - plugin fitur/saluran menggunakan `api.runtime.videoGeneration.*`

Untuk helper runtime media-understanding, plugin dapat memanggil:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

Untuk transkripsi audio, plugin dapat menggunakan runtime media-understanding
atau alias STT yang lebih lama:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Catatan:

- `api.runtime.mediaUnderstanding.*` adalah surface bersama yang diutamakan untuk
  pemahaman gambar/audio/video.
- Menggunakan konfigurasi audio media-understanding core (`tools.media.audio`) dan urutan fallback provider.
- Mengembalikan `{ text: undefined }` ketika tidak ada keluaran transkripsi yang dihasilkan (misalnya input dilewati/tidak didukung).
- `api.runtime.stt.transcribeAudioFile(...)` tetap tersedia sebagai alias kompatibilitas.

Plugin juga dapat meluncurkan eksekusi subagent latar belakang melalui `api.runtime.subagent`:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Catatan:

- `provider` dan `model` adalah override per-eksekusi yang opsional, bukan perubahan sesi persisten.
- OpenClaw hanya menghormati field override tersebut untuk pemanggil tepercaya.
- Untuk eksekusi fallback milik plugin, operator harus melakukan opt-in dengan `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Gunakan `plugins.entries.<id>.subagent.allowedModels` untuk membatasi plugin tepercaya ke target `provider/model` kanonis tertentu, atau `"*"` untuk secara eksplisit mengizinkan target apa pun.
- Eksekusi subagent plugin yang tidak tepercaya tetap berfungsi, tetapi permintaan override ditolak alih-alih diam-diam fallback.

Untuk pencarian web, plugin dapat menggunakan helper runtime bersama alih-alih
menjangkau wiring alat agen:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Plugin juga dapat mendaftarkan provider web-search melalui
`api.registerWebSearchProvider(...)`.

Catatan:

- Pertahankan pemilihan provider, resolusi kredensial, dan semantik permintaan bersama di core.
- Gunakan provider web-search untuk transport pencarian khusus vendor.
- `api.runtime.webSearch.*` adalah surface bersama yang diutamakan untuk plugin fitur/saluran yang membutuhkan perilaku pencarian tanpa bergantung pada wrapper alat agen.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: menghasilkan gambar menggunakan rantai provider image-generation yang dikonfigurasi.
- `listProviders(...)`: menampilkan provider image-generation yang tersedia dan kapabilitasnya.

## Rute HTTP Gateway

Plugin dapat mengekspos endpoint HTTP dengan `api.registerHttpRoute(...)`.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Field rute:

- `path`: path rute di bawah server HTTP gateway.
- `auth`: wajib. Gunakan `"gateway"` untuk memerlukan auth gateway normal, atau `"plugin"` untuk auth/verifikasi webhook yang dikelola plugin.
- `match`: opsional. `"exact"` (default) atau `"prefix"`.
- `replaceExisting`: opsional. Memungkinkan plugin yang sama mengganti registrasi rute miliknya sendiri yang sudah ada.
- `handler`: kembalikan `true` ketika rute menangani permintaan.

Catatan:

- `api.registerHttpHandler(...)` telah dihapus dan akan menyebabkan error saat pemuatan plugin. Gunakan `api.registerHttpRoute(...)` sebagai gantinya.
- Rute plugin harus mendeklarasikan `auth` secara eksplisit.
- Konflik `path + match` yang sama ditolak kecuali `replaceExisting: true`, dan satu plugin tidak dapat mengganti rute milik plugin lain.
- Rute yang tumpang tindih dengan level `auth` berbeda ditolak. Pertahankan rantai fallthrough `exact`/`prefix` hanya pada level auth yang sama.
- Rute `auth: "plugin"` **tidak** menerima scope runtime operator secara otomatis. Rute ini ditujukan untuk webhook/verifikasi signature yang dikelola plugin, bukan panggilan helper Gateway berprivileg.
- Rute `auth: "gateway"` berjalan di dalam scope runtime permintaan Gateway, tetapi scope itu sengaja konservatif:
  - auth bearer shared-secret (`gateway.auth.mode = "token"` / `"password"`) menjaga scope runtime rute plugin tetap terpasang pada `operator.write`, bahkan jika pemanggil mengirim `x-openclaw-scopes`
  - mode HTTP tepercaya yang membawa identitas (misalnya `trusted-proxy` atau `gateway.auth.mode = "none"` pada ingress privat) hanya menghormati `x-openclaw-scopes` ketika header tersebut memang ada
  - jika `x-openclaw-scopes` tidak ada pada permintaan rute plugin yang membawa identitas tersebut, scope runtime akan fallback ke `operator.write`
- Aturan praktis: jangan menganggap rute plugin dengan auth gateway sebagai surface admin implisit. Jika rute Anda membutuhkan perilaku admin-only, wajibkan mode auth yang membawa identitas dan dokumentasikan kontrak header `x-openclaw-scopes` yang eksplisit.

## Path impor Plugin SDK

Gunakan subpath SDK alih-alih impor `openclaw/plugin-sdk` monolitik saat
menulis plugin:

- `openclaw/plugin-sdk/plugin-entry` untuk primitif registrasi plugin.
- `openclaw/plugin-sdk/core` untuk kontrak generic shared yang menghadap plugin.
- `openclaw/plugin-sdk/config-schema` untuk ekspor schema Zod root `openclaw.json`
  (`OpenClawSchema`).
- Primitif saluran stabil seperti `openclaw/plugin-sdk/channel-setup`,
  `openclaw/plugin-sdk/setup-runtime`,
  `openclaw/plugin-sdk/setup-adapter-runtime`,
  `openclaw/plugin-sdk/setup-tools`,
  `openclaw/plugin-sdk/channel-pairing`,
  `openclaw/plugin-sdk/channel-contract`,
  `openclaw/plugin-sdk/channel-feedback`,
  `openclaw/plugin-sdk/channel-inbound`,
  `openclaw/plugin-sdk/channel-lifecycle`,
  `openclaw/plugin-sdk/channel-reply-pipeline`,
  `openclaw/plugin-sdk/command-auth`,
  `openclaw/plugin-sdk/secret-input`, dan
  `openclaw/plugin-sdk/webhook-ingress` untuk wiring setup/auth/reply/webhook
  bersama. `channel-inbound` adalah rumah bersama untuk debounce, mention matching,
  helper inbound mention-policy, pemformatan envelope, dan helper konteks
  inbound envelope.
  `channel-setup` adalah seam setup optional-install yang sempit.
  `setup-runtime` adalah surface setup yang aman untuk runtime yang digunakan oleh `setupEntry` /
  startup tertunda, termasuk adapter patch setup yang aman untuk impor.
  `setup-adapter-runtime` adalah seam adapter account-setup yang sadar env.
  `setup-tools` adalah seam helper CLI/arsip/dokumen kecil (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Subpath domain seperti `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-gateway-runtime`,
  `openclaw/plugin-sdk/approval-handler-adapter-runtime`,
  `openclaw/plugin-sdk/approval-handler-runtime`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store`, dan
  `openclaw/plugin-sdk/directory-runtime` untuk helper runtime/konfigurasi bersama.
  `telegram-command-config` adalah seam publik sempit untuk normalisasi/validasi perintah
  kustom Telegram dan tetap tersedia meskipun
  surface kontrak Telegram bundled sementara tidak tersedia.
  `text-runtime` adalah seam teks/markdown/logging bersama, termasuk
  penghapusan teks yang terlihat oleh asisten, helper render/chunking markdown, helper redaction,
  helper directive-tag, dan utilitas safe-text.
- Seam saluran khusus approval sebaiknya memilih satu kontrak `approvalCapability`
  pada plugin. Core kemudian membaca auth approval, pengiriman, render,
  native-routing, dan perilaku lazy native-handler melalui satu kapabilitas itu
  alih-alih mencampur perilaku approval ke field plugin yang tidak terkait.
- `openclaw/plugin-sdk/channel-runtime` sudah deprecated dan hanya tersisa sebagai
  shim kompatibilitas untuk plugin lama. Kode baru sebaiknya mengimpor primitif generik yang lebih sempit,
  dan kode repo tidak boleh menambahkan impor baru dari
  shim tersebut.
- Internal ekstensi bundled tetap privat. Plugin eksternal sebaiknya hanya menggunakan
  subpath `openclaw/plugin-sdk/*`. Kode core/test OpenClaw dapat menggunakan entry point publik repo
  di bawah root package plugin seperti `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js`, dan file yang di-scope sempit seperti
  `login-qr-api.js`. Jangan pernah mengimpor `src/*` dari package plugin dari core atau dari
  ekstensi lain.
- Pembagian entry point repo:
  `<plugin-package-root>/api.js` adalah barrel helper/types,
  `<plugin-package-root>/runtime-api.js` adalah barrel khusus runtime,
  `<plugin-package-root>/index.js` adalah entry plugin bundled,
  dan `<plugin-package-root>/setup-entry.js` adalah entry plugin setup.
- Contoh provider bundled saat ini:
  - Anthropic menggunakan `api.js` / `contract-api.js` untuk helper stream Claude seperti
    `wrapAnthropicProviderStream`, helper beta-header, dan parsing `service_tier`.
  - OpenAI menggunakan `api.js` untuk builder provider, helper model default, dan
    builder provider realtime.
  - OpenRouter menggunakan `api.js` untuk builder provider plus helper onboarding/konfigurasi,
    sementara `register.runtime.js` masih dapat mengekspor ulang helper generik
    `plugin-sdk/provider-stream` untuk penggunaan lokal repo.
- Entry point publik yang dimuat melalui facade mengutamakan snapshot config runtime aktif
  jika ada, lalu fallback ke file konfigurasi yang telah di-resolve di disk ketika
  OpenClaw belum menyajikan snapshot runtime.
- Primitif bersama generik tetap menjadi kontrak SDK publik yang diutamakan. Sekelompok kecil
  seam helper bermerek saluran bundled yang dicadangkan untuk kompatibilitas masih
  ada. Perlakukan itu sebagai seam pemeliharaan/kompatibilitas bundled, bukan target impor pihak ketiga baru; kontrak lintas saluran baru tetap harus masuk ke
  subpath generik `plugin-sdk/*` atau barrel lokal plugin `api.js` /
  `runtime-api.js`.

Catatan kompatibilitas:

- Hindari barrel root `openclaw/plugin-sdk` untuk kode baru.
- Utamakan primitif stabil yang sempit terlebih dahulu. Subpath setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool yang lebih baru adalah kontrak yang dituju untuk pekerjaan
  plugin bundled dan eksternal yang baru.
  Parsing/pencocokan target harus berada di `openclaw/plugin-sdk/channel-targets`.
  Gerbang aksi pesan dan helper reaction message-id harus berada di
  `openclaw/plugin-sdk/channel-actions`.
- Barrel helper khusus ekstensi bundled tidak stabil secara default. Jika sebuah
  helper hanya dibutuhkan oleh ekstensi bundled, pertahankan helper itu di belakang
  seam lokal `api.js` atau `runtime-api.js` milik ekstensi tersebut alih-alih menaikkannya ke
  `openclaw/plugin-sdk/<extension>`.
- Seam helper bersama yang baru sebaiknya generik, bukan bermerek saluran. Parsing target bersama
  berada di `openclaw/plugin-sdk/channel-targets`; internal khusus saluran
  tetap berada di belakang seam lokal `api.js` atau `runtime-api.js` milik plugin tersebut.
- Subpath khusus kapabilitas seperti `image-generation`,
  `media-understanding`, dan `speech` ada karena plugin bundled/native menggunakannya
  saat ini. Kehadirannya sendiri tidak otomatis berarti setiap helper yang diekspor adalah
  kontrak eksternal jangka panjang yang dibekukan.

## Schema alat message

Plugin sebaiknya memiliki kontribusi schema `describeMessageTool(...)` yang khusus saluran.
Pertahankan field khusus provider di plugin, bukan di core bersama.

Untuk fragmen schema portabel bersama, gunakan kembali helper generik yang diekspor melalui
`openclaw/plugin-sdk/channel-actions`:

- `createMessageToolButtonsSchema()` untuk payload gaya grid tombol
- `createMessageToolCardSchema()` untuk payload kartu terstruktur

Jika bentuk schema hanya masuk akal untuk satu provider, definisikan di source
plugin tersebut sendiri alih-alih menaikkannya ke SDK bersama.

## Resolusi target saluran

Plugin saluran sebaiknya memiliki semantik target khusus saluran. Pertahankan host
outbound bersama tetap generik dan gunakan surface adapter perpesanan untuk aturan provider:

- `messaging.inferTargetChatType({ to })` memutuskan apakah target yang telah dinormalisasi
  harus diperlakukan sebagai `direct`, `group`, atau `channel` sebelum lookup direktori.
- `messaging.targetResolver.looksLikeId(raw, normalized)` memberi tahu core apakah sebuah
  input harus langsung melewati ke resolusi mirip-id alih-alih pencarian direktori.
- `messaging.targetResolver.resolveTarget(...)` adalah fallback plugin ketika
  core membutuhkan resolusi akhir milik provider setelah normalisasi atau setelah
  direktori tidak menemukan hasil.
- `messaging.resolveOutboundSessionRoute(...)` memiliki konstruksi rute sesi
  khusus provider setelah sebuah target di-resolve.

Pemisahan yang direkomendasikan:

- Gunakan `inferTargetChatType` untuk keputusan kategori yang seharusnya terjadi sebelum
  mencari peer/group.
- Gunakan `looksLikeId` untuk pemeriksaan "perlakukan ini sebagai id target eksplisit/native".
- Gunakan `resolveTarget` untuk fallback normalisasi khusus provider, bukan untuk
  pencarian direktori yang luas.
- Pertahankan id native provider seperti chat id, thread id, JID, handle, dan room
  id di dalam nilai `target` atau parameter khusus provider, bukan di field SDK generik.

## Direktori berbasis konfigurasi

Plugin yang menurunkan entri direktori dari konfigurasi sebaiknya menjaga logika itu di
plugin dan menggunakan kembali helper bersama dari
`openclaw/plugin-sdk/directory-runtime`.

Gunakan ini ketika sebuah saluran membutuhkan peer/group berbasis konfigurasi seperti:

- peer DM yang digerakkan oleh allowlist
- peta channel/group yang dikonfigurasi
- fallback direktori statis yang di-scope ke akun

Helper bersama di `directory-runtime` hanya menangani operasi generik:

- pemfilteran query
- penerapan limit
- helper deduplikasi/normalisasi
- membangun `ChannelDirectoryEntry[]`

Inspeksi akun khusus saluran dan normalisasi id sebaiknya tetap berada di
implementasi plugin.

## Katalog provider

Plugin provider dapat mendefinisikan katalog model untuk inferensi dengan
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` mengembalikan bentuk yang sama dengan yang ditulis OpenClaw ke
`models.providers`:

- `{ provider }` untuk satu entri provider
- `{ providers }` untuk beberapa entri provider

Gunakan `catalog` ketika plugin memiliki model id, default base URL, atau metadata
model yang bergantung pada auth dan khusus provider.

`catalog.order` mengontrol kapan katalog sebuah plugin di-merge relatif terhadap
provider implisit bawaan OpenClaw:

- `simple`: provider berbasis API-key atau env biasa
- `profile`: provider yang muncul ketika profil auth ada
- `paired`: provider yang mensintesis beberapa entri provider terkait
- `late`: lintasan terakhir, setelah provider implisit lainnya

Provider yang lebih akhir menang pada tabrakan key, sehingga plugin dapat secara sengaja
menimpa entri provider bawaan dengan provider id yang sama.

Kompatibilitas:

- `discovery` masih berfungsi sebagai alias legacy
- jika `catalog` dan `discovery` keduanya didaftarkan, OpenClaw menggunakan `catalog`

## Inspeksi saluran read-only

Jika plugin Anda mendaftarkan sebuah saluran, utamakan implementasi
`plugin.config.inspectAccount(cfg, accountId)` di samping `resolveAccount(...)`.

Mengapa:

- `resolveAccount(...)` adalah jalur runtime. Jalur ini boleh mengasumsikan bahwa kredensial
  sudah sepenuhnya dimaterialisasi dan dapat gagal cepat ketika secret yang diperlukan tidak ada.
- Jalur perintah read-only seperti `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve`, dan alur doctor/perbaikan
  konfigurasi tidak seharusnya perlu mematerialisasi kredensial runtime hanya untuk
  menjelaskan konfigurasi.

Perilaku `inspectAccount(...)` yang direkomendasikan:

- Hanya kembalikan state akun yang deskriptif.
- Pertahankan `enabled` dan `configured`.
- Sertakan field sumber/status kredensial jika relevan, seperti:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Anda tidak perlu mengembalikan nilai token mentah hanya untuk melaporkan ketersediaan
  read-only. Mengembalikan `tokenStatus: "available"` (dan field source yang cocok)
  sudah cukup untuk perintah bergaya status.
- Gunakan `configured_unavailable` ketika sebuah kredensial dikonfigurasi melalui SecretRef tetapi
  tidak tersedia pada jalur perintah saat ini.

Ini memungkinkan perintah read-only melaporkan "dikonfigurasi tetapi tidak tersedia pada jalur perintah ini"
alih-alih crash atau salah melaporkan akun sebagai tidak dikonfigurasi.

## Package pack

Sebuah direktori plugin dapat menyertakan `package.json` dengan `openclaw.extensions`:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Setiap entri menjadi sebuah plugin. Jika pack mencantumkan beberapa extension, plugin id
menjadi `name/<fileBase>`.

Jika plugin Anda mengimpor dependensi npm, instal dependensi tersebut di direktori itu agar
`node_modules` tersedia (`npm install` / `pnpm install`).

Pagar pengaman keamanan: setiap entri `openclaw.extensions` harus tetap berada di dalam direktori plugin
setelah resolusi symlink. Entri yang keluar dari direktori package akan
ditolak.

Catatan keamanan: `openclaw plugins install` menginstal dependensi plugin dengan
`npm install --omit=dev --ignore-scripts` (tanpa lifecycle script, tanpa dev dependencies saat runtime). Jaga pohon dependensi plugin tetap "pure JS/TS" dan hindari package yang memerlukan build `postinstall`.

Opsional: `openclaw.setupEntry` dapat menunjuk ke modul ringan khusus setup.
Saat OpenClaw membutuhkan surface setup untuk plugin saluran yang dinonaktifkan, atau
ketika plugin saluran diaktifkan tetapi masih belum dikonfigurasi, OpenClaw memuat `setupEntry`
alih-alih entry plugin penuh. Ini menjaga startup dan setup tetap lebih ringan
ketika entry plugin utama Anda juga memasang alat, hook, atau kode lain yang hanya untuk runtime.

Opsional: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
dapat mengikutsertakan plugin saluran ke jalur `setupEntry` yang sama selama fase
startup pre-listen gateway, bahkan ketika saluran sudah dikonfigurasi.

Gunakan ini hanya ketika `setupEntry` sepenuhnya mencakup surface startup yang harus ada
sebelum gateway mulai mendengarkan. Dalam praktiknya, itu berarti entry setup
harus mendaftarkan setiap kapabilitas milik saluran yang dibutuhkan startup, seperti:

- registrasi saluran itu sendiri
- setiap rute HTTP yang harus tersedia sebelum gateway mulai mendengarkan
- setiap metode gateway, alat, atau layanan yang harus ada selama jendela yang sama

Jika entry penuh Anda masih memiliki kapabilitas startup yang wajib, jangan aktifkan
flag ini. Pertahankan plugin pada perilaku default dan biarkan OpenClaw memuat
entry penuh saat startup.

Saluran bundled juga dapat memublikasikan helper contract-surface khusus setup yang dapat
dikonsultasikan core sebelum runtime saluran penuh dimuat. Surface promosi setup saat ini
adalah:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Core menggunakan surface tersebut ketika perlu mempromosikan konfigurasi saluran
legacy akun tunggal ke `channels.<id>.accounts.*` tanpa memuat entry plugin penuh.
Matrix adalah contoh bundled saat ini: plugin ini hanya memindahkan key auth/bootstrap ke
akun bernama yang dipromosikan ketika named account sudah ada, dan plugin ini dapat
mempertahankan key default-account non-kanonis yang telah dikonfigurasi alih-alih selalu membuat
`accounts.default`.

Adapter patch setup tersebut menjaga discovery contract-surface bundled tetap lazy. Waktu impor
tetap ringan; surface promosi dimuat hanya saat pertama kali digunakan alih-alih
masuk kembali ke startup saluran bundled saat impor modul.

Ketika surface startup tersebut mencakup metode RPC gateway, pertahankan metode itu pada
prefiks khusus plugin. Namespace admin core (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) tetap dicadangkan dan selalu di-resolve
ke `operator.admin`, bahkan jika sebuah plugin meminta scope yang lebih sempit.

Contoh:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Metadata katalog saluran

Plugin saluran dapat mengiklankan metadata setup/discovery melalui `openclaw.channel` dan
petunjuk instalasi melalui `openclaw.install`. Ini menjaga data katalog core tetap kosong.

Contoh:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Chat self-hosted melalui bot webhook Nextcloud Talk.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Field `openclaw.channel` yang berguna di luar contoh minimal:

- `detailLabel`: label sekunder untuk surface katalog/status yang lebih kaya
- `docsLabel`: menimpa teks tautan untuk tautan dokumen
- `preferOver`: id plugin/saluran prioritas lebih rendah yang harus dikalahkan oleh entri katalog ini
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: kontrol copy surface pemilihan
- `markdownCapable`: menandai saluran sebagai mampu markdown untuk keputusan pemformatan outbound
- `exposure.configured`: sembunyikan saluran dari surface daftar saluran yang dikonfigurasi saat disetel ke `false`
- `exposure.setup`: sembunyikan saluran dari picker setup/konfigurasi interaktif saat disetel ke `false`
- `exposure.docs`: tandai saluran sebagai internal/privat untuk surface navigasi dokumen
- `showConfigured` / `showInSetup`: alias legacy masih diterima untuk kompatibilitas; utamakan `exposure`
- `quickstartAllowFrom`: ikutsertakan saluran ke alur `allowFrom` quickstart standar
- `forceAccountBinding`: wajibkan binding akun eksplisit bahkan ketika hanya ada satu akun
- `preferSessionLookupForAnnounceTarget`: utamakan lookup sesi saat me-resolve target announce

OpenClaw juga dapat menggabungkan **katalog saluran eksternal** (misalnya, ekspor registri MPM
). Letakkan file JSON di salah satu lokasi berikut:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Atau arahkan `OPENCLAW_PLUGIN_CATALOG_PATHS` (atau `OPENCLAW_MPM_CATALOG_PATHS`) ke
satu atau lebih file JSON (dipisahkan koma/titik koma/`PATH`). Setiap file harus
berisi `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. Parser juga menerima `"packages"` atau `"plugins"` sebagai alias legacy untuk key `"entries"`.

## Plugin mesin konteks

Plugin mesin konteks memiliki orkestrasi konteks sesi untuk ingest, assembly,
dan compaction. Daftarkan plugin tersebut dari plugin Anda dengan
`api.registerContextEngine(id, factory)`, lalu pilih mesin aktif dengan
`plugins.slots.contextEngine`.

Gunakan ini ketika plugin Anda perlu mengganti atau memperluas pipeline konteks default
alih-alih hanya menambahkan pencarian memory atau hook.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Jika mesin Anda **tidak** memiliki algoritme compaction, tetap implementasikan `compact()`
dan delegasikan secara eksplisit:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Menambahkan kapabilitas baru

Ketika sebuah plugin membutuhkan perilaku yang tidak cocok dengan API saat ini, jangan melewati
sistem plugin dengan akses privat langsung. Tambahkan kapabilitas yang belum ada.

Urutan yang direkomendasikan:

1. definisikan kontrak core
   Putuskan perilaku bersama apa yang harus dimiliki core: kebijakan, fallback, merge konfigurasi,
   lifecycle, semantik yang menghadap saluran, dan bentuk helper runtime.
2. tambahkan surface registrasi/runtime plugin yang bertipe
   Perluas `OpenClawPluginApi` dan/atau `api.runtime` dengan surface kapabilitas bertipe
   terkecil yang berguna.
3. hubungkan konsumen core + saluran/fitur
   Plugin saluran dan fitur harus menggunakan kapabilitas baru melalui core,
   bukan dengan mengimpor implementasi vendor secara langsung.
4. daftarkan implementasi vendor
   Plugin vendor kemudian mendaftarkan backend mereka terhadap kapabilitas tersebut.
5. tambahkan cakupan kontrak
   Tambahkan pengujian agar bentuk kepemilikan dan registrasi tetap eksplisit seiring waktu.

Beginilah OpenClaw tetap tegas tanpa menjadi hardcoded pada
sudut pandang satu provider. Lihat [Capability Cookbook](/id/plugins/architecture)
untuk checklist file yang konkret dan contoh yang sudah dikerjakan.

### Checklist kapabilitas

Ketika Anda menambahkan kapabilitas baru, implementasinya biasanya harus menyentuh
surface berikut secara bersamaan:

- tipe kontrak core di `src/<capability>/types.ts`
- helper runner/runtime core di `src/<capability>/runtime.ts`
- surface registrasi API plugin di `src/plugins/types.ts`
- wiring registri plugin di `src/plugins/registry.ts`
- eksposur runtime plugin di `src/plugins/runtime/*` ketika plugin fitur/saluran
  perlu menggunakannya
- helper capture/test di `src/test-utils/plugin-registration.ts`
- asersi kepemilikan/kontrak di `src/plugins/contracts/registry.ts`
- dokumen operator/plugin di `docs/`

Jika salah satu surface tersebut hilang, itu biasanya merupakan tanda bahwa kapabilitasnya
belum sepenuhnya terintegrasi.

### Template kapabilitas

Pola minimal:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Pola pengujian kontrak:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Ini menjaga aturannya tetap sederhana:

- core memiliki kontrak kapabilitas + orkestrasi
- plugin vendor memiliki implementasi vendor
- plugin fitur/saluran menggunakan helper runtime
- pengujian kontrak menjaga kepemilikan tetap eksplisit
