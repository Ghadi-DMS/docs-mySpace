---
read_when:
    - OpenClaw içinde Matrix kurulumu
    - Matrix E2EE ve doğrulamayı yapılandırma
summary: Matrix destek durumu, kurulum ve yapılandırma örnekleri
title: Matrix
x-i18n:
    generated_at: "2026-04-09T01:29:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28fc13c7620c1152200315ae69c94205da6de3180c53c814dd8ce03b5cb1758f
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix, OpenClaw için paketlenmiş bir kanal eklentisidir.
Resmi `matrix-js-sdk` kullanır ve DM'leri, odaları, thread'leri, medyayı, tepkileri, anketleri, konumu ve E2EE'yi destekler.

## Paketlenmiş eklenti

Matrix, güncel OpenClaw sürümlerinde paketlenmiş bir eklenti olarak gelir, bu yüzden normal
paketlenmiş derlemelerde ayrı bir kurulum gerekmez.

Eski bir derlemeyi veya Matrix'i dışlayan özel bir kurulumu kullanıyorsanız, onu
elle yükleyin:

npm'den yükleme:

```bash
openclaw plugins install @openclaw/matrix
```

Yerel bir checkout'tan yükleme:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Eklenti davranışı ve kurulum kuralları için [Plugins](/tr/tools/plugin) bölümüne bakın.

## Kurulum

1. Matrix eklentisinin kullanılabilir olduğundan emin olun.
   - Güncel paketlenmiş OpenClaw sürümleri bunu zaten içerir.
   - Eski/özel kurulumlar bunu yukarıdaki komutlarla elle ekleyebilir.
2. Homeserver'ınızda bir Matrix hesabı oluşturun.
3. `channels.matrix` yapılandırmasını şu seçeneklerden biriyle ayarlayın:
   - `homeserver` + `accessToken`, veya
   - `homeserver` + `userId` + `password`.
4. Gateway'i yeniden başlatın.
5. Bot ile bir DM başlatın veya onu bir odaya davet edin.
   - Yeni Matrix davetleri yalnızca `channels.matrix.autoJoin` buna izin verdiğinde çalışır.

Etkileşimli kurulum yolları:

```bash
openclaw channels add
openclaw configure --section channels
```

Matrix sihirbazı şunları sorar:

- homeserver URL'si
- kimlik doğrulama yöntemi: erişim belirteci veya parola
- kullanıcı kimliği (yalnızca parola kimlik doğrulaması)
- isteğe bağlı cihaz adı
- E2EE'nin etkinleştirilip etkinleştirilmeyeceği
- oda erişimi ve davet otomatik katılımının yapılandırılıp yapılandırılmayacağı

Sihirbazın temel davranışları:

- Matrix kimlik doğrulama ortam değişkenleri zaten varsa ve bu hesap için yapılandırmada henüz kimlik doğrulama kaydedilmemişse, sihirbaz kimlik doğrulamayı ortam değişkenlerinde tutmak için bir ortam değişkeni kısayolu sunar.
- Hesap adları hesap kimliğine normalize edilir. Örneğin, `Ops Bot`, `ops-bot` olur.
- DM izin listesi girdileri doğrudan `@user:server` kabul eder; görünen adlar yalnızca canlı dizin araması tam bir eşleşme bulduğunda çalışır.
- Oda izin listesi girdileri doğrudan oda kimliklerini ve takma adları kabul eder. `!room:server` veya `#alias:server` tercih edin; çözümlenmemiş adlar çalışma zamanında izin listesi çözümlemesi tarafından yok sayılır.
- Davet otomatik katılımı izin listesi modunda yalnızca kararlı davet hedeflerini kullanın: `!roomId:server`, `#alias:server` veya `*`. Düz oda adları reddedilir.
- Kaydetmeden önce oda adlarını çözümlemek için `openclaw channels resolve --channel matrix "Project Room"` komutunu kullanın.

<Warning>
`channels.matrix.autoJoin` varsayılan olarak `off` değerindedir.

Bunu ayarlamazsanız, bot davet edilen odalara veya yeni DM tarzı davetlere katılmaz; bu nedenle önce elle katılmazsanız yeni gruplarda veya davet edilen DM'lerde görünmez.

Hangi davetleri kabul edeceğini kısıtlamak için `autoJoinAllowlist` ile birlikte `autoJoin: "allowlist"` ayarlayın ya da her davete katılmasını istiyorsanız `autoJoin: "always"` ayarlayın.

`allowlist` modunda `autoJoinAllowlist` yalnızca `!roomId:server`, `#alias:server` veya `*` kabul eder.
</Warning>

İzin listesi örneği:

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Her davete katıl:

```json5
{
  channels: {
    matrix: {
      autoJoin: "always",
    },
  },
}
```

En az belirteç tabanlı kurulum:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Parola tabanlı kurulum (girişten sonra belirteç önbelleğe alınır):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

Matrix önbelleğe alınmış kimlik bilgilerini `~/.openclaw/credentials/matrix/` içinde saklar.
Varsayılan hesap `credentials.json` kullanır; adlandırılmış hesaplar `credentials-<account>.json` kullanır.
Orada önbelleğe alınmış kimlik bilgileri varsa, mevcut kimlik doğrulama doğrudan yapılandırmada ayarlı olmasa bile OpenClaw, kurulum, doctor ve kanal durumu keşfi için Matrix'i yapılandırılmış kabul eder.

Ortam değişkeni karşılıkları (yapılandırma anahtarı ayarlı olmadığında kullanılır):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Varsayılan olmayan hesaplar için hesap kapsamlı ortam değişkenlerini kullanın:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

`ops` hesabı için örnek:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

Normalize edilmiş `ops-bot` hesap kimliği için şunu kullanın:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix, hesap kimliklerindeki noktalamayı, hesap kapsamlı ortam değişkenlerinin çakışmasız kalması için kaçışlar.
Örneğin, `-`, `_X2D_` olur; dolayısıyla `ops-prod`, `MATRIX_OPS_X2D_PROD_*` ile eşlenir.

Etkileşimli sihirbaz, bu kimlik doğrulama ortam değişkenleri zaten mevcutsa ve seçilen hesap için yapılandırmada Matrix kimlik doğrulaması henüz kaydedilmemişse ortam değişkeni kısayolunu sunar.

## Yapılandırma örneği

Bu, DM eşleme, oda izin listesi ve etkin E2EE içeren pratik bir temel yapılandırmadır:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

`autoJoin`, DM tarzı davetler dahil tüm Matrix davetleri için geçerlidir. OpenClaw,
davet anında davet edilen bir odayı güvenilir biçimde DM mi yoksa grup mu diye
sınıflandıramaz; bu yüzden tüm davetler önce `autoJoin` üzerinden geçer. `dm.policy`,
bot katıldıktan ve oda DM olarak sınıflandırıldıktan sonra uygulanır.

