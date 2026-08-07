# Challenge SC04E01 — Centralisation des logs avec Winston et MongoDB

> 21/07/2026

---

## 🎯 Objectif du challenge

Mettre en place un système de journalisation pour l’API O’Quiz, puis commencer un microservice séparé permettant de stocker et de consulter les logs dans une base de données MongoDB.

L’objectif général était de comprendre le parcours d’un log :

```text
Requête reçue par l’API
→ création du log avec Winston
→ écriture du log
→ stockage dans MongoDB
→ consultation depuis le logs-service
```

---

## 🏫 Mise en place réalisée pendant le cours

### 1. Ajout de Winston dans l’API

J’ai installé et configuré Winston afin de remplacer progressivement les simples `console.log()` par un véritable système de journalisation.

J’ai créé un fichier dédié :

```text
src/api/src/lib/logger.ts
```

Ce fichier centralise la configuration du logger.

J’ai également ajouté un middleware :

```text
src/api/src/middlewares/logrequest.middleware.ts
```

Son rôle est de créer automatiquement un log lorsqu’une requête HTTP arrive dans l’API.

J’ai ensuite connecté ce middleware à l’application Express dans `app.ts`.

---

### 2. Configuration des fichiers de logs

La configuration de Winston permet de produire des fichiers comme :

```text
combined.log
```

J’ai compris que ces fichiers sont générés pendant l’exécution de l’application et qu’ils ne doivent pas être envoyés sur Git.

J’ai donc complété le fichier `.gitignore` situé à la racine du projet :

```gitignore
.DS_Store
.env
*.log
node_modules/
```

Comme ce fichier est placé à la racine du dépôt, ces règles s’appliquent également aux sous-dossiers du projet.

---

## 📌 Ce que j’ai développé pendant le challenge

### 3. Création d’un microservice pour les logs

J’ai commencé un nouveau service séparé dans :

```text
src/logs-service
```

J’ai préparé la structure suivante :

```text
logs-service/
├── .env
├── .env.example
├── index.ts
├── package.json
├── package-lock.json
├── README.md
├── tsconfig.json
└── src
    ├── app.ts
    ├── controllers
    │   └── logs.controller.ts
    ├── lib
    │   └── db.ts
    ├── routes
    │   ├── index.router.ts
    │   └── logs.router.ts
    └── services
        └── logs.service.ts
```

Cette séparation m’a permis de retrouver une organisation proche de celle de l’API principale :

```text
index
→ app
→ router
→ controller
→ accès à la base de données
```

Le service Express écoute sur le port `3010`.

---

### 4. Configuration TypeScript du microservice

J’ai rencontré une confusion concernant les extensions utilisées dans les imports.

Le fichier réel est bien un fichier TypeScript :

```text
src/app.ts
```

Mais avec la configuration `NodeNext` et un projet utilisant les modules ESM, l’import est écrit avec l’extension du futur fichier JavaScript :

```ts
import { app } from "./src/app.js";
```

Je pensais d’abord devoir renommer le fichier `app.ts` en `app.js`.

J’ai finalement compris la distinction suivante :

```text
Fichier écrit pendant le développement : app.ts
Chemin utilisé dans l’import : app.js
Fichier produit après compilation : app.js
```

J’ai également rencontré cette erreur :

```text
'Request' is a type and must be imported using a type-only import
when 'verbatimModuleSyntax' is enabled.
```

Mon import initial était :

```ts
import { Request, Response } from "express";
```

Je l’ai corrigé ainsi :

```ts
import type { Request, Response } from "express";
```

`Request` et `Response` sont uniquement utilisés comme types. Avec `verbatimModuleSyntax`, TypeScript demande que cela soit indiqué explicitement.

Après la correction, la vérification TypeScript s’est terminée sans erreur :

```bash
npx tsc --noEmit
```

---

## 🧠 Mise en place de MongoDB

### 1. Première tentative de connexion

J’ai essayé de connecter l’extension MongoDB de VS Code à l’adresse :

```text
localhost:27018
```

La connexion a échoué avec l’erreur :

```text
ECONNREFUSED 127.0.0.1:27018
```

Au début, je cherchais le problème dans mon code et dans la configuration de la connexion.

