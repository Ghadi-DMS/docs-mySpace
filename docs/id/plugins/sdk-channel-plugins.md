---
read_when:
    - Anda sedang membangun Plugin saluran pesan baru
    - Anda ingin menghubungkan OpenClaw ke platform pesan
    - Anda perlu memahami permukaan adaptor ChannelPlugin
sidebarTitle: Channel Plugins
summary: Panduan langkah demi langkah untuk membangun Plugin saluran pesan untuk OpenClaw
title: Membangun Plugin Saluran
x-i18n:
    generated_at: "2026-04-15T09:14:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: a7f4c746fe3163a8880e14c433f4db4a1475535d91716a53fb879551d8d62f65
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# Membangun Plugin Saluran

Panduan ini memandu Anda membangun plugin saluran yang menghubungkan OpenClaw ke
platform pesan. Pada akhirnya Anda akan memiliki saluran yang berfungsi dengan keamanan DM,
pairing, thread balasan, dan pesan keluar.

<Info>
  Jika Anda belum pernah membangun plugin OpenClaw sebelumnya, baca
  [Memulai](/id/plugins/building-plugins) terlebih dahulu untuk struktur paket
  dasar dan penyiapan manifest.
</Info>

## Cara kerja plugin saluran

Plugin saluran tidak memerlukan alat kirim/edit/react mereka sendiri. OpenClaw menyimpan satu
alat `message` bersama di core. Plugin Anda memiliki:

- **Config** — resolusi akun dan wizard penyiapan
- **Keamanan** — kebijakan DM dan allowlist
- **Pairing** — alur persetujuan DM
- **Tata bahasa sesi** — bagaimana id percakapan khusus penyedia dipetakan ke chat dasar, id thread, dan fallback induk
- **Keluar** — mengirim teks, media, dan polling ke platform
- **Threading** — bagaimana balasan di-thread

Core memiliki alat message bersama, wiring prompt, bentuk outer session-key,
pembukuan generik `:thread:`, dan dispatch.

Jika saluran Anda menambahkan param alat message yang membawa sumber media, tampilkan
nama-nama param tersebut melalui `describeMessageTool(...).mediaSourceParams`. Core menggunakan
daftar eksplisit itu untuk normalisasi path sandbox dan kebijakan akses media keluar,
sehingga plugin tidak memerlukan special case shared-core untuk param avatar,
lampiran, atau cover image yang khusus penyedia.
Lebih baik mengembalikan map yang dikunci oleh action seperti
`{ "set-profile": ["avatarUrl", "avatarPath"] }` agar action yang tidak terkait tidak
mewarisi argumen media milik action lain. Array datar tetap berfungsi untuk param yang
memang sengaja dibagikan di setiap action yang diekspos.

Jika platform Anda menyimpan scope tambahan di dalam id percakapan, simpan parsing itu
di plugin dengan `messaging.resolveSessionConversation(...)`. Itu adalah hook kanonis
untuk memetakan `rawId` ke id percakapan dasar, id thread opsional,
`baseConversationId` eksplisit, dan `parentConversationCandidates` apa pun.
Saat Anda mengembalikan `parentConversationCandidates`, jaga urutannya dari
induk yang paling sempit ke percakapan dasar/paling luas.

Plugin bawaan yang memerlukan parsing yang sama sebelum registry saluran boot
juga dapat mengekspos file `session-key-api.ts` tingkat atas dengan ekspor
`resolveSessionConversation(...)` yang sesuai. Core menggunakan permukaan yang aman untuk bootstrap itu
hanya saat registry plugin runtime belum tersedia.

`messaging.resolveParentConversationCandidates(...)` tetap tersedia sebagai
fallback kompatibilitas lama ketika plugin hanya memerlukan fallback induk di atas
id generik/raw. Jika kedua hook ada, core menggunakan
`resolveSessionConversation(...).parentConversationCandidates` terlebih dahulu dan hanya
fallback ke `resolveParentConversationCandidates(...)` saat hook kanonis
tidak menyertakannya.