## Akış önizlemeleri

Matrix yanıt akışı isteğe bağlıdır.

OpenClaw'un tek bir canlı önizleme yanıtı göndermesini, model metin üretirken bu önizlemeyi yerinde düzenlemesini ve yanıt tamamlandığında sonlandırmasını istiyorsanız `channels.matrix.streaming` değerini `"partial"` olarak ayarlayın:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` varsayılandır. OpenClaw son yanıtı bekler ve bir kez gönderir.
- `streaming: "partial"`, mevcut asistan bloğu için normal Matrix metin mesajları kullanarak düzenlenebilir bir önizleme mesajı oluşturur. Bu, Matrix'in eski önizleme-önce bildirim davranışını korur; bu nedenle standart istemciler bitmiş blok yerine ilk akış önizleme metni için bildirim gönderebilir.
- `streaming: "quiet"`, mevcut asistan bloğu için düzenlenebilir tek bir sessiz önizleme bildirimi oluşturur. Bunu yalnızca alıcı push kurallarını sonlandırılmış önizleme düzenlemeleri için de yapılandırdığınızda kullanın.
- `blockStreaming: true` ayrı Matrix ilerleme mesajlarını etkinleştirir. Önizleme akışı etkinleştirildiğinde Matrix, geçerli blok için canlı taslağı tutar ve tamamlanmış blokları ayrı mesajlar olarak korur.
- Önizleme akışı açıkken `blockStreaming` kapalıysa, Matrix canlı taslağı yerinde düzenler ve blok veya tur bittiğinde aynı olayı sonlandırır.
- Önizleme artık tek bir Matrix olayına sığmazsa, OpenClaw önizleme akışını durdurur ve normal son teslimata geri döner.
- Medya yanıtları yine ekleri normal şekilde gönderir. Eski bir önizleme artık güvenle yeniden kullanılamıyorsa, OpenClaw son medya yanıtını göndermeden önce onu redact eder.
- Önizleme düzenlemeleri ek Matrix API çağrıları gerektirir. En muhafazakar rate-limit davranışını istiyorsanız akışı kapalı bırakın.

`blockStreaming` tek başına taslak önizlemelerini etkinleştirmez.
Önizleme düzenlemeleri için `streaming: "partial"` veya `streaming: "quiet"` kullanın; ardından yalnızca tamamlanmış asistan bloklarının ayrı ilerleme mesajları olarak görünür kalmasını da istiyorsanız `blockStreaming: true` ekleyin.

Özel push kuralları olmadan standart Matrix bildirimlerine ihtiyacınız varsa, önizleme-önce davranışı için `streaming: "partial"` kullanın veya yalnızca son teslimat için `streaming` kapalı bırakın. `streaming: "off"` ile:

- `blockStreaming: true`, biten her bloğu normal bildirim veren bir Matrix mesajı olarak gönderir.
- `blockStreaming: false`, yalnızca son tamamlanmış yanıtı normal bildirim veren bir Matrix mesajı olarak gönderir.

### Sessiz sonlandırılmış önizlemeler için self-hosted push kuralları

Kendi Matrix altyapınızı çalıştırıyorsanız ve sessiz önizlemelerin yalnızca bir blok veya
son yanıt tamamlandığında bildirim göndermesini istiyorsanız, `streaming: "quiet"` ayarlayın ve sonlandırılmış önizleme düzenlemeleri için kullanıcı başına bir push kuralı ekleyin.

Bu genellikle homeserver genelinde bir yapılandırma değişikliği değil, alıcı kullanıcı tarafında yapılan bir kurulumdur:

Başlamadan önce hızlı eşleme:

- alıcı kullanıcı = bildirimi alması gereken kişi
- bot kullanıcısı = yanıtı gönderen OpenClaw Matrix hesabı
- aşağıdaki API çağrıları için alıcı kullanıcının erişim belirtecini kullanın
- push kuralındaki `sender` için bot kullanıcısının tam MXID'siyle eşleştirin

1. Sessiz önizlemeleri kullanacak şekilde OpenClaw'u yapılandırın:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. Alıcı hesabın zaten normal Matrix push bildirimleri aldığından emin olun. Sessiz önizleme
   kuralları yalnızca bu kullanıcının zaten çalışan pusher'ları/cihazları varsa çalışır.

3. Alıcı kullanıcının erişim belirtecini alın.
   - Botun belirtecini değil, alıcı kullanıcının belirtecini kullanın.
   - Mevcut bir istemci oturum belirtecini yeniden kullanmak genellikle en kolay yoldur.
   - Yeni bir belirteç üretmeniz gerekiyorsa, standart Matrix Client-Server API üzerinden giriş yapabilirsiniz:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. Alıcı hesabın zaten pusher'lara sahip olduğunu doğrulayın:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Bu, etkin pusher/cihaz döndürmüyorsa aşağıdaki OpenClaw
kuralını eklemeden önce önce normal Matrix bildirimlerini düzeltin.

OpenClaw, sonlandırılmış yalnızca metin içeren önizleme düzenlemelerini şu şekilde işaretler:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Bu bildirimleri alması gereken her alıcı hesap için bir override push kuralı oluşturun:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

Komutu çalıştırmadan önce şu değerleri değiştirin:

- `https://matrix.example.org`: homeserver temel URL'niz
- `$USER_ACCESS_TOKEN`: alıcı kullanıcının erişim belirteci
- `openclaw-finalized-preview-botname`: bu alıcı kullanıcı için bu bota özgü benzersiz bir kural kimliği
- `@bot:example.org`: alıcı kullanıcının MXID'si değil, OpenClaw Matrix bot MXID'niz

Çok botlu kurulumlar için önemli:

- Push kuralları `ruleId` ile anahtarlanır. Aynı kural kimliğine karşı `PUT` komutunu yeniden çalıştırmak, o tek kuralı günceller.
- Bir alıcı kullanıcı birden fazla OpenClaw Matrix bot hesabı için bildirim alacaksa, her `sender` eşleşmesi için benzersiz bir kural kimliğiyle bot başına bir kural oluşturun.
- Basit bir desen `openclaw-finalized-preview-<botname>` biçimidir; örneğin `openclaw-finalized-preview-ops` veya `openclaw-finalized-preview-support`.

Kural olay gönderenine göre değerlendirilir:

- alıcı kullanıcının belirteciyle kimlik doğrulayın
- `sender` değerini OpenClaw bot MXID'siyle eşleştirin

6. Kuralın var olduğunu doğrulayın:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Akışlı bir yanıtı test edin. Sessiz modda oda sessiz bir taslak önizleme göstermeli ve son
   yerinde düzenleme blok veya tur tamamlandığında tek bir kez bildirim göndermelidir.

Daha sonra kuralı kaldırmanız gerekirse, aynı kural kimliğini alıcı kullanıcının belirteciyle silin:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Notlar:

