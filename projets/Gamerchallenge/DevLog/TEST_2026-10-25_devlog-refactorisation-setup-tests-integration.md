# DevLog — Mutualisation du setup des tests d’intégration

**Date : 25/10/2026**

## Contexte

Après avoir terminé les principaux tests d’intégration du backend, j’ai comparé ma structure avec un fichier utilisé dans le projet du formateur.

Ce fichier regroupait notamment :

- le démarrage du serveur Express ;
- la préparation de la base de test avec Docker ;
- le nettoyage des tables avant chaque test ;
- l’arrêt du serveur et de la base après les tests.

Cette comparaison m’a permis de remarquer que mes fichiers de test répétaient tous les mêmes hooks `before`, `beforeEach` et `after`.

## Réflexion avant la modification

Je disposais déjà de la structure suivante :

```text
src/tests/
├── config/
├── helpers/
│   ├── auth.ts
│   ├── challenges.ts
│   ├── cleanup-database.ts
│   └── participations.ts
└── ensure-test-database.ts
```

J’ai d’abord vérifié si `cleanup-database.ts` était correctement placé dans `helpers/`. Ce fichier a bien le même rôle général que les helpers de création : il fournit une fonction réutilisable pour préparer ou remettre à zéro l’état nécessaire aux tests.

Il n’était donc pas nécessaire de déplacer ce fichier ni de créer une nouvelle fonction de nettoyage.

J’ai également distingué deux sujets différents :

1. mutualiser le cycle de vie commun des tests ;
2. ajouter plus tard une base PostgreSQL exécutée avec Docker.

Le code Docker du formateur n’a pas été repris pendant cette refactorisation. Le système actuel utilisant une base locale dédiée fonctionne déjà et dispose d’une protection contre l’utilisation accidentelle de la base de développement.

## Refactorisation réalisée

Création du fichier :

```text
src/tests/setup-integration.ts
```

Ce fichier centralise maintenant :

- le lancement du serveur Express avant les tests d’un fichier ;
- le nettoyage de la base avant chaque scénario avec `clearTestDatabase()` ;
- le dernier nettoyage après les tests ;
- l’arrêt du serveur HTTP ;
- la fermeture de la connexion Prisma ;
- les constantes communes `TEST_PORT` et `API_URL`.

Chaque fichier de test appelle désormais :

```ts
setupIntegrationTests();
```

Les modifications ont été appliquées aux fichiers suivants :

```text
src/controllers/auth.controller.spec.test.ts
src/controllers/challenges.controller.spec.test.ts
src/controllers/participations.controller.spec.test.ts
src/controllers/votes.controller.spec.test.ts
```

## Séparation des responsabilités

Le setup commun ne crée pas les données propres à un scénario.

Par exemple, le nettoyage de la base est commun :

```ts
beforeEach(async () => {
  await clearTestDatabase();
});
```

En revanche, la création d’un auteur, d’un participant et d’un challenge reste dans le fichier de test des participations, car ces données ne concernent pas tous les contrôleurs.

La structure obtenue est donc :

```text
setup-integration.ts
→ cycle de vie commun du serveur et de la base

helpers/
→ création et nettoyage des données de test

*.spec.test.ts
→ préparation spécifique et scénarios Arrange / Act / Assert
```

## Méthode de vérification

Les fichiers ont été modifiés progressivement afin de limiter les erreurs :

1. refactorisation du test des participations ;
2. exécution de ce fichier seul ;
3. refactorisation du test des votes ;
4. nouvelle vérification ;
5. même travail pour les challenges puis l’authentification.

Commande utilisée pour exécuter un seul fichier :

```bash
node --env-file=src/tests/config/.env.test --import tsx --test --test-concurrency=1 src/controllers/participations.controller.spec.test.ts
```

Cette méthode permettait de savoir immédiatement quel changement avait provoqué une éventuelle erreur.

## Incident rencontré

Pendant la modification, le contenu du fichier de test Auth a été effacé accidentellement et `Ctrl + Z` ne permettait pas de le restaurer.

Comme le fichier avait déjà été commit et poussé, sa version enregistrée dans Git a d’abord été vérifiée avec :

```bash
git show HEAD:src/controllers/auth.controller.spec.test.ts
```

Le fichier a ensuite pu être restauré sans toucher aux autres modifications en cours :

```bash
git restore src/controllers/auth.controller.spec.test.ts
```

Cet incident m’a permis de voir concrètement que Git ne sert pas seulement à partager le code : il permet également de récupérer précisément un fichier sans annuler tout le travail en cours.

## Bilan

Les scénarios de test n’ont pas été modifiés. La refactorisation a uniquement supprimé la duplication liée au serveur, au nettoyage de la base et à la fermeture des connexions.

Le résultat est plus lisible : chaque fichier `spec.test.ts` se concentre maintenant sur les données et les comportements propres au contrôleur testé.

L’ajout de Docker reste une étape séparée. Il pourra être réalisé plus tard sans mélanger la gestion du conteneur PostgreSQL avec les scénarios de test eux-mêmes.
