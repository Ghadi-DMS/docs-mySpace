---
read_when:
    - Appairage ou reconnexion du nœud iOS
    - Exécution de l'app iOS depuis le code source
    - Débogage de la découverte de gateway ou des commandes canvas
summary: 'App iOS de nœud : connexion à la gateway, appairage, canvas et dépannage'
title: App iOS
x-i18n:
    generated_at: "2026-04-07T06:51:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: f3e0a6e33e72d4c9f1f17ef70a1b67bae9ebe4a2dca16677ea6b28d0ddac1b4e
    source_path: platforms/ios.md
    workflow: 15
---

# App iOS (nœud)

Disponibilité : préversion interne. L'app iOS n'est pas encore distribuée publiquement.

## Ce qu'elle fait

- Se connecte à une gateway via WebSocket (LAN ou tailnet).
- Expose des capacités de nœud : Canvas, instantané d'écran, capture caméra, localisation, mode Talk, Voice wake.
- Reçoit les commandes `node.invoke` et signale les événements d'état du nœud.

## Exigences

- Gateway en cours d'exécution sur un autre appareil (macOS, Linux ou Windows via WSL2).
- Chemin réseau :
  - Même LAN via Bonjour, **ou**
  - Tailnet via DNS-SD unicast (domaine d'exemple : `openclaw.internal.`), **ou**
  - Hôte/port manuel (repli).

## Démarrage rapide (appairer + connecter)

1. Démarrez la gateway :

```bash
openclaw gateway --port 18789
```

2. Dans l'app iOS, ouvrez Réglages et choisissez une gateway découverte (ou activez Manual Host et saisissez l'hôte/le port).

3. Approuvez la demande d'appairage sur l'hôte gateway :

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Si l'app réessaie l'appairage avec des détails d'authentification modifiés (rôle/scopes/clé publique),
la demande en attente précédente est remplacée et un nouveau `requestId` est créé.
Exécutez `openclaw devices list` à nouveau avant l'approbation.

4. Vérifiez la connexion :

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## Push adossé à un relais pour les builds officiels

Les builds iOS officiels distribués utilisent le relais push externe au lieu de publier le jeton APNs brut
vers la gateway.

Exigence côté gateway :

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

Fonctionnement du flux :

- L'app iOS s'enregistre auprès du relais à l'aide d'App Attest et du reçu de l'app.
- Le relais renvoie un handle de relais opaque ainsi qu'un droit d'envoi à portée d'enregistrement.
- L'app iOS récupère l'identité de la gateway appairée et l'inclut dans l'enregistrement auprès du relais, afin que l'enregistrement adossé au relais soit délégué à cette gateway spécifique.
- L'app transmet cet enregistrement adossé au relais à la gateway appairée avec `push.apns.register`.
- La gateway utilise ce handle de relais stocké pour `push.test`, les réveils en arrière-plan et les nudges de réveil.
- L'URL de base du relais de la gateway doit correspondre à l'URL du relais intégrée au build iOS officiel/TestFlight.
- Si l'app se connecte ensuite à une autre gateway ou à un build avec une autre URL de base de relais, elle actualise l'enregistrement du relais au lieu de réutiliser l'ancienne liaison.

Ce dont la gateway **n'a pas** besoin pour ce chemin :

- Aucun jeton de relais à l'échelle du déploiement.
- Aucune clé APNs directe pour les envois officiels/TestFlight adossés au relais.

Flux opérateur attendu :

1. Installez le build iOS officiel/TestFlight.
2. Définissez `gateway.push.apns.relay.baseUrl` sur la gateway.
3. Appairez l'app avec la gateway et laissez-la terminer la connexion.
4. L'app publie automatiquement `push.apns.register` une fois qu'elle dispose d'un jeton APNs, que la session opérateur est connectée et que l'enregistrement auprès du relais réussit.
5. Après cela, `push.test`, les réveils de reconnexion et les nudges de réveil peuvent utiliser l'enregistrement adossé au relais stocké.

Note de compatibilité :

- `OPENCLAW_APNS_RELAY_BASE_URL` fonctionne toujours comme remplacement temporaire par variable d'environnement pour la gateway.

## Flux d'authentification et de confiance

Le relais existe pour imposer deux contraintes qu'un APNs direct sur la gateway ne peut pas fournir pour
les builds iOS officiels :

- Seuls les builds iOS OpenClaw authentiques distribués via Apple peuvent utiliser le relais hébergé.
- Une gateway ne peut envoyer des pushes adossés au relais qu'aux appareils iOS appairés avec cette
  gateway spécifique.

Saut par saut :

1. `iOS app -> gateway`
   - L'app s'appaire d'abord avec la gateway via le flux normal d'authentification Gateway.
   - Cela donne à l'app une session de nœud authentifiée ainsi qu'une session opérateur authentifiée.
   - La session opérateur est utilisée pour appeler `gateway.identity.get`.

2. `iOS app -> relay`
   - L'app appelle les points de terminaison d'enregistrement du relais via HTTPS.
   - L'enregistrement inclut la preuve App Attest ainsi que le reçu de l'app.
   - Le relais valide le bundle ID, la preuve App Attest et le reçu Apple, et exige le
     chemin de distribution officiel/production.
   - C'est ce qui empêche les builds Xcode/dev locaux d'utiliser le relais hébergé. Un build local peut être
     signé, mais il ne satisfait pas à la preuve de distribution Apple officielle attendue par le relais.

