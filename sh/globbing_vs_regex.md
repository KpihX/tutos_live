# Globbing (Shell) vs Regex (Texte) : Le Guide Complet

## 1. Philosophie : L'Ingrédient vs La Recette

La confusion entre `*` en ligne de commande et `*` dans un éditeur de texte est l'erreur n°1 sous Linux.

### L'origine des mots
*   **Wildcard (Joker) :** Vient du Poker. Une carte qui peut remplacer n'importe quelle autre. C'est le symbole (`*`).
*   **Globbing :** Vient d'un vieux programme Unix (`/etc/glob`). C'est l'action que fait le Shell pour transformer `*.txt` en une liste de fichiers (`a.txt`, `b.txt`).

## 2. La différence fondamentale

| Concept | Globbing (Wildcards) | Expressions Régulières (Regex) |
| :--- | :--- | :--- |
| **Cible** | Les **Fichiers** sur le disque | Le **Contenu** (texte, chaînes) |
| **Interprète** | Le **Shell** (Bash, Zsh) | Le **Programme** (grep, sed, python) |
| **Symbole `*`** | "N'importe quelle suite de caractères" | "Répéter le précédent 0 à l'infini" |
| **Tout sélectionner** | `*` | `.*` |
| **Un seul caractère** | `?` | `.` |

## 3. Le Guide de Survie : Quel outil utilise quoi ?

### Groupe A : Les Outils de Fichiers (Globbing)
Ils manipulent des "boîtes" (fichiers), pas ce qu'il y a dedans.
*   `ls *.txt`
*   `rm *.bak`
*   `cp *.jpg /backup`
*   `dpkg -i *.deb`

### Groupe B : Les Outils de Texte (Regex)
Ils lisent et analysent le texte.
*   `grep "motif" fichier`
*   `sed`
*   `awk`
*   `vim` (recherche)

### Le Piège : `grep`
`grep` utilise les deux !
```bash
grep "ca.*t" *.txt
#    ^^^^^^  ^^^^^
#    Regex   Globbing (géré par le Shell avant que grep ne démarre)
```

## 4. Mise en pratique avec Python

Python distingue aussi très clairement ces deux concepts via deux modules différents.

### A. Le Globbing en Python (Module `glob`)
Pour lister des fichiers, on n'utilise pas de Regex, mais le module `glob` (ou `pathlib`).

```python
import glob

# Équivalent de 'ls *.py'
# Renvoie une liste : ['script.py', 'test.py']
fichiers_py = glob.glob('*.py')

print(f"J'ai trouvé {len(fichiers_py)} scripts Python.")
```

*Avec `pathlib` (plus moderne) :*
```python
from pathlib import Path

# Chercher tous les fichiers jpg dans le dossier courant
for image in Path('.').glob('*.jpg'):
    print(image.name)
```

### B. Les Regex en Python (Module `re`)
Pour analyser du texte, extraire des données ou vérifier un format.

Rappel : En Regex, `*` ne marche pas tout seul, il quantifie ce qu'il y a avant.

```python
import re

texte = "J'ai un chat, un cat et un caaaat."

# Motif : "c" suivi de "a" (0 ou plusieurs fois), suivi de "t"
# Équivalent shell (approximatif) : c*t
motif = r"ca*t" 

# Trouver toutes les occurrences
resultats = re.findall(motif, texte)

print(resultats) 
# Sortie : ['chat', 'cat', 'caaaat']
# Notez que 'h' de chat est ignoré car le motif est strict : c + a(s) + t.
# Pour inclure 'chat', il faudrait : r"c[ha]*t"
```

### Résumé Python
*   Je veux des **fichiers** ? -> `import glob`
*   Je veux fouiller du **texte** ? -> `import re`
