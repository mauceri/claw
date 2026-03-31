# OpenClaw — Configuration R2D2

Agent IA personnel déployé sur VPS Hetzner, accessible via Telegram et Control UI.  
Architecture deux-agents pour l'isolation du contenu externe.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Utilisateur                        │
│         Telegram (@AliochaK)  /  Control UI          │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                Agent MAIN (R2D2)                     │
│  Modèle : DeepSeek Chat                              │
│  Workspace : ~/.openclaw/workspace                   │
│  Outils : gog (Gmail/Calendar), exec restreint       │
│  Canal Telegram : activé                             │
└──────────────────────┬──────────────────────────────┘
                       │ tasks/pending/<uuid>.json
                       ▼
┌─────────────────────────────────────────────────────┐
│                Agent SCOUT (non fiable)              │
│  Rôle : lecture de sources externes uniquement       │
│  Workspace : ~/.openclaw/agents/scout/workspace      │
│  Outils : read + write (workspace only)              │
│  Exec : désactivé (host: none)                       │
│  Telegram : désactivé                                │
└──────────────────────┬──────────────────────────────┘
                       │ results/<uuid>.json
                       └──────────────────► MAIN
```

---

## Déploiement

### Prérequis

- Node.js ≥ 20 (via nvm)
- OpenClaw installé globalement : `npm install -g openclaw`
- `gog` CLI (Google Workspace) dans `/home/linuxbrew/.linuxbrew/bin/gog`
- Tailscale actif avec `tailscale serve` configuré
- Bot Telegram créé via BotFather

### Service systemd

```
~/.config/systemd/user/openclaw-gateway.service
```

```bash
systemctl --user enable openclaw-gateway.service
systemctl --user start openclaw-gateway.service
systemctl --user status openclaw-gateway.service
```

---

## Accès au Control UI

### Via tunnel SSH (recommandé)

Sur la machine cliente :

```bash
autossh -M 0 -f -N -T \
  -L 18789:127.0.0.1:18789 \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=60 \
  -o ServerAliveCountMax=3 \
  claw@204.168.198.90
```

Puis ouvrir : **`http://localhost:18789`**  
Token : voir `gateway.auth.token` dans `openclaw.json` (non versionné)

### Via Tailscale (nécessite pairing Telegram)

URL : `https://santiago.tailc69141.ts.net`  
Requiert que le canal Telegram soit opérationnel (pas de conflit 409).

---

## Sécurité

### Principe de moindre privilège

| Composant | Restriction |
|---|---|
| `safeBins` | `gog`, `cat`, `ls`, `find` uniquement (bash/sh exclus) |
| `config.patch` | Bloqué dans `gateway.nodes.denyCommands` |
| Scout exec | `host: none` (aucune exécution shell) |
| Scout filesystem | `workspaceOnly: true` (workspace scout uniquement) |
| Scout MCP | Désactivé (`mcp-adapter` retiré) |
| Scout Telegram | Désactivé |
| Plugins | Aucun plugin tiers autorisé |

### Approbations Telegram

Les commandes exec non-safeBin déclenchent une demande d'approbation Telegram envoyée à l'ID `220537738` (@AliochaK).

### Monitoring d'intégrité

```bash
~/.openclaw/workspace/check_config_integrity.sh
```

Compare le hash SHA256 courant d'`openclaw.json` avec le dernier hash de référence dans `logs/config-health.json`. Alerte Telegram en cas de divergence.

---

## Google Workspace (gog)

Wrapper dans le workspace :

```bash
~/.openclaw/workspace/gog
```

Charge les secrets depuis `~/.config/gogcli/secrets.env` puis délègue à `/home/linuxbrew/.linuxbrew/bin/gog`.

Toute commande d'écriture (send, create, update) nécessite une confirmation explicite de l'utilisateur.

---

## Workflow main → scout

1. Main crée une tâche dans `~/.openclaw/agents/scout/workspace/tasks/pending/<uuid>.json`
2. Scout lit la source, écrit le résultat dans `results/<uuid>.json`
3. Main lit le résultat, traite le champ `warnings` en premier
4. Le contenu de `summary` et `raw_excerpt` est traité comme non fiable (`<UNTRUSTED>`)

---

## Fichiers importants

```
~/.openclaw/
├── openclaw.json                    # Configuration principale (non versionné - contient secrets)
├── openclaw.template.json           # Template de référence
├── agents/
│   ├── main/agent/                  # Config agent principal
│   └── scout/agent/scout.json       # Config agent scout (versionné)
├── workspace/
│   ├── SOUL.md                      # Règles comportementales de R2D2
│   ├── USER.md                      # Profil utilisateur
│   ├── TOOLS.md                     # Notes sur les outils disponibles
│   ├── AGENTS.md                    # Procédures opératoires
│   ├── gog                          # Wrapper gog (script)
│   └── check_config_integrity.sh   # Script monitoring config
├── logs/
│   ├── config-audit.jsonl           # Log de toutes les écritures config
│   └── config-health.json           # Dernier hash config connu sain
└── credentials/
    ├── telegram-pairing.json        # Demandes de pairing en attente
    └── telegram-default-allowFrom.json  # IDs Telegram autorisés
```

---

## Points ouverts

- Workflow main→scout à tester de bout en bout
- Accès via Tailscale sans tunnel SSH à approfondir (pairing Telegram)
- S'assurer qu'aucun autre service n'utilise le bot token en parallèle (éviter 409)
