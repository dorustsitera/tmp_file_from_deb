# Workflow de contribution patch — Linux Kernel

> Récapitulatif personnel basé sur le cours Linux Foundation (patch development workflow) + les points clarifiés en pratique.

---

## 0. Vue d'ensemble

```
work branch créée
      │
      ▼
┌─────────────────────────────┐
│   Cycle "rinse, lather,     │
│        repeat"              │
│  1. Édite le code           │
│  2. Compile                 │
│  3. checkpatch.pl sur le    │
│     diff brut                │
│  4. Corrige erreurs/warns   │  ◄── on boucle ici jusqu'à 0 erreur
└──────────────┬───────────────┘
               ▼
        Tests fonctionnels
               ▼
            git commit
               ▼
       get_maintainer.pl
               ▼
   git format-patch (--to/--cc)
               ▼
   checkpatch.pl sur le .patch
               ▼
   git send-email --dry-run
               ▼
   git send-email (envoi réel)
```

---

## 1. Setup des remotes et branches

```bash
git remote add origin git@github.com:<toi>/linux.git   # ton fork
git remote add linux  https://git.kernel.org/.../torvalds/linux.git  # mainline
```

> ⚠️ Le nom du remote canonique (`upstream` vs `linux`) n'a aucune importance technique — seulement une convention. Si la doc parle d'`upstream`, adapte avec ton nom réel.

Deux branches distinctes :

| Branche | Rôle |
|---|---|
| `master` | Suit `linux/master` (ou `upstream/master`), **jamais** de commit direct dessus |
| `work` | Branche de développement — tout le cycle ci-dessous se fait ici |

```bash
git checkout -b work linux/master
```

