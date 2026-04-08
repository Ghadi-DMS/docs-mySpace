---
read_when:
    - Discord kanal özellikleri üzerinde çalışırken
summary: Discord bot desteğinin durumu, yetenekleri ve yapılandırması
title: Discord
x-i18n:
    generated_at: "2026-04-08T08:45:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3cd2886fad941ae2129e681911309539e9a65a2352b777b538d7f4686a68f73f
    source_path: channels/discord.md
    workflow: 15
---

# Discord (Bot API)

Durum: resmi Discord ağ geçidi üzerinden DM'ler ve sunucu kanalları için hazır.

<CardGroup cols={3}>
  <Card title="Eşleştirme" icon="link" href="/tr/channels/pairing">
    Discord DM'leri varsayılan olarak eşleştirme modundadır.
  </Card>
  <Card title="Eğik çizgi komutları" icon="terminal" href="/tr/tools/slash-commands">
    Yerel komut davranışı ve komut kataloğu.
  </Card>
  <Card title="Kanal sorun giderme" icon="wrench" href="/tr/channels/troubleshooting">
    Kanallar arası tanılama ve onarım akışı.
  </Card>
</CardGroup>

## Hızlı kurulum

Bot içeren yeni bir uygulama oluşturmanız, botu sunucunuza eklemeniz ve OpenClaw ile eşleştirmeniz gerekir. Botunuzu kendi özel sunucunuza eklemenizi öneririz. Henüz bir sunucunuz yoksa, önce [bir tane oluşturun](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (**Create My Own > For me and my friends** seçin).

<Steps>
  <Step title="Bir Discord uygulaması ve bot oluşturun">
    [Discord Developer Portal](https://discord.com/developers/applications) sayfasına gidin ve **New Application** seçeneğine tıklayın. Adını "OpenClaw" gibi bir şey yapın.

    Kenar çubuğunda **Bot** seçeneğine tıklayın. **Username** alanını OpenClaw ajanınıza verdiğiniz adla ayarlayın.

  </Step>

  <Step title="Ayrıcalıklı intent'leri etkinleştirin">
    Hâlâ **Bot** sayfasındayken, aşağı kaydırarak **Privileged Gateway Intents** bölümüne gelin ve şunları etkinleştirin:

    - **Message Content Intent** (gerekli)
    - **Server Members Intent** (önerilir; rol izin listeleri ve addan kimliğe eşleştirme için gereklidir)
    - **Presence Intent** (isteğe bağlı; yalnızca varlık güncellemeleri için gereklidir)

  </Step>

  <Step title="Bot token'ınızı kopyalayın">
    **Bot** sayfasında tekrar yukarı kaydırın ve **Reset Token** seçeneğine tıklayın.

    <Note>
    İsmine rağmen bu işlem ilk token'ınızı oluşturur — hiçbir şey "sıfırlanmıyor".
    </Note>

    Token'ı kopyalayın ve bir yere kaydedin. Bu sizin **Bot Token** değerinizdir ve birazdan buna ihtiyacınız olacak.

  </Step>

  <Step title="Bir davet URL'si oluşturun ve botu sunucunuza ekleyin">
    Kenar çubuğunda **OAuth2** seçeneğine tıklayın. Botu sunucunuza eklemek için doğru izinlere sahip bir davet URL'si oluşturacaksınız.

    Aşağı kaydırarak **OAuth2 URL Generator** bölümüne gelin ve şunları etkinleştirin:

    - `bot`
    - `applications.commands`

    Aşağıda bir **Bot Permissions** bölümü görünecek. Şunları etkinleştirin:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (isteğe bağlı)

    Alttaki oluşturulan URL'yi kopyalayın, tarayıcınıza yapıştırın, sunucunuzu seçin ve bağlamak için **Continue** seçeneğine tıklayın. Artık botunuzu Discord sunucusunda görmelisiniz.

  </Step>

  <Step title="Developer Mode'u etkinleştirin ve kimliklerinizi toplayın">
    Tekrar Discord uygulamasına dönün; dahili kimlikleri kopyalayabilmek için Developer Mode'u etkinleştirmeniz gerekir.

    1. **User Settings** (avatarınızın yanındaki dişli simgesi) → **Advanced** → **Developer Mode** seçeneğini açın
    2. Kenar çubuğunda **server icon** öğenize sağ tıklayın → **Copy Server ID**
    3. Kendi **avatar** öğenize sağ tıklayın → **Copy User ID**

    **Server ID** ve **User ID** değerlerinizi Bot Token ile birlikte kaydedin — sonraki adımda üçünü de OpenClaw'a göndereceksiniz.

  </Step>

  <Step title="Sunucu üyelerinden gelen DM'lere izin verin">
    Eşleştirmenin çalışması için Discord'un botunuzun size DM göndermesine izin vermesi gerekir. **server icon** öğenize sağ tıklayın → **Privacy Settings** → **Direct Messages** seçeneğini açın.

    Bu, sunucu üyelerinin (botlar dahil) size DM göndermesine izin verir. OpenClaw ile Discord DM'lerini kullanmak istiyorsanız bunu etkin bırakın. Yalnızca sunucu kanallarını kullanmayı planlıyorsanız, eşleştirmeden sonra DM'leri kapatabilirsiniz.

  </Step>

  <Step title="Bot token'ınızı güvenli şekilde ayarlayın (sohbette göndermeyin)">
    Discord bot token'ınız bir gizlidir (parola gibi). Ajanınıza mesaj göndermeden önce bunu OpenClaw çalıştıran makinede ayarlayın.

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set channels.discord.enabled true --strict-json
openclaw gateway
```

    OpenClaw zaten arka plan hizmeti olarak çalışıyorsa, OpenClaw Mac uygulaması üzerinden veya `openclaw gateway run` sürecini durdurup yeniden başlatarak yeniden başlatın.

  </Step>

  <Step title="OpenClaw'ı yapılandırın ve eşleştirin">

    <Tabs>
      <Tab title="Ajanınıza sorun">
        OpenClaw ajanınızla mevcut herhangi bir kanalda (ör. Telegram) sohbet edin ve bunu söyleyin. Discord ilk kanalınızsa bunun yerine CLI / config sekmesini kullanın.

        > "Discord bot token'ımı zaten config içinde ayarladım. Lütfen User ID `<user_id>` ve Server ID `<server_id>` ile Discord kurulumunu tamamla."
      </Tab>
      <Tab title="CLI / config">
        Dosya tabanlı yapılandırmayı tercih ediyorsanız, şunu ayarlayın:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        Varsayılan hesap için env fallback:

```bash
DISCORD_BOT_TOKEN=...
```

        Düz metin `token` değerleri desteklenir. `channels.discord.token` için env/file/exec sağlayıcılarında SecretRef değerleri de desteklenir. Bkz. [Secrets Management](/tr/gateway/secrets).

      </Tab>
    </Tabs>

  </Step>

  <Step title="İlk DM eşleştirmesini onaylayın">
    Ağ geçidi çalışana kadar bekleyin, ardından Discord'da botunuza DM gönderin. Size bir eşleştirme koduyla yanıt verecektir.

    <Tabs>
      <Tab title="Ajanınıza sorun">
        Eşleştirme kodunu mevcut kanalınızdaki ajanınıza gönderin:

        > "Bu Discord eşleştirme kodunu onayla: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    Eşleştirme kodlarının süresi 1 saat sonra dolar.

    Artık Discord'da ajanınızla DM üzerinden sohbet edebilmelisiniz.

  </Step>
</Steps>

<Note>
Token çözümlemesi hesap farkındalığına sahiptir. Config içindeki token değerleri env fallback değerlerinin önüne geçer. `DISCORD_BOT_TOKEN` yalnızca varsayılan hesap için kullanılır.
Gelişmiş giden çağrılar için (message tool/channel actions), açık bir çağrı başına `token` o çağrı için kullanılır. Bu, gönderme ve okuma/probe tarzı eylemler için geçerlidir (örneğin read/search/fetch/thread/pins/permissions). Hesap politikası/yeniden deneme ayarları ise yine etkin çalışma zamanı anlık görüntüsünde seçilen hesaptan gelir.
</Note>

## Önerilen: Bir sunucu çalışma alanı kurun

DM'ler çalıştıktan sonra, Discord sunucunuzu her kanalın kendi bağlamına sahip kendi ajan oturumunu aldığı tam bir çalışma alanı olarak ayarlayabilirsiniz. Bu, yalnızca siz ve botunuzun bulunduğu özel sunucular için önerilir.

<Steps>
  <Step title="Sunucunuzu sunucu izin listesine ekleyin">
    Bu, ajanınızın yalnızca DM'lerde değil, sunucunuzdaki herhangi bir kanalda yanıt vermesini sağlar.

    <Tabs>
      <Tab title="Ajanınıza sorun">
        > "Discord Server ID `<server_id>` değerimi sunucu izin listesine ekle"
      </Tab>
      <Tab title="Config">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="@mention olmadan yanıtlara izin verin">
    Varsayılan olarak ajanınız sunucu kanallarında yalnızca @mention yapıldığında yanıt verir. Özel bir sunucu için muhtemelen her mesaja yanıt vermesini istersiniz.

    <Tabs>
      <Tab title="Ajanınıza sorun">
        > "Ajanımın bu sunucuda @mention yapılmasına gerek kalmadan yanıt vermesine izin ver"
      </Tab>
      <Tab title="Config">
        Sunucu yapılandırmanızda `requireMention: false` ayarlayın:

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Sunucu kanallarında bellek kullanımını planlayın">
    Varsayılan olarak uzun vadeli bellek (MEMORY.md) yalnızca DM oturumlarında yüklenir. Sunucu kanalları MEMORY.md'yi otomatik olarak yüklemez.

    <Tabs>
      <Tab title="Ajanınıza sorun">
        > "Discord kanallarında soru sorduğumda, MEMORY.md içindeki uzun vadeli bağlama ihtiyaç duyarsan memory_search veya memory_get kullan."
      </Tab>
      <Tab title="Elle">
        Her kanalda paylaşılan bağlama ihtiyacınız varsa, kalıcı yönergeleri `AGENTS.md` veya `USER.md` içine koyun (bunlar her oturuma enjekte edilir). Uzun vadeli notları `MEMORY.md` içinde tutun ve gerektiğinde memory araçlarıyla erişin.
      </Tab>
    </Tabs>

  </Step>
</Steps>

Şimdi Discord sunucunuzda birkaç kanal oluşturun ve sohbete başlayın. Ajanınız kanal adını görebilir ve her kanal kendi yalıtılmış oturumunu alır — böylece `#coding`, `#home`, `#research` veya iş akışınıza uyan her neyse onları kurabilirsiniz.

## Çalışma zamanı modeli

- Discord bağlantısının sahibi gateway'dir.
- Yanıt yönlendirmesi deterministiktir: Discord'dan gelen yanıtlar tekrar Discord'a gider.
- Varsayılan olarak (`session.dmScope=main`), doğrudan sohbetler ajanın ana oturumunu paylaşır (`agent:main:main`).
- Sunucu kanalları yalıtılmış oturum anahtarlarıdır (`agent:<agentId>:discord:channel:<channelId>`).
- Grup DM'leri varsayılan olarak yok sayılır (`channels.discord.dm.groupEnabled=false`).
- Yerel slash komutları yalıtılmış komut oturumlarında çalışır (`agent:<agentId>:discord:slash:<userId>`), ancak yine de `CommandTargetSessionKey` değerini yönlendirilen konuşma oturumuna taşır.

## Forum kanalları

Discord forum ve medya kanalları yalnızca ileti dizisi gönderilerini kabul eder. OpenClaw bunları oluşturmak için iki yolu destekler:

- Bir ileti dizisini otomatik oluşturmak için forum üst öğesine (`channel:<forumId>`) mesaj gönderin. İleti dizisi başlığı, mesajınızın ilk boş olmayan satırını kullanır.
- Doğrudan bir ileti dizisi oluşturmak için `openclaw message thread create` kullanın. Forum kanalları için `--message-id` geçmeyin.

Örnek: ileti dizisi oluşturmak için forum üst öğesine gönderme

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

Örnek: açıkça bir forum ileti dizisi oluşturma

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

Forum üst öğeleri Discord bileşenlerini kabul etmez. Bileşenlere ihtiyacınız varsa, ileti dizisinin kendisine (`channel:<threadId>`) gönderin.

## Etkileşimli bileşenler

OpenClaw, ajan mesajları için Discord components v2 kapsayıcılarını destekler. `components` yüküyle message tool kullanın. Etkileşim sonuçları normal gelen mesajlar olarak ajana geri yönlendirilir ve mevcut Discord `replyToMode` ayarlarını izler.

Desteklenen bloklar:

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- Eylem satırları en fazla 5 düğmeye veya tek bir seçim menüsüne izin verir
- Seçim türleri: `string`, `user`, `role`, `mentionable`, `channel`

Varsayılan olarak bileşenler tek kullanımlıktır. Düğmelerin, seçimlerin ve formların süreleri dolana kadar birden fazla kez kullanılmasına izin vermek için `components.reusable=true` ayarlayın.

Bir düğmeye kimlerin tıklayabileceğini kısıtlamak için o düğmede `allowedUsers` ayarlayın (Discord kullanıcı kimlikleri, etiketler veya `*`). Yapılandırıldığında, eşleşmeyen kullanıcılar geçici bir ret alır.

`/model` ve `/models` slash komutları, sağlayıcı ve model açılır menülerinin yanı sıra bir Submit adımı içeren etkileşimli bir model seçiciyi açar. Seçici yanıtı ephemeral olur ve yalnızca komutu çağıran kullanıcı bunu kullanabilir.

Dosya ekleri:

- `file` blokları bir ek başvurusuna işaret etmelidir (`attachment://<filename>`)
- Eki `media`/`path`/`filePath` ile sağlayın (tek dosya); birden çok dosya için `media-gallery` kullanın
- Yükleme adının ek başvurusu ile eşleşmesi gerektiğinde bunu geçersiz kılmak için `filename` kullanın

Modal formlar:

- En fazla 5 alanla `components.modal` ekleyin
- Alan türleri: `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- OpenClaw otomatik olarak bir tetikleyici düğme ekler

Örnek:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "İsteğe bağlı fallback metni",
  components: {
    reusable: true,
    text: "Bir yol seçin",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Onayla",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Reddet", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Bir seçenek seçin",
          options: [
            { label: "Seçenek A", value: "a" },
            { label: "Seçenek B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Ayrıntılar",
      triggerLabel: "Formu aç",
      fields: [
        { type: "text", label: "Talep eden" },
        {
          type: "select",
          label: "Öncelik",
          options: [
            { label: "Düşük", value: "low" },
            { label: "Yüksek", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## Erişim denetimi ve yönlendirme

<Tabs>
  <Tab title="DM ilkesi">
    `channels.discord.dmPolicy` DM erişimini denetler (eski: `channels.discord.dm.policy`):

    - `pairing` (varsayılan)
    - `allowlist`
    - `open` (`channels.discord.allowFrom` içinde `"*"` bulunmasını gerektirir; eski: `channels.discord.dm.allowFrom`)
    - `disabled`

    DM ilkesi açık değilse, bilinmeyen kullanıcılar engellenir (veya `pairing` modunda eşleştirme istenir).

    Çoklu hesap önceliği:

    - `channels.discord.accounts.default.allowFrom` yalnızca `default` hesabına uygulanır.
    - Adlandırılmış hesaplar, kendi `allowFrom` değerleri ayarlanmamışsa `channels.discord.allowFrom` değerini devralır.
    - Adlandırılmış hesaplar `channels.discord.accounts.default.allowFrom` değerini devralmaz.

    Teslimat için DM hedef biçimi:

    - `user:<id>`
    - `<@id>` mention

    Düz sayısal kimlikler belirsizdir ve açık bir user/channel hedef türü sağlanmadıkça reddedilir.

  </Tab>

  <Tab title="Sunucu ilkesi">
    Sunucu işleme `channels.discord.groupPolicy` ile denetlenir:

    - `open`
    - `allowlist`
    - `disabled`

    `channels.discord` mevcut olduğunda güvenli temel çizgi `allowlist` olur.

    `allowlist` davranışı:

    - sunucu `channels.discord.guilds` ile eşleşmelidir (`id` tercih edilir, slug kabul edilir)
    - isteğe bağlı gönderici izin listeleri: `users` (kalıcı kimlikler önerilir) ve `roles` (yalnızca rol kimlikleri); bunlardan biri yapılandırılmışsa, göndericilere `users` VEYA `roles` ile eşleşiyorlarsa izin verilir
    - doğrudan ad/etiket eşleştirmesi varsayılan olarak devre dışıdır; yalnızca acil durum uyumluluk modu olarak `channels.discord.dangerouslyAllowNameMatching: true` etkinleştirin
    - `users` için adlar/etiketler desteklenir, ancak kimlikler daha güvenlidir; `openclaw security audit`, ad/etiket girdileri kullanıldığında uyarır
    - bir sunucuda `channels` yapılandırılmışsa, listede olmayan kanallar reddedilir
    - bir sunucuda `channels` bloğu yoksa, izin listesindeki o sunucudaki tüm kanallara izin verilir

    Örnek:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    Yalnızca `DISCORD_BOT_TOKEN` ayarlayıp `channels.discord` bloğu oluşturmazsanız, çalışma zamanı fallback değeri `groupPolicy="allowlist"` olur (günlüklerde bir uyarıyla), `channels.defaults.groupPolicy` değeri `open` olsa bile.

  </Tab>

  <Tab title="Mention'lar ve grup DM'leri">
    Sunucu mesajları varsayılan olarak mention kapılıdır.

    Mention algılaması şunları içerir:

    - açık bot mention'ı
    - yapılandırılmış mention desenleri (`agents.list[].groupChat.mentionPatterns`, fallback `messages.groupChat.mentionPatterns`)
    - desteklenen durumlarda örtük bota-yanıt davranışı

    `requireMention` sunucu/kanal başına yapılandırılır (`channels.discord.guilds...`).
    `ignoreOtherMentions`, başka bir kullanıcıya/role mention yapan ancak bottan bahsetmeyen mesajları isteğe bağlı olarak düşürür (@everyone/@here hariç).

    Grup DM'leri:

    - varsayılan: yok sayılır (`dm.groupEnabled=false`)
    - isteğe bağlı izin listesi: `dm.groupChannels` (kanal kimlikleri veya slug'lar)

  </Tab>
</Tabs>

### Role dayalı ajan yönlendirme

Discord sunucu üyelerini rol kimliğine göre farklı ajanlara yönlendirmek için `bindings[].match.roles` kullanın. Role dayalı bağlamalar yalnızca rol kimliklerini kabul eder ve peer veya parent-peer bağlamalarından sonra, yalnızca sunucu bağlamalarından önce değerlendirilir. Bir bağlama başka eşleşme alanları da ayarlarsa (örneğin `peer` + `guildId` + `roles`), yapılandırılmış tüm alanlar eşleşmelidir.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Developer Portal kurulumu

<AccordionGroup>
  <Accordion title="Uygulama ve bot oluşturun">

    1. Discord Developer Portal -> **Applications** -> **New Application**
    2. **Bot** -> **Add Bot**
    3. Bot token'ını kopyalayın

  </Accordion>

  <Accordion title="Ayrıcalıklı intent'ler">
    **Bot -> Privileged Gateway Intents** bölümünde şunları etkinleştirin:

    - Message Content Intent
    - Server Members Intent (önerilir)

    Presence intent isteğe bağlıdır ve yalnızca varlık güncellemelerini almak istiyorsanız gereklidir. Bot varlığını ayarlamak (`setPresence`) üyeler için varlık güncellemelerini etkinleştirmeyi gerektirmez.

  </Accordion>

  <Accordion title="OAuth kapsamları ve temel izinler">
    OAuth URL oluşturucu:

    - kapsamlar: `bot`, `applications.commands`

    Tipik temel izinler:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (isteğe bağlı)

    Açıkça gerekmedikçe `Administrator` kullanmaktan kaçının.

  </Accordion>

  <Accordion title="Kimlikleri kopyalayın">
    Discord Developer Mode'u etkinleştirin, ardından şunları kopyalayın:

    - sunucu kimliği
    - kanal kimliği
    - kullanıcı kimliği

    Güvenilir denetimler ve probe'lar için OpenClaw config içinde sayısal kimlikleri tercih edin.

  </Accordion>
</AccordionGroup>

## Yerel komutlar ve komut kimlik doğrulaması

- `commands.native` varsayılan olarak `"auto"` değerindedir ve Discord için etkindir.
- Kanal başına geçersiz kılma: `channels.discord.commands.native`.
- `commands.native=false`, önceden kaydedilmiş Discord yerel komutlarını açıkça temizler.
- Yerel komut kimlik doğrulaması, normal mesaj işlemeyle aynı Discord izin listelerini/ilkelerini kullanır.
- Komutlar, yetkili olmayan kullanıcılar için Discord arayüzünde yine de görünür olabilir; yürütme yine de OpenClaw kimlik doğrulamasını uygular ve "not authorized" döndürür.

Komut kataloğu ve davranış için bkz. [Slash commands](/tr/tools/slash-commands).

Varsayılan slash komut ayarları:

- `ephemeral: true`

## Özellik ayrıntıları

<AccordionGroup>
  <Accordion title="Yanıt etiketleri ve yerel yanıtlar">
    Discord, ajan çıktısında yanıt etiketlerini destekler:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    `channels.discord.replyToMode` ile denetlenir:

    - `off` (varsayılan)
    - `first`
    - `all`
    - `batched`

    Not: `off`, örtük yanıt ileti dizilemesini devre dışı bırakır. Açık `[[reply_to_*]]` etiketlerine yine de uyulur.
    `first`, tur için ilk giden Discord mesajına örtük yerel yanıt başvurusunu her zaman ekler.
    `batched`, Discord'un örtük yerel yanıt başvurusunu yalnızca
    gelen turun birden çok mesajdan oluşan gecikmeli bir toplu işlem olduğu durumda ekler. Bu,
    yerel yanıtları çoğunlukla belirsiz ve patlamalı sohbetlerde istemeniz, her
    tek mesajlık turda istememeniz durumunda faydalıdır.

    Mesaj kimlikleri bağlam/geçmiş içinde görünür, böylece ajanlar belirli mesajları hedefleyebilir.

  </Accordion>

  <Accordion title="Canlı akış önizlemesi">
    OpenClaw, geçici bir mesaj gönderip metin geldikçe düzenleyerek taslak yanıtları akış halinde verebilir.

    - `channels.discord.streaming` önizleme akışını denetler (`off` | `partial` | `block` | `progress`, varsayılan: `off`).
    - Varsayılan `off` olarak kalır çünkü Discord önizleme düzenlemeleri özellikle aynı hesap veya sunucu trafiğini birden çok bot ya da gateway paylaştığında hızla rate limit'e takılabilir.
    - `progress`, kanallar arası tutarlılık için kabul edilir ve Discord'da `partial` değerine eşlenir.
    - `channels.discord.streamMode` eski bir takma addır ve otomatik olarak taşınır.
    - `partial`, token'lar geldikçe tek bir önizleme mesajını düzenler.
    - `block`, taslak boyutunda parçalar üretir (boyutu ve bölünme noktalarını ayarlamak için `draftChunk` kullanın).

    Örnek:

```json5
{
  channels: {
    discord: {
      streaming: "partial",
    },
  },
}
```

    `block` modu parça varsayılanları (`channels.discord.textChunkLimit` ile sınırlandırılır):

```json5
{
  channels: {
    discord: {
      streaming: "block",
      draftChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph",
      },
    },
  },
}
```

    Önizleme akışı yalnızca metin içindir; medya yanıtları normal teslimata fallback yapar.

    Not: önizleme akışı, blok akışından ayrıdır. Discord için blok akışı açıkça
    etkinleştirildiğinde, OpenClaw çift akıştan kaçınmak için önizleme akışını atlar.

  </Accordion>

  <Accordion title="Geçmiş, bağlam ve ileti dizisi davranışı">
    Sunucu geçmiş bağlamı:

    - `channels.discord.historyLimit` varsayılan `20`
    - fallback: `messages.groupChat.historyLimit`
    - `0` devre dışı bırakır

    DM geçmiş denetimleri:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    İleti dizisi davranışı:

    - Discord ileti dizileri kanal oturumları olarak yönlendirilir
    - üst ileti dizisi meta verileri, üst-oturum bağlantısı için kullanılabilir
    - ileti dizisine özgü bir girdi yoksa ileti dizisi config'i üst kanal config'ini devralır

    Kanal konuları **güvenilmeyen** bağlam olarak enjekte edilir (system prompt olarak değil).
    Yanıt ve alıntılanan mesaj bağlamı şu anda alındığı gibi kalır.
    Discord izin listeleri öncelikle ajanın kim tarafından tetiklenebileceğini denetler, tam bir ek-bağlam redaksiyon sınırı değildir.

  </Accordion>

  <Accordion title="Alt ajanlar için ileti dizisine bağlı oturumlar">
    Discord, bir ileti dizisini oturum hedefine bağlayabilir; böylece o ileti dizisindeki takip mesajları aynı oturuma yönlenmeye devam eder (alt ajan oturumları dahil).

    Komutlar:

    - `/focus <target>` geçerli/yeni ileti dizisini bir alt ajan/oturum hedefine bağla
    - `/unfocus` geçerli ileti dizisi bağını kaldır
    - `/agents` etkin çalıştırmaları ve bağlanma durumunu göster
    - `/session idle <duration|off>` odaklı bağlamalar için hareketsizlikte otomatik odak kaldırmayı incele/güncelle
    - `/session max-age <duration|off>` odaklı bağlamalar için katı azami yaşı incele/güncelle

    Config:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // isteğe bağlı olarak açılır
      },
    },
  },
}
```

    Notlar:

    - `session.threadBindings.*` genel varsayılanları ayarlar.
    - `channels.discord.threadBindings.*` Discord davranışını geçersiz kılar.
    - `sessions_spawn({ thread: true })` için ileti dizilerini otomatik oluşturmak/bağlamak üzere `spawnSubagentSessions` true olmalıdır.
    - ACP için ileti dizilerini otomatik oluşturmak/bağlamak üzere `spawnAcpSessions` true olmalıdır (`/acp spawn ... --thread ...` veya `sessions_spawn({ runtime: "acp", thread: true })`).
    - Bir hesap için ileti dizisi bağlamaları devre dışıysa, `/focus` ve ilgili ileti dizisi bağlama işlemleri kullanılamaz.

    Bkz. [Sub-agents](/tr/tools/subagents), [ACP Agents](/tr/tools/acp-agents) ve [Configuration Reference](/tr/gateway/configuration-reference).

  </Accordion>

  <Accordion title="Kalıcı ACP kanal bağlamaları">
    Kararlı, "her zaman açık" ACP çalışma alanları için Discord konuşmalarını hedefleyen üst düzey türlenmiş ACP bağlamaları yapılandırın.

    Config yolu:

    - `type: "acp"` ve `match.channel: "discord"` ile `bindings[]`

    Örnek:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    Notlar:

    - `/acp spawn codex --bind here`, mevcut Discord kanalını veya ileti dizisini yerinde bağlar ve gelecekteki mesajların aynı ACP oturumuna yönlenmesini sağlar.
    - Bu yine de "yeni bir Codex ACP oturumu başlat" anlamına gelebilir, ancak kendi başına yeni bir Discord ileti dizisi oluşturmaz. Mevcut kanal sohbet yüzeyi olarak kalır.
    - Codex yine de diskte kendi `cwd` veya backend çalışma alanında çalışabilir. Bu çalışma alanı Discord ileti dizisi değil, çalışma zamanı durumudur.
    - İleti dizisi mesajları üst kanal ACP bağını devralabilir.
    - Bağlı bir kanal veya ileti dizisinde `/new` ve `/reset`, aynı ACP oturumunu yerinde sıfırlar.
    - Geçici ileti dizisi bağlamaları yine çalışır ve etkinken hedef çözümlemesini geçersiz kılabilir.
    - `spawnAcpSessions`, yalnızca OpenClaw'ın `--thread auto|here` üzerinden bir alt ileti dizisini oluşturması/bağlaması gerektiğinde gerekir. Mevcut kanalda `/acp spawn ... --bind here` için gerekli değildir.

    Bağlama davranışı ayrıntıları için bkz. [ACP Agents](/tr/tools/acp-agents).

  </Accordion>

  <Accordion title="Tepki bildirimleri">
    Sunucu başına tepki bildirim modu:

    - `off`
    - `own` (varsayılan)
    - `all`
    - `allowlist` (`guilds.<id>.users` kullanır)

    Tepki olayları system event'lere dönüştürülür ve yönlendirilen Discord oturumuna eklenir.

  </Accordion>

  <Accordion title="Onay tepkileri">
    `ackReaction`, OpenClaw gelen bir mesajı işlerken bir onay emojisi gönderir.

    Çözümleme sırası:

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - ajan kimlik emojisi fallback'i (`agents.list[].identity.emoji`, yoksa "👀")

    Notlar:

    - Discord unicode emoji veya özel emoji adlarını kabul eder.
    - Bir kanal veya hesap için tepkiyi devre dışı bırakmak üzere `""` kullanın.

  </Accordion>

  <Accordion title="Config yazımları">
    Kanal tarafından başlatılan config yazımları varsayılan olarak etkindir.

    Bu, `/config set|unset` akışlarını etkiler (komut özellikleri etkin olduğunda).

    Devre dışı bırakma:

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Gateway proxy">
    Discord gateway WebSocket trafiğini ve başlangıç REST aramalarını (uygulama kimliği + izin listesi çözümlemesi) `channels.discord.proxy` ile bir HTTP(S) proxy üzerinden yönlendirin.

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    Hesap başına geçersiz kılma:

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="PluralKit desteği">
    Proxy'lenen mesajları sistem üyesi kimliğine eşlemek için PluralKit çözümlemesini etkinleştirin:

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // isteğe bağlı; özel sistemler için gereklidir
      },
    },
  },
}
```

    Notlar:

    - izin listeleri `pk:<memberId>` kullanabilir
    - üye görünen adları yalnızca `channels.discord.dangerouslyAllowNameMatching: true` olduğunda ad/slug ile eşleştirilir
    - aramalar özgün mesaj kimliğini kullanır ve zaman penceresiyle sınırlıdır
    - arama başarısız olursa, proxy'lenen mesajlar bot mesajı olarak kabul edilir ve `allowBots=true` olmadığı sürece düşürülür

  </Accordion>

  <Accordion title="Presence yapılandırması">
    Presence güncellemeleri, bir durum veya etkinlik alanı ayarladığınızda ya da otomatik presence'i etkinleştirdiğinizde uygulanır.

    Yalnızca durum örneği:

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    Etkinlik örneği (özel durum varsayılan etkinlik türüdür):

```json5
{
  channels: {
    discord: {
      activity: "Odak zamanı",
      activityType: 4,
    },
  },
}
```

    Akış örneği:

```json5
{
  channels: {
    discord: {
      activity: "Canlı kodlama",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    Etkinlik türü eşlemesi:

    - 0: Playing
    - 1: Streaming (`activityUrl` gerektirir)
    - 2: Listening
    - 3: Watching
    - 4: Custom (etkinlik metnini durum durumu olarak kullanır; emoji isteğe bağlıdır)
    - 5: Competing

    Otomatik presence örneği (çalışma zamanı sağlık sinyali):

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token tükendi",
      },
    },
  },
}
```

    Otomatik presence, çalışma zamanı kullanılabilirliğini Discord durumuna eşler: sağlıklı => online, bozulmuş veya bilinmiyor => idle, tükenmiş veya kullanılamıyor => dnd. İsteğe bağlı metin geçersiz kılmaları:

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (`{reason}` placeholder'ını destekler)

  </Accordion>

  <Accordion title="Discord içinde onaylar">
    Discord, DM'lerde düğme tabanlı onay işlemeyi destekler ve isteğe bağlı olarak onay istemlerini kaynak kanalda yayımlayabilir.

    Config yolu:

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (isteğe bağlı; mümkün olduğunda `commands.ownerAllowFrom` değerine fallback yapar)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`, varsayılan: `dm`)
    - `agentFilter`, `sessionFilter`, `cleanupAfterResolve`

    Discord, `enabled` ayarlanmamış veya `"auto"` olduğunda ve en az bir onaylayıcı çözümlenebildiğinde yerel exec onaylarını otomatik etkinleştirir; bu ya `execApprovals.approvers` içinden ya da `commands.ownerAllowFrom` içinden olabilir. Discord, exec onaylayıcılarını kanal `allowFrom`, eski `dm.allowFrom` veya doğrudan mesaj `defaultTo` değerinden çıkarmaz. Discord'u yerel onay istemcisi olarak açıkça devre dışı bırakmak için `enabled: false` ayarlayın.

    `target` değeri `channel` veya `both` olduğunda onay istemi kanalda görünür olur. Düğmeleri yalnızca çözümlenmiş onaylayıcılar kullanabilir; diğer kullanıcılar ephemeral bir ret alır. Onay istemleri komut metnini içerir, bu nedenle kanal teslimatını yalnızca güvenilen kanallarda etkinleştirin. Kanal kimliği oturum anahtarından türetilemezse OpenClaw DM teslimatına fallback yapar.

    Discord ayrıca diğer sohbet kanallarının kullandığı ortak onay düğmelerini de işler. Yerel Discord bağdaştırıcısı esas olarak onaylayıcı DM yönlendirmesi ve kanal yayılımı ekler.
    Bu düğmeler mevcut olduğunda birincil onay UX'i onlardır; OpenClaw
    yalnızca araç sonucu sohbet onaylarının kullanılamadığını söylediğinde veya
    tek yol manuel onaysa elle `/approve` komutu eklemelidir.

    Bu işleyici için gateway kimlik doğrulaması, diğer Gateway istemcileriyle aynı paylaşılan kimlik bilgisi çözümleme sözleşmesini kullanır:

    - env-öncelikli yerel kimlik doğrulama (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`, ardından `gateway.auth.*`)
    - yerel modda, `gateway.auth.*` ayarlanmamışsa `gateway.remote.*` yalnızca fallback olarak kullanılabilir; yapılandırılmış ancak çözümlenmemiş yerel SecretRef'ler fail closed olur
    - uygulanabildiğinde `gateway.remote.*` ile uzaktan mod desteği
    - URL geçersiz kılmaları override-safe'dir: CLI geçersiz kılmaları örtük kimlik bilgilerini yeniden kullanmaz ve env geçersiz kılmaları yalnızca env kimlik bilgilerini kullanır

    Onay çözümleme davranışı:

    - `plugin:` öneki taşıyan kimlikler `plugin.approval.resolve` üzerinden çözülür.
    - Diğer kimlikler `exec.approval.resolve` üzerinden çözülür.
    - Discord burada ekstra bir exec-to-plugin fallback sıçraması yapmaz; hangi gateway yöntemini çağıracağını kimlik öneki belirler.

    Exec onaylarının varsayılan süresi 30 dakika sonra dolar. Onaylar
    bilinmeyen onay kimlikleriyle başarısız olursa, onaylayıcı çözümlemesini, özellik etkinliğini ve
    teslim edilen onay kimliği türünün bekleyen istekle eşleştiğini doğrulayın.

    İlgili belgeler: [Exec approvals](/tr/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## Araçlar ve eylem kapıları

Discord mesaj eylemleri; mesajlaşma, kanal yönetimi, moderasyon, presence ve meta veri eylemlerini içerir.

Temel örnekler:

- mesajlaşma: `sendMessage`, `readMessages`, `editMessage`, `deleteMessage`, `threadReply`
- tepkiler: `react`, `reactions`, `emojiList`
- moderasyon: `timeout`, `kick`, `ban`
- presence: `setPresence`

`event-create` eylemi, planlanan etkinlik kapak görselini ayarlamak için isteğe bağlı bir `image` parametresini (URL veya yerel dosya yolu) kabul eder.

Eylem kapıları `channels.discord.actions.*` altında bulunur.

Varsayılan kapı davranışı:

| Eylem grubu                                                                                                                                                              | Varsayılan |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | etkin      |
| roles                                                                                                                                                                    | devre dışı |
| moderation                                                                                                                                                               | devre dışı |
| presence                                                                                                                                                                 | devre dışı |

## Components v2 UI

OpenClaw, exec onayları ve bağlamlar arası işaretçiler için Discord components v2 kullanır. Discord mesaj eylemleri özel UI için `components` de kabul edebilir (ileri düzey; discord tool üzerinden bir bileşen yükü oluşturmayı gerektirir), ancak eski `embeds` hâlâ kullanılabilir ve önerilmez.

- `channels.discord.ui.components.accentColor`, Discord bileşen kapsayıcılarında kullanılan vurgu rengini ayarlar (hex).
- Hesap başına `channels.discord.accounts.<id>.ui.components.accentColor` ile ayarlayın.
- components v2 mevcut olduğunda `embeds` yok sayılır.

Örnek:

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## Ses kanalları

OpenClaw, gerçek zamanlı ve sürekli konuşmalar için Discord ses kanallarına katılabilir. Bu, sesli mesaj eklerinden ayrıdır.

Gereksinimler:

- Yerel komutları etkinleştirin (`commands.native` veya `channels.discord.commands.native`).
- `channels.discord.voice` yapılandırın.
- Botun hedef ses kanalında Connect + Speak izinlerine ihtiyacı vardır.

Oturumları denetlemek için yalnızca Discord'a özel yerel `/vc join|leave|status` komutunu kullanın. Komut, hesabın varsayılan ajanını kullanır ve diğer Discord komutlarıyla aynı izin listesi ve sunucu ilkesi kurallarını izler.

Otomatik katılma örneği:

```json5
{
  channels: {
    discord: {
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
    },
  },
}
```

Notlar:

- `voice.tts`, yalnızca ses oynatımı için `messages.tts` değerini geçersiz kılar.
- Ses transkript turları, sahip durumunu Discord `allowFrom` (veya `dm.allowFrom`) üzerinden türetir; sahip olmayan konuşmacılar yalnızca sahibin kullanabildiği araçlara erişemez (örneğin `gateway` ve `cron`).
- Ses varsayılan olarak etkindir; devre dışı bırakmak için `channels.discord.voice.enabled=false` ayarlayın.
- `voice.daveEncryption` ve `voice.decryptionFailureTolerance`, `@discordjs/voice` katılma seçeneklerine olduğu gibi aktarılır.
- `@discordjs/voice` varsayılanları, ayarlanmamışsa `daveEncryption=true` ve `decryptionFailureTolerance=24` değerleridir.
- OpenClaw ayrıca alma tarafındaki şifre çözme hatalarını izler ve kısa bir zaman penceresinde yinelenen hatalardan sonra ses kanalından ayrılıp yeniden katılarak otomatik kurtarma yapar.
- Alma günlükleri tekrar tekrar `DecryptionFailed(UnencryptedWhenPassthroughDisabled)` gösteriyorsa, bu [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) içinde izlenen yukarı akış `@discordjs/voice` alma hatası olabilir.

## Sesli mesajlar

Discord sesli mesajları bir dalga biçimi önizlemesi gösterir ve OGG/Opus ses ile meta veri gerektirir. OpenClaw dalga biçimini otomatik olarak üretir, ancak ses dosyalarını incelemek ve dönüştürmek için gateway ana makinesinde `ffmpeg` ve `ffprobe` erişilebilir olmalıdır.

Gereksinimler ve kısıtlar:

- Bir **yerel dosya yolu** sağlayın (URL'ler reddedilir).
- Metin içeriğini atlayın (Discord aynı yükte metin + sesli mesaja izin vermez).
- Her türlü ses biçimi kabul edilir; gerektiğinde OpenClaw bunu OGG/Opus'a dönüştürür.

Örnek:

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## Sorun giderme

<AccordionGroup>
  <Accordion title="İzin verilmeyen intent'ler kullanıldı veya bot sunucu mesajlarını görmüyor">

    - Message Content Intent'i etkinleştirin
    - kullanıcı/üye çözümlemesine dayanıyorsanız Server Members Intent'i etkinleştirin
    - intent'leri değiştirdikten sonra gateway'i yeniden başlatın

  </Accordion>

  <Accordion title="Sunucu mesajları beklenmedik şekilde engelleniyor">

    - `groupPolicy` değerini doğrulayın
    - `channels.discord.guilds` altındaki sunucu izin listesini doğrulayın
    - sunucu `channels` eşlemesi varsa, yalnızca listelenen kanallara izin verilir
    - `requireMention` davranışını ve mention desenlerini doğrulayın

    Yararlı kontroller:

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="Require mention false ama yine de engelleniyor">
    Yaygın nedenler:

    - eşleşen sunucu/kanal izin listesi olmadan `groupPolicy="allowlist"`
    - `requireMention` yanlış yerde yapılandırılmış (mutlaka `channels.discord.guilds` veya kanal girdisi altında olmalıdır)
    - gönderici sunucu/kanal `users` izin listesi tarafından engelleniyor

  </Accordion>

  <Accordion title="Uzun süren işleyiciler zaman aşımına uğruyor veya yinelenen yanıtlar oluşuyor">

    Tipik günlükler:

    - `Listener DiscordMessageListener timed out after 30000ms for event MESSAGE_CREATE`
    - `Slow listener detected ...`
    - `discord inbound worker timed out after ...`

    Listener bütçe ayarı:

    - tek hesap: `channels.discord.eventQueue.listenerTimeout`
    - çoklu hesap: `channels.discord.accounts.<accountId>.eventQueue.listenerTimeout`

    Worker çalışma zaman aşımı ayarı:

    - tek hesap: `channels.discord.inboundWorker.runTimeoutMs`
    - çoklu hesap: `channels.discord.accounts.<accountId>.inboundWorker.runTimeoutMs`
    - varsayılan: `1800000` (30 dakika); devre dışı bırakmak için `0` ayarlayın

    Önerilen temel yapılandırma:

```json5
{
  channels: {
    discord: {
      accounts: {
        default: {
          eventQueue: {
            listenerTimeout: 120000,
          },
          inboundWorker: {
            runTimeoutMs: 1800000,
          },
        },
      },
    },
  },
}
```

    Yavaş listener kurulumu için `eventQueue.listenerTimeout`, kuyruklanan ajan turları için ayrı bir güvenlik supabı istiyorsanız ise yalnızca `inboundWorker.runTimeoutMs` kullanın.

  </Accordion>

  <Accordion title="İzin denetimi uyumsuzlukları">
    `channels status --probe` izin kontrolleri yalnızca sayısal kanal kimlikleri için çalışır.

    Slug anahtarları kullanıyorsanız çalışma zamanı eşleştirmesi yine de çalışabilir, ancak probe izinleri tam olarak doğrulayamaz.

  </Accordion>

  <Accordion title="DM ve eşleştirme sorunları">

    - DM devre dışı: `channels.discord.dm.enabled=false`
    - DM ilkesi devre dışı: `channels.discord.dmPolicy="disabled"` (eski: `channels.discord.dm.policy`)
    - `pairing` modunda eşleştirme onayı bekleniyor

  </Accordion>

  <Accordion title="Bottan bota döngüler">
    Varsayılan olarak bot tarafından yazılmış mesajlar yok sayılır.

    `channels.discord.allowBots=true` ayarlarsanız, döngü davranışını önlemek için katı mention ve izin listesi kuralları kullanın.
    Yalnızca bottan mention alan bot mesajlarını kabul etmek için `channels.discord.allowBots="mentions"` tercih edin.

  </Accordion>

  <Accordion title="Voice STT, DecryptionFailed(...) ile düşüyor">

    - Discord ses alma kurtarma mantığının mevcut olduğundan emin olmak için OpenClaw'ı güncel tutun (`openclaw update`)
    - `channels.discord.voice.daveEncryption=true` değerini doğrulayın (varsayılan)
    - `channels.discord.voice.decryptionFailureTolerance=24` (yukarı akış varsayılanı) ile başlayın ve yalnızca gerekirse ayarlayın
    - günlüklerde şunları izleyin:
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - otomatik yeniden katılmadan sonra da hatalar sürerse günlükleri toplayın ve [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) ile karşılaştırın

  </Accordion>
</AccordionGroup>

## Yapılandırma başvuru işaretçileri

Birincil başvuru:

- [Configuration reference - Discord](/tr/gateway/configuration-reference#discord)

Yüksek sinyalli Discord alanları:

- başlangıç/kimlik doğrulama: `enabled`, `token`, `accounts.*`, `allowBots`
- ilke: `groupPolicy`, `dm.*`, `guilds.*`, `guilds.*.channels.*`
- komut: `commands.native`, `commands.useAccessGroups`, `configWrites`, `slashCommand.*`
- olay kuyruğu: `eventQueue.listenerTimeout` (listener bütçesi), `eventQueue.maxQueueSize`, `eventQueue.maxConcurrency`
- inbound worker: `inboundWorker.runTimeoutMs`
- yanıt/geçmiş: `replyToMode`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- teslimat: `textChunkLimit`, `chunkMode`, `maxLinesPerMessage`
- akış: `streaming` (eski takma ad: `streamMode`), `draftChunk`, `blockStreaming`, `blockStreamingCoalesce`
- medya/yeniden deneme: `mediaMaxMb`, `retry`
  - `mediaMaxMb` giden Discord yüklemelerini sınırlar (varsayılan: `100MB`)
- eylemler: `actions.*`
- presence: `activity`, `status`, `activityType`, `activityUrl`
- UI: `ui.components.accentColor`
- özellikler: `threadBindings`, üst düzey `bindings[]` (`type: "acp"`), `pluralkit`, `execApprovals`, `intents`, `agentComponents`, `heartbeat`, `responsePrefix`

## Güvenlik ve işlemler

- Bot token'larını gizli bilgiler olarak ele alın (denetimli ortamlarda `DISCORD_BOT_TOKEN` tercih edilir).
- En az ayrıcalık ilkesine göre Discord izinleri verin.
- Komut dağıtımı/durumu bayatsa, gateway'i yeniden başlatın ve `openclaw channels status --probe` ile tekrar kontrol edin.

## İlgili

- [Eşleştirme](/tr/channels/pairing)
- [Gruplar](/tr/channels/groups)
- [Kanal yönlendirme](/tr/channels/channel-routing)
- [Güvenlik](/tr/gateway/security)
- [Çoklu ajan yönlendirme](/tr/concepts/multi-agent)
- [Sorun giderme](/tr/channels/troubleshooting)
- [Slash commands](/tr/tools/slash-commands)