- Kuralı botun değil, alıcı kullanıcının erişim belirteciyle oluşturun.
- Yeni kullanıcı tanımlı `override` kuralları varsayılan bastırma kurallarının önüne eklenir; bu nedenle ek bir sıralama parametresi gerekmez.
- Bu yalnızca OpenClaw'un güvenle yerinde sonlandırabildiği yalnızca metin içeren önizleme düzenlemelerini etkiler. Medya geri dönüşleri ve eski önizleme geri dönüşleri hâlâ normal Matrix teslimatını kullanır.
- `GET /_matrix/client/v3/pushers` hiçbir pusher göstermiyorsa, kullanıcının bu hesap/cihaz için henüz çalışan Matrix push teslimatı yoktur.

#### Synapse

Synapse için yukarıdaki kurulum genellikle tek başına yeterlidir:

- Sonlandırılmış OpenClaw önizleme bildirimleri için özel bir `homeserver.yaml` değişikliği gerekmez.
- Synapse dağıtımınız zaten normal Matrix push bildirimleri gönderiyorsa, kullanıcı belirteci + yukarıdaki `pushrules` çağrısı temel kurulum adımıdır.
- Synapse'i bir reverse proxy veya workers arkasında çalıştırıyorsanız, `/_matrix/client/.../pushrules/` yolunun Synapse'e doğru ulaştığından emin olun.
- Synapse workers kullanıyorsanız, pusher'ların sağlıklı olduğundan emin olun. Push teslimatı ana süreç veya `synapse.app.pusher` / yapılandırılmış pusher worker'ları tarafından yönetilir.

#### Tuwunel

Tuwunel için yukarıda gösterilen aynı kurulum akışını ve push-rule API çağrısını kullanın:

- Sonlandırılmış önizleme işaretleyicisinin kendisi için Tuwunel'e özgü bir yapılandırma gerekmez.
- Normal Matrix bildirimleri bu kullanıcı için zaten çalışıyorsa, kullanıcı belirteci + yukarıdaki `pushrules` çağrısı temel kurulum adımıdır.
- Kullanıcı başka bir cihazda etkinken bildirimler kayboluyor gibi görünüyorsa, `suppress_push_when_active` etkin mi kontrol edin. Tuwunel bu seçeneği 12 Eylül 2025 tarihinde Tuwunel 1.4.2'de ekledi ve bir cihaz etkinken diğer cihazlara yapılan push'ları bilinçli olarak bastırabilir.

## Bottan bota odalar

Varsayılan olarak, yapılandırılmış diğer OpenClaw Matrix hesaplarından gelen Matrix mesajları yok sayılır.

