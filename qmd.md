# 🧠 qmd - Recherche Sémantique Locale pour tes Notes

> **Machine:** KpihX-Ubuntu | **Date:** 2026-03-24

J'ai une grande quantité de notes en Markdown dans `~/Work/tutos_live` et `~/Work/Homelab/presentation`. Utiliser `grep` fonctionne, mais c'est limité à la recherche par mot-clé exact. Je voulais quelque chose de plus intelligent, capable de comprendre l'intention de ma recherche. C'est là que j'ai découvert `qmd`.

`qmd` est un moteur de recherche local qui utilise des modèles d'IA pour "comprendre" le sens de tes notes et te fournir des résultats beaucoup plus pertinents. Voici comment je l'ai installé et configuré pour qu'il devienne une brique essentielle de ma "mémoire externe".

## Installation et Configuration

L'installation se fait via `bun` (mon gestionnaire de paquets JS/TS de choix).

```bash
bun install -g @tobilu/qmd
```

Une fois installé, la première étape est de lui dire quels dossiers indexer. J'ai deux sources de connaissances principales :

1.  `~/Work/tutos_live`
2.  `~/Work/Homelab/presentation`

J'ai donc créé deux "collections" `qmd` :

```bash
# Crée la collection "tutos_live" et spécifie les fichiers à inclure
qmd collection add ~/Work/tutos_live/ -n tutos_live --mask "**/*.{md,yml,yaml,json,py,toml,sh}"

# Crée la collection "presentation" pour la doc du Homelab
qmd collection add ~/Work/Homelab/presentation/ -n presentation --mask "**/*.{md,yml,yaml,json,py,toml,sh}"
```
Le flag `--mask` est crucial pour dire à `qmd` d'indexer non seulement les `.md`, mais aussi les fichiers de configuration et les scripts.

Enfin, il faut lancer l'indexation sémantique. Cette commande va télécharger les modèles d'IA si c'est la première fois et transformer tes documents en vecteurs.
```bash
qmd embed
```
Ta base de connaissances est maintenant prête à être interrogée.

## Utilisation : De la Recherche Simple à la Recherche Structurée

Voici comment j'utilise `qmd` au quotidien.

### La Recherche Hybride (`qmd query`)

C'est la commande par défaut. Elle est parfaite pour la plupart des cas.

```bash
# Chercher comment j'ai sécurisé Traefik
qmd query "traefik security middleware" -c presentation
```

### La Recherche Structurée (Mode Expert)

Quand je veux être plus précis, je combine la recherche par mot-clé (`lex:`), par concept (`vec:`) et par document hypothétique (`hyde:`).

```bash
# Problème : Trouver le guide qui explique comment configurer un service systemd pour un script Python.
qmd query $'hyde: A tutorial that explains how to create a systemd service file to run a Python script as a background daemon on Ubuntu.' -c tutos_live
```

## Automatisation avec les Hooks Git

Pour que l'index `qmd` soit toujours à jour, j'ai mis en place un hook Git `post-commit` dans chaque dépôt. Ce script lance automatiquement `qmd update` après chaque commit qui modifie des fichiers pertinents.

**Fichier :** `.git/hooks/post-commit` (dans chaque dépôt)
```bash
#!/bin/sh
# Hook post-commit pour mettre à jour l'index qmd.

echo "--- [QMD Hook] Running post-commit checks ---"

if ! command -v qmd > /dev/null; then
    echo "--- [QMD Hook] Error: 'qmd' command not found. Skipping update."
    exit 0
fi

# Vérifie si des fichiers pertinents ont été modifiés
if git diff --name-only HEAD~1 HEAD | grep -qE "\.(md|yml|yaml|json|py|toml|sh)$"; then
    echo "--- [QMD Hook] Relevant files were modified. Updating QMD index..."
    qmd update
    echo "--- [QMD Hook] QMD index update complete. ---"
else
    echo "--- [QMD Hook] No relevant files modified. Skipping update."
fi

exit 0
```
N'oublie pas de le rendre exécutable : `chmod +x .git/hooks/post-commit`.

Avec cette configuration, `qmd` est devenu un outil de recherche puissant et parfaitement intégré à mon workflow, me permettant d'accéder à ma propre base de connaissances de manière intelligente et instantanée.
