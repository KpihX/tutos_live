### **Tutoriel Complet sur `sed` : Le Maître de la Transformation de Texte**

#### **Introduction : Le Problème Originel**

Imaginez que vous êtes sur un serveur distant, sans interface graphique. Vous devez modifier un fichier de configuration de 5000 lignes. Votre tâche : commenter toutes les lignes contenant le mot `beta`, remplacer toutes les occurrences de `old_database_ip` par `new_database_ip`, et supprimer les lignes de débogage vides.

Comment feriez-vous ?

1.  **L'approche manuelle :** Ouvrir le fichier avec un éditeur comme `vim` ou `nano`. Chercher chaque occurrence, faire la modification, sauvegarder. C'est lent, fastidieux, et vous risquez de faire des erreurs. Si vous devez le faire sur 10 serveurs, c'est un cauchemar.
2.  **L'approche scriptée :** Vous pourriez écrire un script en Python ou Perl pour lire le fichier ligne par ligne, effectuer les changements et écrire dans un nouveau fichier. C'est une bonne solution, mais elle peut être lourde pour des tâches simples et nécessite de connaître un langage de programmation.

C'est précisément pour combler ce vide qu'un outil comme `sed` a été créé.

#### **Partie 1 : Le "Pourquoi" - La Philosophie de `sed`**

`sed` est l'acronyme de **S**tream **ED**itor (Éditeur de Flux). Ce nom contient toute sa philosophie :

1.  **Stream (Flux) :** `sed` n'édite pas le fichier directement (par défaut). Il lit le fichier (ou n'importe quelle entrée, comme la sortie d'une autre commande) ligne par ligne, comme un "flux" de données qui passe à travers lui. Pour chaque ligne, il applique un ensemble de règles que vous lui donnez. Puis, il affiche le résultat dans la sortie standard (votre terminal).

    > **Analogie :** Imaginez une chaîne de montage. Les lignes du fichier sont les pièces sur un tapis roulant (`le flux`). `sed` est un bras robotique qui effectue une opération sur chaque pièce (`l'édition`). La pièce modifiée continue son chemin vers la sortie.

2.  **Editor (Éditeur) :** Il est conçu pour effectuer des opérations d'édition : substitution de texte, suppression, insertion, etc.

**Les grands avantages de cette approche :**

*   **Non-destructif :** Comme `sed` écrit sur la sortie standard, votre fichier original reste intact. C'est une sécurité immense. Vous pouvez voir le résultat avant de décider d'écraser le fichier.
*   **"Pipe-able" :** `sed` s'intègre parfaitement dans la philosophie UNIX. Vous pouvez "tuyauter" (pipe `|`) la sortie d'une commande dans `sed`, et la sortie de `sed` dans une autre commande.
    ```bash
    # Exemple : Lister les processus, filtrer avec grep, puis modifier la sortie avec sed
    ps aux | grep 'nginx' | sed 's/\/usr\/sbin\/nginx/NGINX_PROCESS/'
    ```
*   **Scriptable :** Toutes les opérations sont des commandes textuelles. Vous pouvez donc les sauvegarder dans un fichier et les réutiliser, les automatiser, les intégrer dans des scripts shell. C'est l'automatisation à son apogée.

---

#### **Partie 2 : Le "Comment" - La Syntaxe et les Commandes de Base**

La structure générale d'une commande `sed` est :

```bash
sed [OPTIONS] 'script' [fichier...]
```

*   `[OPTIONS]`: Des options pour changer le comportement (ex: `-i` pour éditer sur place).
*   `'script'`: C'est le cœur de `sed`. C'est une ou plusieurs commandes d'édition. On l'entoure de guillemets simples `'` pour éviter que le shell n'interprète des caractères spéciaux.
*   `[fichier...]`: Le ou les fichiers à traiter. Si aucun n'est fourni, `sed` lit l'entrée standard.

##### **L'Espace de Travail (Pattern Space)**

Pour chaque ligne lue, `sed` la place dans une zone mémoire appelée "espace de travail". C'est sur cette copie de la ligne que les commandes du script sont appliquées. Une fois toutes les commandes appliquées, `sed` affiche (par défaut) le contenu de l'espace de travail.

##### **La Commande la plus Importante : `s` (Substitution)**

C'est 90% de l'utilisation de `sed`. Elle remplace du texte.

**Syntaxe :** `s/motif/remplacement/flags`

