---
read_when:
    - OpenClaw'ı Ollama üzerinden bulut veya yerel modellerle çalıştırmak istiyorsunuz
    - Ollama kurulumu ve yapılandırması konusunda rehberliğe ihtiyacınız var
summary: OpenClaw'ı Ollama ile çalıştırın (bulut ve yerel modeller)
title: Ollama
x-i18n:
    generated_at: "2026-04-08T08:43:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: d3295a7c879d3636a2ffdec05aea6e670e54a990ef52bd9b0cae253bc24aa3f7
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama, makinenizde açık kaynaklı modelleri çalıştırmayı kolaylaştıran yerel bir LLM çalışma zamanıdır. OpenClaw, Ollama'nın yerel API'si (`/api/chat`) ile entegre olur, akış ve araç çağırmayı destekler ve `OLLAMA_API_KEY` (veya bir kimlik doğrulama profili) ile etkinleştirdiğinizde ve açık bir `models.providers.ollama` girdisi tanımlamadığınızda yerel Ollama modellerini otomatik olarak keşfedebilir.

<Warning>
**Uzak Ollama kullanıcıları**: OpenClaw ile `/v1` OpenAI uyumlu URL'yi (`http://host:11434/v1`) kullanmayın. Bu, araç çağırmayı bozar ve modeller ham araç JSON'unu düz metin olarak çıktılayabilir. Bunun yerine yerel Ollama API URL'sini kullanın: `baseUrl: "http://host:11434"` (`/v1` olmadan).
</Warning>

## Hızlı başlangıç

### İlk kurulum (önerilir)

Ollama'yı kurmanın en hızlı yolu ilk kurulum sürecini kullanmaktır:

```bash
openclaw onboard
```

Sağlayıcı listesinden **Ollama** seçin. İlk kurulum şunları yapacaktır:

1. Örneğinize erişilebilen Ollama temel URL'sini sorar (varsayılan `http://127.0.0.1:11434`).
2. **Cloud + Local** (bulut modelleri ve yerel modeller) veya **Local** (yalnızca yerel modeller) seçmenize izin verir.
3. **Cloud + Local** seçerseniz ve `ollama.com` üzerinde oturum açmamışsanız tarayıcıda bir oturum açma akışı başlatır.
4. Kullanılabilir modelleri keşfeder ve varsayılanlar önerir.
5. Seçilen model yerel olarak mevcut değilse onu otomatik olarak çeker.

Etkileşimsiz mod da desteklenir:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

İsteğe bağlı olarak özel bir temel URL veya model belirtebilirsiniz:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

### Elle kurulum

1. Ollama'yı yükleyin: [https://ollama.com/download](https://ollama.com/download)

2. Yerel çıkarım istiyorsanız bir yerel model çekin:

```bash
ollama pull gemma4
# veya
ollama pull gpt-oss:20b
# veya
ollama pull llama3.3
```

3. Bulut modelleri de istiyorsanız oturum açın:

```bash
ollama signin
```

4. İlk kurulumu çalıştırın ve `Ollama` seçin:

```bash
openclaw onboard
```

- `Local`: yalnızca yerel modeller
- `Cloud + Local`: yerel modeller ve bulut modelleri
- `kimi-k2.5:cloud`, `minimax-m2.7:cloud` ve `glm-5.1:cloud` gibi bulut modelleri yerel bir `ollama pull` gerektirmez

OpenClaw şu anda şunları önerir:

- yerel varsayılan: `gemma4`
- bulut varsayılanları: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`

5. Elle kurulum tercih ediyorsanız, Ollama'yı doğrudan OpenClaw için etkinleştirin (herhangi bir değer çalışır; Ollama gerçek bir anahtar gerektirmez):

```bash
# Ortam değişkeni ayarla
export OLLAMA_API_KEY="ollama-local"

# Veya yapılandırma dosyanızda ayarlayın
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. Modelleri inceleyin veya değiştirin:

```bash
openclaw models list
openclaw models set ollama/gemma4
```

