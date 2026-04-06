---
read_when:
    - 모델 provider를 선택하려고 합니다
    - 지원되는 LLM 백엔드에 대한 빠른 개요가 필요합니다
summary: OpenClaw가 지원하는 모델 provider(LLM)
title: Provider 디렉터리
x-i18n:
    generated_at: "2026-04-06T03:11:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7271157a6ab5418672baff62bfd299572fd010f75aad529267095c6e55903882
    source_path: providers/index.md
    workflow: 15
---

# 모델 Providers

OpenClaw는 많은 LLM provider를 사용할 수 있습니다. provider를 선택하고, 인증한 다음
기본 모델을 `provider/model`로 설정하세요.

채팅 채널 문서(WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/기타)를 찾고 있나요? [Channels](/ko/channels)를 참조하세요.

## 빠른 시작

1. provider로 인증합니다(대개 `openclaw onboard`를 통해).
2. 기본 모델을 설정합니다:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Provider 문서

- [Alibaba Model Studio](/providers/alibaba)
- [Amazon Bedrock](/ko/providers/bedrock)
- [Anthropic (API + Claude CLI)](/ko/providers/anthropic)
- [BytePlus (International)](/ko/concepts/model-providers#byteplus-international)
- [Chutes](/ko/providers/chutes)
- [ComfyUI](/providers/comfy)
- [Cloudflare AI Gateway](/ko/providers/cloudflare-ai-gateway)
- [DeepSeek](/ko/providers/deepseek)
- [fal](/providers/fal)
- [Fireworks](/ko/providers/fireworks)
- [GitHub Copilot](/ko/providers/github-copilot)
- [GLM models](/ko/providers/glm)
- [Google (Gemini)](/ko/providers/google)
- [Groq (LPU inference)](/ko/providers/groq)
- [Hugging Face (Inference)](/ko/providers/huggingface)
- [Kilocode](/ko/providers/kilocode)
- [LiteLLM (통합 gateway)](/ko/providers/litellm)
- [MiniMax](/ko/providers/minimax)
- [Mistral](/ko/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/ko/providers/moonshot)
- [NVIDIA](/ko/providers/nvidia)
- [Ollama (클라우드 + 로컬 모델)](/ko/providers/ollama)
- [OpenAI (API + Codex)](/ko/providers/openai)
- [OpenCode](/ko/providers/opencode)
- [OpenCode Go](/ko/providers/opencode-go)
- [OpenRouter](/ko/providers/openrouter)
- [Perplexity (웹 검색)](/ko/providers/perplexity-provider)
- [Qianfan](/ko/providers/qianfan)
- [Qwen Cloud](/ko/providers/qwen)
- [Runway](/providers/runway)
- [SGLang (로컬 모델)](/ko/providers/sglang)
- [StepFun](/ko/providers/stepfun)
- [Synthetic](/ko/providers/synthetic)
- [Together AI](/ko/providers/together)
- [Venice (Venice AI, 개인정보 보호 중심)](/ko/providers/venice)
- [Vercel AI Gateway](/ko/providers/vercel-ai-gateway)
- [Vydra](/providers/vydra)
- [vLLM (로컬 모델)](/ko/providers/vllm)
- [Volcengine (Doubao)](/ko/providers/volcengine)
- [xAI](/ko/providers/xai)
- [Xiaomi](/ko/providers/xiaomi)
- [Z.AI](/ko/providers/zai)

## 공통 개요 페이지

- [Additional bundled variants](/ko/providers/models#additional-bundled-provider-variants) - Anthropic Vertex, Copilot Proxy, 그리고 Gemini CLI OAuth
- [Image Generation](/ko/tools/image-generation) - 공통 `image_generate` 도구, provider 선택, 그리고 failover
- [Music Generation](/tools/music-generation) - 공통 `music_generate` 도구, provider 선택, 그리고 failover
- [Video Generation](/tools/video-generation) - 공통 `video_generate` 도구, provider 선택, 그리고 failover

## 전사 provider

- [Deepgram (오디오 전사)](/ko/providers/deepgram)

## 커뮤니티 도구

- [Claude Max API Proxy](/ko/providers/claude-max-api-proxy) - Claude 구독 자격 증명을 위한 커뮤니티 프록시(사용 전에 Anthropic 정책/약관을 확인하세요)

전체 provider 카탈로그(xAI, Groq, Mistral 등)와 고급 구성을 보려면
[Model providers](/ko/concepts/model-providers)를 참조하세요.
