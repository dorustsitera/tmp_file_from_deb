# Guide de référence — Git Workflow pour LKM out-of-tree (solo)

> **Dépôt :** `git@github.com:dorustsitera/linux-kernel-module.git`
> **Modèle :** Solo Flow (Git Flow allégé)
> **Dernière mise à jour :** 2025-06

---

## Table des matières

1. [Vue d'ensemble du modèle](#1-vue-densemble-du-modèle)
2. [Structure des branches](#2-structure-des-branches)
3. [Convention des messages de commit](#3-convention-des-messages-de-commit)
4. [Workflows courants](#4-workflows-courants)
5. [Commandes de référence rapide](#5-commandes-de-référence-rapide)
6. [Checklist avant chaque commit](#6-checklist-avant-chaque-commit)
7. [Versioning (tags)](#7-versioning-tags)
8. [Résolution de problèmes courants](#8-résolution-de-problèmes-courants)
9. [Structure réelle du dépôt](#9-structure-réelle-du-dépôt)

---

## 1. Vue d'ensemble du modèle

```
main ──●──────────────────────────────────●──────── (stable, taggée)
        \                                /
dev      ●────●────●────●────●────●────●           (travail quotidien)
                    \              /
feature              ●────●────●                   (branche courte)
```

**Règle d'or :** `main` est toujours propre, buildable, et chargeable avec `insmod`.

| Branche | Durée de vie | Peut être cassée ? | Push direct ? |
|---|---|---|---|
| `main` | Permanente | ❌ Non | ❌ Jamais |
| `dev` | Permanente | ✅ Oui | ✅ Oui |
| `feature/*` | Courte (1-5 jours) | ✅ Oui | ✅ Oui |
| `fix/*` | Très courte | ✅ Oui | ✅ Oui |

---

## 2. Structure des branches

### `main`

- Contient uniquement du code **validé et testé** (`insmod`/`rmmod` propres)
- Chaque commit sur `main` correspond à un **tag de version** (`v0.1`, `v1.0`...)
- Jamais de commit direct — uniquement via merge depuis `dev`

### `dev`

- Branche de **travail quotidien**
- Peut contenir du code temporairement non fonctionnel
- Commits petits et fréquents
- Rebaser régulièrement si `main` avance

### `feature/<nom>` et `fix/<nom>`

- Créée depuis `dev`, fusionnée dans `dev`
- Nommage : `feature/proc-interface`, `fix/null-ptr-oops`, `feature/netfilter-hook`
- Supprimée après merge
- Utilisée quand la modification touche **une interface ou un sous-système entier**

---

## 3. Convention des messages de commit

Format : `<type>(<scope optionnel>): <description courte>`

### Types

| Type | Usage |
|---|---|
| `feat` | Nouvelle fonctionnalité (nouveau hook, nouveau ioctl...) |
| `fix` | Correction de bug (oops, memory leak, undefined symbol) |
| `build` | Makefile, Kbuild, DKMS |
| `test` | Ajout ou correction de tests manuels / scripts de test |
| `refactor` | Refactoring sans changement de comportement |
| `docs` | Documentation, commentaires |
| `chore` | .gitignore, CI, config |
| `release` | Commit de merge vers main pour une release |

### Exemples concrets

```
feat(netfilter): add NF_HOOK_OPS for egress filtering on eth0

fix(proc): fix null pointer dereference in read handler

build: set KDIR to support cross-compilation for arm64

test: add insmod/rmmod stress test script

release: v0.3 - netfilter hook + proc interface
```

### Corps du message (optionnel mais recommandé)

```
feat(ioctl): add MYMODULE_GET_STATS command

Returns cumulative packet count and byte count since module load.
Tested on Linux 6.8.0-45, x86_64 and Linux 6.6.0 LTS.
insmod/rmmod: clean (verified with kmemleak).
No regressions on existing ioctls.
```

---

## 4. Workflows courants

### 4.1 Début de session de travail

```bash
# Toujours partir de dev à jour
git checkout dev
git pull origin dev
```

### 4.2 Commit quotidien sur `dev` (modification simple)

```bash
# Après modification de lkm_template.c (ou tout autre fichier source)
make clean && make FNAME_C=lkm_template      # compiler
sudo insmod lkm_template.ko                  # charger
sudo dmesg | tail -20                        # vérifier les logs kernel
sudo rmmod lkm_template                      # décharger proprement

git status
git add -p                                   # ajouter de façon sélective (recommandé)
git commit -m "fix(init): correct module_init return code"
git push origin dev
```

### 4.3 Nouvelle fonctionnalité (feature branch)

```bash
# 1. Créer la branche depuis dev
git checkout dev
git checkout -b feature/proc-interface

# 2. Modifier les fichiers source à la racine
nvim lkm_template.c       # ou le fichier concerné

# 3. Compiler et tester
make clean && make FNAME_C=lkm_template
sudo insmod lkm_template.ko
cat /proc/lkm_template                       # tester l'interface
sudo dmesg | tail -20
sudo rmmod lkm_template

# 4. Commiter
git add -p
git commit -m "feat(proc): add /proc/lkm_template read handler"

# continuer les commits sur cette branche...
git commit -m "feat(proc): add write support for runtime config"

# 5. Fusionner dans dev (rebase pour historique propre)
git checkout dev
git rebase feature/proc-interface

# OU squash si les commits intermédiaires sont du bruit :
# git merge --squash feature/proc-interface
# git commit -m "feat(proc): add /proc interface (read + write)"

# 6. Supprimer la branche
git branch -d feature/proc-interface
git push origin dev
git push origin --delete feature/proc-interface   # si elle a été poussée
```

### 4.4 Hotfix urgent (bug critique sur dev)

```bash
git checkout dev
git checkout -b fix/null-ptr-init

# Corriger le bug
nvim lkm_template.c

make clean && make FNAME_C=lkm_template
sudo insmod lkm_template.ko
sudo dmesg | tail -5                         # confirmer que le fix tient
sudo rmmod lkm_template

git add -p
git commit -m "fix(init): fix null ptr deref when kmalloc fails at init"

git checkout dev
git rebase fix/null-ptr-init
git branch -d fix/null-ptr-init
git push origin dev
```

### 4.5 Release : merger `dev` dans `main` et taguer

```bash
# S'assurer que dev est propre et testée
git checkout dev
git pull origin dev
make clean && make FNAME_C=lkm_template
sudo insmod lkm_template.ko
# effectuer tous les tests manuels...
sudo rmmod lkm_template

# Relire les commits qui vont merger
git log --oneline dev ^main

# Merger dans main
git checkout main
git merge --no-ff dev -m "release: v0.2 - proc interface + netfilter hook"

# Taguer
git tag -a v0.2 -m "Version 0.2
- /proc/lkm_template interface (read + write)
- NF_HOOK_OPS for egress filtering
- Fix null ptr on kmalloc failure
Tested: Linux 6.8.0-45 x86_64"

# Pousser main + le tag
git push origin main
git push origin v0.2

# Retourner sur dev
git checkout dev
```

---

## 5. Commandes de référence rapide

### Build avec le Makefile Kaiwan (FNAME_C obligatoire)

```bash
make FNAME_C=lkm_template                   # builder
make FNAME_C=lkm_template clean             # nettoyer
make FNAME_C=lkm_template install           # installer dans /lib/modules/
make FNAME_C=lkm_template MYDEBUG=y         # build en mode debug
make FNAME_C=lkm_template ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-  # cross-compile
make FNAME_C=lkm_template sa               # toutes les analyses statiques
make FNAME_C=lkm_template checkpatch       # vérifier le style kernel
make FNAME_C=lkm_template help             # voir toutes les cibles disponibles
```

### Charger / décharger

```bash
sudo insmod lkm_template.ko                 # charger le module
sudo rmmod lkm_template                     # décharger
sudo modinfo lkm_template.ko                # infos sur le module
lsmod | grep lkm_template                   # vérifier si chargé
sudo dmesg | tail -20                       # logs kernel
sudo dmesg -wH                              # logs en temps réel (Ctrl+C pour quitter)
```

### Navigation Git

```bash
git branch -a                               # toutes les branches
git log --oneline --graph --all             # vue graphe complet
git log --oneline -10                       # 10 derniers commits
git diff HEAD~1                             # diff avec le commit précédent
git show v0.1                               # détails d'un tag
```

### Staging sélectif (recommandé)

```bash
git add -p                                  # ajouter morceau par morceau (hunk)
git diff --staged                           # voir ce qui va être commité
git restore --staged <fichier>              # désindexer un fichier
```

### Annuler / corriger

```bash
git commit --amend                          # modifier le dernier commit (avant push)
git restore <fichier>                       # abandonner les modifs locales
git reset HEAD~1                            # défaire le dernier commit (garder les modifs)
git push origin dev --force                 # forcer le push après reset (avec précaution)
git stash                                   # mettre de côté les modifs en cours
git stash pop                               # récupérer le stash
```

### Tags

```bash
git tag                                     # lister tous les tags
git tag -a v0.1 -m "Version initiale"      # créer un tag annoté
git push origin --tags                      # pousser tous les tags
git checkout v0.1                           # revenir à une version précise
```

---

## 6. Checklist avant chaque commit

Avant tout `git commit`, vérifier :

- [ ] `make clean && make FNAME_C=<module>` : compilation sans warning
- [ ] `sudo insmod <module>.ko` : chargement sans erreur
- [ ] `sudo dmesg | tail -20` : aucun oops, aucun warning kernel
- [ ] `sudo rmmod <module>` : déchargement propre
- [ ] `sudo dmesg | tail -5` : aucun message d'erreur au rmmod
- [ ] `git diff --staged` : relire le diff avant de commiter
- [ ] Message de commit : type + scope + description courte

### Avant une release (merge → main)

- [ ] Tous les points ci-dessus
- [ ] `uname -r` : noter la version kernel testée dans le tag
- [ ] `make FNAME_C=<module> checkpatch` : style kernel propre
- [ ] Optionnel : `make FNAME_C=<module> sa` : analyse statique propre
- [ ] Optionnel : kmemleak propre (`sudo cat /sys/kernel/debug/kmemleak`)
- [ ] `git log --oneline dev ^main` : relire tous les commits qui vont merger
- [ ] Rédiger les notes de release dans le message du tag

---

## 7. Versioning (tags)

Utilise le versioning sémantique adapté aux modules kernel :

```
v<MAJEUR>.<MINEUR>.<PATCH>
```

| Niveau | Incrémenter quand... |
|---|---|
| MAJEUR | Changement d'interface (ioctl ABI, sysfs attrs, proc format) |
| MINEUR | Nouvelle fonctionnalité, nouveau sous-système supporté |
| PATCH | Bugfix, amélioration sans nouvelle interface |

**Exemples :**

```
v0.1   — template de base, init/exit propres
v0.2   — ajout interface /proc
v0.3   — ajout hook netfilter
v1.0   — interface stable, testé sur plusieurs kernels LTS
v1.1   — ajout support arm64
v1.1.1 — fix memory leak sur arm64
```

---

## 8. Résolution de problèmes courants

### "nothing to commit" mais des fichiers .ko traînent

```bash
cat .gitignore                              # vérifier que *.ko est ignoré
git status --ignored                        # voir les fichiers ignorés
```

### `make` échoue avec "No rule to make target"

```bash
# FNAME_C est obligatoire avec ce Makefile
make FNAME_C=lkm_template                  # toujours spécifier le nom
```

### Merger feature dans dev crée des conflits

```bash
git checkout dev
git rebase feature/ma-branche
# En cas de conflit :
nvim fichier_en_conflit.c                  # éditer et résoudre
git add fichier_en_conflit.c
git rebase --continue
# Abandonner le rebase :
git rebase --abort
```

### Annuler un merge dans main avant le push

```bash
git checkout main
git reset --hard HEAD~1                    # défaire le merge
# Ne jamais faire ça APRÈS git push origin main
```

### Retrouver un commit perdu

```bash
git reflog                                 # historique de tous les mouvements HEAD
git checkout <sha>                         # revenir à ce commit
```

### Module qui refuse de se charger après un rebase

```bash
make clean && make FNAME_C=lkm_template   # recompiler complètement
sudo dmesg | tail -30
sudo modprobe -v lkm_template             # plus verbeux que insmod
```

### `insmod: ERROR: could not insert module`

```bash
sudo dmesg | tail -10                     # lire le vrai message d'erreur
# Causes fréquentes :
# - module déjà chargé → sudo rmmod lkm_template d'abord
# - version kernel mismatch → make clean && make FNAME_C=lkm_template
# - symbole manquant → vérifier les #include dans le .c
```

---

## 9. Structure réelle du dépôt

```
linux-kernel-module/          ← racine du dépôt (root dir du module)
├── .gitignore
├── LICENSE
├── Makefile                  ← Makefile "better" de Kaiwan N Billimoria
├── lkm_template.c            ← fichier source principal
└── docs/
    └── LKM-GIT-WORKFLOW.md   ← ce fichier
```

> Les fichiers source `.c` et `.h` sont placés **directement à la racine**,
> pas dans des sous-dossiers. Le Makefile cible le fichier via `FNAME_C`.

### `.gitignore` pour LKM out-of-tree

```gitignore
# Kernel build artifacts
*.ko
*.mod
*.mod.c
*.mod.o
*.o
*.a
*.s
*.symtypes
*.order
*.cmd

# Répertoires kbuild
.tmp_versions/
__pycache__/
bkp/

# Fichiers kbuild racine
Module.symvers
modules.order

# Éditeurs
*.swp
*~
.clangd/
compile_commands.json
.vscode/
```

---

*Guide généré pour le dépôt `dorustsitera/linux-kernel-module` — à adapter selon l'évolution du projet.*
