---
read_when:
    - Membangun atau men-debug plugin OpenClaw native
    - Memahami model capability plugin atau batas kepemilikan
    - Mengerjakan pipeline pemuatan plugin atau registry
    - Mengimplementasikan hook runtime provider atau plugin channel
sidebarTitle: Internals
summary: 'Internal plugin: model capability, kepemilikan, kontrak, pipeline pemuatan, dan helper runtime'
title: Internal Plugin
x-i18n:
    generated_at: "2026-04-06T03:11:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: d39158455701dedfb75f6c20b8c69fd36ed9841f1d92bed1915f448df57fd47b
    source_path: plugins/architecture.md
    workflow: 15
---

# Internal Plugin

<Info>
  Ini adalah **referensi arsitektur mendalam**. Untuk panduan praktis, lihat:
  - [Install and use plugins](/id/tools/plugin) — panduan pengguna
  - [Getting Started](/id/plugins/building-plugins) — tutorial plugin pertama
  - [Channel Plugins](/id/plugins/sdk-channel-plugins) — bangun channel perpesanan
  - [Provider Plugins](/id/plugins/sdk-provider-plugins) — bangun provider model
  - [SDK Overview](/id/plugins/sdk-overview) — peta import dan API registrasi
</Info>

Halaman ini membahas arsitektur internal sistem plugin OpenClaw.

## Model capability publik

Capability adalah model **plugin native** publik di dalam OpenClaw. Setiap
plugin OpenClaw native mendaftar terhadap satu atau lebih tipe capability:

