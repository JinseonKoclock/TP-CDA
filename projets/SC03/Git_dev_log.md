# Git - Organisation du travail avec Git

**Date :** 15/07/2026

---

# 🎯 Contexte

Depuis plusieurs challenges, je rencontrais toujours le même problème.

À chaque fois que je souhaitais récupérer le code du formateur, je craignais de perdre une partie de mon travail personnel, notamment mes DevLogs.

Cette situation revenait régulièrement et m'obligeait à hésiter avant d'utiliser certaines commandes Git (`reset --hard`, changement de branche, récupération du dépôt du formateur...).

Je passais alors beaucoup de temps à essayer de protéger mes fichiers de documentation au lieu de me concentrer sur le challenge.

Le 15/07/2026, j'ai finalement identifié l'origine du problème : mes DevLogs n'avaient pas leur place dans le dépôt Git du projet.

Ils constituent une documentation personnelle destinée au Titre Professionnel et non une partie du code source.

---

# ⚠️ Difficultés rencontrées

Au début, j'ai confondu plusieurs notions :

- les branches locales ;
- les branches distantes ;
- `origin` et `prof` ;
- `git fetch` et `git pull`.

J'ai également créé une branche à partir de `prof/SC03E01-CI` alors que le code utilisé pendant le cours avait en réalité été poussé sur `prof/main`.

Pendant plusieurs minutes, je pensais que le code du formateur n'avait pas été publié.

Après vérification des commits, j'ai finalement constaté que le dernier commit se trouvait bien sur :

```text
prof/main
```

---

# ✅ Procédure retenue

Avant toute modification :

```bash
git status
```

Vérifier les changements éventuels.

Si des fichiers personnels sont présents :

```bash
git stash
```

Récupérer les dernières informations du dépôt du formateur :

```bash
git fetch prof
```

Vérifier les derniers commits :

```bash
git log --oneline prof/main -5
```

Replacer le dépôt sur la version du formateur :

```bash
git switch master
git reset --hard prof/main
```

Créer ensuite une branche personnelle pour le challenge :

```bash
git switch -c Challenge_SC03E01
```

Ainsi :

- le dépôt est identique à celui du formateur ;
- le développement est réalisé sur une branche personnelle ;
- les modifications restent indépendantes.

---

# 💡 Ce que j'ai appris

J'ai compris que :

- `git fetch` récupère uniquement les informations du dépôt distant ;
- `git reset --hard prof/main` remplace le contenu de la branche courante par celui du formateur ;
- il est préférable de développer sur une branche personnelle plutôt que directement sur `master`.

J'ai également compris que le code du formateur peut être publié sur une branche différente de celle que j'imaginais. Il est donc important de vérifier les derniers commits avant de conclure que le dépôt n'a pas été mis à jour.

---

# 📝 Nouvelle organisation

Cette séance m'a permis de modifier complètement mon organisation de travail.

Désormais :

- les dépôts Git ne contiendront que le code source du projet ;
- les DevLogs, les notes personnelles, les captures d'écran et les documents du Titre Professionnel seront conservés dans un dossier indépendant (`TP`).

Cette séparation me permettra de récupérer facilement les mises à jour du formateur sans risquer de perdre ma documentation personnelle.

Avec le recul, ce n'était pas un problème Git, mais un problème d'organisation de mes fichiers.

# Ce que j'ai appris

Au cours de cette séance, j'ai également clarifié plusieurs notions que je mélangeais encore.

Au départ, j'avais tendance à considérer Git comme un langage informatique.

En réalité, Git est un **système de gestion de versions (Version Control System)**.

Son rôle est de conserver l'historique des modifications d'un projet, de créer des branches, de revenir à une version précédente et de faciliter le travail en équipe.

J'ai également mieux compris le rôle de GitHub.

GitHub n'est pas Git : c'est une plateforme qui héberge des dépôts Git afin de sauvegarder le code et de collaborer avec d'autres développeurs.

Cette distinction m'a aidé à mieux comprendre pourquoi je travaillais avec deux dépôts distants différents (`origin` et `prof`) pendant les challenges.

Cette séance ne m'a pas seulement appris de nouvelles commandes Git.

Elle m'a surtout permis de mieux comprendre l'organisation générale d'un projet Git et la différence entre Git, GitHub, les dépôts distants (`origin`, `prof`) et les branches de travail.