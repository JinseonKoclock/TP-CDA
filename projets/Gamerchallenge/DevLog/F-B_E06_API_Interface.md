# BACK_E06 — Connexion BDD, API et Front-end

## Objectif

Connecter progressivement PostgreSQL au back-end Express, puis vérifier que les données peuvent être récupérées depuis le front-end React.

L'objectif principal était de comprendre concrètement le chemin parcouru par les données entre la base de données, l'API et l'interface.

---

## 1. Scripts de gestion de la BDD

Le fichier SQL permettant de créer les tables a été renommé :

`mpd.sql` → `create.sql`

Ce nom permet de distinguer le MPD, qui représente le modèle de données, du script SQL réellement exécuté pour créer les tables.

Des scripts npm ont également été préparés pour faciliter la gestion de la base :

- `db:create` : création des tables ;
- `db:seed` : insertion des données de test ;
- `db:reset` : recréation puis remplissage de la BDD.

---

## 2. Utilisation de `tsx`

Le serveur de développement est lancé avec `tsx`.

J'ai compris la différence principale :

- `tsc` compile TypeScript en JavaScript ;
- `tsx` permet d'exécuter directement les fichiers TypeScript pendant le développement.

---

## 3. Connexion à PostgreSQL avec `pg`

Installation de `pg` et de ses types TypeScript.

Création de `src/database/db_client.ts`.

Ce fichier crée la connexion entre l'application Node.js et PostgreSQL.

Pour le moment, je n'utilise volontairement pas de Data Mapper afin de comprendre d'abord le fonctionnement direct d'une requête SQL depuis le back-end.

---

## 4. Première requête depuis un Controller

Création d'un controller permettant de récupérer les challenges depuis PostgreSQL.

Une première requête limitée à trois challenges a permis de vérifier que :

- la connexion PostgreSQL fonctionne ;
- le controller peut exécuter une requête SQL ;
- les résultats peuvent être transformés en réponse JSON.

---

## 5. Mise en place des routes

Création d'une route dédiée aux challenges et connexion de celle-ci au router principal.

J'ai choisi de préfixer les routes de l'API avec `/api`.

L'endpoint testé est donc :

`GET /api/challenges`

Cela permet de distinguer clairement les routes du front-end des endpoints du back-end.

---

## 6. Erreur 404 rencontrée

Lors du premier test, j'ai obtenu :

`Cannot GET /api/challenges`

Le problème venait du chemin de la route.

Cette erreur m'a permis de comprendre qu'une route Express dépend de l'assemblage des différents niveaux de routers et du préfixe défini dans l'application principale.

Après correction, l'endpoint a retourné correctement trois challenges provenant de PostgreSQL.

---

## 7. Connexion avec React

Dans `HomePage`, j'ai utilisé `useEffect()` pour déclencher une requête `fetch()` vers l'API au chargement de la page.

L'adresse du back-end est définie dans le fichier `.env` du front avec :

`VITE_API_URL`

Les données reçues sont ensuite enregistrées dans un state React avec `setChallenges()`.

---

## 8. Erreur CORS rencontrée

Lors du premier appel depuis React, le navigateur affichait une erreur liée à la requête réseau / CORS.

Le problème ne venait finalement pas de la configuration CORS : le serveur back-end n'était simplement pas lancé.

Après avoir lancé le back-end avec :

`npm run dev`

la requête du front-end vers `/api/challenges` a fonctionné correctement.

J'ai compris qu'une erreur affichée autour d'une requête CORS ne signifie pas automatiquement que la configuration CORS est incorrecte. Il faut également vérifier que le serveur cible est démarré et accessible.

---

## 9. Résultat obtenu

Les trois challenges provenant de PostgreSQL sont maintenant visibles dans la console du navigateur côté React.

Le chemin complet des données fonctionne donc :

PostgreSQL  
→ `pg` / `db_client.ts`  
→ Controller  
→ Router  
→ Express API  
→ requête HTTP avec `fetch()`  
→ React  
→ `setChallenges()`

C'est la première fois dans ce projet que j'ai mis en place et vérifié moi-même l'ensemble de ce chemin.

---

## 10. Préparation de l'affichage des données

Le composant `ChallengeCard` actuel utilise encore des données statiques et ses propriétés ne correspondent pas exactement aux données retournées par ma BDD.

La version réalisée précédemment par Marie-Laure a été conservée séparément afin de ne pas la supprimer pendant mes essais.

J'ai commencé à créer un type TypeScript `IChallenge` correspondant aux données provenant de l'API.

Une erreur a été repérée dans cette première version : la propriété `description` a été déclarée deux fois et devra être corrigée.

---

## Ce que j'ai compris

- comment connecter Node.js à PostgreSQL avec `pg` ;
- comment un controller récupère les données de la BDD ;
- comment les routers conduisent une requête HTTP jusqu'au controller ;
- pourquoi utiliser un préfixe `/api` ;
- la différence entre `console.log()` et une réponse HTTP avec `res.json()` ;
- comment React appelle l'API avec `fetch()` ;
- comment les données reçues sont enregistrées dans un state ;
- pourquoi une erreur réseau/CORS ne vient pas nécessairement de CORS ;
- comment les différentes parties étudiées séparément commencent à former une application complète.

---

## État actuel

- [x] BDD PostgreSQL créée et seedée
- [x] Connexion PostgreSQL avec `pg`
- [x] Controller `getAll`
- [x] Route `GET /api/challenges`
- [x] Réponse JSON vérifiée
- [x] Appel de l'API depuis React
- [x] Données reçues dans le front-end
- [x] Début du type `IChallenge`
- [ ] Corriger et terminer `IChallenge`
- [ ] Typer le state `challenges`
- [ ] Adapter `ChallengeCard` aux données réelles
- [ ] Générer les cartes avec `.map()`
- [ ] Afficher les données PostgreSQL réellement dans la HomePage