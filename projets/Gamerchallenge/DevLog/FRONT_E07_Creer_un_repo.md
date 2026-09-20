# Créer un repository personnel à partir d'un projet existant

## Objectif

Créer une copie personnelle d'un projet Git existant afin de pouvoir expérimenter et développer librement sans modifier le repository de l'équipe.

Dans mon cas :

```text
gamerChallenge/
├── front/       → repository Front de l'équipe
├── back/        → repository Back de l'équipe
├── frontend/    → mon Front personnel
└── backend/     → mon Back personnel
```

Le projet `frontend` est une copie locale du projet `front`, mais il possède ensuite son propre repository GitHub.

---

# 1. Copier le projet

Depuis le gestionnaire de fichiers, copier le dossier :

```text
front
```

puis renommer la copie :

```text
frontend
```

Cette méthode conserve les fichiers présents localement, y compris les modifications non encore commit.

Avant la copie, il est préférable de sauvegarder tous les fichiers ouverts dans VS Code.

---

# 2. Vérifier le repository Git copié

Se placer dans la copie :

```bash
cd /var/www/html/gamerChallenge/frontend
```

Vérifier le remote :

```bash
git remote -v
```

Dans mon cas, le résultat était encore :

```text
origin  git@github.com:O-clock-Jakarta/projet-gamerchallenges-front.git (fetch)
origin  git@github.com:O-clock-Jakarta/projet-gamerchallenges-front.git (push)
```

Cela signifie que le dossier `.git` a également été copié.

Le nouveau dossier `frontend` est donc encore lié au repository Git de l'équipe.

---

# 3. Supprimer l'ancien historique Git

Avant cette commande, vérifier impérativement que le terminal se trouve dans le bon dossier :

```bash
pwd
```

Résultat attendu :

```text
/var/www/html/gamerChallenge/frontend
```

Supprimer ensuite uniquement les informations Git :

```bash
rm -rf .git
```

Cette commande supprime :

- l'historique Git local ;
- les branches Git ;
- les remotes ;
- la relation avec l'ancien repository.

Elle ne supprime pas le code source du projet.

---

# 4. Vérifier que le projet n'est plus un repository Git

```bash
git status
```

Résultat attendu :

```text
fatal: not a git repository
```

Le code est toujours présent, mais le dossier n'est maintenant plus lié à Git.

---

# 5. Initialiser mon nouveau repository local

```bash
git init
```

Définir `main` comme branche principale :

```bash
git branch -M main
```

Vérifier :

```bash
git status
```

Le projet doit maintenant être un nouveau repository Git sans aucun commit.

---

# 6. Vérifier le `.gitignore`

Avant d'ajouter les fichiers, vérifier que les fichiers sensibles ou inutiles sont ignorés.

Exemples :

```gitignore
node_modules
.env
dist
```

Le fichier :

```text
.env
```

ne doit pas être envoyé sur GitHub.

En revanche, un fichier :

```text
.env.example
```

peut être versionné afin d'indiquer les variables nécessaires au fonctionnement du projet sans exposer les valeurs sensibles.

---

# 7. Ajouter les fichiers

```bash
git add .
```

Puis vérifier :

```bash
git status
```

Avant de continuer, vérifier notamment que `.env` n'apparaît pas parmi les fichiers qui seront commit.

---

# 8. Créer le premier commit

```bash
git commit -m "chore: Initialiser le frontend personnel"
```

Le nouveau repository possède maintenant son propre historique Git.

---

# 9. Créer un repository vide sur GitHub

Sur GitHub :

1. créer un nouveau repository ;
2. choisir un nom, par exemple :

```text
Frontend_Jinseon
```

3. ne pas créer automatiquement de README ;
4. ne pas ajouter de `.gitignore` ;
5. ne pas ajouter de licence.

Le repository GitHub doit être vide car le projet existe déjà localement.

---

# 10. Connecter le repository local au nouveau GitHub

Ajouter le nouveau remote :

```bash
git remote add origin git@github.com:JinseonKoclock/Frontend_Jinseon.git
```

