# BACK_E06 — Connexion entre l'API et PostgreSQL

## Objectif

Après avoir terminé la création et le seeding de la base de données, j'ai commencé à connecter mon back-end Express à PostgreSQL.

J'avais déjà utilisé séparément plusieurs éléments pendant la formation : Express, les routers, les controllers, PostgreSQL, SQL et `fetch()`.

Cependant, je voulais maintenant comprendre moi-même le chemin complet d'une requête, depuis l'URL appelée jusqu'à la base de données, puis le retour de la réponse.

Pour commencer simplement, mon premier objectif est de faire fonctionner :

```text
GET /api/challenges
```

avant de développer les autres fonctionnalités.

---

## 1. Préparation des commandes de gestion de la BDD

J'ai ajouté dans `package.json` quelques commandes permettant de gérer plus facilement la base de données pendant le développement :

```text
db:create
db:seed
db:reset
```

Je me suis inspirée d'anciens projets utilisant Prisma, mais je n'ai pas repris leurs commandes directement.

Mon projet utilise actuellement des fichiers SQL exécutés avec PostgreSQL. J'ai donc conservé uniquement les commandes correspondant réellement à mon fonctionnement actuel.

J'ai également renommé :

```text
mpd.sql
```

en :

```text
create.sql
```

afin de distinguer clairement :

```text
docs/BDD/mpd.md
→ documentation du modèle physique de données

prisma/create.sql
→ script permettant de créer les tables
```

Le script de création peut maintenant supprimer les anciennes tables avant de les recréer.

Le reset de la base suit donc cette logique :

```text
db:reset
   ↓
recréation des tables
   ↓
seeding
```

---

## 2. `tsc` et `tsx`

En travaillant sur les scripts du projet, j'ai également clarifié la différence entre `tsc` et `tsx`.

```text
tsc
→ compile TypeScript vers JavaScript

tsx
→ permet d'exécuter directement du TypeScript pendant le développement
```

Dans mon projet, le serveur de développement est actuellement lancé avec `tsx`.

Cette distinction était importante pour comprendre ce que fait réellement la commande `npm run dev`.

---

## 3. Installation de `pg`

Pour cette première connexion entre Express et PostgreSQL, j'ai choisi d'utiliser directement `pg`.

L'objectif n'est pas forcément de conserver cette solution pour la version finale du projet.

Le cahier des charges prévoit l'utilisation d'un ORM comme Sequelize ou Prisma.

Je souhaite cependant commencer avec `pg` afin de voir directement les requêtes SQL et de comprendre ce que fait la couche d'accès aux données avant d'utiliser l'abstraction fournie par un ORM.

J'ai donc installé :

```text
pg
@types/pg
```

`pg` permet à l'application Node.js de communiquer avec PostgreSQL.

`@types/pg` fournit les types nécessaires pour TypeScript.

---

## 4. Création du client PostgreSQL

J'ai créé :

```text
src/database/db_client.ts
```

Ce fichier centralise la connexion entre le back-end et PostgreSQL.

La configuration de la connexion est récupérée depuis la variable d'environnement :

```text
DB_URL
```

Le fichier `.env` est chargé au démarrage de l'application grâce à `dotenv`.

J'ai également compris que `db_client.ts` n'a pas besoin d'être lancé directement.

Il est chargé lorsqu'une autre partie de l'application l'importe.

Dans mon cas, le controller peut importer le client PostgreSQL puis l'utiliser pour envoyer une requête SQL.

---

## 5. Premier accès à PostgreSQL depuis le controller

Pour cette première étape, j'ai volontairement décidé de ne pas créer de Data Mapper.

Je veux d'abord comprendre le trajet complet avec le moins de couches possible.

Le controller des challenges utilise donc directement le client PostgreSQL pour exécuter une première requête permettant de récupérer quelques challenges.

L'objectif n'est pas encore de construire tout le CRUD.

Je veux d'abord réussir un seul trajet complet :

```text
requête HTTP
      ↓
Router
      ↓
Controller
      ↓
pg
      ↓
PostgreSQL
      ↓
résultat SQL
      ↓
Controller
      ↓
réponse JSON
```

Une fois ce fonctionnement compris, je pourrai décider comment mieux séparer les responsabilités.

---

## 6. Mise en place des routers

J'ai ensuite relié le controller au système de routing d'Express.

