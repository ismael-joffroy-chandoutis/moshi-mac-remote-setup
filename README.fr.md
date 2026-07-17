[English](README.md) · **Français**

# Piloter son Mac depuis son iPhone avec Moshi : la configuration complète (et les pièges que personne ne documente)

Un guide de terrain pour faire fonctionner [Moshi](https://getmoshi.app) (le compagnon terminal + agent IA sur iOS) avec un Mac, de bout en bout :

- une **boîte de réception d'agent** sur votre iPhone et votre Apple Watch (vos sessions Claude Code / Codex, notifications, fins de tour)
- des **noms de session conviviaux** dans Moshi qui correspondent à votre multiplexeur local (cmux), au lieu de `ttys000`
- un **vrai terminal depuis n'importe où** (5G, n'importe quel Wi-Fi) via Tailscale, pour pouvoir taper dans une session active quand vous êtes loin de chez vous

La plupart de ces éléments sont déjà documentés quelque part. Les trois choses qui m'ont coûté des heures, et que vous ne trouverez détaillées nulle part ailleurs, sont dans les sections **Pièges**. Si vous ne devez lire qu'une chose, lisez celles-là.

> Espaces réservés : remplacez `100.x.y.z` par l'adresse IP Tailscale de votre Mac, `youruser` par votre nom d'utilisateur macOS court, et `you@example.com` par votre identifiant Tailscale.

**Guide complémentaire :** pour la vue d'ensemble, comment Moshi se compare au **Remote Control** natif (application Claude / application ChatGPT pour Codex) et aux applications mobiles par outil, quand utiliser lequel, la vérité sur la persistance, et les schémas multi-machines / de parité, voir **[Piloter des agents de code depuis son téléphone](docs/driving-coding-agents-remote.md)** (en anglais).

---

## Ce que vous obtenez

| Capacité | Transport | Fonctionne loin de chez vous ? |
|---|---|---|
| Boîte de réception d'agent, notifications, Apple Watch | démon moshi-hook -> cloud Moshi | Oui (cloud, pas de VPN nécessaire) |
| Noms de session conviviaux | script local + launchd | n/a (local) |
| Taper dans une session active (« Open in Terminal ») | SSH/Mosh via Tailscale | Oui, une fois Tailscale bien configuré |

La première ligne est l'idée essentielle : **la boîte de réception d'agent n'a besoin d'aucun VPN ni port ouvert.** Le démon établit une connexion WebSocket sortante vers le cloud de Moshi, donc les notifications et le triage des sessions fonctionnent en 4G/5G sans rien configurer de plus. Seul le *terminal* a besoin d'une accessibilité réseau vers le Mac.

---

## Prérequis

