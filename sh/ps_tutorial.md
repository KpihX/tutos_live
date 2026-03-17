### **Tutoriel Complet sur `ps` : L'Art d'Inspecter les Processus**

#### **Introduction : Le Problème Originel**

Votre ordinateur ou votre serveur exécute des dizaines, voire des centaines de programmes en même temps. Un programme en cours d'exécution est appelé un **processus**.

Maintenant, imaginez ces situations :
*   Votre serveur est soudainement très lent. Un processus consomme-t-il 100% du CPU ?
*   Vous avez lancé un script en arrière-plan. Est-il toujours en train de tourner ?
*   Le serveur web ne répond plus. Le processus `nginx` ou `apache2` est-il même démarré ?
*   Vous voyez une commande suspecte et vous voulez savoir qui l'a lancée et depuis quand.

Pour répondre à ces questions, vous avez besoin d'un moyen de lister les processus en cours et d'obtenir des informations à leur sujet. C'est exactement le rôle de `ps` (de l'anglais **P**rocess **S**tatus).

**`ps` vs `top`/`htop` : La différence clé**
Pensez à `ps` comme à un **photographe**. Il prend un **cliché instantané** de l'état des processus à un moment T et vous le présente. `top` et `htop`, eux, sont des **cameramen** : ils filment l'activité des processus en continu et rafraîchissent l'affichage en temps réel.

---

#### **Partie 1 : Le "Pourquoi" - La Philosophie (et la Confusion) de `ps`**

La mission de `ps` est simple : afficher des informations sur les processus actifs. Cependant, au fil de l'histoire des systèmes UNIX, différentes versions de `ps` ont vu le jour, chacune avec sa propre syntaxe. C'est la source de la confusion.

Il existe principalement **trois styles d'options** pour `ps` :

1.  **Style UNIX (ou System V) :** Les options sont précédées d'un tiret `-`.
    *   Exemple : `ps -ef`

2.  **Style BSD :** Les options n'ont **pas** de tiret.
    *   Exemple : `ps aux`

3.  **Style GNU "long" :** Les options sont précédées de deux tirets `--`.
    *   Exemple : `ps --forest`

La bonne nouvelle ? Sur la plupart des systèmes Linux modernes, ces styles peuvent coexister. La mauvaise nouvelle ? Cela peut rendre la lecture des commandes d'autrui difficile. La meilleure approche est de mémoriser les deux invocations les plus courantes et de comprendre ce que chacune fait.

---

#### **Partie 2 : Les Deux Commandes que Vous Devez Absolument Connaître**

Oubliez la myriade d'options pour l'instant. 99% du temps, vous utiliserez l'une de ces two commandes.

##### **1. `ps aux` (Style BSD)**

C'est probablement la commande `ps` la plus populaire.

*   `a` : Affiche les processus de **tous** les utilisateurs, pas seulement les vôtres.
*   `u` : Affiche des informations détaillées dans un format "orienté **u**tilisateur".
*   `x` : Affiche aussi les processus qui ne sont pas attachés à un terminal (les "daemons" ou services qui tournent en arrière-plan).

**Sortie typique de `ps aux` :**
```
USER         PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1   0.0  0.1 169324 11332 ?        Ss   Jan09   0:02 /sbin/init
www-data     850   0.2  0.5 834572 45120 ?        Sl   Jan09   5:30 nginx: worker process
kpihx       2150   0.0  0.2 229892 21888 pts/0    Ss   10:30   0:00 -zsh
```
**Les colonnes les plus importantes :**
*   `USER` : L'utilisateur qui a lancé le processus.
*   `PID` : **Process ID**. C'est le numéro d'identification unique du processus. C'est ce que vous utiliserez pour "tuer" (kill) un processus.
*   `%CPU` / `%MEM` : Le pourcentage de CPU et de mémoire vive (RAM) utilisé par le processus. **Idéal pour trouver les processus gourmands.**
*   `COMMAND` : La commande qui a été exécutée pour lancer le processus.

##### **2. `ps -ef` (Style UNIX)**

C'est l'autre grand classique, souvent préféré par les administrateurs système "à l'ancienne".

*   `-e` : Affiche **tous** (`every`) les processus.
*   `-f` : Affiche au format "complet" (`full`), qui a l'avantage de montrer la relation parent-enfant.