## Persetujuan dan kapabilitas saluran

Sebagian besar plugin saluran tidak memerlukan kode khusus persetujuan.

- Core memiliki `/approve` chat yang sama, payload tombol persetujuan bersama, dan pengiriman fallback generik.
- Gunakan satu objek `approvalCapability` pada plugin saluran bila saluran memerlukan perilaku khusus persetujuan.
- `ChannelPlugin.approvals` sudah dihapus. Letakkan fakta pengiriman/native/render/auth persetujuan di `approvalCapability`.
- `plugin.auth` hanya untuk login/logout; core tidak lagi membaca hook auth persetujuan dari objek itu.
- `approvalCapability.authorizeActorAction` dan `approvalCapability.getActionAvailabilityState` adalah seam auth persetujuan yang kanonis.
- Gunakan `approvalCapability.getActionAvailabilityState` untuk ketersediaan auth persetujuan chat yang sama.
- Jika saluran Anda mengekspos persetujuan exec native, gunakan `approvalCapability.getExecInitiatingSurfaceState` untuk status permukaan pemicu/native-client saat berbeda dari auth persetujuan chat yang sama. Core menggunakan hook khusus exec itu untuk membedakan `enabled` vs `disabled`, memutuskan apakah saluran pemicu mendukung persetujuan exec native, dan menyertakan saluran tersebut dalam panduan fallback native-client. `createApproverRestrictedNativeApprovalCapability(...)` mengisi ini untuk kasus umum.
- Gunakan `outbound.shouldSuppressLocalPayloadPrompt` atau `outbound.beforeDeliverPayload` untuk perilaku siklus hidup payload khusus saluran, seperti menyembunyikan prompt persetujuan lokal duplikat atau mengirim indikator mengetik sebelum pengiriman.
- Gunakan `approvalCapability.delivery` hanya untuk perutean persetujuan native atau penekanan fallback.
- Gunakan `approvalCapability.nativeRuntime` untuk fakta persetujuan native yang dimiliki saluran. Jaga agar tetap lazy pada entrypoint saluran yang hot dengan `createLazyChannelApprovalNativeRuntimeAdapter(...)`, yang dapat mengimpor modul runtime Anda sesuai kebutuhan sambil tetap memungkinkan core menyusun siklus hidup persetujuan.
- Gunakan `approvalCapability.render` hanya ketika saluran benar-benar memerlukan payload persetujuan kustom alih-alih renderer bersama.
- Gunakan `approvalCapability.describeExecApprovalSetup` ketika saluran ingin balasan jalur disabled menjelaskan knob config yang tepat untuk mengaktifkan persetujuan exec native. Hook menerima `{ channel, channelLabel, accountId }`; saluran akun bernama harus merender path yang dibatasi akun seperti `channels.<channel>.accounts.<id>.execApprovals.*` alih-alih default tingkat atas.
- Jika saluran dapat menyimpulkan identitas DM yang stabil seperti pemilik dari config yang ada, gunakan `createResolvedApproverActionAuthAdapter` dari `openclaw/plugin-sdk/approval-runtime` untuk membatasi `/approve` chat yang sama tanpa menambahkan logika khusus persetujuan di core.
- Jika saluran memerlukan pengiriman persetujuan native, jaga agar kode saluran tetap berfokus pada normalisasi target ditambah fakta transport/presentasi. Gunakan `createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`, `createChannelApproverDmTargetResolver`, dan `createApproverRestrictedNativeApprovalCapability` dari `openclaw/plugin-sdk/approval-runtime`. Letakkan fakta khusus saluran di belakang `approvalCapability.nativeRuntime`, idealnya melalui `createChannelApprovalNativeRuntimeAdapter(...)` atau `createLazyChannelApprovalNativeRuntimeAdapter(...)`, agar core dapat menyusun handler dan memiliki filter permintaan, perutean, dedupe, expiry, subscription Gateway, dan pemberitahuan dialihkan-ke-tempat-lain. `nativeRuntime` dibagi menjadi beberapa seam yang lebih kecil:
- `availability` — apakah akun sudah dikonfigurasi dan apakah permintaan harus ditangani
- `presentation` — memetakan view model persetujuan bersama ke payload native pending/resolved/expired atau action akhir
- `transport` — menyiapkan target serta mengirim/memperbarui/menghapus pesan persetujuan native
- `interactions` — hook bind/unbind/clear-action opsional untuk tombol atau reaksi native
- `observe` — hook diagnostik pengiriman opsional
- Jika saluran memerlukan objek yang dimiliki runtime seperti client, token, aplikasi Bolt, atau penerima Webhook, daftarkan melalui `openclaw/plugin-sdk/channel-runtime-context`. Registry runtime-context generik memungkinkan core melakukan bootstrap handler berbasis kapabilitas dari status startup saluran tanpa menambahkan glue wrapper khusus persetujuan.
- Gunakan `createChannelApprovalHandler` atau `createChannelNativeApprovalRuntime` tingkat bawah hanya ketika seam berbasis kapabilitas belum cukup ekspresif.
- Saluran persetujuan native harus merutekan `accountId` dan `approvalKind` melalui helper tersebut. `accountId` menjaga agar kebijakan persetujuan multi-akun tetap dibatasi ke akun bot yang benar, dan `approvalKind` menjaga agar perilaku persetujuan exec vs plugin tetap tersedia untuk saluran tanpa cabang yang di-hardcode di core.
- Core sekarang juga memiliki pemberitahuan pengalihan persetujuan. Plugin saluran tidak boleh mengirim pesan tindak lanjut mereka sendiri seperti "approval went to DMs / another channel" dari `createChannelNativeApprovalRuntime`; sebagai gantinya, tampilkan perutean origin + DM approver yang akurat melalui helper kapabilitas persetujuan bersama dan biarkan core mengagregasi pengiriman aktual sebelum memposting pemberitahuan kembali ke chat pemicu.
- Pertahankan jenis id persetujuan yang dikirim dari ujung ke ujung. Client native tidak boleh
  menebak atau menulis ulang perutean persetujuan exec vs plugin dari status lokal saluran.