- macOS avec Homebrew
- [Moshi](https://apps.apple.com/app/id6757859949) sur iPhone (et l'application watchOS, optionnelle)
- `tmux` (ou Zellij/Herdr) pour les sessions terminal
- Optionnel : [cmux](https://github.com/) si vous voulez le pont noms-conviviaux

---

## Partie 1 : les hooks d'agent (boîte de réception, notifications, Apple Watch)

C'est le démon `moshi-hook`. Il installe des hooks dans vos CLI d'agents et transmet les événements de cycle de vie (début/fin de session, fin de tour, demandes de permission, questions) à l'application Moshi.

```bash
brew tap rjyo/moshi
brew install moshi-hook

# Réglages -> Hooks dans l'application Moshi affiche un jeton d'appairage :
moshi-hook pair --token <token-from-the-app>
moshi-hook install            # écrit la config des hooks dans ~/.claude/settings.json, ~/.codex/hooks.json, etc.
brew services start moshi-hook
moshi-hook status             # doit afficher : status: paired
```

Vérifiez que le démon voit bien vos sessions :

```bash
moshi-hook cwd-list           # liste les répertoires de travail d'agents que le démon a enregistrés
```

Si vos projets apparaissent ici, la partie côté hôte est terminée. Les sessions apparaissent dans l'onglet **Agents** de l'application (pas dans l'écran Hooks/réglages).

### Piège n°1 : hooks obsolètes après une mise à jour

Si votre boîte de réception est vide alors que le démon est bien `paired` et en cours d'exécution, votre `settings.json` peut contenir des hooks **obsolètes** (écrits par une version plus ancienne). Le démon le signale dans son journal :

```
WARN "agent hooks missing or stale; rerun install" command="moshi-hook install --target claude"
```

Correction :

```bash
moshi-hook install            # réécrit le jeu de hooks actuel, retire les événements retirés
```

C'est la seule chose qui a fait revivre une boîte de réception vide, dans mon cas. Le journal du démon se trouve dans `~/Library/Application Support/Moshi/hook.log`, mais notez qu'il ne journalise que le sondeur d'usage, **pas** les appels de hooks individuels : un journal vide ne signifie donc pas que les hooks échouent. Le signal fiable, c'est l'application, ou `moshi-hook cwd-list`.

### Remarque : `--dangerously-skip-permissions` supprime les cartes Allow/Deny

Si vous lancez votre agent avec le contournement des permissions, l'agent ne demande jamais de permission, donc l'événement **PermissionRequest** ne se déclenche jamais et vous n'aurez jamais de cartes Allow/Deny dans l'application. Vous recevez quand même les notifications de fin de tour et les cartes de question. C'est une conséquence de votre propre option, pas un bug.

---

## Partie 2 : noms de session conviviaux (cmux -> Moshi)

Si vous encapsulez vos sessions dans [cmux](https://github.com/), Moshi les affiche comme `ttys000`, `ttys001`, ... parce que **cmux ne propage jamais ses titres d'espace de travail conviviaux vers les noms de session tmux sous-jacents.**

Le pont : lire les titres de cmux depuis son RPC, faire correspondre chaque processus `tmux new-session -s ttysNNN` à son `CMUX_WORKSPACE_ID`, et faire un `tmux rename-session` vers le titre convivial. Une tâche launchd réapplique cela toutes les quelques minutes pour que les nouveaux espaces de travail soient aussi nommés.

- `scripts/moshi-name-sessions.sh` fait la correspondance et le renommage (idempotent : il ne touche que les sessions encore nommées `ttysNNN`).
- `scripts/com.example.moshi-name-sessions.plist` l'exécute toutes les 120 secondes.

```bash
cp scripts/moshi-name-sessions.sh ~/.claude/scripts/
chmod +x ~/.claude/scripts/moshi-name-sessions.sh

# éditez le chemin du plist pour votre utilisateur, puis :
cp scripts/com.example.moshi-name-sessions.plist ~/Library/LaunchAgents/com.you.moshi-name-sessions.plist
launchctl load ~/Library/LaunchAgents/com.you.moshi-name-sessions.plist
```

C'est non destructif : renommer une session tmux est un changement d'étiquette, cela ne touche pas au processus en cours d'exécution. Tant que cmux est attaché, le renommage tient (cmux suit les panes, pas les noms de session). Pour revenir en arrière, déchargez la tâche launchd.

---

## Partie 3 : un terminal depuis n'importe où (Tailscale)

Pour que « Open in Terminal » fonctionne quand vous êtes en 5G, Moshi se connecte via SSH/Mosh, et le Mac doit être accessible. Le bon outil est **Tailscale**, pas Cloudflare (le Cloudflare Tunnel pour SSH a besoin d'un proxy côté client qu'une application terminal iOS n'a pas).

Côté Mac :

```bash
sudo systemsetup -setremotelogin on          # ou : moshi-hook host enable-ssh
brew install tailscale && sudo tailscale up   # (ou l'application graphique)
tailscale ip -4                                # l'IP 100.x.y.z de votre Mac
```

Dans Moshi, créez une connexion enregistrée (ou utilisez `moshi-hook host setup` pour l'appairage facile) :

- **Host : l'IP Tailscale `100.x.y.z`** (PAS le nom Bonjour en `.local`, qui ne se résout que sur le réseau local domestique)
- Utilisateur : `youruser`, Port : 22, type : Auto

### Piège n°2 (le gros) : une ACL de tailnet stricte fait silencieusement échouer le téléphone

Symptôme : l'application Tailscale dit **Connected**, `tailscale ping` depuis le Mac reçoit un pong, mais le téléphone ne peut absolument pas joindre le Mac. SSH expire (`os error 60`), tout comme le simple HTTP dans Safari. On dirait un bug de routage ou d'iOS. Ce n'en est pas un.

Si votre tailnet utilise des [ACL](https://tailscale.com/kb/1018/acls) (fréquent dès que vous avez Tailnet Lock ou une politique personnalisée), le Mac **rejette chaque paquet provenant d'un appareil qu'aucune règle n'autorise.** `tailscale ping` fonctionne parce que le ping disco de bas niveau n'est pas soumis au filtre de paquets de données, ce qui explique précisément la confusion.

Diagnostiquez-le depuis le Mac. Ceci affiche le filtre de paquets réellement reçu par le Mac, c'est-à-dire quelles IP source sont autorisées à le joindre :

```bash
tailscale debug netmap | python3 -c "import json,sys; d=json.load(sys.stdin); [print(r.get('Srcs')) for r in d.get('PacketFilter',[])]"
```

Si l'IP `100.x` de votre téléphone n'apparaît dans **aucune** liste `Srcs`, voilà votre réponse.

Correction : faites correspondre le téléphone à une règle. Une ACL propre à base de tags ressemble à ceci, où les appareils mobiles peuvent joindre les appareils de confiance sur les ports qui vous intéressent :

```json
{
  "tagOwners": { "tag:trusted": ["autogroup:admin"], "tag:mobile": ["autogroup:admin"] },
  "grants": [
    { "src": ["tag:trusted"], "dst": ["*"], "ip": ["*"] },
    { "src": ["tag:mobile"], "dst": ["tag:trusted"],
      "ip": ["tcp:22", "tcp:80", "tcp:443", "tcp:5900", "tcp:8080", "udp:60000-61000"] }
  ]
}
```

Ensuite **taguez le téléphone** pour qu'il tombe effectivement sous `tag:mobile` : console admin -> Machines -> votre téléphone -> Edit ACL tags -> ajoutez `tag:mobile`. Dès que vous enregistrez, l'IP du téléphone apparaît dans le filtre `netmap`, et SSH/Moshi se connecte.

Incluez `udp:60000-61000` pour que Moshi puisse utiliser **Mosh**, qui survit aux changements d'adresse constants d'une connexion cellulaire. SSH seul (TCP) fonctionnera mais peut se bloquer quand votre adresse 5G se réattribue.

### Piège n°3 : réinstaller Tailscale sur iOS verrouille l'appareil (Tailnet Lock)

Si votre tailnet a [Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock) activé et que vous supprimez puis réinstallez l'application Tailscale, le téléphone génère une **nouvelle clé de nœud** qui n'est pas signée. Vous obtenez « Admin signing required », et encore une fois : Connected, mais aucun trafic. Ne partez pas à la chasse à un bug iOS.

Signez-la depuis un nœud de signature de confiance (par exemple votre Mac, si c'en est un) :

```bash
tailscale lock status                                  # liste les nœuds verrouillés + leur nodekey, et si CE nœud peut signer
tailscale lock sign nodekey:<the-locked-out-key>
```

Leçon : ne réinstallez pas l'application iOS de Tailscale sauf nécessité. Si vous le faites, signez à nouveau depuis le Mac.

### Garder le Mac éveillé

Si le Mac s'endort chez vous, il est injoignable, aussi parfait que soit le reste.

```bash
sudo pmset -c sleep 0 displaysleep 10     # ne jamais dormir sur secteur ; l'écran peut quand même s'assombrir
```

Gardez-le branché avec le capot ouvert. Un Mac fermé (mode clamshell) sans écran externe s'endort quand même, sauf si vous réglez `pmset -c disablesleep 1`, ce qui désactive aussi la mise en veille sur batterie et peut surchauffer dans un sac : à utiliser délibérément.

---

## Antisèche diagnostic rapide

```bash
# le démon est-il sain et voit-il les sessions ?
moshi-hook status
moshi-hook cwd-list

# SSH est-il vraiment actif ? (lsof sans root ne voit pas le socket sshd appartenant à root : faux négatif)
nc -vz 127.0.0.1 22

# le Mac est-il joignable via Tailscale, et comment (direct vs relais DERP) ?
tailscale ping <phone-name>
tailscale netcheck

# le pare-feu macOS bloque-t-il l'entrant ?
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# celui qui tranche : le filtre ACL autorise-t-il mon téléphone à joindre ce Mac ?
tailscale debug netmap | python3 -c "import json,sys; d=json.load(sys.stdin); [print(r.get('Srcs')) for r in d.get('PacketFilter',[])]"
```

---

## Licence

- Documentation (texte) : [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)
- Code (scripts) : [PolyForm Noncommercial 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0/)

Non affilié à Moshi ni à Tailscale. Les noms d'outils appartiennent à leurs propriétaires respectifs.

Par [Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com).