En avançant légèrement dans le replay du cours, je me suis rendu compte que j’avais manqué une information importante : MongoDB devait d’abord être lancé dans un conteneur Docker.

Cette erreur signifiait donc simplement qu’aucun service MongoDB n’écoutait encore sur le port `27018`.

---

### 2. Lancement du premier conteneur MongoDB

J’ai lancé MongoDB avec :

```bash
docker run -d --name log_db -p 27018:27017 --rm mongo:latest
```

Docker a téléchargé l’image, mais la commande suivante ne montrait aucun conteneur actif :

```bash
docker ps
```

Le problème était difficile à observer parce que l’option `--rm` supprimait automatiquement le conteneur après son arrêt.

J’ai donc relancé le conteneur sans cette option :

```bash
docker run -d --name log_db -p 27018:27017 mongo:latest
```

Puis j’ai affiché tous les conteneurs, y compris ceux qui étaient arrêtés :

```bash
docker ps -a
```

Le résultat indiquait :

```text
Exited (132)
```

Cette fois, le conteneur était conservé et je pouvais consulter ses logs :

```bash
docker logs log_db
```

---

### 3. Incompatibilité entre MongoDB et le Téléporteur

Les logs du conteneur indiquaient :

```text
MongoDB 5.0+ requires a CPU with AVX support
```

La commande donnée dans le replay utilisait :

```text
mongo:latest
```

Elle fonctionnait dans l’environnement du formateur, mais pas dans mon Téléporteur, car son environnement CPU ne fournit pas le support AVX exigé par MongoDB 5 et les versions suivantes.

J’ai donc supprimé le conteneur arrêté :

```bash
docker rm log_db
```

Puis j’ai utilisé une version compatible :

```bash
docker run -d --name log_db -p 27018:27017 mongo:4.4
```

J’ai vérifié son état avec :

```bash
docker ps
```

Cette fois, le conteneur MongoDB est resté actif et la connexion sur le port `27018` a fonctionné.

Ce problème ne venait donc ni de mon code ni du port choisi. Il venait d’une incompatibilité entre la version récente de MongoDB et l’environnement CPU du Téléporteur.

---

## 🧪 Premier test d’insertion dans MongoDB

Pour comprendre MongoDB, j’avais besoin de commencer par une opération concrète :

```text
insérer un document
→ vérifier sa présence
→ le récupérer
```

J’ai retenu les correspondances suivantes :

```text
PostgreSQL : table       → MongoDB : collection
PostgreSQL : ligne       → MongoDB : document
```

J’ai créé un fichier de démonstration contenant une insertion :

```ts
await client
  .db()
  .collection("logs")
  .insertOne({ message: "je suis un log" });
```

Au début, j’ai lancé :

```bash
npm run dev
```

Mais aucun document n’apparaissait dans MongoDB.

Je pensais que l’insertion ou la connexion ne fonctionnait pas. Le véritable problème était que `npm run dev` lançait `index.ts`, tandis que mon fichier de démonstration n’était importé nulle part.

Créer un fichier ne signifie pas que son code est automatiquement exécuté.

J’ai donc lancé directement le fichier concerné :

```bash
npx tsx --env-file=.env src/services/logs.service.ts
```

Cette fois, le document a bien été ajouté dans la collection `logs`.

J’ai ainsi compris la différence entre :

```text
npm run dev
→ lance le point d’entrée prévu par le script

npx tsx chemin/du/fichier.ts
→ exécute directement le fichier indiqué
```

---

## ✅ Lecture des logs à travers l’API

Après avoir réussi l’insertion, j’ai connecté le contrôleur à MongoDB afin de récupérer tous les documents :

```ts
export async function getAllLogs(req: Request, res: Response) {
  const logs = await client
    .db()
    .collection("logs")
    .find()
    .toArray();

  res.status(200).json(logs);
}
```

J’ai relié cette fonction à la route :

```ts
router.get("/logs", logsController.getAllLogs);
```

Puis j’ai envoyé la requête suivante :

```http
@baseURL=http://localhost:3010/logs-service/logs

### Récupérer tous les logs
GET {{baseURL}}
```