- Jenis persetujuan yang berbeda dapat dengan sengaja mengekspos permukaan native yang berbeda.
  Contoh bawaan saat ini:
  - Slack mempertahankan perutean persetujuan native tersedia untuk id exec dan plugin.
  - Matrix mempertahankan perutean DM/saluran native yang sama dan UX reaksi untuk persetujuan exec
    dan plugin, sambil tetap memungkinkan auth berbeda menurut jenis persetujuan.
- `createApproverRestrictedNativeApprovalAdapter` masih ada sebagai wrapper kompatibilitas, tetapi kode baru sebaiknya lebih memilih builder kapabilitas dan mengekspos `approvalCapability` pada plugin.

Untuk entrypoint saluran yang hot, lebih baik gunakan subpath runtime yang lebih sempit saat Anda hanya
memerlukan satu bagian dari keluarga itu:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Demikian juga, lebih baik gunakan `openclaw/plugin-sdk/setup-runtime`,
`openclaw/plugin-sdk/setup-adapter-runtime`,
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference`, dan
`openclaw/plugin-sdk/reply-chunking` saat Anda tidak memerlukan
permukaan payung yang lebih luas.

Khusus untuk penyiapan:

- `openclaw/plugin-sdk/setup-runtime` mencakup helper penyiapan yang aman untuk runtime:
  adaptor patch penyiapan yang aman diimpor (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), keluaran catatan lookup,
  `promptResolvedAllowFrom`, `splitSetupEntries`, dan builder
  setup-proxy yang didelegasikan
- `openclaw/plugin-sdk/setup-adapter-runtime` adalah
  seam adaptor sempit yang sadar env untuk `createEnvPatchedAccountSetupAdapter`
- `openclaw/plugin-sdk/channel-setup` mencakup builder penyiapan instalasi opsional
  ditambah beberapa primitif yang aman untuk penyiapan:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,

Jika saluran Anda mendukung penyiapan atau auth berbasis env dan alur startup/config generik
perlu mengetahui nama env tersebut sebelum runtime dimuat, deklarasikan di
manifest plugin dengan `channelEnvVars`. Simpan `envVars` runtime saluran atau konstanta lokal
untuk salinan yang ditujukan kepada operator saja.
`createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
`createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, dan
`splitSetupEntries`

- gunakan seam `openclaw/plugin-sdk/setup` yang lebih luas hanya ketika Anda juga memerlukan
  helper setup/config bersama yang lebih berat seperti
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

Jika saluran Anda hanya ingin mengiklankan "install plugin ini terlebih dahulu" di permukaan
penyiapan, lebih baik gunakan `createOptionalChannelSetupSurface(...)`. Adaptor/wizard yang dihasilkan
fail closed pada penulisan config dan finalisasi, dan mereka menggunakan kembali pesan wajib-instalasi yang sama
di seluruh validasi, finalisasi, dan salinan tautan docs.

Untuk path saluran hot lainnya, lebih baik gunakan helper sempit daripada permukaan lama yang lebih luas:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution`, dan
  `openclaw/plugin-sdk/account-helpers` untuk config multi-akun dan
  fallback akun default
