# 25/10/2026 — Premier test d'intégration du vote

## Objectif

Automatiser le scénario de vote sur un challenge avec la base PostgreSQL réservée aux tests : un premier appel ajoute le vote, un second appel le retire, puis le nombre final de votes revient à sa valeur initiale.

## Séparation de l'application et du démarrage du serveur

Le fichier `index.ts` contenait à la fois la configuration Express et `app.listen()`. J'ai séparé les responsabilités :

- `app.ts` configure Express, CORS, les cookies, Swagger, les routes et le gestionnaire d'erreurs ;
- `index.ts` charge l'application et démarre le serveur.

Cette séparation permet au test de réutiliser l'application Express et de contrôler lui-même le démarrage et l'arrêt du serveur.

Lors de la vérification, j'ai cru que le nouveau serveur se fermait immédiatement alors que l'API fonctionnait encore. Un serveur était simplement déjà lancé dans un autre terminal. Cette confusion m'a permis de mieux distinguer le processus que je démarre de celui qui occupe réellement le port `3000`.

## Vérification du serveur de test

J'ai ajouté une commande utilisant `.env.test`. Elle commence par vérifier que `DATABASE_URL` cible exactement `gamerchallenge_test`, puis démarre Express sur le port `7357`.

Le serveur de développement et le serveur de test sont désormais séparés :

```text
Développement : port 3000 → base de développement
Tests         : port 7357 → gamerchallenge_test
```

L'appel HTTP sur le port `7357` a fonctionné correctement.

## Préparation des données avec des helpers

J'ai observé la structure utilisée pendant la formation avant de créer mes propres helpers :

- `auth.ts` crée un utilisateur directement dans la base de test et fournit un cookie contenant un access token valide ;
- `challenges.ts` crée un challenge associé à son auteur ;
- `participations.ts` prépare une participation pour les futurs tests de votes sur les participations.

Ces helpers ne réalisent pas les assertions et ne constituent pas les scénarios de test. Ils préparent uniquement les données nécessaires afin d'éviter de répéter le même code dans chaque fichier de test.

L'utilisateur authentifié est créé directement en base car le scénario actuel teste le vote, pas la connexion. La route de connexion devra avoir ses propres tests.

## Utilisation de `randomUUID()`

Je ne me souvenais plus du rôle du module `node:crypto`. Ici, `randomUUID()` ne sert pas à hasher le mot de passe. Il génère une valeur aléatoire utilisée dans le pseudo, l'adresse email et le titre du challenge.

Cela évite les conflits avec les contraintes `@unique` lorsque plusieurs tests créent des données similaires. Le mot de passe reste hashé séparément avec Argon2.

## Nettoyage de la base de test

J'ai créé `cleanup-database.ts` pour supprimer les données générées par les tests. Le nettoyage commence seulement après l'exécution de la sécurité qui vérifie le nom de la base.

L'ordre de suppression respecte les dépendances entre les tables :

```text
VoteParticipation → Participation → Challenge
VoteChallenge     → Challenge
RefreshToken      → User
Challenge         → User
Participation     → User
```

Les tables enfants sont donc vidées avant les tables parentes afin d'éviter les erreurs de clés étrangères. Le nettoyage n'est pas automatique : le fichier de test appelle explicitement `clearTestDatabase()` avant chaque scénario et une dernière fois à la fin.

## Organisation des commandes de test

J'ai séparé les commandes selon leur rôle :

- `test:unit` exécute les tests de fonctions sans base de données ;
- `test:integration` charge `.env.test`, vérifie la base, puis exécute les tests HTTP avec une concurrence limitée à un seul test ;
- `test` enchaîne les tests unitaires et les tests d'intégration.

Cette séparation empêche un test d'intégration d'utiliser la configuration de développement par erreur. L'exécution séquentielle empêche également deux scénarios de nettoyer la base en même temps.

## Premier scénario Vote

Le fichier `votes.controller.spec.test.ts` suit la logique AAA :

```text
Arrange → nettoyer la base, créer l'utilisateur, le cookie et le challenge
Act     → envoyer une première requête, puis une deuxième requête
Assert  → vérifier les réponses HTTP et les données enregistrées avec Prisma
```

Le scénario vérifie :

1. le premier appel retourne `liked: true` et ajoute un vote ;
2. le deuxième appel retourne `liked: false` et retire ce vote ;
3. le nombre final de votes est égal au nombre initial.

Le test ne se limite donc pas à lire la réponse HTTP : il vérifie aussi directement l'état de la base avec Prisma.

## Résultat

La commande suivante exécute maintenant toute la vérification automatiquement :

```bash
npm test
```

Les trois tests unitaires existants et le premier test d'intégration du vote passent. Le serveur de test démarre avant le scénario, puis la base, le serveur HTTP et la connexion Prisma sont correctement nettoyés ou fermés à la fin.

## Ce que j'ai nouvellement compris

- Swagger sert à vérifier manuellement une route, tandis qu'un test d'intégration rejoue automatiquement le même scénario.
- Un helper prépare les données ; le fichier `*.spec.test.ts` exécute le scénario et réalise les assertions.
- `fetch` permet de tester l'ensemble du parcours HTTP sans ajouter Supertest.
- `randomUUID()` évite les collisions entre les données créées par plusieurs exécutions.
- Le nettoyage doit respecter les relations de clés étrangères.
- La séparation entre `app.ts` et `index.ts` permet au test de contrôler le cycle de vie du serveur.

## Prochaines étapes

- Ajouter le test `401` pour une requête sans cookie d'authentification.
- Ajouter le test `404` pour un challenge inexistant.
- Tester séparément le `403` sur la modification ou la suppression d'un challenge appartenant à un autre utilisateur.
