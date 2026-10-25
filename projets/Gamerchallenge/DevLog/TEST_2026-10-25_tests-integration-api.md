# DevLog — Tests d’intégration de l’API GamerChallenge

**Date : 25/10/2026**

## Objectif

Sécuriser les principales routes de l’API avec des tests automatisés, sans toucher à la base de développement.

## Démarche et décisions

- Création d’une base PostgreSQL réservée aux tests.
- Ajout d’une vérification de sécurité qui bloque l’exécution si la base utilisée n’est pas `gamerchallenge_test`.
- Application des migrations Prisma sur cette base dédiée.
- Séparation de la configuration Express dans `src/app.ts` et du démarrage du serveur dans `src/index.ts`, afin que les tests puissent importer l’application.
- Conservation de la méthode vue en cours : `node:test`, `node:assert` et `fetch`, sans ajouter Supertest.
- Exécution séquentielle des tests d’intégration avec `--test-concurrency=1`, car chaque fichier utilise le même port et la même base de test.

## Préparation des données de test

Création de helpers dans `src/tests/helpers/` :

- `auth.ts` : créer un utilisateur authentifié ou le connecter par la véritable route `/login` ;
- `challenges.ts` : créer un challenge lié à son auteur ;
- `participations.ts` : créer une participation ;
- `cleanup-database.ts` : supprimer les données dans un ordre compatible avec les clés étrangères.

`randomUUID()` est utilisé pour produire des valeurs uniques et éviter les conflits sur les champs soumis à une contrainte d’unicité.

## Tests réalisés

### Votes

- ajout d’un vote : `liked: true` ;
- second appel : retrait du vote avec `liked: false` ;
- retour du nombre de votes à sa valeur initiale ;
- refus sans authentification : `401` ;
- challenge inexistant : `404`.

### Participations

- création d’une participation ;
- suppression par son auteur ;
- refus de suppression par un autre utilisateur : `403`.

### Challenges

- création, modification et suppression ;
- refus de modification par un autre utilisateur : `403` ;
- contrôle de la base après les requêtes pour vérifier que les données interdites n’ont pas été modifiées.

### Authentification

- inscription et enregistrement du mot de passe sous forme de hash Argon2 ;
- connexion et création des cookies `accessToken` et `refreshToken` ;
- refus d’un mot de passe incorrect : `401` ;
- rotation du refresh token ;
- déconnexion, suppression du refresh token en base et nettoyage des cookies ;
- refus d’une inscription avec une adresse email déjà utilisée : `409`.

## Points de compréhension consolidés

Les tests suivent la structure AAA :

1. **Arrange** : préparer les utilisateurs, cookies et données nécessaires ;
2. **Act** : envoyer une vraie requête HTTP avec `fetch` ;
3. **Assert** : vérifier la réponse HTTP puis l’état réel de PostgreSQL avec Prisma.

Le helper `createAuthenticatedUser()` insère directement un utilisateur et fabrique un access token. Il est adapté aux tests des routes protégées, mais pas au test de l’inscription elle-même, qui doit obligatoirement passer par `/api/auth/register`.

Pour les tests de refresh et de logout, un second helper passe réellement par `/api/auth/login`, car ces scénarios nécessitent également la création d’un refresh token en base.

## Difficultés et corrections

- Une erreur d’import Prisma a été repérée en comparant le code proposé avec la structure réelle du projet. Le chemin correct est `src/database/prisma.ts`.
- Un premier fichier ZIP avait été créé depuis le dossier parent et incluait aussi le frontend. L’archive a ensuite été recréée depuis le dossier `backend` uniquement.
- Les fonctions de préparation de session avaient d’abord été placées dans le fichier de test Auth. Après relecture, elles ont été déplacées dans `helpers/auth.ts` afin de conserver une séparation claire entre préparation et scénario de test.
- Les réponses HTTP seules ne suffisent pas : les tests vérifient également que les créations, suppressions, refus et rotations sont réellement reflétés dans la base.

## Vérification finale

Les commandes suivantes passent sans erreur :

```bash
npm run typecheck
npm test
```

Les modifications ont ensuite été commit et poussées sur le dépôt.

## Bilan

Les principales fonctions du MVP disposent maintenant de tests d’intégration reproductibles sur une base isolée. Cette étape a aussi permis de clarifier la différence entre un helper de préparation, une requête HTTP réellement testée et une assertion portant sur l’état final de la base.
