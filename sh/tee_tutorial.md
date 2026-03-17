### **Tutoriel Complet sur `tee` : Le Maître de la Bifurcation de Flux**

#### **Introduction : Le Problème Originel**

Imaginez que vous exécutez un script d'installation ou une longue compilation qui dure 20 minutes. Vous voulez absolument voir ce qui se passe à l'écran en temps réel pour détecter d'éventuelles erreurs, mais vous voulez aussi sauvegarder toute la sortie dans un fichier de log pour l'analyser plus tard ou l'envoyer à un collègue.

Comment feriez-vous ?

1.  **L'approche par redirection standard (`>`) :** 
    ```bash
    ./mon_script.sh > install.log
    ```
    Le problème ? Vous ne voyez plus rien à l'écran. Vous devez attendre la fin ou ouvrir un autre terminal pour faire un `tail -f install.log`. C'est frustrant.

2.  **L'approche par copier-coller :**
    Laisser le script finir, puis copier tout le contenu de votre terminal dans un fichier. C'est archaïque, limité par la taille du buffer de votre terminal, et impossible à automatiser.

C'est ici qu'intervient `tee`.

#### **Partie 1 : Le "Pourquoi" - La Philosophie de `tee`**

La commande `tee` tire son nom d'une pièce de plomberie : le **raccord en T**. Sa philosophie est simple mais cruciale : **la bifurcation**.

1.  **Le Raccord en T :** Tout comme un raccord de tuyauterie prend un flux d'eau et le divise en deux directions, la commande `tee` prend l'entrée standard (`stdin`) et la copie vers deux destinations simultanément :
    *   La **sortie standard** (`stdout`), c'est-à-dire votre terminal.
    *   Un ou plusieurs **fichiers**.

    > **Analogie :** Imaginez un greffier dans un tribunal. Il écoute les débats (`le flux`). Tout en laissant les gens parler (le son continue vers l'audience), il écrit simultanément tout ce qui est dit dans son registre (`le fichier`).

**Les grands avantages de cette approche :**

*   **Visibilité et Persistance :** Vous gardez un œil sur l'exécution tout en garantissant une trace écrite.
*   **Maillon de Chaîne :** Comme `tee` renvoie ce qu'il reçoit vers `stdout`, il peut être inséré n'importe où au milieu d'un pipeline sans l'interrompre.
*   **Élévation de Privilèges :** C'est l'un des moyens les plus sûrs d'écrire dans un fichier protégé par `root` tout en restant dans un pipeline utilisateur.

---

#### **Partie 2 : Le "Comment" - La Syntaxe et l'Usage de Base**

La structure générale d'une commande `tee` est :

```bash
commande | tee [OPTIONS] [FICHIER...]
```

*   `commande |`: `tee` lit presque toujours depuis un tube (pipe).
*   `[OPTIONS]`: Pour modifier la façon d'écrire (ex: ajouter au lieu d'écraser).
*   `[FICHIER...]`: Le nom du ou des fichiers où sauvegarder la sortie.

##### **L'usage le plus simple**

```bash
ls -l | tee liste.txt
```
Ici, la liste des fichiers s'affiche dans votre terminal ET est sauvegardée dans `liste.txt`. Si `liste.txt` existait déjà, il est **écrasé**.

##### **L'option indispensable : `-a` (append)**

Par défaut, `tee` se comporte comme `>`, il écrase le fichier. Pour qu'il se comporte comme `>>` (ajouter à la fin), on utilise l'option `-a`.

```bash
echo "Nouvelle entrée de log" | tee -a journal.log
```

##### **Écrire dans plusieurs fichiers à la fois**

`tee` peut arroser plusieurs cibles simultanément :

```bash
echo "Alerte Système" | tee log1.txt log2.txt log3.txt
```

---

#### **Partie 3 : Le Cas d'Usage "Sudo" (Le plus célèbre)**

C'est l'utilisation la plus astucieuse de `tee`. Imaginez que vous voulez modifier un fichier système protégé :

```bash
# CELA NE FONCTIONNERA PAS :
sudo echo "127.0.0.1 monsite.local" >> /etc/hosts
```
**Pourquoi ?** Parce que le `sudo` s'applique à `echo`, mais la redirection `>>` est gérée par votre shell actuel, qui n'a pas les droits `root`.

**La solution avec `tee` :**
```bash
echo "127.0.0.1 monsite.local" | sudo tee -a /etc/hosts > /dev/null
```
Ici, `echo` tourne avec vos droits, mais `tee` tourne avec `sudo`. Il reçoit le texte et a la permission de l'écrire dans `/etc/hosts`. Le `> /dev/null` à la fin sert juste à ne pas afficher la ligne dans le terminal si vous voulez rester discret.

---

#### **Partie 4 : Astuces Avancées**

##### **Ignorer les interruptions avec `-i`**

Si vous ne voulez pas que `tee` s'arrête si vous appuyez sur `Ctrl+C` (SIGINT), utilisez l'option `-i` (**ignore interrupts**). C'est utile pour s'assurer que les logs sont bien écrits même si vous interrompez l'affichage.

```bash
./longue_tache.sh | tee -i record.log
```

##### **Utilisation avec les Process Substitutions**

Vous pouvez utiliser `tee` pour envoyer des données à deux commandes différentes simultanément :

```bash
cat données.csv | tee >(process_compta) >(process_marketing) > /dev/null
```

---

#### **Partie 5 : Pratique - Exemples concrets**

**Cas 1 : Benchmarker et Logguer**
Utilisons ton script de benchmark :
```bash
./benchmark.sh phi3.5 | tee results_phi3.5.log
```
Tu vois les scores en direct et tu gardes le fichier pour tes archives.

**Cas 2 : Déboguer un pipeline complexe**
Si vous avez un pipeline qui ne donne pas le résultat escompté, insérez `tee` au milieu pour voir l'état des données à cette étape précise :
```bash
cat data.txt | grep "error" | tee /tmp/step1.txt | awk '{print $3}' | sort
```
Vous pourrez inspecter `/tmp/step1.txt` pour voir si le `grep` a bien fonctionné avant le `awk`.

**Cas 3 : Sauvegarder l'historique d'une installation**
```bash
sudo apt upgrade -y | tee upgrade_$(date +%F).log
```

---

#### **Conclusion**

`tee` est un outil simple, mais il est le garant de la visibilité dans le monde de l'automatisation. 

*   Il **double** l'information.
*   Il **résout** les problèmes de droits avec `sudo`.
*   Il est le **miroir** de vos pipelines.

La prochaine fois que vous lancerez une commande dont vous voulez garder une trace sans sacrifier votre affichage, pensez à `tee`. C'est le petit raccord qui change tout.
