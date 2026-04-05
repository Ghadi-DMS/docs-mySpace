---
x-i18n:
    generated_at: "2026-04-05T17:46:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6e1cf417b0c04d001bc494fbe03ac2fcb66866f759e21646dbfd1a9c3a968bff
    source_path: .i18n/README.md
    workflow: 15
---

# OpenClaw 文档 i18n 资源

此文件夹存储源文档仓库的翻译配置。

生成的语言页面和实时语言翻译记忆现在位于发布仓库（`openclaw/docs`，本地同级检出目录为 `~/Projects/openclaw-docs`）中。

## 文件

- `glossary.<lang>.json` — 首选术语映射（用于提示词引导）。
- `<lang>.tm.jsonl` — 以工作流 + 模型 + 文本哈希为键的翻译记忆（缓存）。在此仓库中，语言 TM 文件会按需生成。

## 词汇表格式

`glossary.<lang>.json` 是一个条目数组：

```json
{
  "source": "troubleshooting",
  "target": "故障排除",
  "ignore_case": true,
  "whole_word": false
}
```

字段：

- `source`: 优先使用的英文（或源语言）短语。
- `target`: 首选翻译输出。

## 说明

- 词汇表条目会作为模型的**提示词引导**传递（不会进行确定性重写）。
- `scripts/docs-i18n` 仍负责翻译生成。
- 源仓库会将英文文档同步到发布仓库；语言生成会在该仓库中按语言在推送、计划任务和发布分发时运行。