*   `s` : La commande de substitution.
*   `/` : Le délimiteur. Vous pouvez utiliser presque n'importe quel autre caractère (`#`, `|`, `:`, etc.), ce qui est très utile si votre texte contient des `/` (comme des chemins de fichiers ou des URLs).
*   `motif` : Une **expression régulière** (regex) pour trouver le texte à remplacer.
*   `remplacement` : Le texte par lequel remplacer le `motif`.
*   `flags` (optionnel) : Pour modifier le comportement.

**Exemple 1 : Remplacer la première occurrence**

Créons un fichier `exemple.txt` :
```
Hello world, this is a test. The world is beautiful.
```

```bash
sed 's/world/planet/' exemple.txt
# Sortie :
# Hello planet, this is a test. The world is beautiful.
```
Seul le premier "world" a été remplacé.

**Les `flags` de substitution :**

*   `g` (global) : Remplace **toutes** les occurrences sur la ligne, pas seulement la première.

    ```bash
    sed 's/world/planet/g' exemple.txt
    # Sortie :
    # Hello planet, this is a test. The planet is beautiful.
    ```

*   `i` (ignore-case) : Ignore la casse (majuscules/minuscules) dans le `motif`. (Note: cette option est une extension GNU, elle peut ne pas marcher sur des systèmes non-Linux comme macOS par défaut).

    ```bash
    # Crée un fichier avec "Hello"
    echo "Hello" > test.txt
    sed 's/hello/Hi/i' test.txt
    # Sortie :
    # Hi
    ```

*   **Un nombre :** Remplace seulement la Nième occurrence.

    ```bash
    sed 's/world/planet/2' exemple.txt
    # Sortie :
    # Hello world, this is a test. The planet is beautiful.
    ```

**Changer le délimiteur :**

```bash
# Remplacer un chemin de fichier
# echo "/usr/local/bin" | sed 's/\/usr\/local/\/opt/' # Difficile à lire avec les échappements
echo "/usr/local/bin" | sed 's#/usr/local#/opt#'    # Beaucoup plus clair !
```

---

#### **Partie 3 : Le Ciblage - Adressage**

La vraie puissance de `sed` est de pouvoir appliquer une commande **uniquement sur certaines lignes**. C'est ce qu'on appelle l'adressage. L'adresse se place juste avant la commande.

**Syntaxe :** `[adresse]commande`

**1. Adressage par numéro de ligne :**

*   Appliquer à une ligne spécifique : `3s/a/b/` (substitue sur la ligne 3 uniquement).
*   Appliquer à un intervalle de lignes : `5,10d` (supprime les lignes 5 à 10).
*   D'une ligne jusqu'à la fin : `15,$s/old/new/` (`$` représente la dernière ligne).

**Exemple :** Fichier `lignes.txt`
```
ligne 1
ligne 2 (à changer)
ligne 3
ligne 4
ligne 5 (à changer)
```
```bash
# Changer uniquement la ligne 2
sed '2s/changer/modifiée/' lignes.txt

# Changer les lignes 2 et 5
sed -e '2s/changer/modifiée/' -e '5s/changer/modifiée/' lignes.txt
# Ou plus court avec un point-virgule
sed '2s/changer/modifiée/; 5s/changer/modifiée/' lignes.txt
```

**2. Adressage par motif (expression régulière) :**

C'est encore plus puissant. La commande ne s'applique qu'aux lignes qui correspondent au motif.

*   `/[Mm]otif/d` : Supprime toutes les lignes contenant "Motif" ou "motif".
*   `/^#/d` : Supprime toutes les lignes commençant par `#` (les commentaires).
*   `/^$/d` : Supprime toutes les lignes vides.

**Exemple :** Fichier `config.txt`
```
# Activer le cache
enable_cache=true

# Désactiver le debug
# debug_mode=true
debug_mode=false
```
```bash
# Commenter la ligne qui active le cache
sed '/enable_cache=true/s/^/#/' config.txt
# Sortie :
# # Activer le cache
#enable_cache=true
# ...
```

**3. Adressage par intervalle de motifs :**

Appliquer une commande d'un motif de début jusqu'à un motif de fin.

```bash
sed '/START/,/END/d' mon_fichier.log
# Supprime la ligne contenant START, celle contenant END, et tout ce qui se trouve entre les deux.
```

**4. Inverser l'adresse avec `!`**

Appliquer une commande à **toutes les lignes SAUF** celles qui correspondent à l'adresse.