3. `gateway identity delegation`
   - Avant l'enregistrement auprès du relais, l'app récupère l'identité de la gateway appairée depuis
     `gateway.identity.get`.
   - L'app inclut cette identité de gateway dans la charge utile d'enregistrement du relais.
   - Le relais renvoie un handle de relais et un droit d'envoi à portée d'enregistrement qui sont délégués à
     cette identité de gateway.

4. `gateway -> relay`
   - La gateway stocke le handle de relais et le droit d'envoi issus de `push.apns.register`.
   - Lors de `push.test`, des réveils de reconnexion et des nudges de réveil, la gateway signe la requête d'envoi avec sa
     propre identité d'appareil.
   - Le relais vérifie à la fois le droit d'envoi stocké et la signature de la gateway par rapport à l'identité de gateway déléguée
     issue de l'enregistrement.
   - Une autre gateway ne peut pas réutiliser cet enregistrement stocké, même si elle obtenait le handle d'une manière ou d'une autre.

5. `relay -> APNs`
   - Le relais possède les identifiants APNs de production et le jeton APNs brut pour le build officiel.
   - La gateway ne stocke jamais le jeton APNs brut pour les builds officiels adossés au relais.
   - Le relais envoie le push final à APNs au nom de la gateway appairée.

Pourquoi cette conception a été créée :

- Pour garder les identifiants APNs de production hors des gateways des utilisateurs.
- Pour éviter de stocker les jetons APNs bruts des builds officiels sur la gateway.
- Pour autoriser l'usage du relais hébergé uniquement pour les builds OpenClaw officiels/TestFlight.
- Pour empêcher une gateway d'envoyer des pushes de réveil à des appareils iOS appartenant à une autre gateway.

Les builds locaux/manuels restent sur APNs direct. Si vous testez ces builds sans le relais, la
gateway a toujours besoin d'identifiants APNs directs :

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

Il s'agit de variables d'environnement d'exécution sur l'hôte gateway, pas de paramètres Fastlane. `apps/ios/fastlane/.env` stocke uniquement
l'authentification App Store Connect / TestFlight comme `ASC_KEY_ID` et `ASC_ISSUER_ID` ; il ne configure pas
la livraison APNs directe pour les builds iOS locaux.

Stockage recommandé sur l'hôte gateway :

```bash
mkdir -p ~/.openclaw/credentials/apns
chmod 700 ~/.openclaw/credentials/apns
mv /path/to/AuthKey_KEYID.p8 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
chmod 600 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
export OPENCLAW_APNS_PRIVATE_KEY_PATH="$HOME/.openclaw/credentials/apns/AuthKey_KEYID.p8"
```

Ne validez pas le fichier `.p8` et ne le placez pas dans l'extraction du dépôt.

## Chemins de découverte

### Bonjour (LAN)

L'app iOS parcourt `_openclaw-gw._tcp` sur `local.` et, lorsqu'il est configuré, le même
domaine de découverte DNS-SD wide-area. Les gateways du même LAN apparaissent automatiquement depuis `local.` ;
la découverte inter-réseau peut utiliser le domaine wide-area configuré sans changer le type de balise.

### Tailnet (inter-réseau)

Si mDNS est bloqué, utilisez une zone DNS-SD unicast (choisissez un domaine ; exemple :
`openclaw.internal.`) et le split DNS de Tailscale.
Voir [Bonjour](/fr/gateway/bonjour) pour l'exemple CoreDNS.

### Hôte/port manuel

Dans Réglages, activez **Manual Host** et saisissez l'hôte gateway + le port (par défaut `18789`).

## Canvas + A2UI

Le nœud iOS rend un canvas WKWebView. Utilisez `node.invoke` pour le piloter :

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

Remarques :

- L'hôte canvas de la gateway sert `/__openclaw__/canvas/` et `/__openclaw__/a2ui/`.
- Il est servi depuis le serveur HTTP de la gateway (même port que `gateway.port`, par défaut `18789`).
- Le nœud iOS navigue automatiquement vers A2UI à la connexion lorsqu'une URL d'hôte canvas est annoncée.
- Revenez au scaffold intégré avec `canvas.navigate` et `{"url":""}`.

### Évaluation / instantané canvas

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## Voice wake + mode Talk

- Voice wake et le mode Talk sont disponibles dans Réglages.
- iOS peut suspendre l'audio en arrière-plan ; considérez les fonctionnalités vocales comme fournies au mieux lorsque l'app n'est pas active.

## Erreurs courantes

- `NODE_BACKGROUND_UNAVAILABLE` : mettez l'app iOS au premier plan (les commandes canvas/camera/screen l'exigent).
- `A2UI_HOST_NOT_CONFIGURED` : la gateway n'a pas annoncé d'URL d'hôte canvas ; vérifiez `canvasHost` dans la [configuration Gateway](/fr/gateway/configuration).
- L'invite d'appairage n'apparaît jamais : exécutez `openclaw devices list` et approuvez manuellement.
- La reconnexion échoue après réinstallation : le jeton d'appairage du Trousseau a été effacé ; réappairez le nœud.

## Documentation associée

- [Appairage](/fr/channels/pairing)
- [Découverte](/fr/gateway/discovery)
- [Bonjour](/fr/gateway/bonjour)