Ajanlar arası Matrix trafiğini bilinçli olarak istiyorsanız `allowBots` kullanın:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true`, izin verilen odalar ve DM'lerde yapılandırılmış diğer Matrix bot hesaplarından gelen mesajları kabul eder.
- `allowBots: "mentions"`, bu mesajları odalarda yalnızca bu bottan görünür biçimde bahsettiklerinde kabul eder. DM'lere yine izin verilir.
- `groups.<room>.allowBots`, hesap düzeyindeki ayarı tek bir oda için geçersiz kılar.
- OpenClaw, kendine yanıt döngülerini önlemek için aynı Matrix kullanıcı kimliğinden gelen mesajları yine yok sayar.
- Matrix burada yerel bir bot bayrağı sunmaz; OpenClaw, "bot tarafından yazılmış" ifadesini "bu OpenClaw gateway'inde yapılandırılmış başka bir Matrix hesabı tarafından gönderilmiş" olarak değerlendirir.

Paylaşılan odalarda bottan bota trafiği etkinleştirirken katı oda izin listeleri ve mention gereksinimleri kullanın.

## Şifreleme ve doğrulama

Şifreli (E2EE) odalarda, giden görsel olayları `thumbnail_file` kullanır; böylece görsel önizlemeleri tam ek ile birlikte şifrelenir. Şifrelenmemiş odalarda hâlâ düz `thumbnail_url` kullanılır. Yapılandırma gerekmez — eklenti E2EE durumunu otomatik olarak algılar.

Şifrelemeyi etkinleştirin:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

Doğrulama durumunu kontrol edin:

```bash
openclaw matrix verify status
```

Ayrıntılı durum (tam tanılama):

```bash
openclaw matrix verify status --verbose
```

Makine tarafından okunabilir çıktıda saklanan kurtarma anahtarını da ekleyin:

```bash
openclaw matrix verify status --include-recovery-key --json
```

Cross-signing ve doğrulama durumunu bootstrap edin:

```bash
openclaw matrix verify bootstrap
```

Ayrıntılı bootstrap tanılaması:

```bash
openclaw matrix verify bootstrap --verbose
```

Bootstrap işleminden önce yeni bir cross-signing kimliği sıfırlamasını zorlayın:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

Bu cihazı bir kurtarma anahtarıyla doğrulayın:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

Ayrıntılı cihaz doğrulama ayrıntıları:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

Oda anahtarı yedeği sağlığını kontrol edin:

```bash
openclaw matrix verify backup status
```

Ayrıntılı yedek sağlığı tanılaması:

```bash
openclaw matrix verify backup status --verbose
```

Oda anahtarlarını sunucu yedeğinden geri yükleyin:

```bash
openclaw matrix verify backup restore
```

Ayrıntılı geri yükleme tanılaması:

```bash
openclaw matrix verify backup restore --verbose
```

Mevcut sunucu yedeğini silin ve yeni bir yedek temel çizgisi oluşturun. Saklanan
yedek anahtarı temiz biçimde yüklenemiyorsa, bu sıfırlama secret storage'ı da yeniden oluşturabilir; böylece
gelecekteki cold start'lar yeni yedek anahtarını yükleyebilir:

```bash
openclaw matrix verify backup reset --yes
```

Tüm `verify` komutları varsayılan olarak kısadır (sessiz dahili SDK günlükleme dahil) ve ayrıntılı tanılamayı yalnızca `--verbose` ile gösterir.
Betik yazarken tam makine tarafından okunabilir çıktı için `--json` kullanın.

Çok hesaplı kurulumlarda Matrix CLI komutları, siz `--account <id>` vermediğiniz sürece örtük Matrix varsayılan hesabını kullanır.
Birden çok adlandırılmış hesap yapılandırırsanız, önce `channels.matrix.defaultAccount` ayarlayın; aksi halde bu örtük CLI işlemleri durur ve sizden açıkça bir hesap seçmenizi ister.
Doğrulama veya cihaz işlemlerinin açıkça adlandırılmış bir hesabı hedeflemesini istediğiniz her durumda `--account` kullanın:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Adlandırılmış bir hesap için şifreleme devre dışıysa veya kullanılamıyorsa, Matrix uyarıları ve doğrulama hataları örneğin `channels.matrix.accounts.assistant.encryption` gibi bu hesabın yapılandırma anahtarını işaret eder.

### "Doğrulanmış" ne anlama gelir

OpenClaw bu Matrix cihazını yalnızca kendi cross-signing kimliğiniz tarafından doğrulandığında doğrulanmış kabul eder.
Pratikte `openclaw matrix verify status --verbose`, üç güven sinyali sunar:

- `Locally trusted`: bu cihaz yalnızca geçerli istemci tarafından güvenilir kabul edilir
- `Cross-signing verified`: SDK, cihazı cross-signing üzerinden doğrulanmış olarak raporlar
- `Signed by owner`: cihaz, kendi self-signing anahtarınız tarafından imzalanmıştır

`Verified by owner`, yalnızca cross-signing doğrulaması veya sahip imzası mevcut olduğunda `yes` olur.
Yalnızca yerel güven, OpenClaw'un cihazı tamamen doğrulanmış kabul etmesi için yeterli değildir.

### Bootstrap ne yapar

`openclaw matrix verify bootstrap`, şifreli Matrix hesapları için onarım ve kurulum komutudur.
Sırayla şunların tümünü yapar:

- mümkün olduğunda mevcut kurtarma anahtarını yeniden kullanarak secret storage'ı bootstrap eder
- cross-signing'i bootstrap eder ve eksik herkese açık cross-signing anahtarlarını yükler
- mevcut cihazı işaretlemeyi ve cross-signing ile imzalamayı dener
- zaten mevcut değilse yeni bir sunucu tarafı oda anahtarı yedeği oluşturur

Homeserver, cross-signing anahtarlarını yüklemek için etkileşimli kimlik doğrulama gerektiriyorsa OpenClaw önce kimlik doğrulama olmadan, sonra `m.login.dummy` ile, `channels.matrix.password` yapılandırılmışsa da `m.login.password` ile yüklemeyi dener.

`--force-reset-cross-signing` seçeneğini yalnızca mevcut cross-signing kimliğini bilinçli olarak silmek ve yenisini oluşturmak istediğinizde kullanın.

Mevcut oda anahtarı yedeğini bilinçli olarak silmek ve gelecekteki mesajlar için yeni
bir yedek temel çizgisi başlatmak istiyorsanız `openclaw matrix verify backup reset --yes` kullanın.
Bunu yalnızca kurtarılamayan eski şifreli geçmişin erişilemez kalacağını
ve OpenClaw'un mevcut yedek sırrı güvenle yüklenemiyorsa secret storage'ı yeniden oluşturabileceğini kabul ediyorsanız yapın.

### Yeni yedek temel çizgisi

Gelecekteki şifreli mesajların çalışmaya devam etmesini istiyor ve kurtarılamayan eski geçmişi kaybetmeyi kabul ediyorsanız, şu komutları sırayla çalıştırın:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Açıkça adlandırılmış bir Matrix hesabını hedeflemek istediğinizde her komuta `--account <id>` ekleyin.

### Başlangıç davranışı

`encryption: true` olduğunda Matrix, `startupVerification` değerini varsayılan olarak `"if-unverified"` yapar.
Başlangıçta bu cihaz hâlâ doğrulanmamışsa Matrix, başka bir Matrix istemcisinde self-verification isteyecek,
zaten beklemede olan bir istek varken yinelenen istekleri atlayacak ve yeniden başlatmalardan sonra tekrar denemeden önce yerel bir cooldown uygulayacaktır.
Başarısız istek denemeleri varsayılan olarak başarılı istek oluşturmaya göre daha erken yeniden denenir.
Otomatik başlangıç isteklerini devre dışı bırakmak için `startupVerification: "off"` ayarlayın veya daha kısa ya da daha uzun bir yeniden deneme penceresi istiyorsanız `startupVerificationCooldownHours` değerini ayarlayın.

Başlangıç ayrıca otomatik olarak muhafazakar bir crypto bootstrap geçişi gerçekleştirir.
Bu geçiş önce mevcut secret storage ve cross-signing kimliğini yeniden kullanmayı dener ve siz açık bir bootstrap onarım akışı çalıştırmadığınız sürece cross-signing'i sıfırlamaktan kaçınır.

Başlangıç bozuk bootstrap durumu bulursa ve `channels.matrix.password` yapılandırılmışsa, OpenClaw daha katı bir onarım yolu deneyebilir.
Geçerli cihaz zaten owner-signed ise OpenClaw bu kimliği otomatik olarak sıfırlamak yerine korur.

Tam yükseltme akışı, sınırlar, kurtarma komutları ve yaygın geçiş mesajları için [Matrix migration](/tr/install/migrating-matrix) bölümüne bakın.

### Doğrulama bildirimleri

Matrix, doğrulama yaşam döngüsü bildirimlerini doğrudan katı DM doğrulama odasına `m.notice` mesajları olarak gönderir.
Buna şunlar dahildir:

- doğrulama isteği bildirimleri
- doğrulama hazır bildirimleri ("Emoji ile doğrula" yönlendirmesi açıkça belirtilir)
- doğrulama başlatma ve tamamlama bildirimleri
- mevcut olduğunda SAS ayrıntıları (emoji ve ondalık)

Başka bir Matrix istemcisinden gelen doğrulama istekleri OpenClaw tarafından izlenir ve otomatik kabul edilir.
Self-verification akışlarında OpenClaw, emoji doğrulaması kullanılabilir olduğunda SAS akışını da otomatik başlatır ve kendi tarafını onaylar.
Başka bir Matrix kullanıcısı/cihazından gelen doğrulama isteklerinde OpenClaw isteği otomatik kabul eder ve ardından SAS akışının normal şekilde ilerlemesini bekler.
Doğrulamayı tamamlamak için yine de Matrix istemcinizde emoji veya ondalık SAS değerini karşılaştırmanız ve orada "Eşleşiyorlar" seçeneğini onaylamanız gerekir.

OpenClaw kendi başlattığı yinelenen akışları körü körüne otomatik kabul etmez. Başlangıç, bir self-verification isteği zaten beklemedeyse yeni bir istek oluşturmayı atlar.

Doğrulama protokolü/sistem bildirimleri ajan sohbet işlem hattına iletilmez, bu yüzden `NO_REPLY` üretmezler.

### Cihaz hijyeni

OpenClaw tarafından yönetilen eski Matrix cihazları hesapta birikebilir ve şifreli oda güvenini anlamayı zorlaştırabilir.
Bunları şu komutla listeleyin:

```bash
openclaw matrix devices list
```

Eski OpenClaw tarafından yönetilen cihazları şu komutla kaldırın:

```bash
openclaw matrix devices prune-stale
```

### Crypto store

Matrix E2EE, Node içinde resmi `matrix-js-sdk` Rust crypto yolunu ve IndexedDB shim olarak `fake-indexeddb` kullanır. Crypto durumu bir anlık görüntü dosyasına (`crypto-idb-snapshot.json`) kalıcı olarak kaydedilir ve başlangıçta geri yüklenir. Anlık görüntü dosyası, kısıtlayıcı dosya izinleriyle saklanan hassas çalışma zamanı durumudur.

Şifreli çalışma zamanı durumu, hesap başına, kullanıcı başına token-hash kökleri altında
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/` dizininde bulunur.
Bu dizin sync store'u (`bot-storage.json`), crypto store'u (`crypto/`),
kurtarma anahtarı dosyasını (`recovery-key.json`), IndexedDB anlık görüntüsünü (`crypto-idb-snapshot.json`),
thread bağlarını (`thread-bindings.json`) ve başlangıç doğrulama durumunu (`startup-verification.json`) içerir.
Belirteç değiştiğinde ama hesap kimliği aynı kaldığında OpenClaw, bu hesap/homeserver/kullanıcı demeti için
mevcut en iyi kökü yeniden kullanır; böylece önceki sync durumu, crypto durumu, thread bağları
ve başlangıç doğrulama durumu görünür kalır.