```bash
sed '/debug/!d' mon_fichier.log
# Supprime toutes les lignes qui ne contiennent PAS 'debug'. Ne garde que les lignes de debug.
```

---

#### **Partie 4 : Autres Commandes Utiles**

*   `d` (delete) : Supprime la ligne entière de l'espace de travail. Elle ne sera donc pas affichée.

    ```bash
    # Supprimer les commentaires et les lignes vides
sed '/^#/d; /^$/d' config.txt
    ```

*   `p` (print) : Affiche le contenu de l'espace de travail. Cela semble inutile car `sed` le fait déjà par défaut. Mais son pouvoir se révèle avec l'option `-n`.

    **L'option `-n` et la commande `p` : le duo `grep`**

    L'option `-n` dit à `sed` de **NE PAS** afficher l'espace de travail à la fin de chaque cycle.
    En combinant `-n` avec la commande `p`, vous pouvez afficher **UNIQUEMENT** les lignes que vous choisissez.

    ```bash
    # Simuler un `grep 'debug'`
sed -n '/debug/p' config.txt
    # Seules les lignes contenant "debug" seront affichées.
    ```

*   `i` (insert) et `a` (append) : Pour ajouter du texte.
    *   `i \texte à insérer` : Insère le texte **avant** la ligne courante.
    *   `a \texte à ajouter` : Ajoute le texte **après** la ligne courante.

    ```bash
    # Ajouter une ligne après la 2ème ligne
sed '2a \## NOUVELLE LIGNE AJOUTEE ##' config.txt
    ```

*   `y` (transliterate) : Remplace des caractères un par un. Utile pour changer la casse.

    ```bash
    # Met tout en majuscules
    echo "bonjour" | sed 'y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/'
    ```

---

#### **Partie 5 : Édition sur Place et Bonnes Pratiques**

Jusqu'à maintenant, nous n'avons pas modifié les fichiers. Pour le faire, on utilise l'option `-i`.

**ATTENTION : L'OPTION `-i` EST DESTRUCTIVE.**

```bash
# Cette commande MODIFIE config.txt directement. Pas de retour en arrière.
sed -i '/^$/d' config.txt
```

**Bonne pratique : Créer un backup**

La plupart des versions de `sed` permettent de créer un backup automatique en ajoutant un suffixe à l'option `-i`.

```bash
# Modifie config.txt et crée un backup config.txt.bak avec le contenu original
sed -i.bak 's/old/new/g' config.txt
```
**C'est une pratique très fortement recommandée !**

---

#### **Partie 6 : Concepts Avancés - Pour Aller Plus Loin**

`sed` a une deuxième zone mémoire appelée **l'espace de rétention (Hold Space)**. C'est un "presse-papiers" interne.

*   `h` : Copie l'espace de travail **vers** l'espace de rétention (écrase ce qu'il y avait).
*   `H` : Ajoute l'espace de travail **vers** l'espace de rétention (ajoute à la suite).
*   `g` : Copie l'espace de rétention **vers** l'espace de travail (écrase ce qu'il y avait).
*   `G` : Ajoute l'espace de rétention **vers** l'espace de travail (ajoute à la suite).
*   `x` : Échange le contenu des deux espaces.

Ceci permet des opérations complexes comme inverser l'ordre des lignes d'un fichier (un exercice classique mais complexe) ou déplacer des blocs de texte.

**Exemple : Dupliquer chaque ligne**
```bash
sed 'h;G' exemple.txt
# Pour chaque ligne :
# 1. `h` : la copie dans l'espace de rétention.
# 2. `G` : ajoute le contenu de l'espace de rétention (qui contient la ligne elle-même) à la fin de l'espace de travail.
# Résultat :
# Hello world, this is a test. The world is beautiful.
# Hello world, this is a test. The world is beautiful.
```

---

#### **Conclusion**

Vous comprenez maintenant `sed` non plus comme une commande obscure, mais comme un outil de traitement de texte en flux, extrêmement puissant et efficace.

*   Il est **sûr** par défaut.
*   Il est **rapide** et **léger**.
*   Il est **universel** (disponible sur presque tous les systèmes UNIX-like).
*   Il est parfait pour **l'automatisation** de tâches d'édition répétitives.

La clé pour maîtriser `sed` est de pratiquer. Commencez par des substitutions simples, puis intégrez l'adressage, et enfin, lorsque vous rencontrerez un problème complexe, souvenez-vous que l'espace de rétention existe pour vous sauver.
