---
read_when:
    - Procurando definições de canais de lançamento públicos
    - Procurando nomenclatura de versões e cadência
summary: Canais de lançamento públicos, nomenclatura de versões e cadência
title: Política de lançamento
x-i18n:
    generated_at: "2026-04-14T05:33:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3eaf9f1786b8c9fd4f5a9c657b623cb69d1a485958e1a9b8f108511839b63587
    source_path: reference/RELEASING.md
    workflow: 15
---

# Política de lançamento

O OpenClaw tem três linhas públicas de lançamento:

- stable: lançamentos com tag que publicam no npm `beta` por padrão, ou no npm `latest` quando solicitado explicitamente
- beta: tags de pré-lançamento que publicam no npm `beta`
- dev: a ponta móvel de `main`

## Nomenclatura de versões

- Versão de lançamento stable: `YYYY.M.D`
  - Tag Git: `vYYYY.M.D`
- Versão de lançamento de correção stable: `YYYY.M.D-N`
  - Tag Git: `vYYYY.M.D-N`
- Versão de pré-lançamento beta: `YYYY.M.D-beta.N`
  - Tag Git: `vYYYY.M.D-beta.N`
- Não use preenchimento com zero para mês ou dia
- `latest` significa o lançamento stable promovido atual do npm
- `beta` significa o destino de instalação beta atual
- Lançamentos stable e de correção stable publicam no npm `beta` por padrão; operadores de lançamento podem direcionar para `latest` explicitamente, ou promover depois uma build beta validada
- Todo lançamento do OpenClaw envia o pacote npm e o app macOS juntos

## Cadência de lançamento

- Os lançamentos passam primeiro por beta
- stable vem somente depois que o beta mais recente é validado
- O procedimento detalhado de lançamento, aprovações, credenciais e notas de recuperação é exclusivo para mantenedores

## Verificações prévias de lançamento

- Execute `pnpm build && pnpm ui:build` antes de `pnpm release:check` para que os artefatos de lançamento esperados em `dist/*` e o bundle da Control UI existam para a etapa de validação do pack
- Execute `pnpm release:check` antes de todo lançamento com tag
- As verificações de lançamento agora são executadas em um workflow manual separado:
  `OpenClaw Release Checks`
- Essa separação é intencional: mantém o caminho real de lançamento no npm curto,
  determinístico e focado em artefatos, enquanto verificações live mais lentas ficam em sua própria linha para não atrasar nem bloquear a publicação