## Profil yönetimi

Seçili hesap için Matrix self-profile'ı şu komutla güncelleyin:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Açıkça adlandırılmış bir Matrix hesabını hedeflemek istediğinizde `--account <id>` ekleyin.

Matrix `mxc://` avatar URL'lerini doğrudan kabul eder. Bir `http://` veya `https://` avatar URL'si verdiğinizde OpenClaw önce bunu Matrix'e yükler ve çözümlenen `mxc://` URL'sini tekrar `channels.matrix.avatarUrl` içine (veya seçilen hesap geçersiz kılmasına) kaydeder.

## Thread'ler

Matrix, hem otomatik yanıtlar hem de mesaj aracı gönderimleri için yerel Matrix thread'lerini destekler.

- `dm.sessionScope: "per-user"` (varsayılan), Matrix DM yönlendirmesini gönderen kapsamlı tutar; böylece aynı eşe çözümlenen birden fazla DM odası tek bir oturumu paylaşabilir.
- `dm.sessionScope: "per-room"`, her Matrix DM odasını kendi oturum anahtarına izole ederken normal DM kimlik doğrulama ve izin listesi denetimlerini kullanmaya devam eder.
- Açık Matrix konuşma bağları yine de `dm.sessionScope` üzerinde önceliklidir; bu yüzden bağlı odalar ve thread'ler seçilmiş hedef oturumlarını korur.
- `threadReplies: "off"`, yanıtları üst düzeyde tutar ve gelen thread'li mesajları ana oturum üzerinde tutar.
- `threadReplies: "inbound"`, yalnızca gelen mesaj zaten o thread içindeyse thread içinde yanıt verir.
- `threadReplies: "always"`, oda yanıtlarını tetikleyici mesaja köklenen bir thread içinde tutar ve o konuşmayı ilk tetikleyici mesajdan itibaren eşleşen thread kapsamlı oturum üzerinden yönlendirir.
- `dm.threadReplies`, yalnızca DM'ler için üst düzey ayarı geçersiz kılar. Örneğin, odalardaki thread'leri izole tutarken DM'leri düz tutabilirsiniz.
- Gelen thread'li mesajlar, thread kök mesajını ek ajan bağlamı olarak içerir.
- Mesaj aracı gönderimleri, açık bir `threadId` verilmediği sürece hedef aynı odaya ya da aynı DM kullanıcı hedefineyse mevcut Matrix thread'ini otomatik devralır.
- Aynı oturumlu DM kullanıcı hedefi yeniden kullanımı yalnızca mevcut oturum meta verisi aynı Matrix hesabındaki aynı DM eşini kanıtlarsa devreye girer; aksi halde OpenClaw normal kullanıcı kapsamlı yönlendirmeye geri döner.
- OpenClaw, bir Matrix DM odasının aynı paylaşılan Matrix DM oturumunda başka bir DM odasıyla çakıştığını gördüğünde, thread bağları etkinse ve `dm.sessionScope` ipucu varsa o odaya bir defalık `/focus` kaçış yolunu içeren bir `m.notice` gönderir.
- Çalışma zamanı thread bağları Matrix için desteklenir. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` ve thread'e bağlı `/acp spawn`, Matrix odalarında ve DM'lerde çalışır.
- Üst düzey Matrix oda/DM `/focus`, `threadBindings.spawnSubagentSessions=true` olduğunda yeni bir Matrix thread'i oluşturur ve bunu hedef oturuma bağlar.
- Mevcut bir Matrix thread'i içinde `/focus` veya `/acp spawn --thread here` çalıştırmak bunun yerine geçerli thread'i bağlar.

## ACP konuşma bağları

Matrix odaları, DM'ler ve mevcut Matrix thread'leri, sohbet yüzeyi değiştirilmeden kalıcı ACP çalışma alanlarına dönüştürülebilir.

Hızlı operatör akışı:

- Kullanmaya devam etmek istediğiniz Matrix DM, oda veya mevcut thread içinde `/acp spawn codex --bind here` çalıştırın.
- Üst düzey bir Matrix DM veya odasında mevcut DM/oda sohbet yüzeyi olarak kalır ve gelecekteki mesajlar oluşturulan ACP oturumuna yönlendirilir.
- Mevcut bir Matrix thread'i içinde `--bind here`, o geçerli thread'i yerinde bağlar.
- `/new` ve `/reset`, aynı bağlı ACP oturumunu yerinde sıfırlar.
- `/acp close`, ACP oturumunu kapatır ve bağı kaldırır.

Notlar:

- `--bind here` bir alt Matrix thread'i oluşturmaz.
- `threadBindings.spawnAcpSessions`, yalnızca OpenClaw'un bir alt Matrix thread'i oluşturması veya bağlaması gereken `/acp spawn --thread auto|here` için gereklidir.

### Thread bağlama yapılandırması

Matrix, genel varsayılanları `session.threadBindings` üzerinden devralır ve kanal başına geçersiz kılmaları da destekler:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

Matrix thread'e bağlı spawn işaretleri isteğe bağlıdır:

- Üst düzey `/focus` komutunun yeni Matrix thread'leri oluşturup bağlamasına izin vermek için `threadBindings.spawnSubagentSessions: true` ayarlayın.
- `/acp spawn --thread auto|here` komutunun ACP oturumlarını Matrix thread'lerine bağlamasına izin vermek için `threadBindings.spawnAcpSessions: true` ayarlayın.

## Tepkiler

Matrix, giden tepki eylemlerini, gelen tepki bildirimlerini ve gelen ack tepkilerini destekler.

- Giden tepki araçları `channels["matrix"].actions.reactions` tarafından denetlenir.
- `react`, belirli bir Matrix olayına tepki ekler.
- `reactions`, belirli bir Matrix olayı için geçerli tepki özetini listeler.
- `emoji=""`, bot hesabının o olay üzerindeki kendi tepkilerini kaldırır.
- `remove: true`, bot hesabından yalnızca belirtilen emoji tepkisini kaldırır.

Ack tepkileri standart OpenClaw çözümleme sırasını kullanır:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- ajan kimliği emoji geri dönüşü

Ack tepki kapsamı şu sırayla çözülür:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

Tepki bildirim modu şu sırayla çözülür:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- varsayılan: `own`

Davranış:

- `reactionNotifications: "own"`, bot tarafından yazılmış Matrix mesajlarını hedeflediklerinde eklenen `m.reaction` olaylarını iletir.
- `reactionNotifications: "off"`, tepki sistem olaylarını devre dışı bırakır.
- Matrix bunları bağımsız `m.reaction` kaldırmaları olarak değil redact işlemleri olarak sunduğu için tepki kaldırmaları sistem olaylarına dönüştürülmez.

## Geçmiş bağlamı

- `channels.matrix.historyLimit`, bir Matrix oda mesajı ajanı tetiklediğinde `InboundHistory` olarak eklenecek son oda mesajı sayısını denetler. `messages.groupChat.historyLimit` değerine geri döner; her ikisi de ayarlı değilse etkin varsayılan `0` olur. Devre dışı bırakmak için `0` ayarlayın.
- Matrix oda geçmişi yalnızca oda içindir. DM'ler normal oturum geçmişini kullanmaya devam eder.
- Matrix oda geçmişi yalnızca pending durumundadır: OpenClaw henüz bir yanıt tetiklememiş oda mesajlarını arabelleğe alır, ardından bir mention veya başka bir tetikleyici geldiğinde bu pencerenin anlık görüntüsünü alır.
- Geçerli tetikleyici mesaj `InboundHistory` içine dahil edilmez; o tur için ana gelen gövdede kalır.
- Aynı Matrix olayının yeniden denemeleri, daha yeni oda mesajlarına doğru kaymak yerine özgün geçmiş anlık görüntüsünü yeniden kullanır.

## Bağlam görünürlüğü

Matrix, alınan yanıt metni, thread kökleri ve pending geçmiş gibi ek oda bağlamı için paylaşılan `contextVisibility` denetimini destekler.

- `contextVisibility: "all"` varsayılandır. Ek bağlam alındığı gibi korunur.
- `contextVisibility: "allowlist"`, ek bağlamı etkin oda/kullanıcı izin listesi denetimleri tarafından izin verilen gönderenlere filtreler.
- `contextVisibility: "allowlist_quote"`, `allowlist` gibi davranır ama yine de açık biçimde alıntılanmış tek bir yanıtı korur.

Bu ayar, ek bağlam görünürlüğünü etkiler; gelen mesajın kendisinin bir yanıtı tetikleyip tetikleyemeyeceğini etkilemez.
Tetikleme yetkilendirmesi hâlâ `groupPolicy`, `groups`, `groupAllowFrom` ve DM politika ayarlarından gelir.

## DM ve oda politikası

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Mention geçitleme ve izin listesi davranışı için [Groups](/tr/channels/groups) bölümüne bakın.

Matrix DM'leri için eşleme örneği:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Onaylanmamış bir Matrix kullanıcısı siz onu onaylamadan önce size mesaj göndermeye devam ederse, OpenClaw aynı bekleyen eşleme kodunu yeniden kullanır ve yeni bir kod üretmek yerine kısa bir cooldown sonrasında yeniden bir hatırlatma yanıtı gönderebilir.

Paylaşılan DM eşleme akışı ve depolama düzeni için [Pairing](/tr/channels/pairing) bölümüne bakın.

## Doğrudan oda onarımı

Doğrudan mesaj durumu eşzamanını kaybederse, OpenClaw canlı DM yerine eski tekil odaları işaret eden eski `m.direct` eşlemeleriyle kalabilir. Bir eş için geçerli eşlemeyi şu komutla inceleyin:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Şu komutla onarın:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

Onarım akışı:

- zaten `m.direct` içinde eşlenmiş katı 1:1 bir DM'yi tercih eder
- bunun yerine o kullanıcıyla şu anda katılım sağlanmış herhangi bir katı 1:1 DM'ye geri döner
- sağlıklı bir DM yoksa yeni bir doğrudan oda oluşturur ve `m.direct` değerini yeniden yazar

Onarım akışı eski odaları otomatik olarak silmez. Yalnızca sağlıklı DM'yi seçer ve eşlemeyi günceller; böylece yeni Matrix gönderimleri, doğrulama bildirimleri ve diğer doğrudan mesaj akışları yeniden doğru odayı hedefler.

## Exec onayları

Matrix, bir Matrix hesabı için yerel bir onay istemcisi gibi davranabilir. Yerel
DM/kanal yönlendirme ayarları yine exec onay yapılandırması altında yer alır:

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (isteğe bağlı; `channels.matrix.dm.allowFrom` değerine geri döner)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, varsayılan: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Onaylayıcılar `@owner:example.org` gibi Matrix kullanıcı kimlikleri olmalıdır. Matrix, `enabled` ayarsız veya `"auto"` olduğunda ve en az bir onaylayıcı çözümlenebildiğinde yerel onayları otomatik etkinleştirir. Exec onayları önce `execApprovals.approvers` kullanır ve `channels.matrix.dm.allowFrom` değerine geri dönebilir. Eklenti onayları `channels.matrix.dm.allowFrom` üzerinden yetkilendirilir. Matrix'i yerel bir onay istemcisi olarak açıkça devre dışı bırakmak için `enabled: false` ayarlayın. Aksi halde onay istekleri diğer yapılandırılmış onay yollarına veya onay geri dönüş politikasına döner.

Matrix yerel yönlendirmesi her iki onay türünü de destekler:

- `channels.matrix.execApprovals.*`, Matrix onay istemleri için yerel DM/kanal fanout modunu denetler.
- Exec onayları `execApprovals.approvers` veya `channels.matrix.dm.allowFrom` içinden exec onaylayıcı kümesini kullanır.
- Eklenti onayları Matrix DM izin listesini `channels.matrix.dm.allowFrom` içinden kullanır.
- Matrix tepki kısayolları ve mesaj güncellemeleri hem exec hem de eklenti onayları için geçerlidir.

Teslimat kuralları:

- `target: "dm"`, onay istemlerini onaylayıcı DM'lerine gönderir
- `target: "channel"`, istemi kaynak Matrix odasına veya DM'ye geri gönderir
- `target: "both"`, onaylayıcı DM'lerine ve kaynak Matrix odasına veya DM'ye gönderir

Matrix onay istemleri, birincil onay mesajında tepki kısayolları başlatır:

- `✅` = bir kez izin ver
- `❌` = reddet
- `♾️` = bu karar etkin exec politikası tarafından izin verildiğinde her zaman izin ver

Onaylayıcılar o mesaja tepki verebilir veya geri dönüş slash komutlarını kullanabilir: `/approve <id> allow-once`, `/approve <id> allow-always` veya `/approve <id> deny`.

Yalnızca çözümlenmiş onaylayıcılar izin verebilir veya reddedebilir. Exec onaylarında kanal teslimatı komut metnini içerir, bu nedenle `channel` veya `both` seçeneklerini yalnızca güvenilir odalarda etkinleştirin.

Hesap başına geçersiz kılma:

- `channels.matrix.accounts.<account>.execApprovals`

İlgili belgeler: [Exec approvals](/tr/tools/exec-approvals)

## Çok hesap

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

Üst düzey `channels.matrix` değerleri, bir hesap bunları geçersiz kılmadıkça adlandırılmış hesaplar için varsayılan görevi görür.
Devralınan oda girdilerini bir Matrix hesabıyla sınırlamak için `groups.<room>.account` kullanabilirsiniz.
`account` içermeyen girdiler tüm Matrix hesapları arasında paylaşımlı kalır ve `account: "default"` içeren girdiler varsayılan hesap doğrudan üst düzey `channels.matrix.*` üzerinde yapılandırıldığında da çalışmaya devam eder.
Kısmi paylaşımlı kimlik doğrulama varsayılanları kendi başlarına ayrı bir örtük varsayılan hesap oluşturmaz. OpenClaw üst düzey `default` hesabını yalnızca o varsayılan hesapta yeni kimlik doğrulama varsa (`homeserver` artı `accessToken` veya `homeserver` artı `userId` ve `password`) sentezler; adlandırılmış hesaplar daha sonra önbelleğe alınmış kimlik bilgileri kimlik doğrulamayı sağladığında `homeserver` artı `userId` üzerinden yine keşfedilebilir kalabilir.
Matrix'te zaten tam olarak bir adlandırılmış hesap varsa veya `defaultAccount` mevcut bir adlandırılmış hesap anahtarını işaret ediyorsa, tek hesaptan çok hesaba onarım/kurulum yükseltmesi yeni bir `accounts.default` girdisi oluşturmak yerine o hesabı korur. Yalnızca Matrix kimlik doğrulama/bootstrap anahtarları bu yükseltilmiş hesaba taşınır; paylaşılan teslimat politikası anahtarları üst düzeyde kalır.
OpenClaw'un örtük yönlendirme, probe ve CLI işlemleri için bir adlandırılmış Matrix hesabını tercih etmesini istiyorsanız `defaultAccount` ayarlayın.
Birden fazla adlandırılmış hesap yapılandırırsanız, örtük hesap seçimine dayanan CLI komutları için `defaultAccount` ayarlayın veya `--account <id>` geçin.
Bu örtük seçimi tek bir komut için geçersiz kılmak istediğinizde `openclaw matrix verify ...` ve `openclaw matrix devices ...` komutlarına `--account <id>` geçin.

Paylaşılan çok hesaplı desen için [Configuration reference](/tr/gateway/configuration-reference#multi-account-all-channels) bölümüne bakın.

## Özel/LAN homeserver'lar

Varsayılan olarak OpenClaw, siz
hesap başına açıkça izin vermediğiniz sürece SSRF koruması için özel/dahili Matrix homeserver'ları engeller.

Homeserver'ınız localhost, bir LAN/Tailscale IP'si veya dahili bir ana makine adında çalışıyorsa,
o Matrix hesabı için `network.dangerouslyAllowPrivateNetwork` seçeneğini etkinleştirin:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

CLI kurulum örneği:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Bu açık onay yalnızca güvenilir özel/dahili hedeflere izin verir. `http://matrix.example.org:8008` gibi
genel düz metin homeserver'lar engellenmiş kalır. Mümkün olduğunda `https://` tercih edin.