**Sortie typique de `ps -ef` :**
```
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 Jan09 ?        00:00:02 /sbin/init
root         849       1  0 Jan09 ?        00:00:00 nginx: master process /usr/sbin/nginx
www-data     850     849  0 Jan09 ?        00:05:30 nginx: worker process
kpihx       2150    2149  0 10:30 pts/0    00:00:00 -zsh
```
**Les colonnes les plus importantes ici :**
*   `UID` : L'utilisateur (sous forme d'ID numérique).
*   `PID` : L'ID du processus (comme avant).
*   `PPID` : **Parent Process ID**. C'est l'ID du processus qui a lancé celui-ci. **C'est la grande force de cette vue.** On voit ici que le processus `nginx: master` (PID 849) a lancé le `worker` (PID 850).
*   `CMD` : La commande (comme avant).

**Lequel choisir ?**
*   Utilisez `ps aux` pour une vue rapide, orientée "ressources" (qui consomme quoi ?).
*   Utilisez `ps -ef` pour une vue "système", pour comprendre la hiérarchie et les dépendances entre processus.

---

#### **Partie 3 : Personnaliser la Sortie - L'option `-o`**

Vous n'êtes pas limité aux formats par défaut. L'option `-o` (ou `o` en style BSD) vous permet de choisir exactement les colonnes que vous voulez voir.

```bash
# Afficher uniquement le PID, l'utilisateur, et la commande de tous les processus
ps -eo pid,user,comm

# L'exemple ultime : trouver les 5 processus les plus gourmands en mémoire
ps -eo pid,user,%mem,comm --sort=-%mem | head -n 6
```
**Analyse :**
*   `-eo pid,user,%mem,comm` : On demande 4 colonnes spécifiques.
*   `--sort=-%mem` : On trie (`sort`) la sortie par la colonne `%mem`. Le `-` indique un tri descendant (du plus grand au plus petit).
*   `| head -n 6` : On envoie le résultat à `head` pour ne garder que les 6 premières lignes (1 pour l'en-tête + 5 processus).

---

#### **Partie 4 : `ps` et `grep` : Le Duo Incontournable (et ses pièges)**

La tâche la plus courante est de trouver le PID d'un processus spécifique. La méthode classique est de combiner `ps` avec `grep`.

```bash
ps aux | grep "nginx"
# Sortie :
# www-data     850     0.2  0.5 834572 45120 ?        Sl   Jan09   5:30 nginx: worker process
# kpihx       2300     0.0  0.0  12345   880 pts/0    S+   11:00   0:00 grep "nginx"
```
**Le Piège :** Remarquez que `grep` se trouve lui-même dans la liste ! La commande `grep "nginx"` contient le mot "nginx".

**L'astuce classique pour éviter ça :**
```bash
ps aux | grep "[n]ginx"
```
L'expression régulière `[n]ginx` correspond bien à "nginx", mais la chaîne de caractères de la commande `grep` elle-même est maintenant `grep "[n]ginx"`, qui ne correspond plus.

**La méthode moderne et plus propre : `pgrep`**
L'outil `pgrep` est fait pour ça !
```bash
# Trouver le PID de nginx
pgrep nginx

# Afficher le PID et la commande complète
pgrep -a nginx
```
En général, préférez `pgrep` quand vous voulez juste trouver un processus.

---

#### **Partie 5 : Exemples Pratiques et Avancés**

**1. Voir l'arborescence des processus :**
Pour comprendre qui a lancé quoi, la vue en arbre est fantastique.
```bash
ps axf
# ou, avec la vue -ef
ps -ef --forest
```

**2. Tuer un processus :**
Le flux de travail classique : trouver, puis tuer.
```bash
# 1. Trouver le PID du script qui pose problème
ps aux | grep "script_fou.py"

# 2. Tuer le processus avec la commande kill
kill 12345  # Remplacez 12345 par le PID trouvé

# Ou encore mieux, avec pkill (qui combine pgrep et kill)
pkill -f "script_fou.py"
```

---

#### **Conclusion**

`ps` est votre fenêtre sur l'âme de votre système. Il vous dit ce qui est en cours, qui l'a lancé, et comment ça se comporte.

*   Retenez les deux commandes reines : `ps aux` (vue ressources) et `ps -ef` (vue hiérarchique).
*   N'oubliez jamais sa nature d'**instantané**, par opposition à la vue en direct de `top`.
*   Apprenez à le combiner avec `grep`, `sort`, `head` et `kill` pour une administration système efficace.
*   Pour des tâches simples de recherche, envisagez les outils modernes comme `pgrep` et `pkill` qui sont plus directs et moins sujets aux erreurs.

Avec `ps` dans votre arsenal, un système lent ou un processus bloqué n'est plus une boîte noire, mais un problème que vous pouvez diagnostiquer et résoudre.