Vérifier sur quelle branche on est : `git branch -a` (l'astérisque `*` indique la branche active).

---

## 2. Le cycle "rinse, lather, repeat"

1. Édite le code (`drivers/...`, etc.)
2. Compile
3. Vérifie le style **sur le diff brut**, *avant* de committer :
   ```bash
   git diff > temp
   scripts/checkpatch.pl temp
   ```
4. Corrige les erreurs/warnings, retour à l'étape 1 si besoin

### ⚠️ Ne jamais lancer checkpatch.pl sur `git show`

`checkpatch.pl` attend un diff brut ou le format produit par `git format-patch` (en-têtes email `From:`, `Subject:`, etc.). `git show`/`git log` casse cette analyse pour deux raisons concrètes (vérifiées dans le source de `checkpatch.pl`) :

- Il cherche une ligne `From:` pour identifier l'auteur — `git show` affiche `Author:`, donc l'auteur détecté reste vide (`''`), d'où l'erreur *"Missing Signed-off-by: line by nominal patch author ''"*.
- `git show` indente le corps du commit de 4 espaces pour l'affichage → déclenche à tort *"Do not use whitespace before Signed-off-by:"*.

→ Toujours checker `git diff` (avant commit) ou le fichier généré par `git format-patch` (après commit) — jamais `git show`.

---

## 3. Tests fonctionnels

Une fois `checkpatch.pl` propre (0 erreur/warning), **teste réellement** le changement avant de committer. Ne pas committer un changement non testé même si le style est bon.

---

## 4. Commit

```bash
git commit -a              # tous les fichiers modifiés
# ou
git commit <fichiers>      # patches séparés
```

Bonnes pratiques du message :
- **Pourquoi**, pas **quoi** (le diff montre déjà le "quoi")
- Sujet court (`AREA: sous-système: résumé`), ligne vide, corps explicatif, ligne vide, `Signed-off-by:`
- `format.signoff = true` dans `.gitconfig` ajoute automatiquement la ligne `Signed-off-by:`

> 💡 Vérifie que le message décrit **exactement** ce que fait ton diff — un mainteneur relève immédiatement une incohérence entre message et code.

---

## 5. Trouver les mainteneurs

```bash
scripts/get_maintainer.pl <fichier modifié>
```

| Rôle dans la sortie | Où le mettre |
|---|---|
| `maintainer`, `supporter`, `commit_signer` | `--to` |
| `open list` (mailing lists) | `--cc` |

---

## 6. Générer le patch (avec destinataires intégrés)

```bash
git format-patch -1 <commit-id ou HEAD> \
  --to="Nom Mainteneur <email>" \
  --cc=liste@vger.kernel.org
```

- `<commit-id>` se récupère avec `git log` / `git log --oneline`, ou simplement `HEAD` pour le dernier commit.
- `-o <dossier>` pour générer ailleurs que le dossier courant.
- Le format `Nom <email>` ou juste `email` fonctionnent identiquement (testé) — la différence est purement l'affichage côté destinataire.

---

## 7. Vérifier le patch généré

```bash
scripts/checkpatch.pl 0001-*.patch
```

Cette fois le fichier est dans le **bon format** (en-têtes email réels) → résultat fiable, contrairement à un check sur `git show`.

---

## 8. Configurer git send-email (Gmail)

```ini
# ~/.gitconfig
[sendemail]
    smtpserver = smtp.gmail.com
    smtpserverport = 587
    smtpencryption = tls
    smtpuser = ton_email@gmail.com
    smtppass = mot_de_passe_application   # optionnel, voir note sécurité
```

```bash
sudo apt install git-email     # le paquet s'appelle git-email, pas git-send-email
chmod 600 ~/.gitconfig         # si smtppass est stocké en clair
```

> 🔒 Stocker `smtppass` en clair n'est raisonnable que sur une machine perso mono-utilisateur. Sinon, omets la ligne — Git le redemandera à chaque envoi (masqué).
>
> Le mot de passe d'application Gmail s'affiche avec des espaces tous les 4 caractères — retire-les à la saisie.

---

## 9. Dry-run puis envoi réel

```bash
git send-email --dry-run 0001-*.patch     # rien n'est envoyé, juste un aperçu
git send-email 0001-*.patch               # envoi réel
```

### Les deux prompts systématiques

À chaque **session** (= chaque exécution de la commande), `git send-email` pose toujours deux questions, qu'importe le contenu du fichier :

```
To whom should the emails be sent (if anyone)?
Message-ID to be used as In-Reply-To for the first email (if any)?
```

→ **Appuie sur Entrée sans rien taper** pour les deux. Le `To:`/`Cc:` déjà intégré dans le `.patch` (via `--to`/`--cc` au moment du `format-patch`) sera quand même utilisé — vérifié dans le code source : les deux listes de destinataires se **combinent**, elles ne s'écrasent pas.

> Si plusieurs fichiers `.patch` sont passés en une seule commande (une série), ces prompts n'apparaissent qu'**une fois** pour toute la session, pas par fichier.

### Le message "Cc list has been expanded"

`send-email` ajoute automatiquement en Cc les gens cités dans le message de commit (`Signed-off-by:`, `Reviewed-by:`, etc.) et demande confirmation. Contrôlé par `sendemail.confirm` :

| Valeur | Comportement |
|---|---|
| `auto` | confirme seulement si la liste a été étendue automatiquement |
| `always` | confirme à chaque envoi |
| `never` | n'envoyer ce réglage qu'en pleine maîtrise du processus |

### Pourquoi le cours s'arrête à `--dry-run` sans dire de l'enlever

Volontaire : ça évite d'envoyer un vrai patch cassé à un mainteneur en suivant un tuto sans réfléchir. Retirer `--dry-run` doit être une décision consciente, prise quand le patch est vraiment prêt — pas une étape automatique.

---

## 10. Pièges retenus pendant l'entraînement

| Piège | Explication courte |
|---|---|
| `checkpatch.pl` sur `git show` | Format non reconnu → faux positifs (auteur vide, whitespace) |
| Patch ne correspondant pas au message | Le message doit décrire fidèlement le diff réel |
| Croire que `--to` embarqué suffit à éviter tout prompt | Le prompt `To whom...` est générique, posé une fois par session — il faut juste valider (Entrée), pas le craindre |
| Package manquant | `git send-email` nécessite le paquet séparé `git-email` |
| `credential.helper = store` | Concerne HTTPS push/pull, **pas** `send-email` (qui lit `sendemail.smtppass`) |

---

## 11. Convention de nommage (non technique, mais suivie par la communauté)

Tous les mainteneurs kernel apparaissent en **Prénom Nom**, casse standard (`Greg Kroah-Hartman`, `Jiri Kosina`...). Aucun outil ne l'impose (vérifié dans `checkpatch.pl` — aucune règle là-dessus), mais s'aligner évite de se démarquer dans une liste de `Signed-off-by`.

```bash
git config --global user.name "Prénom Nom"   # ne change que les futurs commits
```

---

*Document vivant — à compléter au fil de l'apprentissage.*


# Guide de Poche : Workflow Git Professionnel pour le Noyau Linux

Ce document résume le flux de travail (workflow) standard et rigoureux utilisé par les développeurs et mainteneurs du noyau Linux pour travailler sur la branche **Mainline** sans polluer l'historique de l'arbre officiel.

---

## 1. Concepts Clés du Noyau

* **Pas de Merge (`git merge`)** : Le projet Linux s'interdit les commits de fusion sur le Mainline. L'historique doit rester parfaitement linéaire. On utilise uniquement des fichiers `.patch` plats et la commande `git rebase`.
* **Le Rebase Pré-publication** : C'est au développeur de s'adapter au Mainline, jamais l'inverse. Avant de générer un patch, vos commits doivent être déplacés au sommet du Mainline le plus récent.
* **Le Rôle de `git am`** : Utilisée par les mainteneurs, cette commande applique un fichier `.patch` et génère instantanément le commit associé en préservant l'identité de l'auteur d'origine.

---

## 2. Le Workflow de Développement Quotidien

### Étape A : Configuration et Travail Local
On isole chaque modification dans une branche dédiée (`work`) et on effectue des commits chirurgicaux.

```bash
# 1. Créer sa branche de travail à partir du master d'origine
git checkout -b work

# 2. Travailler sur le code (ex: pendant 2 semaines) et commiter proprement
git add drivers/usb/core/hub.c
git commit -m "HID: usbhid: add debug pr_info for endpoint detection"
```

### Étape B : Resynchronisation avec le Mainline (Le Rebase)
Pendant que vous codiez, le Mainline a bougé. Avant de créer vos patchs, vous devez mettre à jour votre code local.

```bash
# 1. Revenir sur master et récupérer les modifications mondiales sur Internet
git checkout master
git pull origin master

# 2. Revenir sur votre branche de travail
git checkout work

# 3. Réinsérer vos commits tout en haut du code officiel mis à jour
git rebase master
```

*Note : Si un conflit survient (modification de la même ligne de code), Git s'arrête. Nettoyez les balises `<<<<<<<` et `=======` dans votre éditeur, puis validez avec :*
```bash
git add <fichier_conflit>
git rebase --continue
```

---

## 3. Boîte à Outils : Modifier l'Historique Local

### Option 1 : Modifier le TOUT DERNIER commit (`HEAD`)
Si vous avez fait une faute de frappe dans le message ou oublié une ligne dans le tout dernier commit, n'utilisez pas le rebase. Utilisez l'amendement.

```bash
git commit --amend
```
*Cette commande ouvre l'éditeur pour modifier directement le message de votre dernier commit.*

### Option 2 : Modifier un commit plus ancien (Rebase Interactif)
Si l'erreur se trouve sur un commit plus profond dans l'historique (par exemple, le commit juste en dessous du HEAD), utilisez le rebase interactif en ciblant le parent du commit visé.

```bash
# Cibler les 2 derniers commits
git rebase -i HEAD~2
```

Dans la liste textuelle qui s'ouvre :
1. Remplacez le mot `pick` par **`reword`** (ou **`r`**) devant le commit à modifier.
2. Sauvegardez et fermez l'éditeur.
3. Dans la seconde fenêtre qui s'ouvre, modifiez le texte de votre message de commit, sauvegardez et fermez.

---

## 4. Génération des Patchs Officiels

Une fois votre branche `work` réalignée sur le `master` mis à jour et vos messages de commits corrigés, générez les fichiers `.patch` standard attendus par la communauté Linux :

```bash
git format-patch master..work
```
Cette commande crée des fichiers numérotés (ex: `0001-XXXX.patch`, `0002-XXXX.patch`) prêts à être analysés par le script `checkpatch.pl` et envoyés par e-mail aux différents mainteneurs.