## Matrix trafiğini proxy ile yönlendirme

Matrix dağıtımınız açık bir giden HTTP(S) proxy gerektiriyorsa, `channels.matrix.proxy` ayarlayın:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Adlandırılmış hesaplar üst düzey varsayılanı `channels.matrix.accounts.<id>.proxy` ile geçersiz kılabilir.
OpenClaw aynı proxy ayarını çalışma zamanı Matrix trafiği ve hesap durum probe'ları için kullanır.

## Hedef çözümleme

Matrix, OpenClaw'un sizden bir oda veya kullanıcı hedefi istediği her yerde şu hedef biçimlerini kabul eder:

- Kullanıcılar: `@user:server`, `user:@user:server` veya `matrix:user:@user:server`
- Odalar: `!room:server`, `room:!room:server` veya `matrix:room:!room:server`
- Takma adlar: `#alias:server`, `channel:#alias:server` veya `matrix:channel:#alias:server`

Canlı dizin araması, oturum açmış Matrix hesabını kullanır:

- Kullanıcı aramaları o homeserver'daki Matrix kullanıcı dizinini sorgular.
- Oda aramaları açık oda kimliklerini ve takma adları doğrudan kabul eder, ardından bu hesap için katılınan oda adlarını aramaya geri döner.
- Katılınan oda adı araması best-effort çalışır. Bir oda adı bir kimliğe veya takma ada çözümlenemezse, çalışma zamanı izin listesi çözümlemesinde yok sayılır.