| Capability             | Metode registrasi                                | Contoh plugin                       |
| ---------------------- | ------------------------------------------------ | ----------------------------------- |
| Inferensi teks         | `api.registerProvider(...)`                      | `openai`, `anthropic`               |
| Speech                 | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`           |
| Realtime transcription | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                            |
| Realtime voice         | `api.registerRealtimeVoiceProvider(...)`         | `openai`                            |
| Pemahaman media        | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                  |
| Pembuatan gambar       | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Pembuatan musik        | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                 |
| Pembuatan video        | `api.registerVideoGenerationProvider(...)`       | `qwen`                              |
| Web fetch              | `api.registerWebFetchProvider(...)`              | `firecrawl`                         |
| Web search             | `api.registerWebSearchProvider(...)`             | `google`                            |
| Channel / perpesanan   | `api.registerChannel(...)`                       | `msteams`, `matrix`                 |

Plugin yang mendaftarkan nol capability tetapi menyediakan hooks, tools, atau
services adalah plugin **legacy hook-only**. Pola itu masih sepenuhnya didukung.

### Sikap kompatibilitas eksternal

Model capability sudah ada di core dan digunakan oleh plugin bundled/native
hari ini, tetapi kompatibilitas plugin eksternal masih memerlukan standar yang
lebih ketat daripada "diekspor, maka dibekukan."

Panduan saat ini:

- **plugin eksternal yang sudah ada:** pertahankan integrasi berbasis hook tetap berfungsi; perlakukan
  ini sebagai baseline kompatibilitas
- **plugin bundled/native baru:** prioritaskan registrasi capability eksplisit daripada
  reach-in khusus vendor atau desain hook-only baru
- **plugin eksternal yang mengadopsi registrasi capability:** diizinkan, tetapi perlakukan
  permukaan helper spesifik capability sebagai sesuatu yang masih berkembang kecuali dokumen secara eksplisit menandai
  kontrak sebagai stabil

Aturan praktis:

- API registrasi capability adalah arah yang dituju
- hook legacy tetap menjadi jalur paling aman dari sisi tanpa-breakage untuk plugin eksternal selama
  transisi
- subpath helper yang diekspor tidak semuanya setara; prioritaskan kontrak sempit yang terdokumentasi,
  bukan ekspor helper insidental

### Bentuk plugin

OpenClaw mengklasifikasikan setiap plugin yang dimuat ke dalam sebuah bentuk berdasarkan
perilaku registrasi aktualnya (bukan hanya metadata statis):

- **plain-capability** -- mendaftarkan tepat satu tipe capability (misalnya plugin
  provider-only seperti `mistral`)
- **hybrid-capability** -- mendaftarkan beberapa tipe capability (misalnya
  `openai` memiliki inferensi teks, speech, pemahaman media, dan pembuatan
  gambar)
- **hook-only** -- hanya mendaftarkan hooks (typed atau custom), tanpa capabilities,
  tools, commands, atau services
- **non-capability** -- mendaftarkan tools, commands, services, atau routes tetapi tanpa
  capabilities

Gunakan `openclaw plugins inspect <id>` untuk melihat bentuk dan rincian
capability sebuah plugin. Lihat [CLI reference](/cli/plugins#inspect) untuk detail.

### Hook legacy

Hook `before_agent_start` tetap didukung sebagai jalur kompatibilitas untuk
plugin hook-only. Plugin nyata legacy masih bergantung padanya.

Arah ke depan:

- pertahankan agar tetap berfungsi
- dokumentasikan sebagai legacy
- prioritaskan `before_model_resolve` untuk pekerjaan override model/provider
- prioritaskan `before_prompt_build` untuk pekerjaan mutasi prompt
- hapus hanya setelah penggunaan nyata menurun dan cakupan fixture membuktikan keamanan migrasi

### Sinyal kompatibilitas

Saat Anda menjalankan `openclaw doctor` atau `openclaw plugins inspect <id>`, Anda mungkin melihat
salah satu label berikut:

| Sinyal                     | Arti                                                        |
| -------------------------- | ----------------------------------------------------------- |
| **config valid**           | Config berhasil diparse dan plugin ter-resolve              |
| **compatibility advisory** | Plugin menggunakan pola yang didukung tetapi lebih lama (mis. `hook-only`) |
| **legacy warning**         | Plugin menggunakan `before_agent_start`, yang sudah deprecated |
| **hard error**             | Config tidak valid atau plugin gagal dimuat                 |

Baik `hook-only` maupun `before_agent_start` tidak akan merusak plugin Anda hari ini --
`hook-only` bersifat advisory, dan `before_agent_start` hanya memicu peringatan. Sinyal-sinyal ini
juga muncul di `openclaw status --all` dan `openclaw plugins doctor`.

## Ikhtisar arsitektur

Sistem plugin OpenClaw memiliki empat lapisan:

1. **Manifest + discovery**
   OpenClaw menemukan plugin kandidat dari path yang dikonfigurasi, root workspace,
   root extension global, dan extension bundled. Discovery membaca
   manifes native `openclaw.plugin.json` serta manifes bundle yang didukung terlebih dahulu.
2. **Enablement + validation**
   Core memutuskan apakah plugin yang ditemukan diaktifkan, dinonaktifkan, diblokir, atau
   dipilih untuk slot eksklusif seperti memory.
3. **Runtime loading**
   Plugin OpenClaw native dimuat in-process melalui jiti dan mendaftarkan
   capability ke registry pusat. Bundle yang kompatibel dinormalisasi menjadi
   record registry tanpa mengimpor kode runtime.
4. **Surface consumption**
   Bagian lain dari OpenClaw membaca registry untuk mengekspos tools, channels, provider
   setup, hooks, HTTP routes, CLI commands, dan services.

Khusus untuk plugin CLI, discovery root command dibagi menjadi dua fase:

- metadata saat parse berasal dari `registerCli(..., { descriptors: [...] })`
- modul CLI plugin yang sebenarnya bisa tetap lazy dan mendaftar saat pemanggilan pertama

Itu menjaga kode CLI milik plugin tetap berada di dalam plugin sambil tetap membiarkan OpenClaw
mencadangkan nama root command sebelum parsing.

Batas desain yang penting:

- discovery + validasi config harus bekerja dari **metadata manifest/schema**
  tanpa mengeksekusi kode plugin
- perilaku runtime native berasal dari path `register(api)` pada modul plugin

Pemisahan itu memungkinkan OpenClaw memvalidasi config, menjelaskan plugin yang hilang/dinonaktifkan, dan
membangun petunjuk UI/schema sebelum runtime penuh aktif.

### Plugin channel dan tool message bersama

Plugin channel tidak perlu mendaftarkan tool kirim/edit/react terpisah untuk
aksi chat normal. OpenClaw mempertahankan satu tool `message` bersama di core, dan
plugin channel memiliki discovery dan eksekusi yang spesifik channel di belakangnya.

Batas saat ini adalah:

- core memiliki host tool `message` bersama, wiring prompt, pencatatan sesi/thread,
  dan dispatch eksekusi
- plugin channel memiliki discovery aksi berlingkup, discovery capability, dan setiap
  fragmen schema spesifik channel
- plugin channel memiliki tata bahasa percakapan sesi yang spesifik provider, seperti
  bagaimana id percakapan mengodekan id thread atau mewarisi dari percakapan induk
- plugin channel mengeksekusi aksi akhir melalui action adapter mereka

Untuk plugin channel, permukaan SDK-nya adalah
`ChannelMessageActionAdapter.describeMessageTool(...)`. Panggilan discovery terpadu
ini memungkinkan plugin mengembalikan aksi yang terlihat, capability, dan kontribusi
schema bersama-sama sehingga bagian-bagian itu tidak saling melenceng.

Core meneruskan scope runtime ke langkah discovery itu. Field penting meliputi:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- `requesterSenderId` inbound tepercaya

Ini penting untuk plugin yang sensitif konteks. Sebuah channel dapat menyembunyikan atau mengekspos
aksi message berdasarkan akun aktif, room/thread/message saat ini, atau
identitas requester tepercaya tanpa melakukan hardcode cabang spesifik channel di
tool `message` core.

Inilah sebabnya perubahan routing embedded-runner tetap merupakan pekerjaan plugin: runner
bertanggung jawab meneruskan identitas chat/session saat ini ke batas
discovery plugin agar tool `message` bersama mengekspos surface milik channel yang tepat
untuk giliran saat ini.

Untuk helper eksekusi yang dimiliki channel, plugin bundled sebaiknya menjaga runtime
eksekusi di dalam modul extension milik mereka sendiri. Core tidak lagi memiliki
runtime message-action Discord, Slack, Telegram, atau WhatsApp di bawah `src/agents/tools`.
Kami tidak menerbitkan subpath `plugin-sdk/*-action-runtime` terpisah, dan plugin
bundled sebaiknya mengimpor kode runtime lokal mereka sendiri langsung dari
modul milik extension mereka.

Batas yang sama berlaku pada seam SDK bernama provider secara umum: core sebaiknya
tidak mengimpor convenience barrel spesifik channel untuk Slack, Discord, Signal,
WhatsApp, atau extension serupa. Jika core membutuhkan suatu perilaku, konsumsi
barrel `api.ts` / `runtime-api.ts` milik plugin bundled itu sendiri atau naikkan kebutuhan tersebut
menjadi capability generik sempit di SDK bersama.

Khusus untuk poll, ada dua path eksekusi:

- `outbound.sendPoll` adalah baseline bersama untuk channel yang sesuai dengan model
  poll umum
- `actions.handleAction("poll")` adalah path yang diprioritaskan untuk semantik poll
  spesifik channel atau parameter poll tambahan

Core sekarang menunda parsing poll bersama hingga setelah dispatch poll plugin menolak
aksi tersebut, sehingga handler poll milik plugin dapat menerima field poll
spesifik channel tanpa diblokir parser poll generik terlebih dahulu.

Lihat [Load pipeline](#load-pipeline) untuk urutan startup lengkap.

## Model kepemilikan capability

OpenClaw memperlakukan plugin native sebagai batas kepemilikan untuk sebuah **company** atau
sebuah **feature**, bukan sebagai kumpulan integrasi tak terkait.

Artinya:

- plugin company biasanya sebaiknya memiliki semua surface OpenClaw yang menghadap
  company tersebut
- plugin feature biasanya sebaiknya memiliki surface feature penuh yang diperkenalkannya
- channel sebaiknya mengonsumsi capability core bersama alih-alih mengimplementasikan ulang
  perilaku provider secara ad hoc

Contoh:

- plugin bundled `openai` memiliki perilaku model-provider OpenAI dan perilaku OpenAI
  speech + realtime-voice + media-understanding + image-generation
- plugin bundled `elevenlabs` memiliki perilaku speech ElevenLabs
- plugin bundled `microsoft` memiliki perilaku speech Microsoft
- plugin bundled `google` memiliki perilaku model-provider Google plus Google
  media-understanding + image-generation + web-search
- plugin bundled `firecrawl` memiliki perilaku web-fetch Firecrawl
- plugin bundled `minimax`, `mistral`, `moonshot`, dan `zai` memiliki backend
  media-understanding mereka
- plugin `voice-call` adalah plugin feature: ia memiliki transport panggilan, tools,
  CLI, routes, dan bridging media-stream Twilio, tetapi ia mengonsumsi capability speech bersama
  plus realtime-transcription dan realtime-voice alih-alih
  mengimpor plugin vendor secara langsung

Keadaan akhir yang dituju adalah:

- OpenAI berada dalam satu plugin bahkan jika mencakup model teks, speech, gambar, dan
  video di masa depan
- vendor lain dapat melakukan hal yang sama untuk area permukaannya sendiri
- channel tidak peduli plugin vendor mana yang memiliki provider; mereka mengonsumsi
  kontrak capability bersama yang diekspos oleh core

Inilah perbedaan kuncinya:

- **plugin** = batas kepemilikan
- **capability** = kontrak core yang dapat diimplementasikan atau dikonsumsi oleh beberapa plugin

Jadi jika OpenClaw menambahkan domain baru seperti video, pertanyaan pertama bukan
"provider mana yang harus melakukan hardcode penanganan video?" Pertanyaan pertama adalah "apa
kontrak capability video core?" Setelah kontrak itu ada, plugin vendor
dapat mendaftar terhadapnya dan plugin channel/feature dapat mengonsumsinya.

Jika capability belum ada, langkah yang benar biasanya adalah:

1. definisikan capability yang hilang di core
2. ekspos melalui API/runtime plugin secara typed
3. hubungkan channel/features terhadap capability itu
4. biarkan plugin vendor mendaftarkan implementasi

Ini menjaga kepemilikan tetap eksplisit sambil menghindari perilaku core yang bergantung pada
satu vendor atau satu path kode spesifik plugin.

### Pelapisan capability

Gunakan model mental ini saat memutuskan di mana kode harus berada:

- **lapisan capability core**: orkestrasi bersama, kebijakan, fallback, aturan merge
  config, semantik delivery, dan kontrak typed
- **lapisan plugin vendor**: API spesifik vendor, auth, katalog model, speech
  synthesis, image generation, backend video masa depan, endpoint usage
- **lapisan plugin channel/feature**: integrasi Slack/Discord/voice-call/dll.
  yang mengonsumsi capability core dan menyajikannya di sebuah surface

Sebagai contoh, TTS mengikuti bentuk ini:

- core memiliki kebijakan reply-time TTS, urutan fallback, preferensi, dan delivery channel
- `openai`, `elevenlabs`, dan `microsoft` memiliki implementasi synthesis
- `voice-call` mengonsumsi helper runtime telephony TTS

Pola yang sama sebaiknya diprioritaskan untuk capability di masa depan.

### Contoh plugin company multi-capability

Plugin company seharusnya terasa kohesif dari luar. Jika OpenClaw memiliki
kontrak bersama untuk models, speech, realtime transcription, realtime voice, media
understanding, image generation, video generation, web fetch, dan web search,
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

Yang penting bukan nama helper persisnya. Bentuknya yang penting:

- satu plugin memiliki surface vendor
- core tetap memiliki kontrak capability
- channel dan plugin feature mengonsumsi helper `api.runtime.*`, bukan kode vendor
- contract test dapat menegaskan bahwa plugin mendaftarkan capability yang
  diklaimnya dimiliki

### Contoh capability: pemahaman video

OpenClaw sudah memperlakukan pemahaman gambar/audio/video sebagai satu
capability bersama. Model kepemilikan yang sama berlaku di sana:

1. core mendefinisikan kontrak media-understanding
2. plugin vendor mendaftarkan `describeImage`, `transcribeAudio`, dan
   `describeVideo` sesuai kebutuhan
3. channel dan plugin feature mengonsumsi perilaku core bersama alih-alih
   melakukan wiring langsung ke kode vendor

Ini menghindari asumsi video dari satu provider tertanam di core. Plugin memiliki
surface vendor; core memiliki kontrak capability dan perilaku fallback.

Pembuatan video sudah menggunakan urutan yang sama: core memiliki
kontrak capability typed dan helper runtime, dan plugin vendor mendaftarkan
implementasi `api.registerVideoGenerationProvider(...)` terhadapnya.

Butuh checklist rollout konkret? Lihat
[Capability Cookbook](/id/plugins/architecture).

## Kontrak dan penegakan

Permukaan API plugin sengaja dibuat typed dan terpusat di
`OpenClawPluginApi`. Kontrak itu mendefinisikan titik registrasi yang didukung dan
helper runtime yang boleh diandalkan oleh plugin.

Mengapa ini penting:

- penulis plugin mendapatkan satu standar internal yang stabil
- core dapat menolak kepemilikan ganda seperti dua plugin yang mendaftarkan provider id yang sama
- startup dapat menampilkan diagnostik yang dapat ditindaklanjuti untuk registrasi yang salah bentuk
- contract test dapat menegakkan kepemilikan plugin bundled dan mencegah drift diam-diam

Ada dua lapisan penegakan:

1. **penegakan registrasi runtime**
   Plugin registry memvalidasi registrasi saat plugin dimuat. Contoh:
   duplicate provider id, duplicate speech provider id, dan registrasi yang salah bentuk
   menghasilkan diagnostik plugin alih-alih perilaku tak terdefinisi.
2. **contract tests**
   Plugin bundled ditangkap di registry kontrak selama test run sehingga
   OpenClaw dapat menegaskan kepemilikan secara eksplisit. Saat ini ini digunakan untuk model
   providers, speech providers, web search providers, dan kepemilikan registrasi bundled.

Efek praktisnya adalah bahwa OpenClaw mengetahui sejak awal plugin mana yang memiliki surface mana.
Itu memungkinkan core dan channel berkomposisi secara mulus karena kepemilikan
dideklarasikan, bertipe, dan dapat diuji alih-alih implisit.

### Apa yang seharusnya masuk ke dalam kontrak

Kontrak plugin yang baik adalah:

- typed
- kecil
- spesifik capability
- dimiliki oleh core
- dapat digunakan ulang oleh beberapa plugin
- dapat dikonsumsi oleh channels/features tanpa pengetahuan vendor

Kontrak plugin yang buruk adalah:

- kebijakan spesifik vendor yang disembunyikan di core
- escape hatch plugin satu kali yang melewati registry
- kode channel yang langsung menjangkau implementasi vendor
- objek runtime ad hoc yang bukan bagian dari `OpenClawPluginApi` atau
  `api.runtime`

Jika ragu, naikkan tingkat abstraksinya: definisikan capability terlebih dahulu, lalu
biarkan plugin masuk ke dalamnya.

## Model eksekusi

Plugin OpenClaw native berjalan **in-process** dengan Gateway. Mereka tidak
di-sandbox. Plugin native yang dimuat memiliki batas kepercayaan tingkat proses yang sama
dengan kode core.

Implikasi:

- plugin native dapat mendaftarkan tools, network handlers, hooks, dan services
- bug plugin native dapat membuat gateway crash atau tidak stabil
- plugin native yang berbahaya setara dengan eksekusi kode arbitrer di dalam proses OpenClaw

Bundle yang kompatibel lebih aman secara default karena OpenClaw saat ini memperlakukannya
sebagai paket metadata/konten. Dalam rilis saat ini, itu sebagian besar berarti
Skills bundled.

Gunakan allowlist dan path install/load eksplisit untuk plugin non-bundled. Perlakukan
plugin workspace sebagai kode waktu pengembangan, bukan default produksi.

Untuk nama package workspace bundled, pertahankan plugin id tertambat di nama
npm: `@openclaw/<id>` secara default, atau suffix typed yang disetujui seperti
`-provider`, `-plugin`, `-speech`, `-sandbox`, atau `-media-understanding` ketika
package dengan sengaja mengekspos peran plugin yang lebih sempit.

Catatan kepercayaan penting:

- `plugins.allow` mempercayai **plugin ids**, bukan asal sumber.
- Plugin workspace dengan id yang sama seperti plugin bundled dengan sengaja membayangi
  salinan bundled ketika plugin workspace itu diaktifkan/di-allowlist.
- Ini normal dan berguna untuk pengembangan lokal, patch testing, dan hotfix.

## Batas ekspor

OpenClaw mengekspor capability, bukan convenience implementasi.

Pertahankan registrasi capability tetap publik. Pangkas ekspor helper non-kontrak:

- subpath helper spesifik plugin bundled
- subpath plumbing runtime yang tidak dimaksudkan sebagai API publik
- helper convenience spesifik vendor
- helper setup/onboarding yang merupakan detail implementasi

Beberapa subpath helper plugin bundled masih ada dalam peta ekspor SDK yang dihasilkan
untuk kompatibilitas dan pemeliharaan plugin bundled. Contoh saat ini mencakup
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, dan beberapa seam `plugin-sdk/matrix*`. Perlakukan itu sebagai
ekspor detail implementasi yang dicadangkan, bukan sebagai pola SDK yang direkomendasikan untuk
plugin pihak ketiga baru.

## Load pipeline

Saat startup, OpenClaw kira-kira melakukan ini:

1. menemukan root plugin kandidat
2. membaca manifes native atau bundle yang kompatibel dan metadata package
3. menolak kandidat yang tidak aman
4. menormalisasi config plugin (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. memutuskan enablement untuk setiap kandidat
6. memuat modul native yang diaktifkan melalui jiti
7. memanggil hook native `register(api)` (atau `activate(api)` — alias legacy) dan mengumpulkan registrasi ke dalam plugin registry
8. mengekspos registry ke surface commands/runtime

<Note>
`activate` adalah alias legacy untuk `register` — loader me-resolve mana pun yang ada (`def.register ?? def.activate`) dan memanggilnya pada titik yang sama. Semua plugin bundled menggunakan `register`; prioritaskan `register` untuk plugin baru.
</Note>

Gate keamanan terjadi **sebelum** eksekusi runtime. Kandidat diblokir
ketika entry keluar dari root plugin, path dapat ditulis oleh world, atau
kepemilikan path tampak mencurigakan untuk plugin non-bundled.

### Perilaku manifest-first

Manifest adalah sumber kebenaran control-plane. OpenClaw menggunakannya untuk:

- mengidentifikasi plugin
- menemukan declared channels/skills/config schema atau capability bundle
- memvalidasi `plugins.entries.<id>.config`
- menambah label/placeholder Control UI
- menampilkan metadata install/catalog

Untuk plugin native, modul runtime adalah bagian data-plane. Ia mendaftarkan
perilaku aktual seperti hooks, tools, commands, atau alur provider.

### Apa yang di-cache oleh loader

OpenClaw menyimpan cache in-process jangka pendek untuk:

- hasil discovery
- data manifest registry
- registry plugin yang dimuat

Cache ini mengurangi startup bursty dan overhead perintah yang berulang. Mereka aman
untuk dipahami sebagai cache performa berumur pendek, bukan persistence.

Catatan performa:

- Setel `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` atau
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` untuk menonaktifkan cache ini.
- Atur jendela cache dengan `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` dan
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Model registry

Plugin yang dimuat tidak langsung memodifikasi global core acak. Mereka mendaftar ke sebuah
plugin registry pusat.

Registry melacak:

- record plugin (identitas, sumber, asal, status, diagnostik)
- tools
- hook legacy dan hook typed
- channels
- providers
- gateway RPC handlers
- HTTP routes
- CLI registrars
- background services
- commands milik plugin

Fitur core kemudian membaca dari registry itu alih-alih berbicara dengan modul plugin
secara langsung. Ini menjaga pemuatan tetap satu arah:

- modul plugin -> registrasi registry
- runtime core -> konsumsi registry

Pemisahan itu penting untuk maintainability. Artinya sebagian besar surface core hanya
memerlukan satu titik integrasi: "baca registry", bukan "spesialkan setiap modul plugin".

## Callback pengikatan percakapan

Plugin yang mengikat sebuah percakapan dapat bereaksi saat sebuah persetujuan diselesaikan.

Gunakan `api.onConversationBindingResolved(...)` untuk menerima callback setelah permintaan bind
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
- `binding`: binding yang di-resolve untuk permintaan yang disetujui
- `request`: ringkasan permintaan asli, petunjuk detach, sender id, dan
  metadata percakapan

Callback ini hanya notifikasi. Ia tidak mengubah siapa yang diizinkan mengikat sebuah
percakapan, dan dijalankan setelah penanganan persetujuan core selesai.

## Hook runtime provider

Plugin provider sekarang memiliki dua lapisan:

- metadata manifest: `providerAuthEnvVars` untuk lookup env-auth murah sebelum
  runtime dimuat, plus `providerAuthChoices` untuk label onboarding/auth-choice murah
  dan metadata flag CLI sebelum runtime dimuat
- hook waktu config: `catalog` / legacy `discovery` plus `applyConfigDefaults`
- hook runtime: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
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

OpenClaw tetap memiliki loop agen generik, failover, penanganan transcript, dan
kebijakan tool. Hook-hook ini adalah permukaan extension untuk perilaku spesifik provider tanpa
memerlukan transport inferensi kustom penuh.

Gunakan manifest `providerAuthEnvVars` ketika provider memiliki kredensial berbasis env
yang harus dilihat oleh jalur auth/status/model-picker generik tanpa memuat runtime plugin.
Gunakan manifest `providerAuthChoices` ketika surface CLI onboarding/auth-choice
perlu mengetahui choice id provider, group label, dan wiring auth satu-flag sederhana tanpa
memuat runtime provider. Pertahankan runtime provider `envVars` untuk petunjuk yang menghadap operator
seperti label onboarding atau variabel setup OAuth
client-id/client-secret.

### Urutan hook dan penggunaan

Untuk plugin model/provider, OpenClaw memanggil hook dalam urutan kasar berikut.
Kolom "When to use" adalah panduan keputusan cepat.

| #   | Hook                              | Apa yang dilakukan                                                                     | Kapan digunakan                                                                                                                             |
| --- | --------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Mempublikasikan config provider ke `models.providers` selama generasi `models.json`   | Provider memiliki katalog atau default base URL                                                                                             |
| 2   | `applyConfigDefaults`             | Menerapkan default config milik provider selama materialisasi config                   | Default bergantung pada mode auth, env, atau semantik family model provider                                                                 |
| --  | _(lookup model bawaan)_           | OpenClaw mencoba path registry/catalog normal terlebih dahulu                          | _(bukan hook plugin)_                                                                                                                       |
| 3   | `normalizeModelId`                | Menormalkan alias model-id legacy atau preview sebelum lookup                          | Provider memiliki pembersihan alias sebelum resolusi model kanonis                                                                          |
| 4   | `normalizeTransport`              | Menormalkan family `api` / `baseUrl` provider sebelum assembly model generik          | Provider memiliki pembersihan transport untuk provider id kustom dalam family transport yang sama                                          |
| 5   | `normalizeConfig`                 | Menormalkan `models.providers.<id>` sebelum resolusi runtime/provider                  | Provider membutuhkan pembersihan config yang sebaiknya hidup bersama plugin; helper family Google bundled juga menjadi backstop untuk entri config Google yang didukung |
| 6   | `applyNativeStreamingUsageCompat` | Menerapkan penulisan ulang kompatibilitas usage native streaming ke config provider    | Provider membutuhkan perbaikan metadata usage native streaming yang digerakkan endpoint                                                     |
| 7   | `resolveConfigApiKey`             | Me-resolve auth env-marker untuk config provider sebelum pemuatan auth runtime         | Provider memiliki resolusi API-key env-marker milik provider; `amazon-bedrock` juga memiliki resolver env-marker AWS bawaan di sini       |
| 8   | `resolveSyntheticAuth`            | Mengekspos auth lokal/self-hosted atau berbasis config tanpa menyimpan plaintext       | Provider dapat beroperasi dengan penanda kredensial synthetic/lokal                                                                         |
| 9   | `shouldDeferSyntheticProfileAuth` | Menurunkan prioritas placeholder profil synthetic yang disimpan di bawah auth berbasis env/config | Provider menyimpan profil placeholder synthetic yang seharusnya tidak menang prioritas                                                      |
| 10  | `resolveDynamicModel`             | Fallback sinkron untuk model id milik provider yang belum ada di registry lokal        | Provider menerima model id upstream arbitrer                                                                                                |
| 11  | `prepareDynamicModel`             | Warm-up async, lalu `resolveDynamicModel` dijalankan lagi                              | Provider membutuhkan metadata jaringan sebelum me-resolve id yang tidak dikenal                                                            |
| 12  | `normalizeResolvedModel`          | Penulisan ulang final sebelum embedded runner menggunakan model yang di-resolve        | Provider membutuhkan penulisan ulang transport tetapi tetap menggunakan transport core                                                      |
| 13  | `contributeResolvedModelCompat`   | Menyumbang flag compat untuk model vendor di balik transport kompatibel lain           | Provider mengenali modelnya sendiri pada transport proxy tanpa mengambil alih provider                                                     |
| 14  | `capabilities`                    | Metadata transcript/tooling milik provider yang digunakan oleh logika core bersama     | Provider membutuhkan kekhasan transcript/provider-family                                                                                    |
| 15  | `normalizeToolSchemas`            | Menormalkan tool schema sebelum embedded runner melihatnya                             | Provider membutuhkan pembersihan schema family transport                                                                                   |
| 16  | `inspectToolSchemas`              | Menampilkan diagnostik schema milik provider setelah normalisasi                       | Provider menginginkan peringatan keyword tanpa mengajarkan aturan spesifik provider ke core                                               |
| 17  | `resolveReasoningOutputMode`      | Memilih kontrak output reasoning native vs tagged                                      | Provider membutuhkan output reasoning/final bertag alih-alih field native                                                                 |
| 18  | `prepareExtraParams`              | Normalisasi request-param sebelum wrapper opsi stream generik                          | Provider membutuhkan default request params atau pembersihan param per provider                                                             |
| 19  | `createStreamFn`                  | Mengganti sepenuhnya path stream normal dengan transport kustom                        | Provider membutuhkan wire protocol kustom, bukan sekadar wrapper                                                                           |
| 20  | `wrapStreamFn`                    | Wrapper stream setelah wrapper generik diterapkan                                     | Provider membutuhkan wrapper header/body/model compat request tanpa transport kustom                                                       |
| 21  | `resolveTransportTurnState`       | Melampirkan header atau metadata transport native per turn                             | Provider ingin transport generik mengirim identitas turn native provider                                                                   |
| 22  | `resolveWebSocketSessionPolicy`   | Melampirkan header WebSocket native atau kebijakan cool-down session                   | Provider ingin transport WS generik menyesuaikan header session atau kebijakan fallback                                                    |
| 23  | `formatApiKey`                    | Formatter auth-profile: profil tersimpan menjadi string `apiKey` runtime               | Provider menyimpan metadata auth tambahan dan membutuhkan bentuk token runtime kustom                                                      |
| 24  | `refreshOAuth`                    | Override refresh OAuth untuk endpoint refresh kustom atau kebijakan kegagalan refresh  | Provider tidak cocok dengan refresher `pi-ai` bersama                                                                                      |
| 25  | `buildAuthDoctorHint`             | Petunjuk perbaikan yang ditambahkan saat refresh OAuth gagal                           | Provider membutuhkan panduan perbaikan auth milik provider setelah kegagalan refresh                                                      |
| 26  | `matchesContextOverflowError`     | Matcher overflow context-window milik provider                                         | Provider memiliki error overflow mentah yang terlewat oleh heuristik generik                                                               |
| 27  | `classifyFailoverReason`          | Klasifikasi alasan failover milik provider                                             | Provider dapat memetakan error API/transport mentah ke rate-limit/overload/dll.                                                           |
| 28  | `isCacheTtlEligible`              | Kebijakan prompt-cache untuk provider proxy/backhaul                                   | Provider membutuhkan gate TTL cache spesifik proxy                                                                                         |
| 29  | `buildMissingAuthMessage`         | Pengganti pesan pemulihan missing-auth generik                                         | Provider membutuhkan petunjuk pemulihan missing-auth spesifik provider                                                                     |
| 30  | `suppressBuiltInModel`            | Penekanan model upstream usang plus petunjuk error opsional yang menghadap pengguna    | Provider perlu menyembunyikan baris upstream usang atau menggantinya dengan petunjuk vendor                                               |
| 31  | `augmentModelCatalog`             | Baris katalog synthetic/final yang ditambahkan setelah discovery                       | Provider membutuhkan baris forward-compat synthetic di `models list` dan picker                                                           |
| 32  | `isBinaryThinking`                | Toggle reasoning on/off untuk provider binary-thinking                                 | Provider hanya mengekspos thinking biner on/off                                                                                            |
| 33  | `supportsXHighThinking`           | Dukungan reasoning `xhigh` untuk model tertentu                                        | Provider menginginkan `xhigh` hanya pada subset model tertentu                                                                             |
| 34  | `resolveDefaultThinkingLevel`     | Level `/think` default untuk family model tertentu                                     | Provider memiliki kebijakan default `/think` untuk family model                                                                            |
| 35  | `isModernModelRef`                | Matcher modern-model untuk filter live profile dan pemilihan smoke                     | Provider memiliki pencocokan preferred-model live/smoke                                                                                    |
| 36  | `prepareRuntimeAuth`              | Menukar kredensial yang dikonfigurasi menjadi token/key runtime aktual tepat sebelum inferensi | Provider membutuhkan pertukaran token atau kredensial request berumur pendek                                                               |
| 37  | `resolveUsageAuth`                | Me-resolve kredensial usage/billing untuk `/usage` dan surface status terkait          | Provider membutuhkan parsing token usage/quota kustom atau kredensial usage yang berbeda                                                  |
| 38  | `fetchUsageSnapshot`              | Mengambil dan menormalkan snapshot usage/quota spesifik provider setelah auth di-resolve | Provider membutuhkan endpoint usage spesifik provider atau parser payload                                                                  |
| 39  | `createEmbeddingProvider`         | Membangun adapter embedding milik provider untuk memory/search                         | Perilaku memory embedding seharusnya berada bersama plugin provider                                                                        |
| 40  | `buildReplayPolicy`               | Mengembalikan kebijakan replay yang mengontrol penanganan transcript untuk provider     | Provider membutuhkan kebijakan transcript kustom (misalnya, stripping thinking-block)                                                      |
| 41  | `sanitizeReplayHistory`           | Menulis ulang replay history setelah pembersihan transcript generik                    | Provider membutuhkan penulisan ulang replay spesifik provider di luar helper compaction bersama                                            |
| 42  | `validateReplayTurns`             | Validasi atau pembentukan ulang replay-turn final sebelum embedded runner              | Transport provider membutuhkan validasi turn yang lebih ketat setelah sanitasi generik                                                    |
| 43  | `onModelSelected`                 | Menjalankan efek samping pasca-pemilihan milik provider                                | Provider membutuhkan telemetri atau state milik provider saat model menjadi aktif                                                          |

`normalizeModelId`, `normalizeTransport`, dan `normalizeConfig` pertama-tama memeriksa
plugin provider yang cocok, lalu melanjutkan ke plugin provider lain yang mampu-hook
sampai salah satu benar-benar mengubah model id atau transport/config. Itu menjaga
shim alias/compat provider tetap berfungsi tanpa mengharuskan pemanggil mengetahui plugin
bundled mana yang memiliki penulisan ulang. Jika tidak ada hook provider yang menulis ulang entri
config family Google yang didukung, normalizer config Google bundled tetap menerapkan
pembersihan kompatibilitas itu.

Jika provider membutuhkan wire protocol kustom penuh atau eksekutor request kustom,
itu adalah kelas extension yang berbeda. Hook-hook ini untuk perilaku provider
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
  dan `wrapStreamFn` karena ia memiliki forward-compat Claude 4.6,
  petunjuk provider-family, panduan perbaikan auth, integrasi endpoint usage,
  kelayakan prompt-cache, default config yang sadar-auth, kebijakan thinking
  default/adaptif Claude, dan stream shaping khusus Anthropic untuk
  beta headers, `/fast` / `serviceTier`, dan `context1m`.
- Helper stream khusus Claude milik Anthropic untuk sementara tetap berada di
  seam publik `api.ts` / `contract-api.ts` milik plugin bundled itu sendiri. Surface package
  itu mengekspor `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, dan builder wrapper
  Anthropic level bawah alih-alih memperlebar SDK generik di sekitar aturan
  beta-header satu provider.
- OpenAI menggunakan `resolveDynamicModel`, `normalizeResolvedModel`, dan
  `capabilities` plus `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking`, dan `isModernModelRef`
  karena ia memiliki forward-compat GPT-5.4, normalisasi langsung OpenAI
  `openai-completions` -> `openai-responses`, petunjuk auth yang sadar Codex,
  penekanan Spark, baris daftar OpenAI synthetic, dan kebijakan thinking /
  live-model GPT-5; family stream `openai-responses-defaults` memiliki
  wrapper Responses OpenAI native bersama untuk attribution headers,
  `/fast`/`serviceTier`, text verbosity, native Codex web search,
  reasoning-compat payload shaping, dan manajemen context Responses.
- OpenRouter menggunakan `catalog` plus `resolveDynamicModel` dan
  `prepareDynamicModel` karena provider bersifat pass-through dan dapat mengekspos
  model id baru sebelum katalog statis OpenClaw diperbarui; ia juga menggunakan
  `capabilities`, `wrapStreamFn`, dan `isCacheTtlEligible` untuk menjaga
  request headers spesifik provider, metadata routing, patch reasoning, dan
  kebijakan prompt-cache tetap di luar core. Kebijakan replay-nya berasal dari
  family `passthrough-gemini`, sedangkan family stream `openrouter-thinking`
  memiliki injeksi reasoning proxy dan skip unsupported-model / `auto`.
- GitHub Copilot menggunakan `catalog`, `auth`, `resolveDynamicModel`, dan
  `capabilities` plus `prepareRuntimeAuth` dan `fetchUsageSnapshot` karena ia
  membutuhkan device login milik provider, perilaku fallback model, kekhasan
  transcript Claude, pertukaran token GitHub -> token Copilot, dan endpoint
  usage milik provider.
- OpenAI Codex menggunakan `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth`, dan `augmentModelCatalog` plus
  `prepareExtraParams`, `resolveUsageAuth`, dan `fetchUsageSnapshot` karena ia
  masih berjalan pada transport OpenAI core tetapi memiliki normalisasi
  transport/base URL, kebijakan fallback refresh OAuth, pilihan transport
  default, baris katalog Codex synthetic, dan integrasi endpoint usage ChatGPT; ia
  berbagi family stream `openai-responses-defaults` yang sama dengan OpenAI langsung.
- Google AI Studio dan Gemini CLI OAuth menggunakan `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn`, dan `isModernModelRef` karena family replay
  `google-gemini` memiliki fallback forward-compat Gemini 3.1,
  validasi replay Gemini native, sanitasi replay bootstrap, mode output
  reasoning bertag, dan pencocokan modern-model, sedangkan family stream
  `google-thinking` memiliki normalisasi payload thinking Gemini;
  Gemini CLI OAuth juga menggunakan `formatApiKey`, `resolveUsageAuth`, dan
  `fetchUsageSnapshot` untuk formatting token, parsing token, dan wiring
  endpoint quota.
- Anthropic Vertex menggunakan `buildReplayPolicy` melalui
  family replay `anthropic-by-model` sehingga pembersihan replay khusus Claude tetap
  berlingkup pada id Claude alih-alih setiap transport `anthropic-messages`.
- Amazon Bedrock menggunakan `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason`, dan `resolveDefaultThinkingLevel` karena ia memiliki
  klasifikasi error throttle/not-ready/context-overflow spesifik Bedrock
  untuk traffic Anthropic-on-Bedrock; kebijakan replay-nya tetap berbagi guard
  `anthropic-by-model` khusus Claude yang sama.
- OpenRouter, Kilocode, Opencode, dan Opencode Go menggunakan `buildReplayPolicy`
  melalui family replay `passthrough-gemini` karena mereka mem-proxy model Gemini
  melalui transport yang kompatibel OpenAI dan membutuhkan sanitasi
  thought-signature Gemini tanpa validasi replay Gemini native atau
  penulisan ulang bootstrap.
- MiniMax menggunakan `buildReplayPolicy` melalui
  family replay `hybrid-anthropic-openai` karena satu provider memiliki baik semantik
  Anthropic-message maupun OpenAI-compatible; ia menjaga dropping thinking-block
  khusus Claude di sisi Anthropic sambil mengoverride mode output reasoning kembali ke native, dan family stream `minimax-fast-mode` memiliki penulisan ulang model
  fast-mode pada path stream bersama.
- Moonshot menggunakan `catalog` plus `wrapStreamFn` karena ia tetap menggunakan
  transport OpenAI bersama tetapi membutuhkan normalisasi payload thinking milik provider; family stream `moonshot-thinking` memetakan config plus state `/think` ke payload thinking biner native miliknya.
- Kilocode menggunakan `catalog`, `capabilities`, `wrapStreamFn`, dan
  `isCacheTtlEligible` karena ia membutuhkan request headers milik provider,
  normalisasi payload reasoning, petunjuk transcript Gemini, dan gating
  cache-TTL Anthropic; family stream `kilocode-thinking` menjaga injeksi
  thinking Kilo pada path stream proxy bersama sambil melewati `kilo/auto` dan
  model id proxy lain yang tidak mendukung payload reasoning eksplisit.
- Z.AI menggunakan `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth`, dan `fetchUsageSnapshot` karena ia memiliki fallback GLM-5,
  default `tool_stream`, UX thinking biner, pencocokan modern-model, dan baik
  auth usage maupun pengambilan kuota; family stream `tool-stream-default-on` menjaga
  wrapper `tool_stream` default-on tetap di luar glue tulisan tangan per-provider.
- xAI menggunakan `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel`, dan `isModernModelRef`
  karena ia memiliki normalisasi transport native xAI Responses, penulisan ulang
  alias Grok fast-mode, default `tool_stream`, pembersihan strict-tool / reasoning-payload,
  penggunaan ulang auth fallback untuk tool milik plugin, resolusi model
  Grok forward-compat, dan patch compat milik provider seperti profile tool-schema xAI,
  unsupported schema keywords, native `web_search`, dan dekode argumen tool-call entitas HTML.
- Mistral, OpenCode Zen, dan OpenCode Go hanya menggunakan `capabilities`
  untuk menjaga kekhasan transcript/tooling tetap di luar core.
- Provider bundled yang hanya katalog seperti `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway`, dan `volcengine` menggunakan
  `catalog` saja.
- Qwen menggunakan `catalog` untuk provider teksnya plus registrasi media-understanding dan
  video-generation bersama untuk surface multimodal-nya.
- MiniMax dan Xiaomi menggunakan `catalog` plus hook usage karena perilaku `/usage`
  mereka dimiliki plugin meskipun inferensi tetap berjalan melalui transport bersama.

## Helper runtime

Plugin dapat mengakses helper core tertentu melalui `api.runtime`. Untuk TTS:

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

- `textToSpeech` mengembalikan payload output TTS core normal untuk surface file/voice-note.
- Menggunakan konfigurasi core `messages.tts` dan pemilihan provider.
- Mengembalikan buffer audio PCM + sample rate. Plugin harus melakukan resample/encode untuk provider.
- `listVoices` bersifat opsional per provider. Gunakan untuk voice picker atau setup flow milik vendor.
- Daftar voice dapat mencakup metadata yang lebih kaya seperti locale, gender, dan tag personality untuk picker yang sadar provider.
- OpenAI dan ElevenLabs mendukung telephony saat ini. Microsoft tidak.

Plugin juga dapat mendaftarkan speech providers melalui `api.registerSpeechProvider(...)`.

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

- Pertahankan kebijakan TTS, fallback, dan delivery balasan di core.
- Gunakan speech providers untuk perilaku synthesis milik vendor.
- Input legacy Microsoft `edge` dinormalisasi ke id provider `microsoft`.
- Model kepemilikan yang diprioritaskan berorientasi company: satu plugin vendor dapat memiliki
  provider teks, speech, gambar, dan media masa depan saat OpenClaw menambahkan
  kontrak capability tersebut.

Untuk pemahaman gambar/audio/video, plugin mendaftarkan satu provider
media-understanding typed alih-alih bag key/value generik:

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

- Pertahankan orkestrasi, fallback, config, dan wiring channel di core.
- Pertahankan perilaku vendor di plugin provider.
- Ekspansi aditif sebaiknya tetap typed: method opsional baru, field hasil opsional baru, capability opsional baru.
- Pembuatan video sudah mengikuti pola yang sama:
  - core memiliki kontrak capability dan helper runtime
  - plugin vendor mendaftarkan `api.registerVideoGenerationProvider(...)`
  - plugin feature/channel mengonsumsi `api.runtime.videoGeneration.*`

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

- `api.runtime.mediaUnderstanding.*` adalah surface bersama yang diprioritaskan untuk
  pemahaman gambar/audio/video.
- Menggunakan konfigurasi audio media-understanding core (`tools.media.audio`) dan urutan fallback provider.
- Mengembalikan `{ text: undefined }` ketika tidak ada output transkripsi yang dihasilkan (misalnya input dilewati/tidak didukung).
- `api.runtime.stt.transcribeAudioFile(...)` tetap tersedia sebagai alias kompatibilitas.

Plugin juga dapat meluncurkan run subagent background melalui `api.runtime.subagent`:

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

- `provider` dan `model` adalah override per-run opsional, bukan perubahan sesi persisten.
- OpenClaw hanya menghormati field override tersebut untuk pemanggil tepercaya.
- Untuk run fallback milik plugin, operator harus opt in dengan `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Gunakan `plugins.entries.<id>.subagent.allowedModels` untuk membatasi plugin tepercaya ke target kanonis `provider/model` tertentu, atau `"*"` untuk mengizinkan target apa pun secara eksplisit.
- Run subagent plugin yang tidak tepercaya tetap berfungsi, tetapi permintaan override ditolak alih-alih diam-diam fallback.

Untuk web search, plugin dapat mengonsumsi helper runtime bersama alih-alih
menjangkau ke wiring tool agen:

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

Plugin juga dapat mendaftarkan web-search providers melalui
`api.registerWebSearchProvider(...)`.

Catatan:

- Pertahankan pemilihan provider, resolusi kredensial, dan semantik request bersama di core.
- Gunakan web-search providers untuk transport search spesifik vendor.
- `api.runtime.webSearch.*` adalah surface bersama yang diprioritaskan untuk plugin feature/channel yang membutuhkan perilaku search tanpa bergantung pada wrapper tool agen.

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

- `generate(...)`: hasilkan gambar menggunakan rantai provider image-generation yang dikonfigurasi.
- `listProviders(...)`: daftar provider image-generation yang tersedia dan capability-nya.

## Gateway HTTP routes

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

Field route:

- `path`: path route di bawah server HTTP gateway.
- `auth`: wajib. Gunakan `"gateway"` untuk memerlukan auth gateway normal, atau `"plugin"` untuk auth/webhook verification yang dikelola plugin.
- `match`: opsional. `"exact"` (default) atau `"prefix"`.
- `replaceExisting`: opsional. Mengizinkan plugin yang sama mengganti registrasi route miliknya sendiri yang sudah ada.
- `handler`: kembalikan `true` ketika route menangani request.

Catatan:

- `api.registerHttpHandler(...)` telah dihapus dan akan menyebabkan plugin-load error. Gunakan `api.registerHttpRoute(...)` sebagai gantinya.
- Route plugin harus mendeklarasikan `auth` secara eksplisit.
- Konflik `path + match` exact ditolak kecuali `replaceExisting: true`, dan satu plugin tidak dapat mengganti route plugin lain.
- Route yang tumpang tindih dengan level `auth` berbeda ditolak. Pertahankan rantai fallthrough `exact`/`prefix` hanya pada level auth yang sama.
- Route `auth: "plugin"` **tidak** menerima scope runtime operator secara otomatis. Route ini untuk webhook/signature verification yang dikelola plugin, bukan panggilan helper Gateway yang memiliki hak istimewa.
- Route `auth: "gateway"` berjalan di dalam scope runtime request Gateway, tetapi scope itu sengaja konservatif:
  - shared-secret bearer auth (`gateway.auth.mode = "token"` / `"password"`) menjaga scope runtime plugin-route tetap dipatok ke `operator.write`, bahkan jika pemanggil mengirim `x-openclaw-scopes`
  - mode HTTP pembawa identitas tepercaya (misalnya `trusted-proxy` atau `gateway.auth.mode = "none"` pada ingress privat) menghormati `x-openclaw-scopes` hanya ketika header secara eksplisit ada
  - jika `x-openclaw-scopes` tidak ada pada request plugin-route yang membawa identitas itu, scope runtime fallback ke `operator.write`
- Aturan praktis: jangan mengasumsikan route plugin yang diautentikasi gateway adalah surface admin implisit. Jika route Anda membutuhkan perilaku admin-only, wajibkan mode auth yang membawa identitas dan dokumentasikan kontrak header `x-openclaw-scopes` yang eksplisit.

## Path import Plugin SDK

Gunakan subpath SDK alih-alih import monolitik `openclaw/plugin-sdk` saat
menulis plugin:

- `openclaw/plugin-sdk/plugin-entry` untuk primitif registrasi plugin.
- `openclaw/plugin-sdk/core` untuk kontrak generik bersama yang menghadap plugin.
- `openclaw/plugin-sdk/config-schema` untuk ekspor skema Zod `openclaw.json` root
  (`OpenClawSchema`).
- Primitif channel stabil seperti `openclaw/plugin-sdk/channel-setup`,
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
  `openclaw/plugin-sdk/webhook-ingress` untuk wiring setup/auth/reply/webhook bersama.
  `channel-inbound` adalah rumah bersama untuk debounce, pencocokan mention,
  formatting envelope, dan helper konteks envelope inbound.
  `channel-setup` adalah seam setup optional-install yang sempit.
  `setup-runtime` adalah surface setup yang aman untuk runtime yang digunakan oleh `setupEntry` /
  startup tertunda, termasuk adapter patch setup yang aman untuk import.
  `setup-adapter-runtime` adalah seam adapter account-setup yang sadar env.
  `setup-tools` adalah seam helper CLI/archive/docs kecil (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Subpath domain seperti `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
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
  `openclaw/plugin-sdk/directory-runtime` untuk helper runtime/config bersama.
  `telegram-command-config` adalah seam publik sempit untuk normalisasi/validasi
  custom command Telegram dan tetap tersedia meskipun surface kontrak Telegram bundled
  untuk sementara tidak tersedia.
  `text-runtime` adalah seam text/markdown/logging bersama, termasuk
  stripping teks yang terlihat oleh asisten, helper render/chunking markdown, helper redaction,
  helper directive-tag, dan utilitas safe-text.
- Seam channel spesifik persetujuan sebaiknya memilih satu kontrak `approvalCapability`
  pada plugin. Core lalu membaca auth persetujuan, delivery, render, dan
  perilaku native-routing melalui satu capability itu alih-alih mencampur perilaku
  persetujuan ke field plugin yang tidak terkait.
- `openclaw/plugin-sdk/channel-runtime` sudah deprecated dan hanya tersisa sebagai
  shim kompatibilitas untuk plugin yang lebih lama. Kode baru sebaiknya mengimpor primitif generik yang lebih sempit, dan kode repo sebaiknya tidak menambahkan import baru dari shim tersebut.
- Internal extension bundled tetap privat. Plugin eksternal sebaiknya hanya menggunakan
  subpath `openclaw/plugin-sdk/*`. Kode core/test OpenClaw dapat menggunakan entry point
  publik repo di bawah root package plugin seperti `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js`, dan file berlingkup sempit seperti
  `login-qr-api.js`. Jangan pernah mengimpor `src/*` package plugin dari core atau dari extension lain.
- Pemisahan repo entry point:
  `<plugin-package-root>/api.js` adalah barrel helper/types,
  `<plugin-package-root>/runtime-api.js` adalah barrel runtime-only,
  `<plugin-package-root>/index.js` adalah entry plugin bundled,
  dan `<plugin-package-root>/setup-entry.js` adalah entry plugin setup.
- Contoh provider bundled saat ini:
  - Anthropic menggunakan `api.js` / `contract-api.js` untuk helper stream Claude seperti
    `wrapAnthropicProviderStream`, helper beta-header, dan parsing `service_tier`.
  - OpenAI menggunakan `api.js` untuk provider builders, helper default-model, dan
    realtime provider builders.
  - OpenRouter menggunakan `api.js` untuk provider builder-nya plus helper
    onboarding/config, sementara `register.runtime.js` masih dapat mengekspor ulang helper
    generik `plugin-sdk/provider-stream` untuk penggunaan lokal repo.
- Entry point publik yang dimuat melalui facade memprioritaskan snapshot config runtime aktif
  ketika ada, lalu fallback ke file config di disk yang sudah di-resolve saat
  OpenClaw belum melayani snapshot runtime.
- Primitif generik bersama tetap menjadi kontrak SDK publik yang diprioritaskan. Sekelompok kecil
  seam helper bermerk channel bundled yang dicadangkan masih ada. Perlakukan itu sebagai seam pemeliharaan/kompatibilitas bundled, bukan target import pihak ketiga baru; kontrak lintas-channel baru tetap sebaiknya masuk ke subpath `plugin-sdk/*` generik atau barrel `api.js` /
  `runtime-api.js` lokal plugin.

Catatan kompatibilitas:

- Hindari root barrel `openclaw/plugin-sdk` untuk kode baru.
- Prioritaskan primitif stabil yang sempit terlebih dahulu. Subpath setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool yang lebih baru adalah kontrak yang dituju untuk pekerjaan plugin bundled dan eksternal baru.
  Parsing/pencocokan target berada pada `openclaw/plugin-sdk/channel-targets`.
  Gate aksi message dan helper message-id reaction berada pada
  `openclaw/plugin-sdk/channel-actions`.
- Barrel helper khusus extension bundled tidak stabil secara default. Jika sebuah
  helper hanya dibutuhkan oleh extension bundled, pertahankan di balik seam lokal
  `api.js` atau `runtime-api.js` extension itu alih-alih mempromosikannya ke
  `openclaw/plugin-sdk/<extension>`.
- Seam helper bersama baru sebaiknya generik, bukan bermerk channel. Parsing target bersama
  berada pada `openclaw/plugin-sdk/channel-targets`; internal spesifik channel tetap berada di belakang seam lokal `api.js` atau `runtime-api.js` milik plugin tersebut.
- Subpath spesifik capability seperti `image-generation`,
  `media-understanding`, dan `speech` ada karena plugin bundled/native menggunakannya
  hari ini. Kehadirannya sendiri tidak berarti setiap helper yang diekspor adalah
  kontrak eksternal jangka panjang yang dibekukan.

## Schema tool message

Plugin sebaiknya memiliki kontribusi schema `describeMessageTool(...)` yang spesifik channel.
Pertahankan field spesifik provider di plugin, bukan di core bersama.

Untuk fragmen schema portable bersama, gunakan kembali helper generik yang diekspor melalui
`openclaw/plugin-sdk/channel-actions`:

- `createMessageToolButtonsSchema()` untuk payload gaya grid tombol
- `createMessageToolCardSchema()` untuk payload card terstruktur

Jika suatu bentuk schema hanya masuk akal untuk satu provider, definisikan di source
plugin itu sendiri alih-alih mempromosikannya ke SDK bersama.

## Resolusi target channel

Plugin channel sebaiknya memiliki semantik target yang spesifik channel. Pertahankan
host outbound bersama tetap generik dan gunakan surface messaging adapter untuk aturan provider:

- `messaging.inferTargetChatType({ to })` memutuskan apakah target yang telah dinormalisasi
  harus diperlakukan sebagai `direct`, `group`, atau `channel` sebelum lookup direktori.
- `messaging.targetResolver.looksLikeId(raw, normalized)` memberi tahu core apakah sebuah
  input harus langsung melewati ke resolusi seperti-id alih-alih pencarian direktori.
- `messaging.targetResolver.resolveTarget(...)` adalah fallback plugin saat
  core membutuhkan resolusi akhir milik provider setelah normalisasi atau setelah
  direktori tidak menemukan hasil.
- `messaging.resolveOutboundSessionRoute(...)` memiliki konstruksi route sesi
  spesifik provider setelah target di-resolve.

Pemisahan yang direkomendasikan:

- Gunakan `inferTargetChatType` untuk keputusan kategori yang harus terjadi sebelum
  mencari peer/group.
- Gunakan `looksLikeId` untuk pemeriksaan "perlakukan ini sebagai id target
  eksplisit/native".
- Gunakan `resolveTarget` untuk fallback normalisasi spesifik provider, bukan untuk
  pencarian direktori yang luas.
- Pertahankan id native provider seperti chat id, thread id, JID, handle, dan room
  id di dalam nilai `target` atau param spesifik provider, bukan di field SDK generik.

## Direktori berbasis config

Plugin yang menurunkan entri direktori dari config sebaiknya menjaga logika itu di
plugin dan menggunakan kembali helper bersama dari
`openclaw/plugin-sdk/directory-runtime`.

Gunakan ini ketika sebuah channel membutuhkan peers/groups berbasis config seperti:

- peer DM yang digerakkan allowlist
- peta channel/group yang dikonfigurasi
- fallback direktori statis berlingkup akun

Helper bersama di `directory-runtime` hanya menangani operasi generik:

- pemfilteran query
- penerapan limit
- helper deduping/normalisasi
- membangun `ChannelDirectoryEntry[]`

Inspeksi akun dan normalisasi id spesifik channel sebaiknya tetap berada di
implementasi plugin.

## Katalog provider

Plugin provider dapat mendefinisikan katalog model untuk inferensi dengan
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` mengembalikan bentuk yang sama seperti yang ditulis OpenClaw ke dalam
`models.providers`:

- `{ provider }` untuk satu entri provider
- `{ providers }` untuk beberapa entri provider

Gunakan `catalog` ketika plugin memiliki model id spesifik provider, default base URL,
atau metadata model yang digate auth.

`catalog.order` mengontrol kapan katalog plugin di-merge relatif terhadap
provider implisit bawaan OpenClaw:

- `simple`: provider berbasis API-key atau env biasa
- `profile`: provider yang muncul ketika auth profile ada
- `paired`: provider yang mensintesis beberapa entri provider terkait
- `late`: pass terakhir, setelah provider implisit lain

Provider yang lebih akhir menang pada tabrakan key, sehingga plugin dapat dengan sengaja
mengoverride entri provider bawaan dengan provider id yang sama.

Kompatibilitas:

- `discovery` tetap berfungsi sebagai alias legacy
- jika `catalog` dan `discovery` keduanya terdaftar, OpenClaw menggunakan `catalog`

## Inspeksi channel read-only

Jika plugin Anda mendaftarkan sebuah channel, prioritaskan implementasi
`plugin.config.inspectAccount(cfg, accountId)` bersama `resolveAccount(...)`.

Mengapa:

- `resolveAccount(...)` adalah path runtime. Ia boleh berasumsi bahwa kredensial
  sepenuhnya termaterialisasi dan dapat fail fast ketika secret yang dibutuhkan hilang.
- Path perintah read-only seperti `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve`, dan alur
  doctor/config repair seharusnya tidak perlu mematerialisasi kredensial runtime hanya untuk
  mendeskripsikan konfigurasi.

Perilaku `inspectAccount(...)` yang direkomendasikan:

- Kembalikan hanya state akun yang deskriptif.
- Pertahankan `enabled` dan `configured`.
- Sertakan field sumber/status kredensial bila relevan, seperti:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Anda tidak perlu mengembalikan nilai token mentah hanya untuk melaporkan
  ketersediaan read-only. Mengembalikan `tokenStatus: "available"` (dan field sumber yang cocok) sudah cukup untuk perintah bergaya status.
- Gunakan `configured_unavailable` ketika kredensial dikonfigurasi melalui SecretRef tetapi
  tidak tersedia pada path perintah saat ini.

Ini memungkinkan perintah read-only melaporkan "configured but unavailable in this command
path" alih-alih crash atau salah melaporkan akun sebagai tidak dikonfigurasi.

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

Setiap entri menjadi sebuah plugin. Jika pack mencantumkan beberapa extensions, plugin id
menjadi `name/<fileBase>`.

Jika plugin Anda mengimpor dependensi npm, instal dependensi tersebut di direktori itu agar
`node_modules` tersedia (`npm install` / `pnpm install`).

Guardrail keamanan: setiap entri `openclaw.extensions` harus tetap berada di dalam direktori plugin
setelah resolusi symlink. Entri yang keluar dari direktori package akan
ditolak.

Catatan keamanan: `openclaw plugins install` menginstal dependensi plugin dengan
`npm install --omit=dev --ignore-scripts` (tanpa lifecycle scripts, tanpa dev dependencies saat runtime). Pertahankan pohon dependensi plugin "pure JS/TS" dan hindari package yang membutuhkan build `postinstall`.

Opsional: `openclaw.setupEntry` dapat menunjuk ke modul setup-only yang ringan.
Saat OpenClaw membutuhkan surface setup untuk plugin channel yang dinonaktifkan, atau
saat plugin channel diaktifkan tetapi belum dikonfigurasi, ia memuat `setupEntry`
alih-alih entry plugin penuh. Ini menjaga startup dan setup lebih ringan
ketika entry plugin utama Anda juga mewiring tools, hooks, atau kode runtime-only lainnya.

Opsional: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
dapat mengikutsertakan plugin channel ke path `setupEntry` yang sama selama fase startup
pre-listen gateway, bahkan ketika channel sudah dikonfigurasi.

Gunakan ini hanya ketika `setupEntry` sepenuhnya mencakup surface startup yang harus ada
sebelum gateway mulai mendengarkan. Dalam praktiknya, itu berarti entry setup
harus mendaftarkan setiap capability milik channel yang menjadi dependensi startup, seperti:

- registrasi channel itu sendiri
- setiap HTTP route yang harus tersedia sebelum gateway mulai mendengarkan
- setiap gateway method, tool, atau service yang harus ada selama jendela yang sama

Jika entry penuh Anda masih memiliki capability startup yang dibutuhkan, jangan aktifkan
flag ini. Pertahankan plugin pada perilaku default dan biarkan OpenClaw memuat
entry penuh selama startup.

Channel bundled juga dapat menerbitkan helper contract-surface setup-only yang dapat
dikonsultasikan core sebelum runtime channel penuh dimuat. Surface promosi setup
saat ini adalah:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Core menggunakan surface itu ketika perlu mempromosikan config channel legacy single-account
ke `channels.<id>.accounts.*` tanpa memuat entry plugin penuh.
Matrix adalah contoh bundled saat ini: ia memindahkan hanya key auth/bootstrap ke
akun promoted bernama ketika akun bernama sudah ada, dan ia dapat mempertahankan
key default-account non-kanonis yang dikonfigurasi alih-alih selalu membuat
`accounts.default`.

Adapter patch setup tersebut menjaga discovery contract-surface bundled tetap lazy. Waktu import tetap ringan; surface promosi dimuat hanya saat pertama kali digunakan alih-alih
masuk kembali ke startup channel bundled saat import modul.

Ketika surface startup itu mencakup method gateway RPC, pertahankan pada
prefiks spesifik plugin. Namespace admin core (`config.*`,
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

### Metadata katalog channel

Plugin channel dapat mengiklankan metadata setup/discovery melalui `openclaw.channel` dan
petunjuk install melalui `openclaw.install`. Ini menjaga data-free pada katalog core.

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
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
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
- `docsLabel`: override teks link untuk link docs
- `preferOver`: plugin/channel id prioritas lebih rendah yang seharusnya dikalahkan oleh entri katalog ini
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: kontrol copy pada surface selection
- `markdownCapable`: menandai channel sebagai mampu-markdown untuk keputusan formatting outbound
- `exposure.configured`: sembunyikan channel dari surface listing configured-channel ketika disetel ke `false`
- `exposure.setup`: sembunyikan channel dari picker setup/configure interaktif ketika disetel ke `false`
- `exposure.docs`: tandai channel sebagai internal/private untuk surface navigasi docs
- `showConfigured` / `showInSetup`: alias legacy masih diterima demi kompatibilitas; prioritaskan `exposure`
- `quickstartAllowFrom`: ikutsertakan channel ke alur `allowFrom` quickstart standar
- `forceAccountBinding`: wajibkan account binding eksplisit bahkan ketika hanya ada satu akun
- `preferSessionLookupForAnnounceTarget`: prioritaskan session lookup saat me-resolve target announce

OpenClaw juga dapat me-merge **katalog channel eksternal** (misalnya, ekspor registry MPM).
Taruh file JSON di salah satu dari:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Atau arahkan `OPENCLAW_PLUGIN_CATALOG_PATHS` (atau `OPENCLAW_MPM_CATALOG_PATHS`) ke
satu atau lebih file JSON (dipisahkan koma/titik koma/`PATH`). Setiap file seharusnya
berisi `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. Parser juga menerima `"packages"` atau `"plugins"` sebagai alias legacy untuk key `"entries"`.

## Plugin context engine

Plugin context engine memiliki orkestrasi konteks sesi untuk ingest, assembly,
dan compaction. Daftarkan dari plugin Anda dengan
`api.registerContextEngine(id, factory)`, lalu pilih engine aktif dengan
`plugins.slots.contextEngine`.

Gunakan ini ketika plugin Anda perlu mengganti atau memperluas pipeline konteks default
alih-alih hanya menambahkan memory search atau hooks.

```ts
export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Jika engine Anda **tidak** memiliki algoritma compaction, pertahankan `compact()`
tetap diimplementasikan dan delegasikan secara eksplisit:

```ts
import { delegateCompactionToRuntime } from "openclaw/plugin-sdk/core";

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
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Menambahkan capability baru

Saat sebuah plugin membutuhkan perilaku yang tidak cocok dengan API saat ini, jangan
melewati sistem plugin dengan reach-in privat. Tambahkan capability yang hilang.

Urutan yang direkomendasikan:

1. definisikan kontrak core
   Putuskan perilaku bersama apa yang seharusnya dimiliki core: kebijakan, fallback, merge config,
   lifecycle, semantik yang menghadap channel, dan bentuk helper runtime.
2. tambahkan permukaan registrasi/runtime plugin yang typed
   Perluas `OpenClawPluginApi` dan/atau `api.runtime` dengan permukaan capability typed
   terkecil yang berguna.
3. hubungkan konsumen core + channel/feature
   Channel dan plugin feature sebaiknya mengonsumsi capability baru melalui core,
   bukan dengan mengimpor implementasi vendor secara langsung.
4. daftarkan implementasi vendor
   Plugin vendor kemudian mendaftarkan backend mereka terhadap capability tersebut.
5. tambahkan cakupan kontrak
   Tambahkan test agar kepemilikan dan bentuk registrasi tetap eksplisit dari waktu ke waktu.

Beginilah OpenClaw tetap opinionated tanpa menjadi hardcoded pada
pandangan satu provider. Lihat [Capability Cookbook](/id/plugins/architecture)
untuk checklist file konkret dan contoh yang dikerjakan.

### Checklist capability

Saat Anda menambahkan capability baru, implementasinya biasanya seharusnya menyentuh
surface-surface ini bersama-sama:

- tipe kontrak core di `src/<capability>/types.ts`
- runner/helper runtime core di `src/<capability>/runtime.ts`
- surface registrasi API plugin di `src/plugins/types.ts`
- wiring plugin registry di `src/plugins/registry.ts`
- eksposur runtime plugin di `src/plugins/runtime/*` ketika plugin feature/channel
  perlu mengonsumsinya
- helper capture/test di `src/test-utils/plugin-registration.ts`
- penegasan kepemilikan/kontrak di `src/plugins/contracts/registry.ts`
- docs operator/plugin di `docs/`

Jika salah satu surface itu tidak ada, biasanya itu tanda bahwa capability tersebut
belum sepenuhnya terintegrasi.

### Template capability

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

Pola contract test:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Itu menjaga aturannya tetap sederhana:

- core memiliki kontrak capability + orkestrasi
- plugin vendor memiliki implementasi vendor
- plugin feature/channel mengonsumsi helper runtime
- contract tests menjaga kepemilikan tetap eksplisit