Vérifier :

```bash
git remote -v
```

Résultat attendu :

```text
origin  git@github.com:JinseonKoclock/Frontend_Jinseon.git (fetch)
origin  git@github.com:JinseonKoclock/Frontend_Jinseon.git (push)
```

À cette étape, le projet local est maintenant relié à mon repository personnel et non plus au repository de l'équipe.

---

# 11. Envoyer le projet sur GitHub

Pour le premier push :

```bash
git push -u origin main
```

L'option :

```text
-u
```

permet d'associer la branche locale `main` à la branche distante `origin/main`.

Après cette première fois, les prochains push pourront simplement être effectués avec :

```bash
git push
```

---

# Nettoyage du repository de l'équipe

Après avoir vérifié que la copie personnelle contient bien toutes les modifications, il est possible de nettoyer le repository original.

Dans mon cas :

```bash
cd /var/www/html/gamerChallenge/front
```

Les fichiers déjà suivis par Git mais modifiés localement ont été restaurés :

```bash
git restore src/App.tsx src/pages/ChallengesPage/ChallengesPage.tsx src/pages/HomePage/HomePage.tsx
```

Les fichiers créés uniquement pour mes tests ont ensuite été supprimés manuellement.

Après nettoyage :

```bash
git status
```

Résultat :

```text
nothing to commit, working tree clean
```

---

# Suppression de la branche de test

La branche utilisée pour les expérimentations était :

```text
feat/connection_B-F
```

Comme une branche actuellement utilisée ne peut pas être supprimée directement, je suis d'abord retournée sur `develop` :

```bash
git switch develop
```

Puis j'ai vérifié :

```bash
git status
```

Enfin, j'ai supprimé la branche locale devenue inutile :

```bash
git branch -D feat/connection_B-F
```

Le repository de l'équipe est ainsi revenu sur :

```text
develop
```

avec un working tree propre.

---

# Résultat final

```text
gamerChallenge/
│
├── front/
│   └── repository Git de l'équipe
│
├── back/
│   └── repository Git de l'équipe
│
├── frontend/
│   └── mon repository Front personnel
│
└── backend/
    └── mon repository Back personnel
```

Je peux maintenant expérimenter sur :

```text
frontend
backend
```

sans modifier directement les repositories de l'équipe.

---

# Ce que je retiens

## Copier un dossier Git copie aussi `.git`

Une copie classique du dossier peut également copier le dossier caché :

```text
.git
```

La copie reste alors liée au même historique et au même remote Git.

---

## `rm -rf .git` ne supprime pas le code

Cette commande supprime les informations Git du dossier, pas les fichiers du projet.

Elle permet de repartir avec :

```bash
git init
```

et de créer un nouveau repository indépendant.

---

## Les modifications non commit sont des fichiers locaux

Une modification n'a pas besoin d'être commit pour être copiée avec le dossier.

Si le fichier est enregistré sur le disque, il est également présent dans la copie.

---

## Supprimer une branche ne supprime pas forcément les modifications locales

Les modifications non commit appartiennent au working directory.

Elles ne sont pas automatiquement supprimées simplement parce qu'une branche est supprimée ou changée.

Il faut distinguer :

```text
Repository
   │
   ├── commits
   ├── branches
   │
   └── working directory
          ├── fichiers modifiés
          └── fichiers untracked
```

---

## `git restore` et les fichiers untracked sont différents

`git restore` permet de restaurer un fichier déjà suivi par Git.

En revanche, un fichier `untracked` n'existe pas dans l'historique Git.

Il doit donc être supprimé séparément si je ne souhaite pas le conserver.

---

## Toujours vérifier avant de supprimer

Avant d'utiliser :

```bash
rm -rf .git
```

il faut vérifier le dossier courant avec :

```bash
pwd
```

Avant de supprimer les modifications du projet original, il faut également vérifier que la copie personnelle contient bien les fichiers nécessaires.

Cette vérification évite de perdre le travail réalisé localement.