7. Ya da varsayılanı yapılandırmada ayarlayın:

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/gemma4" },
    },
  },
}
```

## Model keşfi (örtük sağlayıcı)

`OLLAMA_API_KEY` (veya bir kimlik doğrulama profili) ayarladığınızda ve `models.providers.ollama` tanımlamadığınızda, OpenClaw yerel Ollama örneğinden `http://127.0.0.1:11434` adresinde modelleri keşfeder:

- `/api/tags` sorgulanır
- Kullanılabildiğinde `contextWindow` değerini okumak ve yetenekleri (görsel dahil) algılamak için en iyi çabayla `/api/show` aramaları kullanılır
- `/api/show` tarafından bildirilen `vision` yeteneğine sahip modeller, görsel destekli (`input: ["text", "image"]`) olarak işaretlenir; böylece OpenClaw bu modeller için isteme görüntüleri otomatik olarak ekler
- `reasoning`, model adı sezgisiyle işaretlenir (`r1`, `reasoning`, `think`)
- `maxTokens`, OpenClaw tarafından kullanılan varsayılan Ollama maksimum token sınırına ayarlanır
- Tüm maliyetler `0` olarak ayarlanır

Bu, katalogu yerel Ollama örneğiyle uyumlu tutarken elle model girişi yapma gereğini ortadan kaldırır.

Hangi modellerin kullanılabilir olduğunu görmek için:

```bash
ollama list
openclaw models list
```

Yeni bir model eklemek için, onu Ollama ile çekmeniz yeterlidir:

```bash
ollama pull mistral
```

Yeni model otomatik olarak keşfedilir ve kullanıma hazır olur.

`models.providers.ollama` değerini açıkça ayarlarsanız otomatik keşif atlanır ve modelleri elle tanımlamanız gerekir (aşağıya bakın).

## Yapılandırma

### Temel kurulum (örtük keşif)

Ollama'yı etkinleştirmenin en basit yolu ortam değişkeni kullanmaktır:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### Açık kurulum (elle modeller)

Şu durumlarda açık yapılandırma kullanın:

- Ollama başka bir ana makine/bağlantı noktasında çalışıyorsa.
- Belirli bağlam pencerelerini veya model listelerini zorlamak istiyorsanız.
- Tamamen elle model tanımları istiyorsanız.

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "gpt-oss:20b",
            name: "GPT-OSS 20B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192 * 10
          }
        ]
      }
    }
  }
}
```

`OLLAMA_API_KEY` ayarlıysa, sağlayıcı girdisinde `apiKey` değerini atlayabilirsiniz; OpenClaw bunu kullanılabilirlik denetimleri için doldurur.

### Özel temel URL (açık yapılandırma)

Ollama farklı bir ana makinede veya bağlantı noktasında çalışıyorsa (açık yapılandırma otomatik keşfi devre dışı bırakır, bu nedenle modelleri elle tanımlayın):

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // /v1 yok - yerel Ollama API URL'sini kullanın
        api: "ollama", // Yerel araç çağırma davranışını garanti etmek için açıkça ayarlayın
      },
    },
  },
}
```

<Warning>
URL'ye `/v1` eklemeyin. `/v1` yolu, araç çağırmanın güvenilir olmadığı OpenAI uyumlu modu kullanır. Yol son eki olmadan temel Ollama URL'sini kullanın.
</Warning>

### Model seçimi

Yapılandırıldıktan sonra tüm Ollama modelleriniz kullanılabilir olur:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## Bulut modelleri

Bulut modelleri, bulutta barındırılan modelleri (örneğin `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`) yerel modellerinizle birlikte çalıştırmanıza olanak tanır.

Bulut modellerini kullanmak için kurulum sırasında **Cloud + Local** modunu seçin. Sihirbaz, oturum açıp açmadığınızı denetler ve gerektiğinde tarayıcıda bir oturum açma akışı başlatır. Kimlik doğrulama doğrulanamazsa sihirbaz yerel model varsayılanlarına geri döner.