## Yapılandırma başvurusu

- `enabled`: kanalı etkinleştir veya devre dışı bırak.
- `name`: hesap için isteğe bağlı etiket.
- `defaultAccount`: birden fazla Matrix hesabı yapılandırıldığında tercih edilen hesap kimliği.
- `homeserver`: homeserver URL'si, örneğin `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: bu Matrix hesabının özel/dahili homeserver'lara bağlanmasına izin ver. Homeserver `localhost`, bir LAN/Tailscale IP'si veya `matrix-synapse` gibi dahili bir ana makineye çözümlendiğinde bunu etkinleştirin.
- `proxy`: Matrix trafiği için isteğe bağlı HTTP(S) proxy URL'si. Adlandırılmış hesaplar üst düzey varsayılanı kendi `proxy` değerleriyle geçersiz kılabilir.
- `userId`: tam Matrix kullanıcı kimliği, örneğin `@bot:example.org`.
- `accessToken`: belirteç tabanlı kimlik doğrulama için erişim belirteci. Düz metin değerleri ve SecretRef değerleri, env/file/exec sağlayıcıları genelinde `channels.matrix.accessToken` ve `channels.matrix.accounts.<id>.accessToken` için desteklenir. Bkz. [Secrets Management](/tr/gateway/secrets).
- `password`: parola tabanlı giriş için parola. Düz metin değerleri ve SecretRef değerleri desteklenir.
- `deviceId`: açık Matrix cihaz kimliği.
- `deviceName`: parola ile giriş için cihaz görünen adı.
- `avatarUrl`: profil eşitleme ve `profile set` güncellemeleri için saklanan self-avatar URL'si.
- `initialSyncLimit`: başlangıç senkronizasyonu sırasında alınan en fazla olay sayısı.
- `encryption`: E2EE'yi etkinleştir.
- `allowlistOnly`: `true` olduğunda `open` oda politikasını `allowlist` seviyesine yükseltir ve `disabled` dışındaki tüm etkin DM politikalarını (`pairing` ve `open` dahil) `allowlist` olmaya zorlar. `disabled` politikalarını etkilemez.
- `allowBots`: yapılandırılmış diğer OpenClaw Matrix hesaplarından gelen mesajlara izin ver (`true` veya `"mentions"`).
- `groupPolicy`: `open`, `allowlist` veya `disabled`.
- `contextVisibility`: ek oda bağlamı görünürlük modu (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: oda trafiği için kullanıcı kimliği izin listesi. Girdiler tam Matrix kullanıcı kimlikleri olmalıdır; çözümlenmemiş adlar çalışma zamanında yok sayılır.
- `historyLimit`: grup geçmiş bağlamı olarak eklenecek en fazla oda mesajı sayısı. `messages.groupChat.historyLimit` değerine geri döner; her ikisi de ayarlı değilse etkin varsayılan `0` olur. Devre dışı bırakmak için `0` ayarlayın.
- `replyToMode`: `off`, `first`, `all` veya `batched`.
- `markdown`: giden Matrix metni için isteğe bağlı Markdown işleme yapılandırması.
- `streaming`: `off` (varsayılan), `"partial"`, `"quiet"`, `true` veya `false`. `"partial"` ve `true`, normal Matrix metin mesajlarıyla önizleme-önce taslak güncellemelerini etkinleştirir. `"quiet"`, self-hosted push-rule kurulumları için bildirim vermeyen önizleme bildirimleri kullanır. `false`, `"off"` ile eşdeğerdir.
- `blockStreaming`: `true`, taslak önizleme akışı etkinken tamamlanmış asistan blokları için ayrı ilerleme mesajlarını etkinleştirir.
- `threadReplies`: `off`, `inbound` veya `always`.
- `threadBindings`: thread'e bağlı oturum yönlendirmesi ve yaşam döngüsü için kanal başına geçersiz kılmalar.
- `startupVerification`: başlangıçta otomatik self-verification istek modu (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: otomatik başlangıç doğrulama isteklerini yeniden denemeden önce bekleme süresi.
- `textChunkLimit`: karakter cinsinden giden mesaj parça boyutu (`chunkMode` değeri `length` olduğunda uygulanır).
- `chunkMode`: `length`, mesajları karakter sayısına göre böler; `newline`, satır sınırlarında böler.
- `responsePrefix`: bu kanal için tüm giden yanıtların başına eklenecek isteğe bağlı dize.
- `ackReaction`: bu kanal/hesap için isteğe bağlı ack tepki geçersiz kılması.
- `ackReactionScope`: isteğe bağlı ack tepki kapsamı geçersiz kılması (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: gelen tepki bildirim modu (`own`, `off`).
- `mediaMaxMb`: giden gönderimler ve gelen medya işleme için MB cinsinden medya boyutu üst sınırı.
- `autoJoin`: davet otomatik katılım politikası (`always`, `allowlist`, `off`). Varsayılan: `off`. DM tarzı davetler dahil tüm Matrix davetleri için geçerlidir.
- `autoJoinAllowlist`: `autoJoin` değeri `allowlist` olduğunda izin verilen odalar/takma adlar. Takma ad girdileri davet işleme sırasında oda kimliklerine çözülür; OpenClaw davet edilen odanın iddia ettiği takma ad durumuna güvenmez.
- `dm`: DM politika bloğu (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- `dm.policy`: OpenClaw odaya katıldıktan ve onu DM olarak sınıflandırdıktan sonra DM erişimini denetler. Bir davetin otomatik katılımını değiştirmaz.
- `dm.allowFrom`: bunları canlı dizin aramasıyla zaten çözümlemediyseniz girdiler tam Matrix kullanıcı kimlikleri olmalıdır.
- `dm.sessionScope`: `per-user` (varsayılan) veya `per-room`. Eş aynı olsa bile her Matrix DM odasının ayrı bağlam tutmasını istiyorsanız `per-room` kullanın.
- `dm.threadReplies`: yalnızca DM için thread politika geçersiz kılması (`off`, `inbound`, `always`). Hem yanıt yerleşimi hem de DM'lerde oturum izolasyonu için üst düzey `threadReplies` ayarını geçersiz kılar.
- `execApprovals`: Matrix yerel exec onay teslimatı (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: exec isteklerini onaylamasına izin verilen Matrix kullanıcı kimlikleri. `dm.allowFrom` zaten onaylayıcıları tanımlıyorsa isteğe bağlıdır.
- `execApprovals.target`: `dm | channel | both` (varsayılan: `dm`).
- `accounts`: adlandırılmış hesap başına geçersiz kılmalar. Üst düzey `channels.matrix` değerleri bu girdiler için varsayılan görevi görür.
- `groups`: oda başına politika eşlemi. Oda kimliklerini veya takma adları tercih edin; çözümlenmemiş oda adları çalışma zamanında yok sayılır. Çözümlemeden sonra oturum/grup kimliği kararlı oda kimliğini kullanır.
- `groups.<room>.account`: çok hesaplı kurulumlarda devralınmış bir oda girdisini belirli bir Matrix hesabıyla sınırla.
- `groups.<room>.allowBots`: yapılandırılmış bot gönderenler için oda düzeyi geçersiz kılma (`true` veya `"mentions"`).
- `groups.<room>.users`: oda başına gönderen izin listesi.
- `groups.<room>.tools`: oda başına araç izin/verme reddetme geçersiz kılmaları.
- `groups.<room>.autoReply`: oda düzeyi mention geçitleme geçersiz kılması. `true`, o oda için mention gereksinimlerini devre dışı bırakır; `false`, bunları yeniden zorlar.
- `groups.<room>.skills`: isteğe bağlı oda düzeyi Skills filtresi.
- `groups.<room>.systemPrompt`: isteğe bağlı oda düzeyi system prompt parçası.
- `rooms`: `groups` için eski takma ad.
- `actions`: eylem başına araç denetimi (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## İlgili

- [Channels Overview](/tr/channels) — desteklenen tüm kanallar
- [Pairing](/tr/channels/pairing) — DM kimlik doğrulaması ve eşleme akışı
- [Groups](/tr/channels/groups) — grup sohbeti davranışı ve mention geçitleme
- [Channel Routing](/tr/channels/channel-routing) — mesajlar için oturum yönlendirmesi
- [Security](/tr/gateway/security) — erişim modeli ve sağlamlaştırma
