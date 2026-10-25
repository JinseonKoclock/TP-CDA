# 25/10/2026 — Création d'une base PostgreSQL dédiée aux tests

## Objectif

Préparer les futurs tests d'intégration du backend GamerChallenge sans modifier les données de la base de développement.

## Réflexion

Swagger permettait déjà de vérifier manuellement les routes, mais les tests automatisés vont créer et supprimer des utilisateurs, des challenges et des votes. J'ai donc choisi d'isoler ces données dans une base réservée aux tests.

La stratégie retenue pour les tests d'intégration est :

```text
node:test + node:assert + fetch + base PostgreSQL de test
```

## Travail réalisé

- Connexion à PostgreSQL avec le compte administrateur.
- Création du rôle `gamerchallenge_test`.
- Vérification de sa présence avec `\du gamerchallenge_test`.
- Constat que `\l gamerchallenge_test` ne retournait aucune ligne : le rôle existait, mais la base n'avait pas encore été créée.
- Remplacement du mot de passe provisoire saisi littéralement par un véritable mot de passe de test.
- Création de la base `gamerchallenge_test` avec `gamerchallenge_test` comme propriétaire.
- Vérification de la connexion avec le compte de test.
- Vérification de la base et de l'utilisateur actifs avec `current_database()` et `current_user`.

## Ce que j'ai compris

`CREATE USER` crée un rôle PostgreSQL autorisé à se connecter, mais ne crée pas automatiquement une base de données. Il faut ensuite créer la base séparément et définir son propriétaire.

La séparation entre la base de développement et la base de test protège les données utilisées pendant le développement. Les futurs tests pourront préparer et nettoyer leurs propres données sans toucher à la base principale.

## Prochaine étape

Créer le fichier `.env.test`, y enregistrer la connexion à `gamerchallenge_test`, puis ajouter une vérification empêchant les tests de s'exécuter sur la base de développement.