La réponse obtenue était :

```text
HTTP/1.1 200 OK
```

Les documents enregistrés dans MongoDB ont été retournés sous la forme d’un tableau JSON.

Le parcours réellement fonctionnel est maintenant :

```text
GET /logs-service/logs
→ logs.router.ts
→ getAllLogs()
→ collection MongoDB "logs"
→ find()
→ toArray()
→ réponse JSON avec le statut 200
```

---

## ⚠️ Les principales difficultés

### Comprendre ce qui était réellement exécuté

Ma principale confusion ne concernait pas MongoDB lui-même, mais le point d’entrée exécuté.

J’avais créé un fichier contenant `insertOne()`, puis lancé le serveur avec `npm run dev`. Je pensais que le simple fait d’avoir créé ce fichier suffisait pour que son code soit exécuté.

J’ai compris qu’un fichier TypeScript doit :

- être importé dans le parcours de l’application ;
- ou être lancé directement.

---

### Distinguer un problème de code d’un problème d’environnement

Lorsque la connexion à MongoDB échouait, plusieurs causes étaient possibles :

- MongoDB n’était pas lancé ;
- le port était incorrect ;
- le conteneur s’arrêtait ;
- la version de MongoDB était incompatible ;
- le fichier d’insertion n’était pas exécuté.

J’ai progressé en vérifiant chaque couche séparément :

```text
docker ps
→ docker ps -a
→ docker logs
→ connexion MongoDB
→ exécution directe du fichier
→ vérification du document
```

Cette méthode m’a évité de modifier le code au hasard.

---

### Suivre le replay alors que certaines informations avaient été manquées

J’avais manqué l’explication indiquant que MongoDB devait être lancé avec Docker.

En voyant `ECONNREFUSED`, j’ai d’abord cherché une erreur dans mon propre travail. En revenant légèrement dans le replay, j’ai retrouvé l’étape manquante.

Cette situation m’a rappelé qu’une erreur de connexion ne signifie pas automatiquement que le code de connexion est incorrect : le service ciblé doit aussi être réellement lancé.

---

### Avancer malgré la fatigue et la précipitation

Je savais qu’il fallait construire le microservice progressivement, mais j’avais l’impression de devoir tout terminer immédiatement.

Cette précipitation rendait la structure plus difficile à lire et augmentait mon inquiétude devant les fichiers encore vides.

J’ai donc utilisé les commits comme points de sauvegarde intermédiaires :

```text
ajout du logger
→ création de la structure du logs-service
→ validation TypeScript
→ test de MongoDB
→ récupération des logs
```

Le service n’avait pas besoin d’être entièrement terminé pour que chaque étape constitue une progression réelle et enregistrable.

---

## 💡 Ce que j’ai nouvellement compris

### Un conteneur peut être créé puis s’arrêter immédiatement

Une longue série de téléchargements Docker suivie de l’affichage d’un identifiant ne garantit pas que le conteneur fonctionne encore.

Il faut vérifier son état :

```bash
docker ps
```

ou, pour voir également les conteneurs arrêtés :

```bash
docker ps -a
```

---

### L’option `--rm` peut compliquer le diagnostic

L’option `--rm` supprime automatiquement le conteneur lorsqu’il s’arrête.

Elle est pratique dans certains cas, mais elle peut empêcher de retrouver facilement le conteneur après une erreur.

Pendant un diagnostic, conserver le conteneur permet d’utiliser :

```bash
docker logs nom_du_conteneur
```

---

### Une image Docker récente n’est pas toujours compatible avec l’environnement

Le tag `latest` ne signifie pas « meilleure version pour tous les ordinateurs ».

Il signifie seulement que Docker utilisera la version actuellement désignée comme la plus récente.

Dans mon environnement, `mongo:latest` était incompatible avec l’absence de support AVX, tandis que `mongo:4.4` fonctionnait.

---

### `find()` et `toArray()` ont deux rôles différents

Dans :

```ts
const logs = await client
  .db()
  .collection("logs")
  .find()
  .toArray();
```

j’ai compris que :

```text
find()
→ prépare la recherche et retourne un curseur

toArray()
→ récupère les résultats sous la forme d’un tableau
```