- `openclaw/plugin-sdk/inbound-envelope` dan
  `openclaw/plugin-sdk/inbound-reply-dispatch` untuk perutean/envelope masuk dan
  wiring catat-dan-dispatch
- `openclaw/plugin-sdk/messaging-targets` untuk parsing/pencocokan target
- `openclaw/plugin-sdk/outbound-media` dan
  `openclaw/plugin-sdk/outbound-runtime` untuk pemuatan media ditambah delegasi
  identitas/pengiriman keluar
- `openclaw/plugin-sdk/thread-bindings-runtime` untuk siklus hidup thread-binding
  dan pendaftaran adaptor
- `openclaw/plugin-sdk/agent-media-payload` hanya ketika tata letak field payload agent/media
  lama masih diperlukan
- `openclaw/plugin-sdk/telegram-command-config` untuk normalisasi perintah kustom Telegram, validasi duplikat/konflik, dan kontrak config perintah yang stabil untuk fallback

Saluran khusus auth biasanya dapat berhenti pada jalur default: core menangani persetujuan dan plugin hanya mengekspos kapabilitas outbound/auth. Saluran persetujuan native seperti Matrix, Slack, Telegram, dan transport chat kustom harus menggunakan helper native bersama alih-alih membuat siklus hidup persetujuan mereka sendiri.

## Kebijakan mention masuk

Pertahankan penanganan mention masuk terbagi menjadi dua lapisan:

- pengumpulan bukti yang dimiliki plugin
- evaluasi kebijakan bersama

Gunakan `openclaw/plugin-sdk/channel-inbound` untuk lapisan bersama.

Cocok untuk logika lokal plugin:

- deteksi reply-to-bot
- deteksi quoted-bot
- pemeriksaan partisipasi thread
- pengecualian service/system-message
- cache native platform yang diperlukan untuk membuktikan partisipasi bot

Cocok untuk helper bersama:

- `requireMention`
- hasil mention eksplisit
- allowlist mention implisit
- bypass perintah
- keputusan skip akhir

Alur yang disarankan:

1. Hitung fakta mention lokal.
2. Masukkan fakta tersebut ke `resolveInboundMentionDecision({ facts, policy })`.
3. Gunakan `decision.effectiveWasMentioned`, `decision.shouldBypassMention`, dan `decision.shouldSkip` di gerbang masuk Anda.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";

