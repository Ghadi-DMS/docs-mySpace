---
read_when:
    - Tam alan düzeyinde yapılandırma anlamlarını veya varsayılanları gerektiğinde
    - Kanal, model, gateway veya araç yapılandırma bloklarını doğrularken
summary: Temel OpenClaw anahtarları, varsayılanlar ve özel alt sistem referanslarına bağlantılar için gateway yapılandırma başvurusu
title: Yapılandırma Başvurusu
x-i18n:
    generated_at: "2026-04-09T01:34:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: a9d6d0c542b9874809491978fdcf8e1a7bb35a4873db56aa797963d03af4453c
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# Yapılandırma Başvurusu

`~/.openclaw/openclaw.json` için temel yapılandırma başvurusu. Görev odaklı bir genel bakış için bkz. [Configuration](/tr/gateway/configuration).

Bu sayfa, ana OpenClaw yapılandırma yüzeylerini kapsar ve bir alt sistemin kendine ait daha derin bir başvurusu olduğunda dışarı bağlantı verir. **Her** kanal/eklentiye ait komut kataloğunu veya her derin bellek/QMD ayarını tek bir sayfada satır içine almaya çalışmaz.

Kod doğruluğu:

- `openclaw config schema`, doğrulama ve Control UI için kullanılan canlı JSON Schema'yı yazdırır; mevcut olduğunda paketlenmiş/eklenti/kanal meta verileri de birleştirilir
- `config.schema.lookup`, ayrıntılı inceleme araçları için bir yol kapsamlı şema düğümü döndürür
- `pnpm config:docs:check` / `pnpm config:docs:gen`, yapılandırma dokümanı temel hash'ini geçerli şema yüzeyine karşı doğrular

Ayrı derin başvurular:

- `agents.defaults.memorySearch.*`, `memory.qmd.*`, `memory.citations` ve `plugins.entries.memory-core.config.dreaming` altındaki dreaming yapılandırması için [Memory configuration reference](/tr/reference/memory-config)
- Güncel yerleşik + paketlenmiş komut kataloğu için [Slash Commands](/tr/tools/slash-commands)
- Kanala özgü komut yüzeyleri için ilgili kanal/eklenti sayfaları

Yapılandırma biçimi **JSON5**'tir (yorumlara + sondaki virgüllere izin verilir). Tüm alanlar isteğe bağlıdır — OpenClaw belirtilmediğinde güvenli varsayılanlar kullanır.

---

## Kanallar

Her kanal, yapılandırma bölümü mevcut olduğunda otomatik olarak başlar (`enabled: false` olmadığı sürece).

### DM ve grup erişimi

Tüm kanallar DM ilkelerini ve grup ilkelerini destekler:

| DM ilkesi           | Davranış                                                       |
| ------------------- | -------------------------------------------------------------- |
| `pairing` (varsayılan) | Bilinmeyen gönderenler tek seferlik bir eşleştirme kodu alır; sahibi onaylamalıdır |
| `allowlist`         | Yalnızca `allowFrom` içindeki gönderenler (veya eşleştirilmiş izin deposu) |
| `open`              | Tüm gelen DM'lere izin ver (`allowFrom: ["*"]` gerektirir)    |
| `disabled`          | Tüm gelen DM'leri yok say                                      |

| Grup ilkesi           | Davranış                                              |
| --------------------- | ----------------------------------------------------- |
| `allowlist` (varsayılan) | Yalnızca yapılandırılmış izin listesiyle eşleşen gruplar |
| `open`                | Grup izin listelerini atla (bahsetme geçidi yine de uygulanır) |
| `disabled`            | Tüm grup/oda mesajlarını engelle                      |

<Note>
`channels.defaults.groupPolicy`, bir sağlayıcının `groupPolicy` değeri ayarlanmadığında varsayılanı belirler.
Eşleştirme kodlarının süresi 1 saat sonra dolur. Bekleyen DM eşleştirme istekleri **kanal başına 3** ile sınırlıdır.
Bir sağlayıcı bloğu tamamen yoksa (`channels.<provider>` yoksa), çalışma zamanındaki grup ilkesi başlangıç uyarısıyla birlikte `allowlist` (fail-closed) olarak geri döner.
</Note>

### Kanal model geçersiz kılmaları

