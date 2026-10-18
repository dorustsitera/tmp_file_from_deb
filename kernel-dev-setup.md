# Configuration d'un workspace de développement kernel (mainline + stable)

Guide pour mettre en place un environnement de développement et de build pour le kernel Linux, avec suivi simultané de mainline et d'une branche stable (ex: 6.18.y), via `git worktree` et `ccache`.

---

## Vue d'ensemble du résultat final

```
~/linux/                  ← clone principal, mainline (master)
~/linux-stable/           ← worktree, branche linux-6.18.y (suit le remote stable)
~/builds/
  ├── mainline/            ← fichiers générés du build mainline (O=)
  └── stable-6.18/         ← fichiers générés du build stable (O=)
~/.ccache/                 ← cache partagé entre les deux builds
```

Aucun fichier généré ne se mélange aux sources. Les deux working trees partagent le même historique git (`.git`), donc pas de duplication d'espace disque pour l'historique.

---

## 1. Cloner mainline

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git ~/linux
cd ~/linux
```

Ce clone télécharge l'historique complet de mainline. C'est le seul vrai clone réseau à faire — tout le reste réutilisera cet historique localement.

---

## 2. Ajouter le remote stable

```bash
git remote add stable https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
git fetch stable
```

`git fetch stable` télécharge les nouvelles refs (branches, tags) du repo stable. Comme une grande partie des commits stable sont déjà présents dans l'historique mainline, ce fetch est rapide et ne duplique pas l'espace disque de manière significative.

Vérifier que la branche `linux-6.18.y` existe bien côté remote :

```bash
git branch -r | grep stable/linux-6.18.y
```

---

## 3. Créer le worktree pour la branche stable

```bash
git worktree add --track -b linux-6.18.y ~/linux-stable stable/linux-6.18.y
```

Ce que fait cette commande :

- crée un nouveau dossier physique `~/linux-stable`
- crée une branche locale `linux-6.18.y`
- configure cette branche pour suivre (`--track`) `stable/linux-6.18.y`
- partage le même `.git` que `~/linux` (pas de duplication d'historique)

Vérifier que les deux worktrees sont bien listés :

```bash
git worktree list
```

```
~/linux         abc1234 [master]
~/linux-stable  def5678 [linux-6.18.y]
```

---

## 4. Workflow de mise à jour de la branche stable

Garder `linux-6.18.y` **vierge de tout commit local** — elle ne sert qu'à suivre les nouvelles releases stable.

```bash
cd ~/linux-stable
git pull
```

Ça fonctionne car la branche a un upstream configuré (`stable/linux-6.18.y`), contrairement à un detached HEAD sur un tag.

### Si tu veux modifier le code dans stable

Ne jamais committer directement sur `linux-6.18.y`. Créer une branche de travail séparée par-dessus :

```bash
cd ~/linux-stable
git checkout -b my-work-6.18 linux-6.18.y
# ... tes commits ici
```

Pour récupérer les nouveaux fixes stable après avoir commencé à travailler :

```bash
git checkout linux-6.18.y
git pull
git checkout my-work-6.18
git rebase linux-6.18.y
```

---

## 5. Configurer ccache (`~/.bashrc`)

```bash
# ─── ccache ───────────────────────────────────────────────
export PATH="/usr/lib/ccache:$PATH"
export CCACHE_DIR=~/.ccache
export KBUILD_BUILD_TIMESTAMP=''

# Compression du cache (réduit l'espace disque utilisé)
export CCACHE_COMPRESS=1
export CCACHE_COMPRESSLEVEL=6   # 1 (rapide) à 9 (max compression)

# Évite les miss dus aux chemins absolus dans les fichiers sources
export CCACHE_BASEDIR=~
# ──────────────────────────────────────────────────────────
```

### Pourquoi chaque ligne

| Variable | Rôle |
|---|---|
| `PATH="/usr/lib/ccache:$PATH"` | Place les symlinks ccache (`gcc`, `g++`, `clang`...) avant les vrais compilateurs dans le PATH. Rien à passer en argument à `make`. |
| `CCACHE_DIR` | Emplacement explicite du cache, plutôt que de dépendre du défaut implicite. |
| `KBUILD_BUILD_TIMESTAMP=''` | Supprime le timestamp embarqué par le build kernel, qui sinon change les flags de compilation à chaque build et provoque 100% de cache miss. |
| `CCACHE_COMPRESS` / `CCACHE_COMPRESSLEVEL` | Compresse les entrées du cache pour réduire l'espace disque, impact négligeable sur la vitesse. |
| `CCACHE_BASEDIR` | Normalise les chemins absolus dans les flags, utile si plusieurs clones/worktrees à des emplacements différents doivent partager le cache. |

### Activer la config

```bash
source ~/.bashrc

# Vérifier que le wrapper est actif
which gcc
# → /usr/lib/ccache/gcc   ✓
```

### Fixer la taille du cache (une seule fois)

```bash
ccache --set-config max_size=25G
```

25 Go est une marge confortable pour couvrir mainline + stable en parallèle, vu le volume de fichiers identiques entre les deux trees.

---

## 6. Builder

Aucun argument supplémentaire à passer à `make` — tout est géré par les variables d'environnement.

```bash
# Mainline
cd ~/linux
make O=~/builds/mainline defconfig
make O=~/builds/mainline -j$(nproc)

# Stable 6.18
cd ~/linux-stable
make O=~/builds/stable-6.18 defconfig
make O=~/builds/stable-6.18 -j$(nproc)
```

`O=` isole les fichiers générés (`.o`, `vmlinux`, `.config`, modules...) dans le dossier indiqué — l'arbre source reste intact dans les deux cas.

---

## 7. Vérifier que ccache travaille

```bash
ccache -s
```

Affiche les statistiques : hits, misses, taille utilisée. Après le deuxième build (stable, une fois mainline déjà compilé), un taux de hit élevé est attendu sur les fichiers communs aux deux versions.

---

## Résumé des commandes, dans l'ordre

```bash
# 1. Clone mainline
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git ~/linux
cd ~/linux

# 2. Remote stable
git remote add stable https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
git fetch stable

# 3. Worktree stable
git worktree add --track -b linux-6.18.y ~/linux-stable stable/linux-6.18.y

# 4. ccache (à ajouter dans ~/.bashrc, puis source ~/.bashrc)
export PATH="/usr/lib/ccache:$PATH"
export CCACHE_DIR=~/.ccache
export KBUILD_BUILD_TIMESTAMP=''
export CCACHE_COMPRESS=1
export CCACHE_COMPRESSLEVEL=6
export CCACHE_BASEDIR=~
ccache --set-config max_size=25G

# 5. Build
make -C ~/linux O=~/builds/mainline defconfig
make -C ~/linux O=~/builds/mainline -j$(nproc)

make -C ~/linux-stable O=~/builds/stable-6.18 defconfig
make -C ~/linux-stable O=~/builds/stable-6.18 -j$(nproc)
```
