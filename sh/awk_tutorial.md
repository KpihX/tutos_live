### **Tutoriel Complet sur `awk` : Le Couteau Suisse du Traitement de Texte**

#### **Introduction : Le Problème Originel**

Imaginons un fichier de log `access.log`, où chaque ligne ressemble à ceci :
`192.168.1.10 - - [10/Jan/2026:10:05:15 +0000] "GET /page.html HTTP/1.1" 200 1542`

Vos tâches :
1.  Afficher uniquement l'adresse IP et la page demandée pour chaque ligne.
2.  Calculer le total des octets transférés (l'avant-dernier champ) pour toutes les requêtes.
3.  Lister toutes les requêtes qui ont renvoyé une erreur 404.

Avec `sed` ou `grep`, c'est compliqué, voire impossible. `grep` peut trouver les lignes avec "404", mais comment extraire et *calculer* une somme ? `sed` peut extraire du texte avec des expressions régulières complexes, mais il n'a aucune notion de "colonne" ou de "champ" et ne sait pas faire d'arithmétique.

C'est là que `awk` entre en scène. Il a été conçu pour comprendre et traiter des données **structurées en colonnes**.

#### **Partie 1 : Le "Pourquoi" - La Philosophie de `awk`**

Le nom `awk` vient de ses créateurs : **A**ho, **W**einberger, et **K**ernighan (le 'K' de "K&R C"). Sa philosophie est radicalement différente de celle de `grep` ou `sed`.

`awk` lit un fichier ligne par ligne (comme les autres), mais pour chaque ligne, il effectue une étape supplémentaire cruciale : il la découpe en **champs** (fields).

**Le modèle de base de `awk` est : `PATERN { ACTION }`**

*   **PATERN (le motif) :** C'est une condition. "Est-ce que cette ligne correspond à ce que je cherche ?". C'est optionnel. Si vous ne mettez pas de motif, l'action s'applique à toutes les lignes.
*   **ACTION (l'action) :** C'est ce qu'il faut faire si la ligne correspond au motif. C'est un bloc de code. Si vous ne mettez pas d'action, l'action par défaut est d'imprimer la ligne entière (`{ print }`).

**Le fonctionnement implicite de `awk` :**

```
Pour chaque ligne du fichier :
  Découper la ligne en champs.
  Est-ce que la ligne correspond au PATERN ?
    Si oui, alors exécuter l'ACTION.
Fin de la boucle.
```
C'est un véritable petit langage de programmation dédié au traitement de texte.

---

#### **Partie 2 : Le "Comment" - Syntaxe, Champs et Actions de Base**

La structure générale :
`awk 'script' [fichier...]`

**Les Champs : La Magie de `$`**

`awk` stocke les champs découpés dans des variables spéciales numérotées avec un `$`: 

*   `$0` : La ligne entière (l'enregistrement complet).
*   `$1` : Le premier champ.
*   `$2` : Le deuxième champ.
*   ... 
*   `$N` : Le Nième champ.

Par défaut, les champs sont séparés par un ou plusieurs espaces ou tabulations.

**Exemple 1 : Afficher des colonnes spécifiques**

Utilisons la sortie de `ls -l` :
```
-rw-r--r-- 1 user group 432 Jan 10 10:20 fichier.txt
drwxr-xr-x 2 user group 4096 Jan 9 09:15 dossier
```
Si nous voulons afficher la taille (5ème champ) et le nom (9ème champ) :
```bash
ls -l | awk '{ print $5, $9 }'
# Sortie :
# 432 fichier.txt
# 4096 dossier
```
**Qu'est-il arrivé ?**
Pour chaque ligne de `ls -l`, `awk` a exécuté l'action `{ print $5, $9 }`. La virgule dans `print` ajoute un séparateur de sortie par défaut (un espace).

---

#### **Partie 3 : Le Pouvoir des `PATERN` - Filtrer les Lignes**

**1. Filtrage par expression régulière :**

Comme `grep`, vous pouvez filtrer les lignes qui correspondent à une regex.

```bash
# Afficher la taille et le nom uniquement pour les dossiers (lignes commençant par 'd')
ls -l | awk '/^d/ { print $5, $9 }'
# Sortie :
# 4096 dossier
```

**2. Filtrage par comparaison de champs :**

C'est là que `awk` surpasse de loin les autres outils. Vous pouvez utiliser des opérateurs de comparaison (`==`, `!=`, `>`, `<`, `>=`, `<=`) sur n'importe quel champ.

```bash
# Afficher les fichiers de plus de 1000 octets
ls -l | awk '$5 > 1000 { print $5, $9 }'

# Résoudre notre problème de log : trouver les erreurs 404
# Le code de statut HTTP est le 9ème champ dans notre exemple de log
awk '$9 == 404 { print $0 }' access.log
```

**3. Combiner les motifs :**

Utilisez `&&` (ET) et `||` (OU) pour des filtres complexes.

```bash
# Erreurs 404 venant d'une IP spécifique
awk '$1 == "192.168.1.50" && $9 == 404 { print $7 }' access.log # Affiche la page
```

---

#### **Partie 4 : Le Pouvoir des `ACTION` - Variables, Calculs et Blocs Spéciaux**

**1. Les variables et les calculs :**

`awk` peut déclarer des variables et effectuer des opérations arithmétiques.

**Exemple : Calculer l'espace total utilisé**
```bash
ls -l | awk '{ total += $5 } END { print "Total:", total, "octets" }'
```
**Analyse de cette commande magique :**
*   `{ total += $5 }` : Pour chaque ligne, on ajoute la valeur du 5ème champ (la taille) à une variable que nous avons appelée `total`. `awk` initialise `total` à 0 automatiquement.
*   `END { ... }` : C'est un **bloc spécial**. Il s'exécute **une seule fois**, après que toutes les lignes ont été lues. C'est l'endroit parfait pour afficher un résultat final.

**2. Les blocs `BEGIN` et `END` :**

*   `BEGIN { ... }` : S'exécute **avant** la lecture de la première ligne. Idéal pour initialiser des variables ou afficher un en-tête.
*   `END { ... }` : S'exécute **après** la lecture de la dernière ligne. Idéal pour les totaux, les moyennes, etc.

```bash
awk 'BEGIN { print "IP\tPAGE" } { print $1, $7 }' access.log
```

**3. Les variables intégrées (Built-in Variables) :**

`awk` fournit des variables très utiles :
*   `NR` (Number of Record) : Le numéro de la ligne courante (depuis le début de tous les fichiers).
*   `NF` (Number of Fields) : Le nombre de champs dans la ligne courante. Très utile !
*   `$NF` : La valeur du **dernier champ**. Extrêmement pratique.
*   `FS` (Field Separator) : Le séparateur de champ (par défaut, l'espace).
*   `OFS` (Output Field Separator) : Le séparateur de champ en sortie (par défaut, l'espace).

**Exemple : Traiter un fichier CSV**

Les fichiers CSV utilisent la virgule comme séparateur. On peut le dire à `awk` avec l'option `-F`.

Fichier `utilisateurs.csv`:
`1,Alice,France`
`2,Bob,USA`
`3,Charlie,France`

```bash
# Afficher le nom des utilisateurs français
awk -F',' '$3 == "France" { print $2 }' utilisateurs.csv
# Sortie :
# Alice
# Charlie
```

**4. Les tableaux associatifs : La fonctionnalité ultime**

`awk` supporte les tableaux associatifs (comme les dictionnaires Python), où les indices peuvent être des chaînes de caractères. C'est parfait pour compter des choses.

**Exemple : Compter le nombre de requêtes par code de statut**
```bash
awk '{ counts[$9]++ } END { for (code in counts) print code, counts[code] }' access.log
```
**Analyse :**
*   `{ counts[$9]++ }` : Pour chaque ligne, on utilise le code de statut (`$9`) comme clé du tableau `counts`. On incrémente sa valeur. Si la clé n'existe pas, `awk` la crée.
*   `END { ... }` : À la fin, on parcourt le tableau `counts` et on affiche chaque clé (le code statut) et sa valeur (le nombre d'occurrences).

---

#### **Partie 5 : `awk` en tant que Langage de Script**

Pour des logiques complexes, vous pouvez écrire votre script dans un fichier et l'exécuter avec l'option `-f`.

Fichier `mon_script.awk` :
```awk
# C'est un commentaire
BEGIN {
  FS = ","
  print "Analyse des utilisateurs..."
}

$3 == "France" {
  francais++
}

END {
  printf "Il y a %d utilisateurs français sur un total de %d lignes.\n", francais, NR
}
```

Exécution :
`awk -f mon_script.awk utilisateurs.csv`

---

#### **Conclusion : Quand utiliser `awk` ?**

`awk` est l'outil à dégainer lorsque vos données ont une structure, même simple.

*   **`grep`** est pour trouver des lignes.
*   **`sed`** est pour éditer des lignes.
*   **`awk`** est pour extraire, filtrer, manipuler et calculer des données à partir de **champs** au sein de ces lignes.

Il remplace avantageusement des scripts Python/Perl/Ruby pour une multitude de tâches d'analyse de logs, de traitement de CSV, ou de reformatage de données. Sa syntaxe est dense mais incroyablement expressive. Une fois que vous maîtrisez le concept `PATERN { ACTION }` et le pouvoir des champs, `awk` devient l'un des outils les plus puissants de votre arsenal en ligne de commande.