Ce tableau peut ensuite être envoyé au client avec :

```ts
res.status(200).json(logs);
```

---

## ✍️ Ce que j’ai surtout formulé plus précisément

Je connaissais déjà l’idée générale d’une route Express reliée à un contrôleur.

Ce challenge m’a permis de décrire plus précisément le chemin suivi par la requête :

```text
URL
→ router
→ fonction du contrôleur
→ accès à MongoDB
→ réponse HTTP
```

J’ai également précisé ma compréhension de la différence entre :

- le fichier TypeScript réellement présent ;
- le chemin `.js` écrit dans un import ESM avec `NodeNext` ;
- le fichier JavaScript qui sera produit lors de la compilation.

---

## 🛠 Commandes importantes utilisées

```bash
npx tsc --noEmit
```

Vérifier les erreurs TypeScript sans produire les fichiers JavaScript.

```bash
docker ps
```

Afficher les conteneurs actuellement actifs.

```bash
docker ps -a
```

Afficher les conteneurs actifs et arrêtés.

```bash
docker logs log_db
```

Lire la cause de l’arrêt du conteneur MongoDB.

```bash
docker run -d --name log_db -p 27018:27017 mongo:4.4
```

Lancer une version de MongoDB compatible avec le Téléporteur.

```bash
npx tsx --env-file=.env src/services/logs.service.ts
```

Exécuter directement le fichier de démonstration avec les variables de `.env`.

---

## 📦 Git

J’ai vérifié que les fichiers sensibles ou générés n’étaient pas inclus dans le commit :

```text
.env
node_modules/
*.log
```

En revanche, `.env.example` devait être conservé dans le dépôt afin de documenter les variables nécessaires.

J’ai créé des commits intermédiaires correspondant à l’état réel du travail, notamment :

```text
feat: add logger and start logs service
```

puis :

```text
test: MongoDB works
```

Les modifications ont ensuite été envoyées sur le dépôt distant.

---

## 🌱 Les notions que je commence à comprendre

- Pourquoi utiliser un logger comme Winston plutôt que seulement `console.log()`.
- Le rôle d’un middleware de journalisation dans une API Express.
- Pourquoi les fichiers `.log` et `.env` ne doivent pas être commités.
- La différence entre une table PostgreSQL et une collection MongoDB.
- La différence entre une ligne relationnelle et un document MongoDB.
- Comment lancer MongoDB dans un conteneur Docker.
- Pourquoi `mongo:latest` ne fonctionnait pas dans mon Téléporteur.
- Comment utiliser `docker ps -a` et `docker logs` pour diagnostiquer un conteneur arrêté.
- Pourquoi un fichier contenant du code ne s’exécute pas s’il n’est ni importé ni lancé directement.
- Comment insérer un document avec `insertOne()`.
- Comment récupérer les documents avec `find().toArray()`.
- Comment relier une route GET, un contrôleur Express et une collection MongoDB.

---

## 🔄 Ce qui reste encore à travailler

Le microservice fonctionne maintenant pour récupérer tous les logs, mais il reste encore plusieurs étapes possibles :

- ajouter une gestion des erreurs avec `try/catch` ;
- valider les données reçues avant une insertion ;
- définir plus précisément la structure d’un log ;
- déplacer les opérations MongoDB dans de véritables fonctions de service ;
- ajouter les autres opérations CRUD si elles sont demandées ;
- connecter automatiquement les logs produits par l’API au stockage MongoDB ;
- ajouter des tests pour les routes du logs-service.

Je n’ai pas encore suffisamment travaillé ces parties pour les considérer comme comprises ou terminées.

---

## 🧭 Prochaine étape concrète

La prochaine étape logique est de séparer correctement les responsabilités :

```text
Controller
→ reçoit la requête et prépare la réponse HTTP

Service
→ effectue les opérations MongoDB
```

Pour le moment, la lecture MongoDB est directement écrite dans le contrôleur. Elle fonctionne, ce qui m’a permis de vérifier tout le parcours.

Je pourrai ensuite la déplacer dans `logs.service.ts`, lorsque je serai prête à mieux structurer le service sans perdre de vue ce que chaque partie exécute.