Doğrudan [ollama.com/signin](https://ollama.com/signin) adresinden de oturum açabilirsiniz.

## Ollama Web Search

OpenClaw ayrıca paketlenmiş bir `web_search`
sağlayıcısı olarak **Ollama Web Search** desteği sunar.

- Yapılandırılmış Ollama ana makinenizi kullanır (`models.providers.ollama.baseUrl` ayarlıysa onu, aksi takdirde `http://127.0.0.1:11434`).
- Anahtar gerektirmez.
- Ollama'nın çalışıyor olmasını ve `ollama signin` ile oturum açılmış olmasını gerektirir.

`openclaw onboard` veya
`openclaw configure --section web` sırasında **Ollama Web Search** seçin ya da şunu ayarlayın:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Tam kurulum ve davranış ayrıntıları için bkz. [Ollama Web Search](/tr/tools/ollama-search).

## Gelişmiş

### Reasoning modelleri

OpenClaw, `deepseek-r1`, `reasoning` veya `think` gibi adlara sahip modelleri varsayılan olarak reasoning destekli kabul eder:

```bash
ollama pull deepseek-r1:32b
```

### Model maliyetleri

Ollama ücretsizdir ve yerel olarak çalışır, bu nedenle tüm model maliyetleri $0 olarak ayarlanır.

### Akış yapılandırması

OpenClaw'ın Ollama entegrasyonu varsayılan olarak **yerel Ollama API**'sini (`/api/chat`) kullanır; bu API akış ve araç çağırmayı aynı anda tam olarak destekler. Özel bir yapılandırma gerekmez.

#### Eski OpenAI uyumlu mod

<Warning>
**Araç çağırma, OpenAI uyumlu modda güvenilir değildir.** Bu modu yalnızca bir proxy için OpenAI biçimine ihtiyacınız varsa ve yerel araç çağırma davranışına bağımlı değilseniz kullanın.
</Warning>

Bunun yerine OpenAI uyumlu uç noktayı kullanmanız gerekiyorsa (ör. yalnızca OpenAI biçimini destekleyen bir proxy arkasında), `api: "openai-completions"` değerini açıkça ayarlayın:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: true, // varsayılan: true
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

Bu mod, akış + araç çağırmayı aynı anda desteklemeyebilir. Model yapılandırmasında `params: { streaming: false }` ile akışı devre dışı bırakmanız gerekebilir.

`api: "openai-completions"` Ollama ile kullanıldığında, OpenClaw varsayılan olarak `options.num_ctx` ekler; böylece Ollama sessizce 4096 bağlam penceresine geri düşmez. Proxy/yukarı akış bilinmeyen `options` alanlarını reddediyorsa bu davranışı devre dışı bırakın:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: false,
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

### Bağlam pencereleri

Otomatik olarak keşfedilen modeller için OpenClaw, kullanılabiliyorsa Ollama tarafından bildirilen bağlam penceresini kullanır; aksi takdirde OpenClaw tarafından kullanılan varsayılan Ollama bağlam penceresine geri döner. Açık sağlayıcı yapılandırmasında `contextWindow` ve `maxTokens` değerlerini geçersiz kılabilirsiniz.

## Sorun giderme

### Ollama algılanmadı

Ollama'nın çalıştığından, `OLLAMA_API_KEY` (veya bir kimlik doğrulama profili) ayarladığınızdan ve açık bir `models.providers.ollama` girdisi tanımlamadığınızdan emin olun:

```bash
ollama serve
```

Ve API'nin erişilebilir olduğunu doğrulayın:

```bash
curl http://localhost:11434/api/tags
```

### Kullanılabilir model yok

Modeliniz listelenmiyorsa şu iki durumdan biri söz konusudur:

- Modeli yerel olarak çekin veya
- Modeli `models.providers.ollama` içinde açıkça tanımlayın.

Model eklemek için:

```bash
ollama list  # Kurulu olanları görün
ollama pull gemma4
ollama pull gpt-oss:20b
ollama pull llama3.3     # Veya başka bir model
```

### Bağlantı reddedildi

Ollama'nın doğru bağlantı noktasında çalıştığını denetleyin:

```bash
# Ollama'nın çalışıp çalışmadığını denetleyin
ps aux | grep ollama

# Ya da Ollama'yı yeniden başlatın
ollama serve
```

## Ayrıca bakın

- [Model Providers](/tr/concepts/model-providers) - Tüm sağlayıcılara genel bakış
- [Model Selection](/tr/concepts/models) - Modeller nasıl seçilir
- [Configuration](/tr/gateway/configuration) - Tam yapılandırma başvurusu