Belirli kanal kimliklerini bir modele sabitlemek için `channels.modelByChannel` kullanın. Değerler `provider/model` veya yapılandırılmış model takma adlarını kabul eder. Bir oturumda zaten model geçersiz kılma yoksa kanal eşlemesi uygulanır (örneğin `/model` ile ayarlanmışsa).

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-4.1",
      },
      telegram: {
        "-1001234567890": "openai/gpt-4.1-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

### Kanal varsayılanları ve heartbeat

Sağlayıcılar arasında paylaşılan grup ilkesi ve heartbeat davranışı için `channels.defaults` kullanın:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`: bir sağlayıcı düzeyindeki `groupPolicy` ayarlı değilse geri dönüş grup ilkesi.
- `channels.defaults.contextVisibility`: tüm kanallar için varsayılan ek bağlam görünürlüğü modu. Değerler: `all` (varsayılan, tüm alıntı/konu/geçmiş bağlamını dahil et), `allowlist` (yalnızca izin verilen gönderenlerden gelen bağlamı dahil et), `allowlist_quote` (allowlist ile aynı ama açık alıntı/yanıt bağlamını koru). Kanal başına geçersiz kılma: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: heartbeat çıktısında sağlıklı kanal durumlarını dahil et.
- `channels.defaults.heartbeat.showAlerts`: heartbeat çıktısında bozulmuş/hata durumlarını dahil et.
- `channels.defaults.heartbeat.useIndicator`: kompakt gösterge tarzı heartbeat çıktısı oluştur.

### WhatsApp

WhatsApp, gateway'in web kanalı üzerinden çalışır (Baileys Web). Bağlı bir oturum mevcut olduğunda otomatik başlar.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // mavi tikler (self-chat modunda false)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
  web: {
    enabled: true,
    heartbeatSeconds: 60,
    reconnect: {
      initialMs: 2000,
      maxMs: 120000,
      factor: 1.4,
      jitter: 0.2,
      maxAttempts: 0,
    },
  },
}
```

<Accordion title="Çok hesaplı WhatsApp">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- Giden komutlar mevcutsa varsayılan olarak `default` hesabını kullanır; aksi halde ilk yapılandırılmış hesap kimliğini (sıralanmış) kullanır.
- İsteğe bağlı `channels.whatsapp.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde bu geri dönüş varsayılan hesap seçimini geçersiz kılar.
- Eski tek hesaplı Baileys kimlik doğrulama dizini, `openclaw doctor` tarafından `whatsapp/default` içine taşınır.
- Hesap başına geçersiz kılmalar: `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: "partial", // off | partial | block | progress (varsayılan: off; preview-edit hız sınırlarını önlemek için açıkça etkinleştirin)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Bot tokenı: `channels.telegram.botToken` veya `channels.telegram.tokenFile` (yalnızca normal dosya; symlink'ler reddedilir), varsayılan hesap için geri dönüş olarak `TELEGRAM_BOT_TOKEN`.
- İsteğe bağlı `channels.telegram.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde varsayılan hesap seçimini geçersiz kılar.
- Çok hesaplı kurulumlarda (2+ hesap kimliği), geri dönüş yönlendirmesini önlemek için açık bir varsayılan ayarlayın (`channels.telegram.defaultAccount` veya `channels.telegram.accounts.default`); bu eksik veya geçersizse `openclaw doctor` uyarır.
- `configWrites: false`, Telegram tarafından başlatılan yapılandırma yazımlarını engeller (supergroup kimliği geçişleri, `/config set|unset`).
- `type: "acp"` içeren üst düzey `bindings[]` girdileri, forum konuları için kalıcı ACP bağlamalarını yapılandırır (`match.peer.id` içinde kanonik `chatId:topic:topicId` kullanın). Alan anlamları [ACP Agents](/tr/tools/acp-agents#channel-specific-settings) içinde paylaşılır.
- Telegram akış önizlemeleri `sendMessage` + `editMessageText` kullanır (doğrudan ve grup sohbetlerinde çalışır).
- Yeniden deneme ilkesi: bkz. [Retry policy](/tr/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Short answers only.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      chunkMode: "length", // length | newline
      streaming: "off", // off | partial | block | progress (progress, Discord'da partial'a eşlenir)
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // sessions_spawn({ thread: true }) için açık katılım
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- Token: `channels.discord.token`, varsayılan hesap için geri dönüş olarak `DISCORD_BOT_TOKEN`.
- Açık bir Discord `token` sağlayan doğrudan giden çağrılar, çağrı için o tokenı kullanır; hesap yeniden deneme/ilke ayarları yine de etkin çalışma zamanı anlık görüntüsünde seçilen hesaptan gelir.
- İsteğe bağlı `channels.discord.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde varsayılan hesap seçimini geçersiz kılar.
- Teslim hedefleri için `user:<id>` (DM) veya `channel:<id>` (guild kanalı) kullanın; çıplak sayısal kimlikler reddedilir.
- Guild slug'ları küçük harflidir ve boşluklar `-` ile değiştirilir; kanal anahtarları slug'lanmış adı kullanır (`#` yok). Guild kimliklerini tercih edin.
- Bot tarafından yazılmış mesajlar varsayılan olarak yok sayılır. `allowBots: true` bunları etkinleştirir; yalnızca bottan bahseden bot mesajlarını kabul etmek için `allowBots: "mentions"` kullanın (kendi mesajları yine de filtrelenir).
- `channels.discord.guilds.<id>.ignoreOtherMentions` (ve kanal geçersiz kılmaları), başka bir kullanıcı veya rolden bahseden ancak bottan bahsetmeyen mesajları düşürür (@everyone/@here hariç).
- `maxLinesPerMessage` (varsayılan 17), 2000 karakterin altında olsa bile uzun mesajları böler.
- `channels.discord.threadBindings`, Discord konuya bağlı yönlendirmeyi kontrol eder:
  - `enabled`: konuya bağlı oturum özellikleri için Discord geçersiz kılması (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` ve bağlı teslim/yönlendirme)
  - `idleHours`: saat cinsinden hareketsizlik nedeniyle otomatik odak kaldırma için Discord geçersiz kılması (`0` devre dışı bırakır)
  - `maxAgeHours`: saat cinsinden kesin maksimum yaş için Discord geçersiz kılması (`0` devre dışı bırakır)
  - `spawnSubagentSessions`: `sessions_spawn({ thread: true })` otomatik konu oluşturma/bağlama için açık katılım anahtarı
- `type: "acp"` içeren üst düzey `bindings[]` girdileri, kanallar ve konular için kalıcı ACP bağlamalarını yapılandırır (`match.peer.id` içinde kanal/konu kimliğini kullanın). Alan anlamları [ACP Agents](/tr/tools/acp-agents#channel-specific-settings) içinde paylaşılır.
- `channels.discord.ui.components.accentColor`, Discord components v2 kapsayıcıları için vurgu rengini ayarlar.
- `channels.discord.voice`, Discord ses kanalı konuşmalarını ve isteğe bağlı otomatik katılma + TTS geçersiz kılmalarını etkinleştirir.
- `channels.discord.voice.daveEncryption` ve `channels.discord.voice.decryptionFailureTolerance`, `@discordjs/voice` DAVE seçeneklerine doğrudan aktarılır (varsayılan olarak `true` ve `24`).
- OpenClaw ayrıca, tekrarlanan şifre çözme hatalarından sonra bir ses oturumundan ayrılıp yeniden katılarak ses alımını kurtarmayı dener.
- `channels.discord.streaming`, kanonik akış modu anahtarıdır. Eski `streamMode` ve boolean `streaming` değerleri otomatik taşınır.
- `channels.discord.autoPresence`, çalışma zamanı erişilebilirliğini bot varlığına eşler (sağlıklı => online, bozulmuş => idle, exhausted => dnd) ve isteğe bağlı durum metni geçersiz kılmalarına izin verir.
- `channels.discord.dangerouslyAllowNameMatching`, değiştirilebilir ad/etiket eşleştirmesini yeniden etkinleştirir (cam kırma uyumluluk modu).
- `channels.discord.execApprovals`: Discord yerel exec onay teslimi ve onaylayıcı yetkilendirmesi.
  - `enabled`: `true`, `false` veya `"auto"` (varsayılan). Otomatik modda exec onayları, `approvers` veya `commands.ownerAllowFrom` içinden onaylayıcılar çözümlenebildiğinde etkinleşir.
  - `approvers`: exec isteklerini onaylamasına izin verilen Discord kullanıcı kimlikleri. Belirtilmezse `commands.ownerAllowFrom`'a geri döner.
  - `agentFilter`: isteğe bağlı ajan kimliği izin listesi. Tüm ajanlar için onay iletmek üzere boş bırakın.
  - `sessionFilter`: isteğe bağlı oturum anahtarı kalıpları (alt dizgi veya regex).
  - `target`: onay istemlerinin nereye gönderileceği. `"dm"` (varsayılan) onaylayıcı DM'lerine gönderir, `"channel"` kaynağın kanalına gönderir, `"both"` her ikisine gönderir. Hedef `"channel"` içerdiğinde düğmeler yalnızca çözümlenmiş onaylayıcılar tarafından kullanılabilir.
  - `cleanupAfterResolve`: `true` olduğunda onay, red veya zaman aşımından sonra onay DM'lerini siler.

**Tepki bildirim modları:** `off` (hiçbiri), `own` (botun mesajları, varsayılan), `all` (tüm mesajlar), `allowlist` (`guilds.<id>.users` içinden tüm mesajlarda).

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dm: {
        enabled: true,
        policy: "pairing",
        allowFrom: ["users/1234567890"],
      },
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- Hizmet hesabı JSON'u: satır içi (`serviceAccount`) veya dosya tabanlı (`serviceAccountFile`).
- Hizmet hesabı SecretRef de desteklenir (`serviceAccountRef`).
- Çevre geri dönüşleri: `GOOGLE_CHAT_SERVICE_ACCOUNT` veya `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- Teslim hedefleri için `spaces/<spaceId>` veya `users/<userId>` kullanın.
- `channels.googlechat.dangerouslyAllowNameMatching`, değiştirilebilir e-posta principal eşleştirmesini yeniden etkinleştirir (cam kırma uyumluluk modu).

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { allow: true, requireMention: true, allowBots: false },
        "#general": {
          allow: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Short answers only.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all | batched
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      textChunkLimit: 4000,
      chunkMode: "length",
      streaming: {
        mode: "partial", // off | partial | block | progress
        nativeTransport: true, // mode=partial olduğunda Slack yerel akış API'sini kullan
      },
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- **Socket mode** hem `botToken` hem `appToken` gerektirir (varsayılan hesap çevre geri dönüşü için `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN`).
- **HTTP mode** `botToken` artı `signingSecret` gerektirir (kök düzeyde veya hesap başına).
- `botToken`, `appToken`, `signingSecret` ve `userToken` düz metin
  dizeleri veya SecretRef nesnelerini kabul eder.
- Slack hesap anlık görüntüleri,
  `botTokenSource`, `botTokenStatus`, `appTokenStatus` ve HTTP modunda
  `signingSecretStatus` gibi kimlik bilgisi kaynağı/durumu alanlarını açığa çıkarır.
  `configured_unavailable`, hesabın
  SecretRef üzerinden yapılandırıldığı ancak mevcut komut/çalışma zamanı yolunun
  gizli değerini çözemediği anlamına gelir.
- `configWrites: false`, Slack tarafından başlatılan yapılandırma yazımlarını engeller.
- İsteğe bağlı `channels.slack.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde varsayılan hesap seçimini geçersiz kılar.
- `channels.slack.streaming.mode`, kanonik Slack akış modu anahtarıdır. `channels.slack.streaming.nativeTransport`, Slack'in yerel akış aktarımını kontrol eder. Eski `streamMode`, boolean `streaming` ve `nativeStreaming` değerleri otomatik taşınır.
- Teslim hedefleri için `user:<id>` (DM) veya `channel:<id>` kullanın.

**Tepki bildirim modları:** `off`, `own` (varsayılan), `all`, `allowlist` (`reactionAllowlist` içinden).

**Konu oturumu yalıtımı:** `thread.historyScope` konu başına (varsayılan) veya kanal genelinde paylaşılır. `thread.inheritParent`, üst kanal dökümünü yeni konulara kopyalar.

- Slack yerel akışı ve Slack asistan tarzı "is typing..." konu durumu bir yanıt konu hedefi gerektirir. Üst düzey DM'ler varsayılan olarak konu dışında kalır; bu nedenle konu tarzı önizleme yerine `typingReaction` veya normal teslim kullanırlar.
- `typingReaction`, bir yanıt çalışırken gelen Slack mesajına geçici bir tepki ekler, ardından tamamlandığında kaldırır. `"hourglass_flowing_sand"` gibi bir Slack emoji kısa kodu kullanın.
- `channels.slack.execApprovals`: Slack yerel exec onay teslimi ve onaylayıcı yetkilendirmesi. Discord ile aynı şema: `enabled` (`true`/`false`/`"auto"`), `approvers` (Slack kullanıcı kimlikleri), `agentFilter`, `sessionFilter` ve `target` (`"dm"`, `"channel"` veya `"both"`).

| Eylem grubu | Varsayılan | Notlar                 |
| ----------- | ---------- | ---------------------- |
| reactions   | etkin      | Tepki ver + tepkileri listele |
| messages    | etkin      | Oku/gönder/düzenle/sil |
| pins        | etkin      | Sabitle/kaldır/listele |
| memberInfo  | etkin      | Üye bilgisi            |
| emojiList   | etkin      | Özel emoji listesi     |

### Mattermost

Mattermost bir eklenti olarak gelir: `openclaw plugins install @openclaw/mattermost`.

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // açık katılım
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Ters proxy/genel dağıtımlar için isteğe bağlı açık URL
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

Sohbet modları: `oncall` (@-mention olduğunda yanıt ver, varsayılan), `onmessage` (her mesaj), `onchar` (tetikleyici önekle başlayan mesajlar).

Mattermost yerel komutları etkin olduğunda:

- `commands.callbackPath` tam URL değil, bir yol olmalıdır (örneğin `/api/channels/mattermost/command`).
- `commands.callbackUrl`, OpenClaw gateway uç noktasına çözülmeli ve Mattermost sunucusundan erişilebilir olmalıdır.
- Yerel slash callback'leri, slash komut kaydı sırasında Mattermost tarafından
  döndürülen komut başına tokenlarla doğrulanır. Kayıt başarısız olursa veya
  hiçbir komut etkinleştirilmezse OpenClaw callback'leri
  `Unauthorized: invalid command token.` ile reddeder.
- Özel/tailnet/iç callback barındırıcıları için Mattermost,
  `ServiceSettings.AllowedUntrustedInternalConnections` içinde callback barındırıcısını/alanını gerektirebilir.
  Tam URL değil, barındırıcı/alan değerleri kullanın.
- `channels.mattermost.configWrites`: Mattermost tarafından başlatılan yapılandırma yazımlarına izin ver veya engelle.
- `channels.mattermost.requireMention`: kanallarda yanıt vermeden önce `@mention` gerektir.
- `channels.mattermost.groups.<channelId>.requireMention`: kanal başına bahsetme geçidi geçersiz kılması (varsayılan için `"*"`).
- İsteğe bağlı `channels.mattermost.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde varsayılan hesap seçimini geçersiz kılar.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // isteğe bağlı hesap bağlama
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**Tepki bildirim modları:** `off`, `own` (varsayılan), `all`, `allowlist` (`reactionAllowlist` içinden).

- `channels.signal.account`: kanal başlangıcını belirli bir Signal hesap kimliğine sabitle.
- `channels.signal.configWrites`: Signal tarafından başlatılan yapılandırma yazımlarına izin ver veya engelle.
- İsteğe bağlı `channels.signal.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde varsayılan hesap seçimini geçersiz kılar.

### BlueBubbles

BlueBubbles önerilen iMessage yoludur (eklenti desteklidir, `channels.bluebubbles` altında yapılandırılır).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, grup denetimleri ve gelişmiş eylemler:
      // bkz. /channels/bluebubbles
    },
  },
}
```

- Burada kapsanan temel anahtar yolları: `channels.bluebubbles`, `channels.bluebubbles.dmPolicy`.
- İsteğe bağlı `channels.bluebubbles.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde varsayılan hesap seçimini geçersiz kılar.
- `type: "acp"` içeren üst düzey `bindings[]` girdileri, BlueBubbles konuşmalarını kalıcı ACP oturumlarına bağlayabilir. `match.peer.id` içinde bir BlueBubbles handle veya hedef dizesi (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) kullanın. Paylaşılan alan anlamları: [ACP Agents](/tr/tools/acp-agents#channel-specific-settings).
- Tam BlueBubbles kanal yapılandırması [BlueBubbles](/tr/channels/bluebubbles) içinde belgelenmiştir.

### iMessage

OpenClaw, `imsg rpc` çalıştırır (stdio üzerinden JSON-RPC). Daemon veya port gerekmez.

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      region: "US",
    },
  },
}
```

- İsteğe bağlı `channels.imessage.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde varsayılan hesap seçimini geçersiz kılar.

- Messages DB için Full Disk Access gerekir.
- `chat_id:<id>` hedeflerini tercih edin. Sohbetleri listelemek için `imsg chats --limit 20` kullanın.
- `cliPath`, SSH sarmalayıcısına işaret edebilir; SCP ek dosya alma için `remoteHost` ayarlayın (`host` veya `user@host`).
- `attachmentRoots` ve `remoteAttachmentRoots`, gelen ek dosya yollarını sınırlar (varsayılan: `/Users/*/Library/Messages/Attachments`).
- SCP sıkı ana bilgisayar anahtarı denetimi kullanır; bu nedenle geçiş barındırıcısı anahtarının zaten `~/.ssh/known_hosts` içinde bulunduğundan emin olun.
- `channels.imessage.configWrites`: iMessage tarafından başlatılan yapılandırma yazımlarına izin ver veya engelle.
- `type: "acp"` içeren üst düzey `bindings[]` girdileri, iMessage konuşmalarını kalıcı ACP oturumlarına bağlayabilir. `match.peer.id` içinde normalize edilmiş handle veya açık sohbet hedefi (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) kullanın. Paylaşılan alan anlamları: [ACP Agents](/tr/tools/acp-agents#channel-specific-settings).

<Accordion title="iMessage SSH sarmalayıcı örneği">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix uzantı desteklidir ve `channels.matrix` altında yapılandırılır.

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- Token kimlik doğrulaması `accessToken` kullanır; parola kimlik doğrulaması `userId` + `password` kullanır.
- `channels.matrix.proxy`, Matrix HTTP trafiğini açık bir HTTP(S) proxy üzerinden yönlendirir. Adlandırılmış hesaplar bunu `channels.matrix.accounts.<id>.proxy` ile geçersiz kılabilir.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork`, özel/iç homeserver'lara izin verir. `proxy` ve bu ağ açık katılımı bağımsız denetimlerdir.
- `channels.matrix.defaultAccount`, çok hesaplı kurulumlarda tercih edilen hesabı seçer.
- `channels.matrix.autoJoin` varsayılan olarak `off` olduğundan davet edilen odalar ve yeni DM tarzı davetler, `autoJoin: "allowlist"` ile `autoJoinAllowlist` veya `autoJoin: "always"` ayarlanana kadar yok sayılır.
- `channels.matrix.execApprovals`: Matrix yerel exec onay teslimi ve onaylayıcı yetkilendirmesi.
  - `enabled`: `true`, `false` veya `"auto"` (varsayılan). Otomatik modda exec onayları, `approvers` veya `commands.ownerAllowFrom` içinden onaylayıcılar çözümlenebildiğinde etkinleşir.
  - `approvers`: exec isteklerini onaylamasına izin verilen Matrix kullanıcı kimlikleri (ör. `@owner:example.org`).
  - `agentFilter`: isteğe bağlı ajan kimliği izin listesi. Tüm ajanlar için onay iletmek üzere boş bırakın.
  - `sessionFilter`: isteğe bağlı oturum anahtarı kalıpları (alt dizgi veya regex).
  - `target`: onay istemlerinin nereye gönderileceği. `"dm"` (varsayılan), `"channel"` (kaynak oda) veya `"both"`.
  - Hesap başına geçersiz kılmalar: `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope`, Matrix DM'lerinin oturumlar hâlinde nasıl gruplandığını kontrol eder: `per-user` (varsayılan) yönlendirilen eşe göre paylaşır, `per-room` ise her DM odasını yalıtır.
- Matrix durum probları ve canlı dizin aramaları, çalışma zamanı trafiğiyle aynı proxy ilkesini kullanır.
- Tam Matrix yapılandırması, hedefleme kuralları ve kurulum örnekleri [Matrix](/tr/channels/matrix) içinde belgelenmiştir.

### Microsoft Teams

Microsoft Teams uzantı desteklidir ve `channels.msteams` altında yapılandırılır.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, takım/kanal ilkeleri:
      // bkz. /channels/msteams
    },
  },
}
```

- Burada kapsanan temel anahtar yolları: `channels.msteams`, `channels.msteams.configWrites`.
- Tam Teams yapılandırması (kimlik bilgileri, webhook, DM/grup ilkesi, takım/kanal başına geçersiz kılmalar) [Microsoft Teams](/tr/channels/msteams) içinde belgelenmiştir.

### IRC

IRC uzantı desteklidir ve `channels.irc` altında yapılandırılır.

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- Burada kapsanan temel anahtar yolları: `channels.irc`, `channels.irc.dmPolicy`, `channels.irc.configWrites`, `channels.irc.nickserv.*`.
- İsteğe bağlı `channels.irc.defaultAccount`, yapılandırılmış bir hesap kimliğiyle eşleştiğinde varsayılan hesap seçimini geçersiz kılar.
- Tam IRC kanal yapılandırması (host/port/TLS/kanallar/izin listeleri/bahsetme geçidi) [IRC](/tr/channels/irc) içinde belgelenmiştir.

### Çok hesaplı (tüm kanallar)

Kanal başına birden fazla hesap çalıştırın (her birinin kendi `accountId` değeri vardır):

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- `default`, `accountId` belirtilmediğinde kullanılır (CLI + yönlendirme).
- Çevre tokenları yalnızca **varsayılan** hesap için geçerlidir.
- Temel kanal ayarları, hesap başına geçersiz kılınmadıkça tüm hesaplara uygulanır.
- Her hesabı farklı bir ajana yönlendirmek için `bindings[].match.accountId` kullanın.
- Tek hesaplı üst düzey kanal yapılandırmasındayken `openclaw channels add` (veya kanal onboarding) ile varsayılan olmayan bir hesap eklerseniz, OpenClaw önce hesap kapsamındaki üst düzey tek hesap değerlerini kanal hesap eşlemesine terfi ettirir; böylece özgün hesap çalışmaya devam eder. Çoğu kanal bunları `channels.<channel>.accounts.default` içine taşır; Matrix bunun yerine mevcut eşleşen adlandırılmış/varsayılan hedefi koruyabilir.
- Mevcut yalnızca kanal düzeyindeki bağlamalar (`accountId` olmadan) varsayılan hesapla eşleşmeye devam eder; hesap kapsamlı bağlamalar isteğe bağlı olmaya devam eder.
- `openclaw doctor --fix` ayrıca karışık şekilleri de onarır; hesap kapsamındaki üst düzey tek hesap değerlerini o kanal için seçilen terfi ettirilmiş hesaba taşır. Çoğu kanal `accounts.default` kullanır; Matrix mevcut eşleşen adlandırılmış/varsayılan hedefi koruyabilir.

### Diğer uzantı kanalları

Birçok uzantı kanalı `channels.<id>` olarak yapılandırılır ve özel kanal sayfalarında belgelenir (örneğin Feishu, Matrix, LINE, Nostr, Zalo, Nextcloud Talk, Synology Chat ve Twitch).
Tam kanal dizinine bakın: [Channels](/tr/channels).

### Grup sohbeti bahsetme geçidi

Grup mesajları varsayılan olarak **bahsetme gerektirir** (meta veri bahsetmesi veya güvenli regex kalıpları). WhatsApp, Telegram, Discord, Google Chat ve iMessage grup sohbetleri için geçerlidir.

**Bahsetme türleri:**

- **Meta veri bahsetmeleri**: Yerel platform @-bahsetmeleri. WhatsApp self-chat modunda yok sayılır.
- **Metin kalıpları**: `agents.list[].groupChat.mentionPatterns` içindeki güvenli regex kalıpları. Geçersiz kalıplar ve güvensiz iç içe tekrarlar yok sayılır.
- Bahsetme geçidi yalnızca tespit mümkün olduğunda uygulanır (yerel bahsetmeler veya en az bir kalıp).

```json5
{
  messages: {
    groupChat: { historyLimit: 50 },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` genel varsayılanı ayarlar. Kanallar `channels.<channel>.historyLimit` (veya hesap başına) ile geçersiz kılabilir. Devre dışı bırakmak için `0` ayarlayın.

#### DM geçmiş sınırları

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

Çözümleme: DM başına geçersiz kılma → sağlayıcı varsayılanı → sınır yok (tamamı tutulur).

Desteklenenler: `telegram`, `whatsapp`, `discord`, `slack`, `signal`, `imessage`, `msteams`.

#### Self-chat modu

Self-chat modunu etkinleştirmek için kendi numaranızı `allowFrom` içine ekleyin (yerel @-bahsetmeleri yok sayar, yalnızca metin kalıplarına yanıt verir):

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### Komutlar (sohbet komutu işleme)

```json5
{
  commands: {
    native: "auto", // desteklendiğinde yerel komutları kaydet
    nativeSkills: "auto", // desteklendiğinde yerel skill komutlarını kaydet
    text: true, // sohbet mesajlarında /commands ayrıştır
    bash: false, // ! izin ver (takma ad: /bash)
    bashForegroundMs: 2000,
    config: false, // /config izin ver
    mcp: false, // /mcp izin ver
    plugins: false, // /plugins izin ver
    debug: false, // /debug izin ver
    restart: true, // /restart + gateway yeniden başlatma aracı izin ver
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="Komut ayrıntıları">

- Bu blok komut yüzeylerini yapılandırır. Güncel yerleşik + paketlenmiş komut kataloğu için bkz. [Slash Commands](/tr/tools/slash-commands).
- Bu sayfa bir **yapılandırma anahtarı başvurusu**dur, tam komut kataloğu değildir. QQ Bot `/bot-ping` `/bot-help` `/bot-logs`, LINE `/card`, device-pair `/pair`, memory `/dreaming`, phone-control `/phone` ve Talk `/voice` gibi kanal/eklentiye ait komutlar kanal/eklenti sayfalarında ve [Slash Commands](/tr/tools/slash-commands) içinde belgelenmiştir.
- Metin komutları, başında `/` bulunan **bağımsız** mesajlar olmalıdır.
- `native: "auto"` Discord/Telegram için yerel komutları açar, Slack'i kapalı bırakır.
- `nativeSkills: "auto"` Discord/Telegram için yerel skill komutlarını açar, Slack'i kapalı bırakır.
- Kanal başına geçersiz kılma: `channels.discord.commands.native` (bool veya `"auto"`). `false`, daha önce kaydedilmiş komutları temizler.
- Yerel skill kaydını `channels.<provider>.commands.nativeSkills` ile kanal başına geçersiz kılın.
- `channels.telegram.customCommands`, ek Telegram bot menü girdileri ekler.
- `bash: true`, ana bilgisayar kabuğu için `! <cmd>` etkinleştirir. `tools.elevated.enabled` ve gönderenin `tools.elevated.allowFrom.<channel>` içinde olması gerekir.
- `config: true`, `/config` etkinleştirir (`openclaw.json` okur/yazar). Gateway `chat.send` istemcileri için kalıcı `/config set|unset` yazımları ayrıca `operator.admin` gerektirir; salt okunur `/config show` normal yazma kapsamlı operator istemcileri için kullanılabilir kalır.
- `mcp: true`, `mcp.servers` altındaki OpenClaw tarafından yönetilen MCP sunucu yapılandırması için `/mcp` etkinleştirir.
- `plugins: true`, eklenti keşfi, kurulum ve etkinleştirme/devre dışı bırakma denetimleri için `/plugins` etkinleştirir.
- `channels.<provider>.configWrites`, kanal başına yapılandırma değişikliklerini denetler (varsayılan: true).
- Çok hesaplı kanallar için `channels.<provider>.accounts.<id>.configWrites`, o hesabı hedefleyen yazımları da denetler (örneğin `/allowlist --config --account <id>` veya `/config set channels.<provider>.accounts.<id>...`).
- `restart: false`, `/restart` ve gateway yeniden başlatma aracı eylemlerini devre dışı bırakır. Varsayılan: `true`.
- `ownerAllowFrom`, yalnızca sahip için olan komutlar/araçlar için açık sahip izin listesidir. `allowFrom`'dan ayrıdır.
- `ownerDisplay: "hash"`, sistem isteminde sahip kimliklerini hash'ler. Hashlemeyi denetlemek için `ownerDisplaySecret` ayarlayın.
- `allowFrom` sağlayıcı başınadır. Ayarlandığında **tek** yetkilendirme kaynağı budur (kanal izin listeleri/eşleştirme ve `useAccessGroups` yok sayılır).
- `useAccessGroups: false`, `allowFrom` ayarlı değilken komutların erişim grubu ilkelerini atlamasına izin verir.
- Komut dokümanları eşlemesi:
  - yerleşik + paketlenmiş katalog: [Slash Commands](/tr/tools/slash-commands)
  - kanala özgü komut yüzeyleri: [Channels](/tr/channels)
  - QQ Bot komutları: [QQ Bot](/tr/channels/qqbot)
  - eşleştirme komutları: [Pairing](/tr/channels/pairing)
  - LINE kart komutu: [LINE](/tr/channels/line)
  - memory dreaming: [Dreaming](/tr/concepts/dreaming)

</Accordion>

---

## Ajan varsayılanları

### `agents.defaults.workspace`

Varsayılan: `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

Sistem isteminin Runtime satırında gösterilen isteğe bağlı depo kökü. Ayarlanmazsa OpenClaw, çalışma alanından yukarı doğru ilerleyerek otomatik algılar.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

`agents.list[].skills` ayarlamayan ajanlar için isteğe bağlı varsayılan skill izin listesi.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // github, weather devralır
      { id: "docs", skills: ["docs-search"] }, // varsayılanların yerine geçer
      { id: "locked-down", skills: [] }, // skill yok
    ],
  },
}
```

- Varsayılan olarak kısıtlanmamış skills için `agents.defaults.skills` alanını atlayın.
- Varsayılanları devralmak için `agents.list[].skills` alanını atlayın.
- Hiç skill olmaması için `agents.list[].skills: []` ayarlayın.
- Boş olmayan bir `agents.list[].skills` listesi, o ajan için son kümedir;
  varsayılanlarla birleştirilmez.

### `agents.defaults.skipBootstrap`

Çalışma alanı bootstrap dosyalarının (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`) otomatik oluşturulmasını devre dışı bırakır.

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

Çalışma alanı bootstrap dosyalarının sistem istemine ne zaman enjekte edildiğini denetler. Varsayılan: `"always"`.

- `"continuation-skip"`: güvenli devam turları (tamamlanmış bir assistant yanıtından sonra), çalışma alanı bootstrap yeniden enjeksiyonunu atlar ve istem boyutunu azaltır. Heartbeat çalışmaları ve sıkıştırma sonrası yeniden denemeler yine de bağlamı yeniden kurar.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

Kesmeden önce çalışma alanı bootstrap dosyası başına maksimum karakter sayısı. Varsayılan: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

Tüm çalışma alanı bootstrap dosyalarından enjekte edilen toplam maksimum karakter sayısı. Varsayılan: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Bootstrap bağlamı kesildiğinde ajana görünür uyarı metnini denetler.
Varsayılan: `"once"`.

- `"off"`: sistem istemine hiçbir zaman uyarı metni enjekte etme.
- `"once"`: benzersiz her kesme imzası için uyarıyı bir kez enjekte et (önerilir).
- `"always"`: kesme mevcut olduğunda her çalıştırmada uyarıyı enjekte et.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

Sağlayıcı çağrılarından önce döküm/araç görsel bloklarında en uzun görsel kenarı için maksimum piksel boyutu.
Varsayılan: `1200`.

Daha düşük değerler genellikle ekran görüntüsü yoğun çalıştırmalarda vision-token kullanımını ve istek yük boyutunu azaltır.
Daha yüksek değerler daha fazla görsel ayrıntıyı korur.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

Sistem istemi bağlamı için saat dilimi (mesaj zaman damgaları değil). Ana makinenin saat dilimine geri döner.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Sistem istemindeki saat biçimi. Varsayılan: `auto` (OS tercihi).

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
        fallbacks: ["google/gemini-3.1-flash-image-preview"],
      },
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-i2v"],
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // genel varsayılan sağlayıcı parametreleri
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 3,
    },
  },
}
```

- `model`: ya bir dize (`"provider/model"`) ya da bir nesne (`{ primary, fallbacks }`) kabul eder.
  - Dize biçimi yalnızca birincil modeli ayarlar.
  - Nesne biçimi birincil modeli artı sıralı failover modellerini ayarlar.
- `imageModel`: ya bir dize (`"provider/model"`) ya da bir nesne (`{ primary, fallbacks }`) kabul eder.
  - `image` araç yolu tarafından vision-model yapılandırması olarak kullanılır.
  - Ayrıca seçili/varsayılan model görsel girdiyi kabul edemediğinde geri dönüş yönlendirmesi olarak kullanılır.
- `imageGenerationModel`: ya bir dize (`"provider/model"`) ya da bir nesne (`{ primary, fallbacks }`) kabul eder.
  - Paylaşılan görsel üretimi yeteneği ve görsel üreten tüm gelecekteki araç/eklenti yüzeyleri tarafından kullanılır.
  - Tipik değerler: yerel Gemini görsel üretimi için `google/gemini-3.1-flash-image-preview`, fal için `fal/fal-ai/flux/dev` veya OpenAI Images için `openai/gpt-image-1`.
  - Doğrudan bir sağlayıcı/model seçerseniz eşleşen sağlayıcı auth/API anahtarını da yapılandırın (örneğin `google/*` için `GEMINI_API_KEY` veya `GOOGLE_API_KEY`, `openai/*` için `OPENAI_API_KEY`, `fal/*` için `FAL_KEY`).
  - Belirtilmezse `image_generate` yine de auth destekli bir sağlayıcı varsayılanı çıkarabilir. Önce mevcut varsayılan sağlayıcıyı, ardından sağlayıcı kimliği sırasına göre kayıtlı kalan görsel üretim sağlayıcılarını dener.
- `musicGenerationModel`: ya bir dize (`"provider/model"`) ya da bir nesne (`{ primary, fallbacks }`) kabul eder.
  - Paylaşılan müzik üretimi yeteneği ve yerleşik `music_generate` aracı tarafından kullanılır.
  - Tipik değerler: `google/lyria-3-clip-preview`, `google/lyria-3-pro-preview` veya `minimax/music-2.5+`.
  - Belirtilmezse `music_generate` yine de auth destekli bir sağlayıcı varsayılanı çıkarabilir. Önce mevcut varsayılan sağlayıcıyı, ardından sağlayıcı kimliği sırasına göre kayıtlı kalan müzik üretim sağlayıcılarını dener.
  - Doğrudan bir sağlayıcı/model seçerseniz eşleşen sağlayıcı auth/API anahtarını da yapılandırın.
- `videoGenerationModel`: ya bir dize (`"provider/model"`) ya da bir nesne (`{ primary, fallbacks }`) kabul eder.
  - Paylaşılan video üretimi yeteneği ve yerleşik `video_generate` aracı tarafından kullanılır.
  - Tipik değerler: `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` veya `qwen/wan2.7-r2v`.
  - Belirtilmezse `video_generate` yine de auth destekli bir sağlayıcı varsayılanı çıkarabilir. Önce mevcut varsayılan sağlayıcıyı, ardından sağlayıcı kimliği sırasına göre kayıtlı kalan video üretim sağlayıcılarını dener.
  - Doğrudan bir sağlayıcı/model seçerseniz eşleşen sağlayıcı auth/API anahtarını da yapılandırın.
  - Paketlenmiş Qwen video üretim sağlayıcısı en fazla 1 çıktı videosu, 1 giriş görseli, 4 giriş videosu, 10 saniye süre ve sağlayıcı düzeyinde `size`, `aspectRatio`, `resolution`, `audio` ve `watermark` seçeneklerini destekler.
- `pdfModel`: ya bir dize (`"provider/model"`) ya da bir nesne (`{ primary, fallbacks }`) kabul eder.
  - `pdf` aracı tarafından model yönlendirmesi için kullanılır.
  - Belirtilmezse PDF aracı önce `imageModel`'e, sonra çözümlenmiş oturum/varsayılan modele geri döner.
- `pdfMaxBytesMb`: çağrı anında `maxBytesMb` geçirilmediğinde `pdf` aracı için varsayılan PDF boyut sınırı.
- `pdfMaxPages`: `pdf` aracında çıkarım geri dönüş modu tarafından dikkate alınan varsayılan maksimum sayfa sayısı.
- `verboseDefault`: ajanlar için varsayılan ayrıntı düzeyi. Değerler: `"off"`, `"on"`, `"full"`. Varsayılan: `"off"`.
- `elevatedDefault`: ajanlar için varsayılan elevated-output düzeyi. Değerler: `"off"`, `"on"`, `"ask"`, `"full"`. Varsayılan: `"on"`.
- `model.primary`: `provider/model` biçimi (ör. `openai/gpt-5.4`). Sağlayıcıyı atarsanız OpenClaw önce bir takma adı, sonra tam o model kimliği için benzersiz yapılandırılmış sağlayıcı eşleşmesini dener ve ancak sonrasında yapılandırılmış varsayılan sağlayıcıya geri döner (kullanımdan kaldırılmış uyumluluk davranışı, bu yüzden açık `provider/model` tercih edin). Bu sağlayıcı artık yapılandırılmış varsayılan modeli sunmuyorsa OpenClaw, eski kaldırılmış sağlayıcı varsayılanını yüzeye çıkarmak yerine ilk yapılandırılmış sağlayıcı/modele geri döner.
- `models`: `/model` için yapılandırılmış model kataloğu ve izin listesi. Her giriş `alias` (kısayol) ve `params` (sağlayıcıya özgü; örneğin `temperature`, `maxTokens`, `cacheRetention`, `context1m`) içerebilir.
- `params`: tüm modellere uygulanan genel varsayılan sağlayıcı parametreleri. `agents.defaults.params` altında ayarlanır (ör. `{ cacheRetention: "long" }`).
- `params` birleştirme önceliği (yapılandırma): `agents.defaults.params` (genel taban), `agents.defaults.models["provider/model"].params` (model başına) tarafından geçersiz kılınır, ardından `agents.list[].params` (eşleşen ajan kimliği) anahtar başına geçersiz kılar. Ayrıntılar için bkz. [Prompt Caching](/tr/reference/prompt-caching).
- Bu alanları değiştiren yapılandırma yazıcıları (örneğin `/models set`, `/models set-image` ve fallback ekle/kaldır komutları) kanonik nesne biçimini kaydeder ve mümkün olduğunda mevcut fallback listelerini korur.
- `maxConcurrent`: oturumlar arasında en fazla paralel ajan çalışması (her oturum yine de seri hâlde). Varsayılan: 4.

**Yerleşik takma ad kısaltmaları** (`agents.defaults.models` içinde model varsa uygulanır):

| Takma ad            | Model                                  |
| ------------------- | -------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`            |
| `sonnet`            | `anthropic/claude-sonnet-4-6`          |
| `gpt`               | `openai/gpt-5.4`                       |
| `gpt-mini`          | `openai/gpt-5.4-mini`                  |
| `gpt-nano`          | `openai/gpt-5.4-nano`                  |
| `gemini`            | `google/gemini-3.1-pro-preview`        |
| `gemini-flash`      | `google/gemini-3-flash-preview`        |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

Yapılandırdığınız takma adlar her zaman varsayılanları geçersiz kılar.

Z.AI GLM-4.x modelleri, `--thinking off` ayarlamadığınız veya `agents.defaults.models["zai/<model>"].params.thinking` değerini kendiniz tanımlamadığınız sürece thinking modunu otomatik etkinleştirir.
Z.AI modelleri, araç çağrısı akışı için varsayılan olarak `tool_stream` etkinleştirir. Devre dışı bırakmak için `agents.defaults.models["zai/<model>"].params.tool_stream` değerini `false` ayarlayın.
Anthropic Claude 4.6 modelleri, açık bir thinking düzeyi ayarlanmadığında varsayılan olarak `adaptive` thinking kullanır.

### `agents.defaults.cliBackends`

Yalnızca metin tabanlı geri dönüş çalıştırmaları için isteğe bağlı CLI arka uçları (araç çağrısı yok). API sağlayıcıları başarısız olduğunda yedek olarak kullanışlıdır.

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          modelArg: "--model",
          sessionArg: "--session",
          sessionMode: "existing",
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
        },
      },
    },
  },
}
```

- CLI arka uçları önce metne yöneliktir; araçlar her zaman devre dışıdır.
- `sessionArg` ayarlanmışsa oturumlar desteklenir.
- `imageArg` dosya yollarını kabul ediyorsa görsel geçişi desteklenir.

### `agents.defaults.systemPromptOverride`

OpenClaw tarafından oluşturulan tüm sistem istemini sabit bir dizeyle değiştirin. Varsayılan düzeyde (`agents.defaults.systemPromptOverride`) veya ajan başına (`agents.list[].systemPromptOverride`) ayarlayın. Ajan başına değerler önceliklidir; boş veya yalnızca boşluk içeren değer yok sayılır. Kontrollü istem deneyleri için kullanışlıdır.

```json5
{
  agents: {
    defaults: {
      systemPromptOverride: "You are a helpful assistant.",
    },
  },
}
```

### `agents.defaults.heartbeat`

Dönemsel heartbeat çalışmaları.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m devre dışı bırakır
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // varsayılan: true; false Heartbeat bölümünü sistem isteminden çıkarır
        lightContext: false, // varsayılan: false; true çalışma alanı bootstrap dosyalarından yalnızca HEARTBEAT.md dosyasını tutar
        isolatedSession: false, // varsayılan: false; true her heartbeat'i yeni bir oturumda çalıştırır (konuşma geçmişi yok)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (varsayılan) | block
        target: "none", // varsayılan: none | seçenekler: last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
      },
    },
  },
}
```

- `every`: süre dizesi (ms/s/m/h). Varsayılan: `30m` (API-key auth) veya `1h` (OAuth auth). Devre dışı bırakmak için `0m` ayarlayın.
- `includeSystemPromptSection`: false olduğunda Heartbeat bölümünü sistem isteminden çıkarır ve `HEARTBEAT.md` enjeksiyonunu bootstrap bağlamından atlar. Varsayılan: `true`.
- `suppressToolErrorWarnings`: true olduğunda heartbeat çalışmaları sırasında araç hata uyarı yüklerini bastırır.
- `directPolicy`: doğrudan/DM teslim ilkesi. `allow` (varsayılan) doğrudan hedef teslimine izin verir. `block` doğrudan hedef teslimini bastırır ve `reason=dm-blocked` üretir.
- `lightContext`: true olduğunda heartbeat çalışmaları hafif bootstrap bağlamı kullanır ve çalışma alanı bootstrap dosyalarından yalnızca `HEARTBEAT.md` kalır.
- `isolatedSession`: true olduğunda her heartbeat, önceki konuşma geçmişi olmadan taze bir oturumda çalışır. Cron `sessionTarget: "isolated"` ile aynı yalıtım düzeni. Heartbeat başına token maliyetini yaklaşık ~100K'dan ~2-5K tokene düşürür.
- Ajan başına: `agents.list[].heartbeat` ayarlayın. Herhangi bir ajan `heartbeat` tanımlarsa, heartbeat'leri **yalnızca o ajanlar** çalıştırır.
- Heartbeat'ler tam ajan turları çalıştırır — daha kısa aralıklar daha fazla token tüketir.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // kayıtlı bir compaction sağlayıcı eklentisinin kimliği (isteğe bağlı)
        timeoutSeconds: 900,
        reserveTokensFloor: 24000,
        identifierPolicy: "strict", // strict | off | custom
        identifierInstructions: "Preserve deployment IDs, ticket IDs, and host:port pairs exactly.", // identifierPolicy=custom olduğunda kullanılır
        postCompactionSections: ["Session Startup", "Red Lines"], // [] yeniden enjeksiyonu devre dışı bırakır
        model: "openrouter/anthropic/claude-sonnet-4-6", // isteğe bağlı yalnızca compaction model geçersiz kılması
        notifyUser: true, // compaction başladığında kullanıcıya kısa bildirim gönder (varsayılan: false)
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 6000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with the exact silent token NO_REPLY if nothing to store.",
        },
      },
    },
  },
}
```

- `mode`: `default` veya `safeguard` (uzun geçmişler için parçalı özetleme). Bkz. [Compaction](/tr/concepts/compaction).
- `provider`: kayıtlı bir compaction sağlayıcı eklentisinin kimliği. Ayarlandığında sağlayıcının `summarize()` işlevi yerleşik LLM özetlemesi yerine çağrılır. Başarısızlıkta yerleşiğe geri döner. Bir sağlayıcı ayarlamak `mode: "safeguard"` değerini zorunlu kılar. Bkz. [Compaction](/tr/concepts/compaction).
- `timeoutSeconds`: OpenClaw'ın bir compaction işlemini iptal etmeden önce izin verdiği tek bir compaction işlemi için azami saniye sayısı. Varsayılan: `900`.
- `identifierPolicy`: `strict` (varsayılan), `off` veya `custom`. `strict`, compaction özetlemesi sırasında yerleşik opak tanımlayıcı koruma yönergesini başa ekler.
- `identifierInstructions`: `identifierPolicy=custom` olduğunda kullanılan isteğe bağlı özel tanımlayıcı koruma metni.
- `postCompactionSections`: compaction sonrası yeniden enjekte edilecek isteğe bağlı AGENTS.md H2/H3 bölüm adları. Varsayılan: `["Session Startup", "Red Lines"]`; devre dışı bırakmak için `[]` ayarlayın. Ayarlanmamışsa veya açıkça bu varsayılan ikiliye ayarlanmışsa eski `Every Session`/`Safety` başlıkları da eski uyumluluk geri dönüşü olarak kabul edilir.
- `model`: yalnızca compaction özetlemesi için isteğe bağlı `provider/model-id` geçersiz kılması. Ana oturum bir modeli kullanırken compaction özetlerinin başka bir modelde çalışmasını istiyorsanız bunu kullanın; ayarlanmazsa compaction oturumun birincil modelini kullanır.
- `notifyUser`: `true` olduğunda compaction başladığında kullanıcıya kısa bir bildirim gönderir (örneğin `"Compacting context..."`). Compaction'ı sessiz tutmak için varsayılan olarak devre dışıdır.
- `memoryFlush`: dayanıklı anıları depolamak için otomatik compaction öncesi sessiz ajan dönüşü. Çalışma alanı salt okunursa atlanır.

### `agents.defaults.contextPruning`

LLM'ye göndermeden önce bellek içi bağlamdan **eski araç sonuçlarını** budar. Diskteki oturum geçmişini **değiştirmez**.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // süre (ms/s/m/h), varsayılan birim: dakika
        keepLastAssistants: 3,
        softTrimRatio: 0.3,
        hardClearRatio: 0.5,
        minPrunableToolChars: 50000,
        softTrim: { maxChars: 4000, headChars: 1500, tailChars: 1500 },
        hardClear: { enabled: true, placeholder: "[Old tool result content cleared]" },
        tools: { deny: ["browser", "canvas"] },
      },
    },
  },
}
```

<Accordion title="cache-ttl mod davranışı">

- `mode: "cache-ttl"` budama geçişlerini etkinleştirir.
- `ttl`, budamanın ne sıklıkla tekrar çalışabileceğini denetler (son önbellek dokunuşundan sonra).
- Budama önce büyük araç sonuçlarını yumuşak keser, sonra gerekirse eski araç sonuçlarını sert temizler.

**Yumuşak kesme** başı + sonu tutar ve ortaya `...` ekler.

**Sert temizleme** tüm araç sonucunu yer tutucuyla değiştirir.

Notlar:

- Görsel blokları asla kırpılmaz/temizlenmez.
- Oranlar tam token sayıları değil, karakter tabanlıdır (yaklaşık).
- `keepLastAssistants` kadar assistant mesajı yoksa budama atlanır.

</Accordion>

Davranış ayrıntıları için bkz. [Session Pruning](/tr/concepts/session-pruning).

### Blok akışı

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom (minMs/maxMs kullanın)
    },
  },
}
```

- Telegram dışındaki kanallar, blok yanıtları etkinleştirmek için açık `*.blockStreaming: true` gerektirir.
- Kanal geçersiz kılmaları: `channels.<channel>.blockStreamingCoalesce` (ve hesap başına varyantlar). Signal/Slack/Discord/Google Chat varsayılanı `minChars: 1500`.
- `humanDelay`: blok yanıtlar arasında rastgele duraklama. `natural` = 800–2500ms. Ajan başına geçersiz kılma: `agents.list[].humanDelay`.

Davranış + parçalara ayırma ayrıntıları için bkz. [Streaming](/tr/concepts/streaming).

### Yazıyor göstergeleri

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- Varsayılanlar: doğrudan sohbetler/bahsetmeler için `instant`, bahsetilmeyen grup sohbetleri için `message`.
- Oturum başına geçersiz kılmalar: `session.typingMode`, `session.typingIntervalSeconds`.

Bkz. [Typing Indicators](/tr/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Gömülü ajan için isteğe bağlı sandboxing. Tam kılavuz için bkz. [Sandboxing](/tr/gateway/sandboxing).

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        backend: "docker", // docker | ssh | openshell
        scope: "agent", // session | agent | shared
        workspaceAccess: "none", // none | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefs / satır içi içerikler de desteklenir:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

<Accordion title="Sandbox ayrıntıları">

**Arka uç:**

- `docker`: yerel Docker çalışma zamanı (varsayılan)
- `ssh`: genel SSH destekli uzak çalışma zamanı
- `openshell`: OpenShell çalışma zamanı

`backend: "openshell"` seçildiğinde çalışma zamanına özgü ayarlar
`plugins.entries.openshell.config` altına taşınır.

**SSH arka uç yapılandırması:**

- `target`: `user@host[:port]` biçiminde SSH hedefi
- `command`: SSH istemci komutu (varsayılan: `ssh`)
- `workspaceRoot`: kapsam başına çalışma alanları için kullanılan mutlak uzak kök
- `identityFile` / `certificateFile` / `knownHostsFile`: OpenSSH'ye geçirilen mevcut yerel dosyalar
- `identityData` / `certificateData` / `knownHostsData`: OpenClaw'un çalışma zamanında geçici dosyalara dönüştürdüğü satır içi içerikler veya SecretRef'ler
- `strictHostKeyChecking` / `updateHostKeys`: OpenSSH host-key ilkesi ayarları

**SSH auth önceliği:**

- `identityData`, `identityFile` değerini geçersiz kılar
- `certificateData`, `certificateFile` değerini geçersiz kılar
- `knownHostsData`, `knownHostsFile` değerini geçersiz kılar
- SecretRef destekli `*Data` değerleri, sandbox oturumu başlamadan önce etkin gizli bilgiler çalışma zamanı anlık görüntüsünden çözümlenir

**SSH arka uç davranışı:**

- oluşturma veya yeniden oluşturma sonrası uzak çalışma alanını bir kez tohumlar
- ardından uzak SSH çalışma alanını kanonik tutar
- `exec`, dosya araçları ve medya yollarını SSH üzerinden yönlendirir
- uzak değişiklikleri ana makineye otomatik eşitlemez
- sandbox tarayıcı kapsayıcılarını desteklemez

**Çalışma alanı erişimi:**

- `none`: `~/.openclaw/sandboxes` altında kapsam başına sandbox çalışma alanı
- `ro`: `/workspace` altında sandbox çalışma alanı, `/agent` altında salt okunur bağlı ajan çalışma alanı
- `rw`: `/workspace` altında okuma/yazma bağlı ajan çalışma alanı

**Kapsam:**

- `session`: oturum başına kapsayıcı + çalışma alanı
- `agent`: ajan başına bir kapsayıcı + çalışma alanı (varsayılan)
- `shared`: paylaşılan kapsayıcı ve çalışma alanı (oturumlar arası yalıtım yok)

**OpenShell eklenti yapılandırması:**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // mirror | remote
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // isteğe bağlı
          gatewayEndpoint: "https://lab.example", // isteğe bağlı
          policy: "strict", // isteğe bağlı OpenShell policy id
          providers: ["openai"], // isteğe bağlı
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**OpenShell modu:**

- `mirror`: exec öncesi yerelden uzağa tohumla, exec sonrası geri eşitle; yerel çalışma alanı kanonik kalır
- `remote`: sandbox oluşturulduğunda uzak tarafı bir kez tohumla, ardından uzak çalışma alanını kanonik tut

`remote` modunda, OpenClaw dışında yapılan ana makine yerel düzenlemeleri tohumlama adımından sonra sandbox içine otomatik eşitlenmez.
Taşıma SSH üzerinden OpenShell sandbox içine yapılır, ancak sandbox yaşam döngüsü ve isteğe bağlı mirror eşitleme eklenti tarafından yönetilir.

**`setupCommand`** kapsayıcı oluşturulduktan sonra bir kez çalışır (`sh -lc` üzerinden). Ağ çıkışı, yazılabilir kök ve root kullanıcı gerektirir.

**Kapsayıcılar varsayılan olarak `network: "none"`** ile gelir — ajan dış erişime ihtiyaç duyuyorsa `"bridge"` (veya özel bir bridge ağı) olarak ayarlayın.
`"host"` engellenir. `"container:<id>"`, yalnızca açıkça
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` ayarlarsanız
varsayılan olarak izin verilir (cam kırma).

**Gelen ek dosyalar**, etkin çalışma alanında `media/inbound/*` içine hazırlanır.

**`docker.binds`**, ek ana makine dizinlerini bağlar; genel ve ajan başına bağlamalar birleştirilir.

**Sandboxed browser** (`sandbox.browser.enabled`): kapsayıcı içinde Chromium + CDP. noVNC URL'si sistem istemine enjekte edilir. `openclaw.json` içinde `browser.enabled` gerektirmez.
noVNC gözlemci erişimi varsayılan olarak VNC auth kullanır ve OpenClaw, parolayı paylaşılan URL'de açığa çıkarmak yerine kısa ömürlü bir token URL üretir.

- `allowHostControl: false` (varsayılan), sandbox'lı oturumların ana makine tarayıcısını hedeflemesini engeller.
- `network` varsayılan olarak `openclaw-sandbox-browser` olur (özel bridge ağı). Genel bridge bağlantısı istiyorsanız yalnızca o durumda `bridge` ayarlayın.
- `cdpSourceRange`, isteğe bağlı olarak CDP girişini kapsayıcı kenarında bir CIDR aralığıyla sınırlar (örneğin `172.21.0.1/32`).
- `sandbox.browser.binds`, ek ana makine dizinlerini yalnızca sandbox tarayıcı kapsayıcısına bağlar. Ayarlandığında (`[]` dahil) tarayıcı kapsayıcısı için `docker.binds` yerine geçer.
- Başlatma varsayılanları `scripts/sandbox-browser-entrypoint.sh` içinde tanımlanır ve kapsayıcı barındırıcıları için ayarlanmıştır:
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<OPENCLAW_BROWSER_CDP_PORT değerinden türetilir>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-3d-apis`
  - `--disable-gpu`
  - `--disable-software-rasterizer`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-features=TranslateUI`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--renderer-process-limit=2`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--disable-extensions` (varsayılan olarak etkin)
  - `--disable-3d-apis`, `--disable-software-rasterizer` ve `--disable-gpu`
    varsayılan olarak etkindir ve
    WebGL/3D kullanımı gerektiriyorsa
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` ile devre dışı bırakılabilir.
  - `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0`, iş akışınız
    bunlara bağlıysa uzantıları yeniden etkinleştirir.
  - `--renderer-process-limit=2`,
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` ile değiştirilebilir;
    Chromium'un varsayılan süreç sınırını kullanmak için `0` ayarlayın.
  - ayrıca `noSandbox` etkin olduğunda `--no-sandbox` ve `--disable-setuid-sandbox`.
  - Varsayılanlar kapsayıcı imajının temelidir; kapsayıcı varsayılanlarını değiştirmek için
    özel giriş noktasına sahip özel tarayıcı imajı kullanın.

</Accordion>

Tarayıcı sandboxing ve `sandbox.docker.binds` yalnızca Docker içindir.

İmajları oluşturun:

```bash
scripts/sandbox-setup.sh           # ana sandbox imajı
scripts/sandbox-browser-setup.sh   # isteğe bağlı tarayıcı imajı
```

### `agents.list` (ajan başına geçersiz kılmalar)

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // veya { primary, fallbacks }
        thinkingDefault: "high", // ajan başına thinking düzeyi geçersiz kılması
        reasoningDefault: "on", // ajan başına reasoning görünürlüğü geçersiz kılması
        fastModeDefault: false, // ajan başına fast mode geçersiz kılması
        params: { cacheRetention: "none" }, // eşleşen defaults.models parametrelerini anahtar başına geçersiz kılar
        skills: ["docs-search"], // ayarlandığında agents.defaults.skills yerine geçer
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`: sabit ajan kimliği (gerekli).
- `default`: birden fazlası ayarlanırsa ilki kazanır (uyarı kaydedilir). Hiçbiri ayarlanmazsa listenin ilk girdisi varsayılandır.
- `model`: dize biçimi yalnızca `primary` değerini geçersiz kılar; nesne biçimi `{ primary, fallbacks }` her ikisini de geçersiz kılar (`[]` genel fallback'leri devre dışı bırakır). Yalnızca `primary` geçersiz kılan cron işleri, `fallbacks: []` ayarlamadıkça varsayılan fallback'leri yine de devralır.
- `params`: `agents.defaults.models` içindeki seçili model girdisi üzerine birleştirilen ajan başına akış parametreleri. Tüm model kataloğunu kopyalamadan `cacheRetention`, `temperature` veya `maxTokens` gibi ajana özgü geçersiz kılmalar için kullanın.
- `skills`: isteğe bağlı ajan başına skill izin listesi. Atlanırsa ajan, ayarlıysa `agents.defaults.skills` değerini devralır; açık bir liste varsayılanlarla birleşmek yerine onların yerine geçer ve `[]` hiç skill olmadığı anlamına gelir.
- `thinkingDefault`: isteğe bağlı ajan başına varsayılan thinking düzeyi (`off | minimal | low | medium | high | xhigh | adaptive`). İleti başına veya oturum geçersiz kılması ayarlı değilse bu ajan için `agents.defaults.thinkingDefault` değerini geçersiz kılar.
- `reasoningDefault`: isteğe bağlı ajan başına varsayılan reasoning görünürlüğü (`on | off | stream`). İleti başına veya oturum reasoning geçersiz kılması ayarlı değilse uygulanır.
- `fastModeDefault`: fast mode için isteğe bağlı ajan başına varsayılan (`true | false`). İleti başına veya oturum fast-mode geçersiz kılması ayarlı değilse uygulanır.
- `runtime`: isteğe bağlı ajan başına çalışma zamanı tanımlayıcısı. Ajanın varsayılan olarak ACP harness oturumları kullanması gerekiyorsa `type: "acp"` ve `runtime.acp` varsayılanları (`agent`, `backend`, `mode`, `cwd`) kullanın.
- `identity.avatar`: çalışma alanı göreli yol, `http(s)` URL'si veya `data:` URI.
- `identity`, varsayılanları türetir: `ackReaction` `emoji`den, `mentionPatterns` `name`/`emoji`den.
- `subagents.allowAgents`: `sessions_spawn` için ajan kimlikleri izin listesi (`["*"]` = herhangi biri; varsayılan: yalnızca aynı ajan).
- Sandbox devralma koruması: istekçi oturum sandbox içindeyse `sessions_spawn`, sandboxsız çalışacak hedefleri reddeder.
- `subagents.requireAgentId`: true olduğunda `agentId` içermeyen `sessions_spawn` çağrılarını engeller (açık profil seçimini zorlar; varsayılan: false).

---

## Çok ajanlı yönlendirme

Bir Gateway içinde birden çok yalıtılmış ajan çalıştırın. Bkz. [Multi-Agent](/tr/concepts/multi-agent).

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### Binding eşleşme alanları

- `type` (isteğe bağlı): normal yönlendirme için `route` (eksik type varsayılan olarak route olur), kalıcı ACP konuşma bağlamaları için `acp`.
- `match.channel` (gerekli)
- `match.accountId` (isteğe bağlı; `*` = herhangi bir hesap; atlanırsa = varsayılan hesap)
- `match.peer` (isteğe bağlı; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (isteğe bağlı; kanala özgü)
- `acp` (isteğe bağlı; yalnızca `type: "acp"` için): `{ mode, label, cwd, backend }`

**Belirleyici eşleşme sırası:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (tam eşleşme, peer/guild/team yok)
5. `match.accountId: "*"` (kanal genelinde)
6. Varsayılan ajan

Her katman içinde ilk eşleşen `bindings` girdisi kazanır.

`type: "acp"` girdileri için OpenClaw tam konuşma kimliğine göre çözümler (`match.channel` + hesap + `match.peer.id`) ve yukarıdaki route binding katman sırasını kullanmaz.

### Ajan başına erişim profilleri

<Accordion title="Tam erişim (sandbox yok)">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Salt okunur araçlar + çalışma alanı">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Dosya sistemi erişimi yok (yalnızca mesajlaşma)">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

Öncelik ayrıntıları için bkz. [Multi-Agent Sandbox & Tools](/tr/tools/multi-agent-sandbox-tools).

---

## Oturum

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    parentForkMaxTokens: 100000, // bunun üzerindeki token sayısında üst konu fork'u atlanır (0 devre dışı bırakır)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // süre veya false
      maxDiskBytes: "500mb", // isteğe bağlı kesin bütçe
      highWaterBytes: "400mb", // isteğe bağlı temizleme hedefi
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // varsayılan hareketsizlikten sonra otomatik odak kaldırma saat cinsinden (`0` devre dışı bırakır)
      maxAgeHours: 0, // varsayılan kesin maksimum yaş saat cinsinden (`0` devre dışı bırakır)
    },
    mainKey: "main", // eski (çalışma zamanı her zaman "main" kullanır)
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Oturum alanı ayrıntıları">

- **`scope`**: grup sohbeti bağlamları için temel oturum gruplama stratejisi.
  - `per-sender` (varsayılan): her gönderen bir kanal bağlamı içinde yalıtılmış bir oturum alır.
  - `global`: bir kanal bağlamındaki tüm katılımcılar tek bir oturumu paylaşır (yalnızca paylaşılan bağlam amaçlandığında kullanın).
- **`dmScope`**: DM'lerin nasıl gruplandığı.
  - `main`: tüm DM'ler ana oturumu paylaşır.
  - `per-peer`: kanallar arasında gönderen kimliğine göre yalıt.
  - `per-channel-peer`: kanal + gönderen başına yalıt (çok kullanıcılı inbox'lar için önerilir).
  - `per-account-channel-peer`: hesap + kanal + gönderen başına yalıt (çok hesaplı kullanım için önerilir).
- **`identityLinks`**: kanallar arası oturum paylaşımı için sağlayıcı önekli eşlere kanonik kimlik eşlemesi.
- **`reset`**: birincil sıfırlama ilkesi. `daily`, yerel saatte `atHour` zamanında sıfırlar; `idle`, `idleMinutes` sonrasında sıfırlar. İkisi de yapılandırılmışsa hangisinin süresi önce dolarsa o kazanır.
- **`resetByType`**: tür başına geçersiz kılmalar (`direct`, `group`, `thread`). Eski `dm`, `direct` için takma ad olarak kabul edilir.
- **`parentForkMaxTokens`**: çatallı konu oturumu oluşturulurken üst oturum `totalTokens` için izin verilen maksimum değer (varsayılan `100000`).
  - Üst `totalTokens` bu değerin üzerindeyse OpenClaw üst döküm geçmişini devralmak yerine yeni bir konu oturumu başlatır.
  - Bu korumayı devre dışı bırakmak ve her zaman üst fork'a izin vermek için `0` ayarlayın.
- **`mainKey`**: eski alan. Çalışma zamanı ana doğrudan sohbet kovası için her zaman `"main"` kullanır.
- **`agentToAgent.maxPingPongTurns`**: ajanlar arası değişim sırasında ajanlar arasındaki azami geri yanıt turu sayısı (tam sayı, aralık: `0`–`5`). `0`, ping-pong zincirini devre dışı bırakır.
- **`sendPolicy`**: `channel`, `chatType` (`direct|group|channel`, eski `dm` takma adıyla), `keyPrefix` veya `rawKeyPrefix` ile eşleştir. İlk deny kazanır.
- **`maintenance`**: oturum deposu temizleme + saklama denetimleri.
  - `mode`: `warn` yalnızca uyarı verir; `enforce` temizliği uygular.
  - `pruneAfter`: eski girdiler için yaş kesim noktası (varsayılan `30d`).
  - `maxEntries`: `sessions.json` içindeki maksimum girdi sayısı (varsayılan `500`).
  - `rotateBytes`: `sessions.json` bu boyutu aşınca döndürülür (varsayılan `10mb`).
  - `resetArchiveRetention`: `*.reset.<timestamp>` döküm arşivleri için saklama süresi. Varsayılan olarak `pruneAfter` değerini kullanır; devre dışı bırakmak için `false` ayarlayın.
  - `maxDiskBytes`: oturum dizini için isteğe bağlı disk bütçesi. `warn` modunda uyarı kaydeder; `enforce` modunda önce en eski artifaktları/oturumları kaldırır.
  - `highWaterBytes`: bütçe temizliği sonrası isteğe bağlı hedef. Varsayılan olarak `maxDiskBytes` değerinin `%80`idir.
- **`threadBindings`**: konuya bağlı oturum özellikleri için genel varsayılanlar.
  - `enabled`: ana varsayılan anahtar (sağlayıcılar geçersiz kılabilir; Discord `channels.discord.threadBindings.enabled` kullanır)
  - `idleHours`: saat cinsinden varsayılan hareketsizlik nedeniyle otomatik odak kaldırma (`0` devre dışı bırakır; sağlayıcılar geçersiz kılabilir)
  - `maxAgeHours`: saat cinsinden varsayılan kesin maksimum yaş (`0` devre dışı bırakır; sağlayıcılar geçersiz kılabilir)

</Accordion>

---

## Mesajlar

```json5
{
  messages: {
    responsePrefix: "🦞", // veya "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all
    removeAckAfterReply: false,
    queue: {
      mode: "collect", // steer | followup | collect | steer-backlog | steer+backlog | queue | interrupt
      debounceMs: 1000,
      cap: 20,
      drop: "summarize", // old | new | summarize
      byChannel: {
        whatsapp: "collect",
        telegram: "collect",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 devre dışı bırakır
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Yanıt öneki

Kanal/hesap başına geçersiz kılmalar: `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Çözümleme (en özeli kazanır): hesap → kanal → genel. `""` devre dışı bırakır ve zinciri durdurur. `"auto"` değeri `[{identity.name}]` türetir.

**Şablon değişkenleri:**

| Değişken         | Açıklama             | Örnek                      |
| ---------------- | -------------------- | -------------------------- |
| `{model}`        | Kısa model adı       | `claude-opus-4-6`          |
| `{modelFull}`    | Tam model tanımlayıcısı | `anthropic/claude-opus-4-6` |
| `{provider}`     | Sağlayıcı adı        | `anthropic`                |
| `{thinkingLevel}`| Geçerli thinking düzeyi | `high`, `low`, `off`    |
| `{identity.name}`| Ajan kimlik adı      | (`"auto"` ile aynı)        |

Değişkenler büyük/küçük harfe duyarsızdır. `{think}`, `{thinkingLevel}` için takma addır.

### Ack tepkisi

- Varsayılan olarak etkin ajanın `identity.emoji` değeri, yoksa `"👀"`. Devre dışı bırakmak için `""` ayarlayın.
- Kanal başına geçersiz kılmalar: `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Çözüm sırası: hesap → kanal → `messages.ackReaction` → kimlik geri dönüşü.
- Kapsam: `group-mentions` (varsayılan), `group-all`, `direct`, `all`.
- `removeAckAfterReply`: Slack, Discord ve Telegram'da yanıt sonrası ack'i kaldırır.
- `messages.statusReactions.enabled`: Slack, Discord ve Telegram'da yaşam döngüsü durum tepkilerini etkinleştirir.
  Slack ve Discord'da ayarlanmazsa, ack tepkileri etkinken durum tepkileri etkin kalır.
  Telegram'da yaşam döngüsü durum tepkilerini etkinleştirmek için bunu açıkça `true` yapın.

### Gelen debounce

Aynı gönderenden hızlı metin tabanlı mesajları tek bir ajan turunda toplar. Medya/ekler hemen flush edilir. Denetim komutları debounce'u atlar.

### TTS (text-to-speech)

```json5
{
  messages: {
    tts: {
      auto: "always", // off | always | inbound | tagged
      mode: "final", // final | all
      provider: "elevenlabs",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: { enabled: true },
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
      elevenlabs: {
        apiKey: "elevenlabs_api_key",
        baseUrl: "https://api.elevenlabs.io",
        voiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      openai: {
        apiKey: "openai_api_key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        voice: "alloy",
      },
    },
  },
}
```

- `auto`, varsayılan otomatik TTS modunu denetler: `off`, `always`, `inbound` veya `tagged`. `/tts on|off` yerel tercihleri geçersiz kılabilir ve `/tts status` etkin durumu gösterir.
- `summaryModel`, otomatik özet için `agents.defaults.model.primary` değerini geçersiz kılar.
- `modelOverrides` varsayılan olarak etkindir; `modelOverrides.allowProvider` varsayılanı `false`tur (açık katılım).
- API anahtarları `ELEVENLABS_API_KEY`/`XI_API_KEY` ve `OPENAI_API_KEY` değerlerine geri döner.
- `openai.baseUrl`, OpenAI TTS uç noktasını geçersiz kılar. Çözümleme sırası yapılandırma, sonra `OPENAI_TTS_BASE_URL`, sonra `https://api.openai.com/v1`.
- `openai.baseUrl`, OpenAI dışı bir uç noktaya işaret ettiğinde OpenClaw bunu OpenAI uyumlu bir TTS sunucusu olarak değerlendirir ve model/ses doğrulamasını gevşetir.

---

## Talk

Talk modu için varsayılanlar (macOS/iOS/Android).

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
    },
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

- `talk.provider`, birden çok Talk sağlayıcısı yapılandırıldığında `talk.providers` içindeki bir anahtarla eşleşmelidir.
- Eski düz Talk anahtarları (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) yalnızca uyumluluk içindir ve `talk.providers.<provider>` içine otomatik taşınır.
- Ses kimlikleri `ELEVENLABS_VOICE_ID` veya `SAG_VOICE_ID` değerlerine geri döner.
- `providers.*.apiKey`, düz metin dizelerini veya SecretRef nesnelerini kabul eder.
- `ELEVENLABS_API_KEY` geri dönüşü yalnızca hiçbir Talk API anahtarı yapılandırılmadığında uygulanır.
- `providers.*.voiceAliases`, Talk yönergelerinin dostça adlar kullanmasını sağlar.
- `silenceTimeoutMs`, Talk modunun kullanıcı sessizliğinden sonra dökümü göndermeden önce ne kadar beklediğini kontrol eder. Ayarlanmamışsa platform varsayılan bekleme penceresi korunur (`macOS ve Android'de 700 ms, iOS'ta 900 ms`).

---

## Araçlar

### Araç profilleri

`tools.profile`, `tools.allow`/`tools.deny` öncesinde temel bir izin listesi ayarlar:

Yerel onboarding, ayarlanmamış yeni yerel yapılandırmalarda varsayılan olarak `tools.profile: "coding"` kullanır (mevcut açık profiller korunur).

| Profil      | İçerir                                                                                                                         |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | yalnızca `session_status`                                                                                                       |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate`, `video_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                                       |
| `full`      | Kısıtlama yok (ayarlanmamışla aynı)                                                                                             |

### Araç grupları

| Grup               | Araçlar                                                                                                                 |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash`, `exec` için takma ad olarak kabul edilir)                                |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                  |
| `group:sessions`   | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                           |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                   |
| `group:ui`         | `browser`, `canvas`                                                                                                     |
| `group:automation` | `cron`, `gateway`                                                                                                       |
| `group:messaging`  | `message`                                                                                                               |
| `group:nodes`      | `nodes`                                                                                                                 |
| `group:agents`     | `agents_list`                                                                                                           |
| `group:media`      | `image`, `image_generate`, `video_generate`, `tts`                                                                      |
| `group:openclaw`   | Tüm yerleşik araçlar (sağlayıcı eklentileri hariç)                                                                      |

### `tools.allow` / `tools.deny`

Genel araç izin/engelleme ilkesi (deny kazanır). Büyük/küçük harfe duyarsızdır, `*` joker karakterlerini destekler. Docker sandbox kapalı olsa bile uygulanır.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

Belirli sağlayıcılar veya modeller için araçları daha da kısıtlar. Sıra: temel profil → sağlayıcı profili → allow/deny.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.elevated`

Sandbox dışındaki elevated exec erişimini denetler:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- Ajan başına geçersiz kılma (`agents.list[].tools.elevated`) yalnızca daha fazla kısıtlama getirebilir.
- `/elevated on|off|ask|full`, durumu oturum başına depolar; satır içi yönergeler tek mesaja uygulanır.
- Elevated `exec`, sandboxing'i atlar ve yapılandırılmış kaçış yolunu kullanır (`gateway` varsayılan, exec hedefi `node` olduğunda `node`).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      applyPatch: {
        enabled: false,
        allowModels: ["gpt-5.4"],
      },
    },
  },
}
```

### `tools.loopDetection`

Araç döngüsü güvenlik kontrolleri varsayılan olarak **devre dışıdır**. Tespiti etkinleştirmek için `enabled: true` ayarlayın.
Ayarlar genel olarak `tools.loopDetection` altında tanımlanabilir ve ajan başına `agents.list[].tools.loopDetection` altında geçersiz kılınabilir.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },
  },
}
```

- `historySize`: döngü analizi için tutulan azami araç çağrısı geçmişi.
- `warningThreshold`: uyarılar için tekrarlayan ilerleme yok kalıp eşiği.
- `criticalThreshold`: kritik döngüleri engellemek için daha yüksek tekrar eşiği.
- `globalCircuitBreakerThreshold`: herhangi bir ilerleme yok çalıştırma için kesin durdurma eşiği.
- `detectors.genericRepeat`: aynı araç/aynı argüman çağrılarının tekrarında uyarı verir.
- `detectors.knownPollNoProgress`: bilinen poll araçlarında (`process.poll`, `command_status` vb.) uyarı/ver engelle.
- `detectors.pingPong`: dönüşümlü ilerleme yok çift kalıplarında uyarı/ver engelle.
- `warningThreshold >= criticalThreshold` veya `criticalThreshold >= globalCircuitBreakerThreshold` ise doğrulama başarısız olur.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // veya BRAVE_API_KEY env
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // isteğe bağlı; otomatik algılama için atlayın
        maxChars: 50000,
        maxCharsCap: 50000,
        maxResponseBytes: 2000000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

### `tools.media`

Gelen medya anlamlandırmasını yapılandırır (görsel/ses/video):

```json5
{
  tools: {
    media: {
      concurrency: 2,
      asyncCompletion: {
        directSend: false, // açık katılım: tamamlanan async müzik/videoyu doğrudan kanala gönder
      },
      audio: {
        enabled: true,
        maxBytes: 20971520,
        scope: {
          default: "deny",
          rules: [{ action: "allow", match: { chatType: "direct" } }],
        },
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          { type: "cli", command: "whisper", args: ["--model", "base", "{{MediaPath}}"] },
        ],
      },
      video: {
        enabled: true,
        maxBytes: 52428800,
        models: [{ provider: "google", model: "gemini-3-flash-preview" }],
      },
    },
  },
}
```

<Accordion title="Medya model girdisi alanları">

**Sağlayıcı girdisi** (`type: "provider"` veya atlanmış):

- `provider`: API sağlayıcı kimliği (`openai`, `anthropic`, `google`/`gemini`, `groq` vb.)
- `model`: model kimliği geçersiz kılması
- `profile` / `preferredProfile`: `auth-profiles.json` profil seçimi

**CLI girdisi** (`type: "cli"`):

- `command`: çalıştırılacak yürütülebilir dosya
- `args`: şablonlu argümanlar (`{{MediaPath}}`, `{{Prompt}}`, `{{MaxChars}}` vb. desteklenir)

**Ortak alanlar:**

- `capabilities`: isteğe bağlı liste (`image`, `audio`, `video`). Varsayılanlar: `openai`/`anthropic`/`minimax` → image, `google` → image+audio+video, `groq` → audio.
- `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: girdi başına geçersiz kılmalar.
- Hatalar sonraki girdiye geri döner.

Sağlayıcı auth, standart sırayı izler: `auth-profiles.json` → env vars → `models.providers.*.apiKey`.

**Async completion alanları:**

- `asyncCompletion.directSend`: `true` olduğunda, tamamlanan async `music_generate`
  ve `video_generate` görevleri önce doğrudan kanal teslimini dener. Varsayılan: `false`
  (eski istekçi-oturum uyandırma/model-teslim yolu).

</Accordion>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

Hangi oturumların oturum araçları tarafından hedeflenebileceğini denetler (`sessions_list`, `sessions_history`, `sessions_send`).

Varsayılan: `tree` (geçerli oturum + onun tarafından oluşturulmuş oturumlar, örneğin subagents).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

Notlar:

- `self`: yalnızca geçerli oturum anahtarı.
- `tree`: geçerli oturum + geçerli oturum tarafından oluşturulmuş oturumlar (subagents).
- `agent`: geçerli ajan kimliğine ait herhangi bir oturum (aynı ajan kimliği altında gönderici başına oturumlar çalıştırıyorsanız diğer kullanıcıları içerebilir).
- `all`: herhangi bir oturum. Ajanlar arası hedefleme yine de `tools.agentToAgent` gerektirir.
- Sandbox clamp: geçerli oturum sandbox içindeyse ve `agents.defaults.sandbox.sessionToolsVisibility="spawned"` ise `tools.sessions.visibility="all"` olsa bile görünürlük `tree` olarak zorlanır.

### `tools.sessions_spawn`

`sessions_spawn` için satır içi ek desteğini denetler.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // açık katılım: satır içi dosya eklerine izin vermek için true yapın
        maxTotalBytes: 5242880, // tüm dosyalar için toplam 5 MB
        maxFiles: 50,
        maxFileBytes: 1048576, // dosya başına 1 MB
        retainOnSessionKeep: false, // cleanup="keep" olduğunda ekleri tut
      },
    },
  },
}
```

Notlar:

- Ekler yalnızca `runtime: "subagent"` için desteklenir. ACP çalışma zamanı bunları reddeder.
- Dosyalar alt çalışma alanında `.openclaw/attachments/<uuid>/` içine `.manifest.json` ile somutlaştırılır.
- Ek içeriği döküm kalıcılığından otomatik olarak redakte edilir.
- Base64 girişleri katı alfabe/padding denetimleri ve decode öncesi boyut koruması ile doğrulanır.
- Dosya izinleri dizinler için `0700`, dosyalar için `0600` olur.
- Temizleme `cleanup` ilkesini izler: `delete` her zaman ekleri kaldırır; `keep`, bunları yalnızca `retainOnSessionKeep: true` olduğunda tutar.

### `tools.experimental`

Deneysel yerleşik araç bayrakları. Çalışma zamanına özgü otomatik etkinleştirme kuralı uygulanmadıkça varsayılan olarak kapalıdır.

```json5
{
  tools: {
    experimental: {
      planTool: true, // deneysel update_plan etkinleştir
    },
  },
}
```

Notlar:

- `planTool`: önemsiz olmayan çok adımlı iş takibi için yapılandırılmış `update_plan` aracını etkinleştirir.
- Varsayılan: OpenAI dışı sağlayıcılar için `false`. OpenAI ve OpenAI Codex çalışmaları, ayarlanmamışsa bunu otomatik etkinleştirir; otomatik etkinleştirmeyi kapatmak için `false` ayarlayın.
- Etkinleştirildiğinde sistem istemi ayrıca kullanım rehberi ekler; böylece model bunu yalnızca önemli işler için kullanır ve en fazla bir adımı `in_progress` tutar.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: oluşturulan alt ajanlar için varsayılan model. Belirtilmezse alt ajanlar çağıranın modelini devralır.
- `allowAgents`: istekçi ajan kendi `subagents.allowAgents` değerini ayarlamadığında `sessions_spawn` için varsayılan hedef ajan kimliği izin listesi (`["*"]` = herhangi biri; varsayılan: yalnızca aynı ajan).
- `runTimeoutSeconds`: araç çağrısı `runTimeoutSeconds` atladığında `sessions_spawn` için varsayılan zaman aşımı (saniye). `0`, zaman aşımı yok anlamına gelir.
- Alt ajan başına araç ilkesi: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Özel sağlayıcılar ve base URL'ler

OpenClaw yerleşik model kataloğunu kullanır. `models.providers` üzerinden yapılandırmada veya `~/.openclaw/agents/<agentId>/agent/models.json` içinde özel sağlayıcılar ekleyin.

```json5
{
  models: {
    mode: "merge", // merge (varsayılan) | replace
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

- Özel auth ihtiyaçları için `authHeader: true` + `headers` kullanın.
- Ajan yapılandırma kökünü `OPENCLAW_AGENT_DIR` ile geçersiz kılın (veya eski çevre değişkeni takma adı olan `PI_CODING_AGENT_DIR`).
- Eşleşen sağlayıcı kimlikleri için birleştirme önceliği:
  - Boş olmayan ajan `models.json` `baseUrl` değerleri kazanır.
  - Boş olmayan ajan `apiKey` değerleri, yalnızca o sağlayıcı mevcut yapılandırma/auth-profile bağlamında SecretRef ile yönetilmiyorsa kazanır.
  - SecretRef ile yönetilen sağlayıcı `apiKey` değerleri, çözümlenmiş gizli değerleri kalıcılaştırmak yerine kaynak işaretleyicilerden (`ENV_VAR_NAME` env başvuruları için, `secretref-managed` file/exec başvuruları için) yenilenir.
  - SecretRef ile yönetilen sağlayıcı header değerleri, kaynak işaretleyicilerden (`secretref-env:ENV_VAR_NAME` env başvuruları için, `secretref-managed` file/exec başvuruları için) yenilenir.
  - Boş veya eksik ajan `apiKey`/`baseUrl` değerleri, yapılandırmadaki `models.providers` değerlerine geri döner.
  - Eşleşen model `contextWindow`/`maxTokens`, açık yapılandırma ile örtük katalog değerleri arasındaki daha yüksek değeri kullanır.
  - Eşleşen model `contextTokens`, varsa açık çalışma zamanı üst sınırını korur; yerel model meta verisini değiştirmeden etkin bağlamı sınırlamak için bunu kullanın.
  - Yapılandırmanın `models.json` dosyasını tamamen yeniden yazmasını istiyorsanız `models.mode: "replace"` kullanın.
  - İşaretleyici kalıcılığı kaynak açısından yetkilidir: işaretleyiciler, çözümlenmiş çalışma zamanı gizli değerlerinden değil, etkin kaynak yapılandırma anlık görüntüsünden (çözümleme öncesi) yazılır.

### Sağlayıcı alanı ayrıntıları

- `models.mode`: sağlayıcı katalog davranışı (`merge` veya `replace`).
- `models.providers`: sağlayıcı kimliğine göre anahtarlanmış özel sağlayıcı eşlemesi.
- `models.providers.*.api`: istek bağdaştırıcısı (`openai-completions`, `openai-responses`, `anthropic-messages`, `google-generative-ai` vb.).
- `models.providers.*.apiKey`: sağlayıcı kimlik bilgisi (SecretRef/env substitution tercih edin).
- `models.providers.*.auth`: auth stratejisi (`api-key`, `token`, `oauth`, `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: Ollama + `openai-completions` için isteklere `options.num_ctx` enjekte et (varsayılan: `true`).
- `models.providers.*.authHeader`: gerektiğinde kimlik bilgisini `Authorization` başlığında taşımayı zorla.
- `models.providers.*.baseUrl`: üst akış API base URL'si.
- `models.providers.*.headers`: proxy/tenant yönlendirmesi için ek statik başlıklar.
- `models.providers.*.request`: model-sağlayıcı HTTP istekleri için taşıma geçersiz kılmaları.
  - `request.headers`: ek başlıklar (sağlayıcı varsayılanlarıyla birleştirilir). Değerler SecretRef kabul eder.
  - `request.auth`: auth stratejisi geçersiz kılması. Modlar: `"provider-default"` (sağlayıcının yerleşik auth'unu kullan), `"authorization-bearer"` (`token` ile), `"header"` (`headerName`, `value`, isteğe bağlı `prefix` ile).
  - `request.proxy`: HTTP proxy geçersiz kılması. Modlar: `"env-proxy"` (`HTTP_PROXY`/`HTTPS_PROXY` env vars kullan), `"explicit-proxy"` (`url` ile). Her iki mod da isteğe bağlı `tls` alt nesnesini kabul eder.
  - `request.tls`: doğrudan bağlantılar için TLS geçersiz kılması. Alanlar: `ca`, `cert`, `key`, `passphrase` (tamamı SecretRef kabul eder), `serverName`, `insecureSkipVerify`.
- `models.providers.*.models`: açık sağlayıcı model katalog girdileri.
- `models.providers.*.models.*.contextWindow`: yerel model bağlam penceresi meta verisi.
- `models.providers.*.models.*.contextTokens`: isteğe bağlı çalışma zamanı bağlam üst sınırı. Modelin yerel `contextWindow` değerinden daha küçük etkin bağlam bütçesi istiyorsanız bunu kullanın.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: isteğe bağlı uyumluluk ipucu. `api: "openai-completions"` ve boş olmayan yerel olmayan `baseUrl` (host `api.openai.com` değil) için OpenClaw bunu çalışma zamanında `false` zorlar. Boş/atlanmış `baseUrl`, varsayılan OpenAI davranışını korur.
- `models.providers.*.models.*.compat.requiresStringContent`: yalnızca dize kabul eden OpenAI uyumlu sohbet uç noktaları için isteğe bağlı uyumluluk ipucu. `true` olduğunda OpenClaw, saf metin `messages[].content` dizilerini göndermeden önce düz dizelere indirger.
- `plugins.entries.amazon-bedrock.config.discovery`: Bedrock otomatik keşif ayarları kökü.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: örtük keşfi aç/kapat.
- `plugins.entries.amazon-bedrock.config.discovery.region`: keşif için AWS bölgesi.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: hedefli keşif için isteğe bağlı provider-id filtresi.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: keşif yenileme için yoklama aralığı.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: keşfedilen modeller için geri dönüş bağlam penceresi.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: keşfedilen modeller için geri dönüş maksimum çıktı tokenı.

### Sağlayıcı örnekleri

<Accordion title="Cerebras (GLM 4.6 / 4.7)">

```json5
{
  env: { CEREBRAS_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: {
        primary: "cerebras/zai-glm-4.7",
        fallbacks: ["cerebras/zai-glm-4.6"],
      },
      models: {
        "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
        "cerebras/zai-glm-4.6": { alias: "GLM 4.6 (Cerebras)" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
          { id: "zai-glm-4.6", name: "GLM 4.6 (Cerebras)" },
        ],
      },
    },
  },
}
```

Cerebras için `cerebras/zai-glm-4.7`; Z.AI doğrudan için `zai/glm-4.7` kullanın.

</Accordion>

<Accordion title="OpenCode">

```json5
{
  agents: {
    defaults: {
      model: { primary: "opencode/claude-opus-4-6" },
      models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
    },
  },
}
```

`OPENCODE_API_KEY` (veya `OPENCODE_ZEN_API_KEY`) ayarlayın. Zen kataloğu için `opencode/...` referanslarını veya Go kataloğu için `opencode-go/...` referanslarını kullanın. Kısayol: `openclaw onboard --auth-choice opencode-zen` veya `openclaw onboard --auth-choice opencode-go`.

</Accordion>

<Accordion title="Z.AI (GLM-4.7)">

```json5
{
  agents: {
    defaults: {
      model: { primary: "zai/glm-4.7" },
      models: { "zai/glm-4.7": {} },
    },
  },
}
```

`ZAI_API_KEY` ayarlayın. `z.ai/*` ve `z-ai/*` kabul edilen takma adlardır. Kısayol: `openclaw onboard --auth-choice zai-api-key`.

- Genel uç nokta: `https://api.z.ai/api/paas/v4`
- Kodlama uç noktası (varsayılan): `https://api.z.ai/api/coding/paas/v4`
- Genel uç nokta için base URL geçersiz kılmalı özel sağlayıcı tanımlayın.

</Accordion>

<Accordion title="Moonshot AI (Kimi)">

```json5
{
  env: { MOONSHOT_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "moonshot/kimi-k2.5" },
      models: { "moonshot/kimi-k2.5": { alias: "Kimi K2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "kimi-k2.5",
            name: "Kimi K2.5",
            reasoning: false,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
        ],
      },
    },
  },
}
```

Çin uç noktası için: `baseUrl: "https://api.moonshot.cn/v1"` veya `openclaw onboard --auth-choice moonshot-api-key-cn`.

Yerel Moonshot uç noktaları, paylaşılan
`openai-completions` taşımasında akış kullanım uyumluluğu ilan eder ve OpenClaw,
bunu yalnızca yerleşik sağlayıcı kimliğine değil uç nokta yeteneklerine göre anahtarlayarak işler.

</Accordion>

<Accordion title="Kimi Coding">

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "kimi/kimi-code" },
      models: { "kimi/kimi-code": { alias: "Kimi Code" } },
    },
  },
}
```

Anthropic uyumlu, yerleşik sağlayıcı. Kısayol: `openclaw onboard --auth-choice kimi-code-api-key`.

</Accordion>

<Accordion title="Synthetic (Anthropic uyumlu)">

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M2.5": { alias: "MiniMax M2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M2.5",
            name: "MiniMax M2.5",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 192000,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

Base URL `/v1` içermemelidir (Anthropic istemcisi bunu ekler). Kısayol: `openclaw onboard --auth-choice synthetic-api-key`.

</Accordion>

<Accordion title="MiniMax M2.7 (doğrudan)">

```json5
{
  agents: {
    defaults: {
      model: { primary: "minimax/MiniMax-M2.7" },
      models: {
        "minimax/MiniMax-M2.7": { alias: "Minimax" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

`MINIMAX_API_KEY` ayarlayın. Kısayollar:
`openclaw onboard --auth-choice minimax-global-api` veya
`openclaw onboard --auth-choice minimax-cn-api`.
Model kataloğu varsayılan olarak yalnızca M2.7 kullanır.
Anthropic uyumlu akış yolunda OpenClaw, siz açıkça `thinking`
ayarlamadığınız sürece MiniMax thinking'i varsayılan olarak devre dışı bırakır. `/fast on` veya
`params.fastMode: true`, `MiniMax-M2.7` değerini
`MiniMax-M2.7-highspeed` olarak yeniden yazar.

</Accordion>

<Accordion title="Yerel modeller (LM Studio)">

Bkz. [Local Models](/tr/gateway/local-models). Kısacası: güçlü donanımda LM Studio Responses API üzerinden büyük bir yerel model çalıştırın; geri dönüş için barındırılan modelleri birleştirilmiş tutun.

</Accordion>

---

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // veya düz metin dize
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: yalnızca paketlenmiş skills için isteğe bağlı izin listesi (yönetilen/çalışma alanı skills etkilenmez).
- `load.extraDirs`: ek paylaşılan skill kökleri (en düşük öncelik).
- `install.preferBrew`: `brew` mevcut olduğunda diğer yükleyici türlerine geri dönmeden önce
  Homebrew yükleyicilerini tercih eder.
- `install.nodeManager`: `metadata.openclaw.install`
  tanımları için node yükleyici tercihi (`npm` | `pnpm` | `yarn` | `bun`).
- `entries.<skillKey>.enabled: false`, bir skill'i paketlenmiş/kurulu olsa bile devre dışı bırakır.
- `entries.<skillKey>.apiKey`: birincil env değişkeni bildiren skills için kolaylık alanı (düz metin dize veya SecretRef nesnesi).

---

## Eklentiler

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-extension"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- `~/.openclaw/extensions`, `<workspace>/.openclaw/extensions` ve `plugins.load.paths` içinden yüklenir.
- Keşif, yerel OpenClaw eklentilerini ve uyumlu Codex paketlerini ve Claude paketlerini, manifest'siz Claude varsayılan düzen paketleri dahil olacak şekilde kabul eder.
- **Yapılandırma değişiklikleri gateway yeniden başlatması gerektirir.**
- `allow`: isteğe bağlı izin listesi (yalnızca listelenen eklentiler yüklenir). `deny` kazanır.
- `plugins.entries.<id>.apiKey`: eklenti düzeyi API anahtarı kolaylık alanı (eklenti destekliyorsa).
- `plugins.entries.<id>.env`: eklenti kapsamlı env değişken eşlemesi.
- `plugins.entries.<id>.hooks.allowPromptInjection`: `false` olduğunda çekirdek `before_prompt_build` işlemini engeller ve eski `before_agent_start` içindeki istem değiştiren alanları yok sayar, ancak eski `modelOverride` ve `providerOverride` değerlerini korur. Yerel