const mentionMatch = matchesMentionWithExplicit(text, {
  mentionRegexes,
  mentionPatterns,
});

const facts = {
  canDetectMention: true,
  wasMentioned: mentionMatch.matched,
  hasAnyMention: mentionMatch.hasExplicitMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    allowedImplicitMentionKinds: requireExplicitMention ? [] : ["reply_to_bot", "quoted_bot"],
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`api.runtime.channel.mentions` mengekspos helper mention bersama yang sama untuk
plugin saluran bawaan yang sudah bergantung pada injeksi runtime:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

Helper lama `resolveMentionGating*` tetap ada di
`openclaw/plugin-sdk/channel-inbound` hanya sebagai ekspor kompatibilitas. Kode baru
harus menggunakan `resolveInboundMentionDecision({ facts, policy })`.

## Panduan langkah demi langkah

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paket dan manifest">
    Buat file plugin standar. Field `channel` di `package.json` adalah
    yang menjadikan ini plugin saluran. Untuk permukaan metadata paket lengkap,
    lihat [Penyiapan dan Config Plugin](/id/plugins/sdk-setup#openclaw-channel):

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "Hubungkan OpenClaw ke Acme Chat."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "kind": "channel",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Plugin saluran Acme Chat",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "acme-chat": {
            "type": "object",
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

  </Step>

  <Step title="Bangun objek plugin saluran">
    Interface `ChannelPlugin` memiliki banyak permukaan adaptor opsional. Mulailah dengan
    yang minimum — `id` dan `setup` — lalu tambahkan adaptor sesuai kebutuhan.

    Buat `src/channel.ts`:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // client API platform Anda

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token wajib diisi");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        setup: {
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
      }),

      // Keamanan DM: siapa yang dapat mengirim pesan ke bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: alur persetujuan untuk kontak DM baru
      pairing: {
        text: {
          idLabel: "username Acme Chat",
          message: "Kirim kode ini untuk memverifikasi identitas Anda:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Kode pairing: ${code}`);
          },
        },
      },

      // Threading: cara balasan dikirim
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: kirim pesan ke platform
      outbound: {
        attachedResults: {
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    <Accordion title="Apa yang dilakukan createChatChannelPlugin untuk Anda">
      Alih-alih mengimplementasikan interface adaptor tingkat rendah secara manual, Anda meneruskan
      opsi deklaratif dan builder akan menyusunnya:

      | Opsi | Yang di-wire |
      | --- | --- |
      | `security.dm` | Resolver keamanan DM yang dibatasi dari field config |
      | `pairing.text` | Alur pairing DM berbasis teks dengan pertukaran kode |
      | `threading` | Resolver mode reply-to (tetap, dibatasi akun, atau kustom) |
      | `outbound.attachedResults` | Fungsi kirim yang mengembalikan metadata hasil (ID pesan) |

      Anda juga dapat meneruskan objek adaptor mentah alih-alih opsi deklaratif
      jika memerlukan kontrol penuh.
    </Accordion>

  </Step>

  <Step title="Wire entry point">
    Buat `index.ts`:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Plugin saluran Acme Chat",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Manajemen Acme Chat");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Manajemen Acme Chat",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    Letakkan descriptor CLI yang dimiliki saluran di `registerCliMetadata(...)` agar OpenClaw
    dapat menampilkannya di bantuan root tanpa mengaktifkan runtime saluran penuh,
    sementara pemuatan penuh normal tetap mengambil descriptor yang sama untuk pendaftaran
    perintah yang sebenarnya. Simpan `registerFull(...)` untuk pekerjaan yang hanya runtime.
    Jika `registerFull(...)` mendaftarkan metode RPC Gateway, gunakan
    prefix khusus plugin. Namespace admin core (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) tetap dicadangkan dan selalu
    di-resolve ke `operator.admin`.
    `defineChannelPluginEntry` menangani pemisahan mode pendaftaran secara otomatis. Lihat
    [Entry Points](/id/plugins/sdk-entrypoints#definechannelpluginentry) untuk semua
    opsi.

  </Step>

  <Step title="Tambahkan entri penyiapan">
    Buat `setup-entry.ts` untuk pemuatan ringan selama onboarding:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw memuat ini alih-alih entry penuh saat saluran dinonaktifkan
    atau belum dikonfigurasi. Ini menghindari pemuatan kode runtime yang berat selama alur penyiapan.
    Lihat [Setup and Config](/id/plugins/sdk-setup#setup-entry) untuk detail.

  </Step>

  <Step title="Tangani pesan masuk">
    Plugin Anda perlu menerima pesan dari platform dan meneruskannya ke
    OpenClaw. Pola yang umum adalah Webhook yang memverifikasi permintaan dan
    mendispatch-nya melalui handler masuk saluran Anda:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // auth yang dikelola plugin (verifikasi signature sendiri)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Handler masuk Anda mendispatch pesan ke OpenClaw.
          // Wiring pastinya bergantung pada SDK platform Anda —
          // lihat contoh nyata di paket plugin Microsoft Teams atau Google Chat bawaan.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      Penanganan pesan masuk bersifat khusus saluran. Setiap plugin saluran memiliki
      pipeline masuknya sendiri. Lihat plugin saluran bawaan
      (misalnya paket plugin Microsoft Teams atau Google Chat) untuk pola nyata.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Uji">
Tulis pengujian colocated di `src/channel.test.ts`:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("plugin acme-chat", () => {
      it("me-resolve akun dari config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("memeriksa akun tanpa mematerialkan secret", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("melaporkan config yang hilang", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    Untuk helper pengujian bersama, lihat [Pengujian](/id/plugins/sdk-testing).

  </Step>
</Steps>

## Struktur file

```
<bundled-plugin-root>/acme-chat/
├── package.json              # metadata openclaw.channel
├── openclaw.plugin.json      # Manifest dengan skema config
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Ekspor publik (opsional)
├── runtime-api.ts            # Ekspor runtime internal (opsional)
└── src/
    ├── channel.ts            # ChannelPlugin melalui createChatChannelPlugin
    ├── channel.test.ts       # Pengujian
    ├── client.ts             # Client API platform
    └── runtime.ts            # Penyimpanan runtime (jika diperlukan)
```

## Topik lanjutan

<CardGroup cols={2}>
  <Card title="Opsi threading" icon="git-branch" href="/id/plugins/sdk-entrypoints#registration-mode">
    Mode balasan tetap, dibatasi akun, atau kustom
  </Card>
  <Card title="Integrasi alat pesan" icon="puzzle" href="/id/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool dan penemuan action
  </Card>
  <Card title="Resolusi target" icon="crosshair" href="/id/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="Helper runtime" icon="settings" href="/id/plugins/sdk-runtime">
    TTS, STT, media, subagent melalui api.runtime
  </Card>
</CardGroup>

<Note>
Beberapa seam helper bawaan masih ada untuk pemeliharaan bundled-plugin dan
kompatibilitas. Itu bukan pola yang direkomendasikan untuk plugin saluran baru;
lebih baik gunakan subpath channel/setup/reply/runtime generik dari permukaan SDK
umum kecuali Anda memelihara keluarga bundled plugin tersebut secara langsung.
</Note>

## Langkah berikutnya

- [Plugin Penyedia](/id/plugins/sdk-provider-plugins) — jika plugin Anda juga menyediakan model
- [Ikhtisar SDK](/id/plugins/sdk-overview) — referensi impor subpath lengkap
- [Pengujian SDK](/id/plugins/sdk-testing) — utilitas pengujian dan pengujian kontrak
- [Manifest Plugin](/id/plugins/manifest) — skema manifest lengkap
