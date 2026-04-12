---
read_when:
    - Vous voulez utiliser GitHub Copilot comme fournisseur de modèles
    - Vous avez besoin du flux `openclaw models auth login-github-copilot`
summary: Connectez-vous à GitHub Copilot depuis OpenClaw à l’aide du flux d’appareil
title: GitHub Copilot
x-i18n:
    generated_at: "2026-04-12T23:30:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 51fee006e7d4e78e37b0c29356b0090b132de727d99b603441767d3fb642140b
    source_path: providers/github-copilot.md
    workflow: 15
---

# GitHub Copilot

GitHub Copilot est l’assistant de codage IA de GitHub. Il donne accès aux modèles Copilot pour votre compte GitHub et votre forfait. OpenClaw peut utiliser Copilot comme fournisseur de modèles de deux façons différentes.

## Deux façons d’utiliser Copilot dans OpenClaw

<Tabs>
  <Tab title="Fournisseur intégré (github-copilot)">
    Utilisez le flux natif de connexion par appareil pour obtenir un jeton GitHub, puis l’échanger contre des jetons d’API Copilot lorsque OpenClaw s’exécute. C’est le chemin **par défaut** et le plus simple, car il ne nécessite pas VS Code.

    <Steps>
      <Step title="Exécuter la commande de connexion">
        ```bash
        openclaw models auth login-github-copilot
        ```

        Il vous sera demandé de visiter une URL et de saisir un code à usage unique. Gardez le terminal ouvert jusqu’à la fin du processus.
      </Step>
      <Step title="Définir un modèle par défaut">
        ```bash
        openclaw models set github-copilot/gpt-4o
        ```

        Ou dans la configuration :

        ```json5
        {
          agents: { defaults: { model: { primary: "github-copilot/gpt-4o" } } },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Plugin Copilot Proxy (copilot-proxy)">
    Utilisez l’extension VS Code **Copilot Proxy** comme pont local. OpenClaw communique avec le point de terminaison `/v1` du proxy et utilise la liste de modèles que vous y configurez.

    <Note>
    Choisissez cette option si vous utilisez déjà Copilot Proxy dans VS Code ou si vous devez passer par lui. Vous devez activer le Plugin et garder l’extension VS Code en cours d’exécution.
    </Note>

  </Tab>
</Tabs>

## Drapeaux facultatifs

| Drapeau       | Description                                                |
| ------------- | ---------------------------------------------------------- |
| `--yes`       | Ignorer la demande de confirmation                         |
| `--set-default` | Appliquer également le modèle par défaut recommandé du fournisseur |

```bash
# Ignorer la confirmation
openclaw models auth login-github-copilot --yes

# Se connecter et définir le modèle par défaut en une seule étape
openclaw models auth login --provider github-copilot --method device --set-default
```

<AccordionGroup>
  <Accordion title="TTY interactif requis">
    Le flux de connexion par appareil nécessite un TTY interactif. Exécutez-le directement dans un terminal, pas dans un script non interactif ni dans un pipeline CI.
  </Accordion>

  <Accordion title="La disponibilité des modèles dépend de votre forfait">
    La disponibilité des modèles Copilot dépend de votre forfait GitHub. Si un modèle est refusé, essayez un autre ID (par exemple `github-copilot/gpt-4.1`).
  </Accordion>

  <Accordion title="Sélection du transport">
    Les ID de modèles Claude utilisent automatiquement le transport Anthropic Messages. Les modèles GPT, série o et Gemini conservent le transport OpenAI Responses. OpenClaw sélectionne le transport correct en fonction de la référence du modèle.
  </Accordion>

  <Accordion title="Ordre de résolution des variables d’environnement">
    OpenClaw résout l’authentification Copilot à partir des variables d’environnement selon l’ordre de priorité suivant :

    | Priorité | Variable               | Remarques                              |
    | -------- | ---------------------- | -------------------------------------- |
    | 1        | `COPILOT_GITHUB_TOKEN` | Priorité la plus élevée, spécifique à Copilot |
    | 2        | `GH_TOKEN`             | Jeton GitHub CLI (secours)             |
    | 3        | `GITHUB_TOKEN`         | Jeton GitHub standard (priorité la plus basse) |

    Lorsque plusieurs variables sont définies, OpenClaw utilise celle ayant la priorité la plus élevée.
    Le flux de connexion par appareil (`openclaw models auth login-github-copilot`) stocke son jeton dans le magasin de profils d’authentification et a priorité sur toutes les variables d’environnement.

  </Accordion>

  <Accordion title="Stockage du jeton">
    La connexion stocke un jeton GitHub dans le magasin de profils d’authentification et l’échange contre un jeton d’API Copilot lorsque OpenClaw s’exécute. Vous n’avez pas besoin de gérer le jeton manuellement.
  </Accordion>
</AccordionGroup>

<Warning>
Nécessite un TTY interactif. Exécutez la commande de connexion directement dans un terminal, pas dans un script sans interface ni dans une tâche CI.
</Warning>

## Liens associés

<CardGroup cols={2}>
  <Card title="Sélection du modèle" href="/fr/concepts/model-providers" icon="layers">
    Choisir les fournisseurs, les références de modèles et le comportement de basculement.
  </Card>
  <Card title="OAuth et authentification" href="/fr/gateway/authentication" icon="key">
    Détails d’authentification et règles de réutilisation des identifiants.
  </Card>
</CardGroup>
