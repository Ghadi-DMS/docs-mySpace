---
read_when:
    - API sağlayıcıları başarısız olduğunda güvenilir bir yedek çözüm istiyorsunuz
    - Codex CLI veya diğer yerel AI CLI'larını çalıştırıyorsunuz ve bunları yeniden kullanmak istiyorsunuz
    - CLI arka uç araç erişimi için MCP loopback köprüsünü anlamak istiyorsunuz
summary: 'CLI arka uçları: isteğe bağlı MCP araç köprüsüyle yerel AI CLI yedek çözümü'
title: CLI Arka Uçları
x-i18n:
    generated_at: "2026-04-09T01:28:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9b458f9fe6fa64c47864c8c180f3dedfd35c5647de470a2a4d31c26165663c20
    source_path: gateway/cli-backends.md
    workflow: 15
---

# CLI arka uçları (yedek çalışma zamanı)

OpenClaw, API sağlayıcıları devre dışı kaldığında, oran sınırına takıldığında
veya geçici olarak hatalı davrandığında **yerel AI CLI'larını** **yalnızca metin içeren bir yedek çözüm** olarak çalıştırabilir. Bu tasarım bilinçli olarak muhafazakardır:

- **OpenClaw araçları doğrudan enjekte edilmez**, ancak `bundleMcp: true` olan arka uçlar
  gateway araçlarını bir loopback MCP köprüsü üzerinden alabilir.
- Bunu destekleyen CLI'lar için **JSONL akışı**.
- **Oturumlar desteklenir** (böylece takip eden dönüşler tutarlı kalır).
- CLI görüntü yollarını kabul ediyorsa **görüntüler geçirilebilir**.

Bu, birincil yol olmaktan çok bir **güvenlik ağı** olarak tasarlanmıştır. Harici API'lere
güvenmeden “her zaman çalışan” metin yanıtları istediğinizde kullanın.

ACP oturum denetimleri, arka plan görevleri,
iş parçacığı/konuşma bağlama ve kalıcı harici kodlama oturumları içeren tam bir harness çalışma zamanı istiyorsanız
bunun yerine [ACP Agents](/tr/tools/acp-agents) kullanın. CLI arka uçları ACP değildir.

## Başlangıç dostu hızlı başlangıç

Codex CLI'ı **hiçbir yapılandırma olmadan** kullanabilirsiniz (paketle gelen OpenAI eklentisi
varsayılan bir arka uç kaydeder):

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

Gateway'iniz launchd/systemd altında çalışıyorsa ve PATH minimal ise, yalnızca
komut yolunu ekleyin:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

Hepsi bu kadar. CLI'ın kendisi dışında anahtar, ek kimlik doğrulama yapılandırması gerekmez.

Bir gateway ana bilgisayarında paketle gelen bir CLI arka ucunu **birincil mesaj sağlayıcısı** olarak kullanırsanız,
OpenClaw artık yapılandırmanız
bir model başvurusunda veya `agents.defaults.cliBackends` altında bu arka uca açıkça
başvurduğunda ilgili paketle gelen eklentiyi otomatik olarak yükler.

## Yedek çözüm olarak kullanma

Bir CLI arka ucunu yedek listenize ekleyin; böylece yalnızca birincil modeller başarısız olduğunda çalışır:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.4"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/gpt-5.4": {},
      },
    },
  },
}
```

Notlar:

- `agents.defaults.models` (izin listesi) kullanıyorsanız, CLI arka uç modellerinizi de oraya eklemeniz gerekir.
- Birincil sağlayıcı başarısız olursa (kimlik doğrulama, oran sınırları, zaman aşımı), OpenClaw
  sırada CLI arka ucunu dener.

## Yapılandırma genel görünümü

Tüm CLI arka uçları şunun altında bulunur:

```
agents.defaults.cliBackends
```

Her giriş bir **sağlayıcı kimliği** ile anahtarlanır (ör. `codex-cli`, `my-cli`).
Sağlayıcı kimliği, model başvurunuzun sol tarafı olur:

```
<provider>/<model>
```

### Örnek yapılandırma

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
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          // Codex tarzı CLI'lar bunun yerine bir istem dosyasına işaret edebilir:
          // systemPromptFileConfigArg: "-c",
          // systemPromptFileConfigKey: "model_instructions_file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          serialize: true,
        },
      },
    },
  },
}
```

## Nasıl çalışır

