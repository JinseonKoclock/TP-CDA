# 25/10/2026 — Sécurisation et migration de la base de test

## Objectif

Préparer une base PostgreSQL isolée pour les futurs tests d'intégration, tout en empêchant une exécution accidentelle sur la base de développement.

## Choix de configuration

La configuration de test est placée dans :

```text
src/tests/config/.env.test
```

La base locale utilise actuellement le port PostgreSQL `5432`. Le port `9546` est conservé dans la configuration pour une future utilisation avec Docker. Le serveur Express de test utilisera le port `7357`.

## Difficulté rencontrée

La première commande cherchait `.env.test` à la racine du backend :

```bash
node --env-file=.env.test --import tsx src/tests/ensure-test-database.ts
```

Elle a échoué avec :

```text
node: .env.test: not found
```

Le fichier existait, mais son chemin réel était `src/tests/config/.env.test`. J'ai donc corrigé le chemin et enregistré les commandes dans `package.json` pour éviter de le ressaisir manuellement.

## Travail réalisé

- Création de `src/tests/ensure-test-database.ts`.
- Lecture de `DATABASE_URL` et extraction du nom de la base.
- Blocage de l'exécution si la base ciblée n'est pas exactement `gamerchallenge_test`.
- Ajout de `test:db:check` pour vérifier la base avant toute opération de test.
- Ajout de `test:db:migrate` pour exécuter la vérification puis appliquer les migrations existantes.
- Exécution réussie de la vérification de sécurité.
- Application réussie des migrations Prisma sur la base de test.
- Ajout de `.env.test` dans `.gitignore`.
- Vérification avec `git status --short --ignored` : `!! src/tests/config/.env.test` confirme que le fichier contenant la configuration n'est pas suivi par Git.

## Ce que j'ai compris

Le chemin passé à `--env-file` est interprété depuis le dossier dans lequel la commande est lancée. Le fichier n'était donc pas introuvable à cause de son contenu, mais parce que la commande utilisait un mauvais chemin.

`prisma migrate deploy` applique les migrations déjà présentes au projet. Il convient ici à la base de test, car le but est de reproduire le schéma existant sans créer une nouvelle migration.

L'exécution de `test:db:migrate` commence par la vérification du nom de la base. Cette étape réduit le risque d'appliquer les futures opérations de nettoyage à la base de développement.

## Prochaine étape

Séparer la configuration Express dans `app.ts` et conserver uniquement le démarrage du serveur avec `app.listen()` dans `index.ts`.