- As verificações de lançamento devem ser disparadas a partir da ref de workflow `main` para que a lógica do workflow e os segredos permaneçam canônicos
- Esse workflow aceita uma tag de lançamento existente ou o SHA completo de 40 caracteres do commit atual de `main`
- No modo de commit SHA, ele aceita apenas o HEAD atual de `origin/main`; use uma tag de lançamento para commits de lançamento mais antigos
- A validação prévia apenas de validação de `OpenClaw NPM Release` também aceita o SHA completo de 40 caracteres do commit atual de `main` sem exigir uma tag enviada
- Esse caminho por SHA é apenas para validação e não pode ser promovido para uma publicação real
- No modo SHA, o workflow sintetiza `v<package.json version>` apenas para a verificação dos metadados do pacote; a publicação real ainda exige uma tag de lançamento real
- Ambos os workflows mantêm o caminho real de publicação e promoção em runners hospedados pelo GitHub, enquanto o caminho de validação não mutável pode usar os runners Linux maiores da Blacksmith
- Esse workflow executa
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  usando os segredos de workflow `OPENAI_API_KEY` e `ANTHROPIC_API_KEY`
- A validação prévia de lançamento no npm não espera mais pela linha separada de verificações de lançamento
- Execute `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  (ou a tag beta/correção correspondente) antes da aprovação
- Depois da publicação no npm, execute
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  (ou a versão beta/correção correspondente) para verificar o caminho de instalação publicado no registro em um prefixo temporário novo
- A automação de lançamento dos mantenedores agora usa pré-verificação seguida de promoção:
  - a publicação real no npm deve passar por um `preflight_run_id` bem-sucedido
  - lançamentos npm stable usam `beta` por padrão
  - a publicação npm stable pode direcionar para `latest` explicitamente por meio da entrada do workflow
  - a promoção npm stable de `beta` para `latest` continua disponível como um modo manual explícito no workflow confiável `OpenClaw NPM Release`
  - publicações stable diretas também podem executar um modo explícito de sincronização de dist-tag que aponta `latest` e `beta` para a versão stable já publicada
  - esses modos de dist-tag ainda precisam de um `NPM_TOKEN` válido no ambiente `npm-release`, porque o gerenciamento de `npm dist-tag` é separado da publicação confiável
  - `macOS Release` público é somente para validação
  - a publicação real privada do mac deve passar por `preflight_run_id` e `validate_run_id` privados do mac bem-sucedidos
  - os caminhos de publicação reais promovem artefatos preparados em vez de reconstruí-los novamente
- Para lançamentos de correção stable como `YYYY.M.D-N`, o verificador pós-publicação também verifica o mesmo caminho de upgrade em prefixo temporário de `YYYY.M.D` para `YYYY.M.D-N`, para que correções de lançamento não possam deixar silenciosamente instalações globais antigas na carga stable base
- A validação prévia de lançamento no npm falha em modo fechado, a menos que o tarball inclua `dist/control-ui/index.html` e uma carga `dist/control-ui/assets/` não vazia, para que não enviemos novamente um painel do navegador vazio
- `pnpm test:install:smoke` também impõe o orçamento de `unpackedSize` do `npm pack` no tarball de atualização candidato, para que o e2e do instalador detecte aumento acidental do pack antes do caminho de publicação de lançamento
- Se o trabalho de lançamento tiver alterado o planejamento de CI, manifestos de tempo de extensões ou matrizes de teste de extensões, regenere e revise as saídas da matriz de workflow `checks-node-extensions` de propriedade do planejador a partir de `.github/workflows/ci.yml` antes da aprovação, para que as notas de lançamento não descrevam um layout de CI desatualizado
- A prontidão de lançamento stable do macOS também inclui as superfícies do atualizador:
  - o lançamento no GitHub deve terminar com os arquivos empacotados `.zip`, `.dmg` e `.dSYM.zip`
  - `appcast.xml` em `main` deve apontar para o novo zip stable após a publicação
  - o app empacotado deve manter um bundle id que não seja de depuração, uma URL de feed Sparkle não vazia e um `CFBundleVersion` igual ou superior ao piso canônico de build do Sparkle para essa versão de lançamento

## Entradas de workflow do npm

`OpenClaw NPM Release` aceita estas entradas controladas pelo operador:

- `tag`: tag de lançamento obrigatória, como `v2026.4.2`, `v2026.4.2-1` ou
  `v2026.4.2-beta.1`; quando `preflight_only=true`, também pode ser o SHA completo de 40 caracteres do commit atual de `main` para uma validação prévia somente de validação
- `preflight_only`: `true` para apenas validação/build/package, `false` para o caminho de publicação real
- `preflight_run_id`: obrigatório no caminho de publicação real para que o workflow reutilize o tarball preparado da execução de pré-verificação bem-sucedida
- `npm_dist_tag`: tag de destino do npm para o caminho de publicação; o padrão é `beta`
- `promote_beta_to_latest`: `true` para pular a publicação e mover uma build stable `beta` já publicada para `latest`
- `sync_stable_dist_tags`: `true` para pular a publicação e apontar `latest` e `beta` para uma versão stable já publicada

`OpenClaw Release Checks` aceita estas entradas controladas pelo operador:

- `ref`: tag de lançamento existente ou o SHA completo de 40 caracteres do commit atual de `main` para validar

Regras:

- Tags stable e de correção podem publicar em `beta` ou `latest`
- Tags de pré-lançamento beta podem publicar somente em `beta`
- A entrada de SHA completo de commit é permitida apenas quando `preflight_only=true`
- O modo por commit SHA das verificações de lançamento também exige o HEAD atual de `origin/main`
- O caminho de publicação real deve usar o mesmo `npm_dist_tag` usado durante a pré-verificação; o workflow verifica esses metadados antes de a publicação continuar
- O modo de promoção deve usar uma tag stable ou de correção, `preflight_only=false`,
  `preflight_run_id` vazio e `npm_dist_tag=beta`
- O modo de sincronização de dist-tag deve usar uma tag stable ou de correção,
  `preflight_only=false`, `preflight_run_id` vazio, `npm_dist_tag=latest`,
  e `promote_beta_to_latest=false`
- Os modos de promoção e sincronização de dist-tag também exigem um `NPM_TOKEN` válido porque `npm dist-tag add` ainda precisa de autenticação npm normal; a publicação confiável cobre apenas o caminho de publicação do pacote

## Sequência de lançamento npm stable

Ao criar um lançamento npm stable:

1. Execute `OpenClaw NPM Release` com `preflight_only=true`
   - Antes de existir uma tag, você pode usar o SHA completo atual de `main` para uma simulação somente de validação do workflow de pré-verificação
2. Escolha `npm_dist_tag=beta` para o fluxo normal beta-first, ou `latest` somente quando quiser intencionalmente uma publicação stable direta
3. Execute `OpenClaw Release Checks` separadamente com a mesma tag ou o SHA completo atual de `main` quando quiser cobertura live de cache de prompt
   - Isso é separado de propósito para que a cobertura live continue disponível sem reacoplar verificações demoradas ou instáveis ao workflow de publicação
4. Salve o `preflight_run_id` bem-sucedido
5. Execute `OpenClaw NPM Release` novamente com `preflight_only=false`, a mesma
   `tag`, o mesmo `npm_dist_tag` e o `preflight_run_id` salvo
6. Se o lançamento chegou a `beta`, execute `OpenClaw NPM Release` mais tarde com a mesma `tag` stable, `promote_beta_to_latest=true`, `preflight_only=false`,
   `preflight_run_id` vazio e `npm_dist_tag=beta` quando quiser mover essa build publicada para `latest`
7. Se o lançamento foi intencionalmente publicado diretamente em `latest` e `beta`
   deve seguir a mesma build stable, execute `OpenClaw NPM Release` com a mesma
   `tag` stable, `sync_stable_dist_tags=true`, `promote_beta_to_latest=false`,
   `preflight_only=false`, `preflight_run_id` vazio e `npm_dist_tag=latest`

Os modos de promoção e sincronização de dist-tag ainda exigem a aprovação do ambiente `npm-release` e um `NPM_TOKEN` válido acessível a essa execução de workflow.

Isso mantém o caminho de publicação direta e o caminho de promoção beta-first ambos documentados e visíveis para o operador.

## Referências públicas

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Os mantenedores usam a documentação privada de lançamento em
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
para o runbook real.