L'organisation actuelle est :

```text
index.ts
   ↓
index.router.ts
   ↓
challenge.router.ts
   ↓
challenges.controller.ts
   ↓
db_client.ts
   ↓
PostgreSQL
```

Cette étape m'a permis de mieux comprendre qu'un controller ne crée pas automatiquement une route.

Le controller contient le traitement à effectuer.

Le router détermine quelle URL et quelle méthode HTTP déclenchent ce traitement.

---

## 7. Problème rencontré : `Cannot GET /api/challenges`

Lors du premier test, le navigateur retournait :

```text
Cannot GET /api/challenges
```

avec une erreur :

```text
404 Not Found
```

J'ai d'abord vérifié mes imports et les différents fichiers du router et du controller.

Le problème venait finalement de l'URL.

Ma route correspondait à :

```text
/challenges
```

alors que je testais :

```text
/api/challenges
```

Le préfixe `/api` n'avait pas encore été défini dans Express.

Cette erreur m'a permis de comprendre qu'une erreur `Cannot GET` ne signifie pas forcément que le serveur, le controller ou PostgreSQL ne fonctionne pas.

Elle peut simplement indiquer qu'Express ne trouve aucune route correspondant à l'URL demandée.

---

## 8. Choix du préfixe `/api`

Le préfixe `/api` n'est pas obligatoire.

J'ai néanmoins décidé de l'utiliser afin de distinguer clairement les routes du front-end des endpoints du back-end.

Par exemple :

```text
Front-end
/challenges

API
/api/challenges
```

Plutôt que de répéter `/api` dans chaque router, il peut être défini au niveau du router principal de l'application.

Les différentes parties de l'URL sont alors assemblées :

```text
/api
+
/challenges
=
/api/challenges
```

Cela permettra également de conserver une organisation cohérente pour les futures routes de l'API.

---

## 9. Différence entre `console.log()` et la réponse HTTP

Pendant le test du controller, j'ai également mieux compris la différence entre afficher les données dans le terminal et les envoyer au client.

```text
console.log()
→ affichage côté serveur dans le terminal

res.json()
→ données envoyées dans la réponse HTTP
```

Une requête PostgreSQL peut donc fonctionner correctement et afficher son résultat dans le terminal sans que ces données soient automatiquement envoyées au navigateur.

C'est le controller qui doit construire la réponse HTTP.

---

## Ce que j'ai compris

Cette étape m'a surtout permis de relier des notions que j'avais déjà vues séparément.

Je comprends maintenant mieux le chemin :

```text
Client HTTP / React
        ↓
URL
        ↓
Express
        ↓
Router
        ↓
Controller
        ↓
Client PostgreSQL
        ↓
PostgreSQL
```

Puis dans l'autre sens :

```text
PostgreSQL
        ↓
résultat de la requête
        ↓
Controller
        ↓
réponse JSON
        ↓
Client HTTP / React
```

J'ai également compris que les routes sont construites à partir de plusieurs niveaux de l'application.

Une erreur dans l'URL peut donc provoquer un `404` même si le controller et la base de données fonctionnent correctement.

Enfin, utiliser directement `pg` me permet pour le moment de voir clairement la requête SQL et le trajet des données.

L'utilisation d'un ORM pourra venir ensuite, une fois ce fonctionnement suffisamment compris.

---

## État actuel

- [x] Scripts de gestion de la BDD ajoutés dans `package.json`
- [x] `mpd.sql` renommé en `create.sql`
- [x] Différence entre `tsc` et `tsx` comprise
- [x] `pg` et ses types TypeScript installés
- [x] Client PostgreSQL créé
- [x] Variable `DB_URL` utilisée pour la connexion
- [x] Premier accès SQL préparé depuis le controller
- [x] Router des challenges créé
- [x] Controller relié au router
- [x] Origine du `404 Cannot GET /api/challenges` identifiée
- [x] Fonctionnement du préfixe `/api` compris
- [x] Différence entre `console.log()` et `res.json()` comprise
- [ ] Vérifier définitivement la réponse de `GET /api/challenges`
- [ ] Connecter React à l'endpoint

## Prochaine étape

Tester :

```text
GET /api/challenges
```

et vérifier que la réponse contient bien les données provenant de PostgreSQL.

Une fois cette étape validée, connecter le front-end React à cet endpoint avec `fetch()`.