1. Sağlayıcı önekine (`codex-cli/...`) göre bir **arka uç seçer**.
2. Aynı OpenClaw istemi + çalışma alanı bağlamını kullanarak bir **sistem istemi oluşturur**.
3. Geçmiş tutarlı kalsın diye CLI'ı bir oturum kimliğiyle (destekleniyorsa) **çalıştırır**.
4. **Çıktıyı ayrıştırır** (JSON veya düz metin) ve son metni döndürür.
5. Takip eden istekler aynı CLI oturumunu yeniden kullansın diye arka uç başına **oturum kimliklerini kalıcı hale getirir**.

<Note>
Paketle gelen Anthropic `claude-cli` arka ucu yeniden destekleniyor. Anthropic çalışanları
OpenClaw tarzı Claude CLI kullanımına yeniden izin verildiğini söyledi; bu nedenle Anthropic
yeni bir politika yayınlayana kadar OpenClaw, bu entegrasyon için `claude -p`
kullanımını onaylı kabul eder.
</Note>

Paketle gelen OpenAI `codex-cli` arka ucu, OpenClaw'ın sistem istemini
Codex'in `model_instructions_file` yapılandırma geçersiz kılmasını (`-c
model_instructions_file="..."`) kullanarak geçirir. Codex, Claude tarzı bir
`--append-system-prompt` bayrağı sunmaz; bu nedenle OpenClaw, her yeni Codex CLI oturumu için
oluşturulan istemi geçici bir dosyaya yazar.

## Oturumlar

- CLI oturumları destekliyorsa, bir oturum kimliği göndermek için `sessionArg` (ör. `--session-id`) veya
  kimliğin birden çok bayrağa eklenmesi gerektiğinde
  `sessionArgs` (yer tutucu `{sessionId}`) ayarlayın.
- CLI farklı bayraklara sahip bir **resume subcommand** kullanıyorsa,
  `resumeArgs` (`args` yerine geçer) ve isteğe bağlı olarak
  `resumeOutput` (JSON olmayan devamlar için) ayarlayın.
- `sessionMode`:
  - `always`: her zaman bir oturum kimliği gönderir (saklanmış yoksa yeni UUID oluşturur).
  - `existing`: yalnızca daha önce bir kimlik saklandıysa bir oturum kimliği gönderir.
  - `none`: asla bir oturum kimliği göndermez.

Serileştirme notları:

- `serialize: true` aynı hat üzerindeki çalıştırmaları sıralı tutar.
- Çoğu CLI tek bir sağlayıcı hattında serileştirme yapar.
- OpenClaw, arka uç kimlik doğrulama durumu değiştiğinde saklanan CLI oturum yeniden kullanımını bırakır; buna yeniden giriş, token döndürme veya değişen bir kimlik doğrulama profili kimlik bilgisi dahildir.

## Görüntüler (geçirme)

CLI'ınız görüntü yollarını kabul ediyorsa, `imageArg` ayarlayın:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw, base64 görüntüleri geçici dosyalara yazar. `imageArg` ayarlanmışsa bu
yollar CLI bağımsız değişkenleri olarak geçirilir. `imageArg` eksikse OpenClaw
dosya yollarını isteme ekler (yol enjeksiyonu); bu, düz yollardan yerel dosyaları otomatik
yükleyen CLI'lar için yeterlidir.

## Girdiler / çıktılar

- `output: "json"` (varsayılan) JSON ayrıştırmayı dener ve metin + oturum kimliğini çıkarır.
- Gemini CLI JSON çıktısı için OpenClaw, `usage` eksikse veya boşsa
  yanıt metnini `response` alanından, kullanım bilgisini ise `stats` alanından okur.
- `output: "jsonl"` JSONL akışlarını ayrıştırır (örneğin Codex CLI `--json`) ve mevcutsa son agent mesajını artı oturum
  tanımlayıcılarını çıkarır.
- `output: "text"` stdout'u son yanıt olarak kabul eder.

Girdi modları:

- `input: "arg"` (varsayılan) istemi son CLI bağımsız değişkeni olarak geçirir.
- `input: "stdin"` istemi stdin üzerinden gönderir.
- İstem çok uzunsa ve `maxPromptArgChars` ayarlanmışsa stdin kullanılır.

## Varsayılanlar (eklenti sahibi)

Paketle gelen OpenAI eklentisi ayrıca `codex-cli` için bir varsayılan kaydeder:

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

Paketle gelen Google eklentisi ayrıca `google-gemini-cli` için bir varsayılan kaydeder:

- `command: "gemini"`
- `args: ["--output-format", "json", "--prompt", "{prompt}"]`
- `resumeArgs: ["--resume", "{sessionId}", "--output-format", "json", "--prompt", "{prompt}"]`
- `imageArg: "@"`
- `imagePathScope: "workspace"`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

Ön koşul: yerel Gemini CLI kurulu olmalı ve `PATH` üzerinde
`gemini` olarak erişilebilir olmalıdır (`brew install gemini-cli` veya
`npm install -g @google/gemini-cli`).

Gemini CLI JSON notları:

- Yanıt metni JSON `response` alanından okunur.
- `usage` yoksa veya boşsa kullanım bilgisi `stats` alanına geri döner.
- `stats.cached`, OpenClaw `cacheRead` olarak normalize edilir.
- `stats.input` eksikse OpenClaw giriş tokenlarını
  `stats.input_tokens - stats.cached` üzerinden türetir.

Yalnızca gerekirse geçersiz kılın (yaygın durum: mutlak `command` yolu).

## Eklenti sahibi varsayılanlar

CLI arka ucu varsayılanları artık eklenti yüzeyinin bir parçasıdır:

- Eklentiler bunları `api.registerCliBackend(...)` ile kaydeder.
- Arka uç `id` değeri, model başvurularında sağlayıcı öneki olur.
- `agents.defaults.cliBackends.<id>` içindeki kullanıcı yapılandırması yine eklenti varsayılanını geçersiz kılar.
- Arka uca özgü yapılandırma temizliği, isteğe bağlı
  `normalizeConfig` hook'u aracılığıyla eklenti sahibinde kalır.

## Bundle MCP katmanları

CLI arka uçları OpenClaw araç çağrılarını **doğrudan** almaz, ancak bir arka uç
`bundleMcp: true` ile oluşturulmuş bir MCP yapılandırma katmanına katılmayı seçebilir.

Mevcut paketle gelen davranış:

- `claude-cli`: oluşturulmuş strict MCP yapılandırma dosyası
- `codex-cli`: `mcp_servers` için satır içi yapılandırma geçersiz kılmaları
- `google-gemini-cli`: oluşturulmuş Gemini sistem ayarları dosyası

Bundle MCP etkinleştirildiğinde OpenClaw:

- gateway araçlarını CLI işlemine açan bir loopback HTTP MCP sunucusu başlatır
- köprünün kimliğini oturum başına bir token ile doğrular (`OPENCLAW_MCP_TOKEN`)
- araç erişimini geçerli oturum, hesap ve kanal bağlamıyla sınırlar
- geçerli çalışma alanı için etkin bundle-MCP sunucularını yükler
- bunları mevcut herhangi bir arka uç MCP yapılandırması/ayar biçimiyle birleştirir
- başlatma yapılandırmasını, ilgili uzantının sahip olduğu entegrasyon modunu kullanarak yeniden yazar

Hiçbir MCP sunucusu etkin değilse bile OpenClaw, bir arka uç
bundle MCP'ye katılmayı seçtiğinde arka plan çalıştırmaları yalıtılmış kalsın diye yine de strict bir yapılandırma enjekte eder.

## Sınırlamalar

- **Doğrudan OpenClaw araç çağrıları yoktur.** OpenClaw, araç çağrılarını
  CLI arka ucu protokolüne enjekte etmez. Arka uçlar gateway araçlarını yalnızca
  `bundleMcp: true` ile katılmayı seçtiklerinde görür.
- **Akış arka uca özeldir.** Bazı arka uçlar JSONL akışı yapar; diğerleri
  çıkışa kadar arabelleğe alır.
- **Yapılandırılmış çıktılar** CLI'ın JSON biçimine bağlıdır.
- **Codex CLI oturumları** metin çıktısı üzerinden devam eder (JSONL yoktur); bu,
  ilk `--json` çalıştırmasından daha az yapılandırılmıştır. OpenClaw oturumları yine de
  normal şekilde çalışır.

## Sorun giderme

- **CLI bulunamadı**: `command` için tam yol ayarlayın.
- **Yanlış model adı**: `provider/model` → CLI modeli eşlemesi için `modelAliases` kullanın.
- **Oturum sürekliliği yok**: `sessionArg` ayarlandığından ve `sessionMode` değerinin
  `none` olmadığından emin olun (Codex CLI şu anda JSON çıktısıyla devam edemez).
- **Görüntüler yok sayılıyor**: `imageArg` ayarlayın (ve CLI'ın dosya yollarını desteklediğini doğrulayın).
