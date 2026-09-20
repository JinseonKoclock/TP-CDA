# DevLog — Reprendre le back-end depuis une base vide

**Date : 16/09/2026**  
**Projet : GamerChallenges**

## Contexte

Aujourd'hui, j'ai décidé de reprendre une partie du back-end de GamerChallenges depuis un dossier presque vide.

Le back-end du projet existe déjà et a été commencé par Guillaume. En le relisant, notamment avec Prisma, Express et Docker, je me suis rendu compte que j'arrivais à reconnaître certains éléments séparément, mais que j'avais plus de difficulté à reconstruire moi-même leur mise en place et leur enchaînement.

J'ai donc choisi de refaire une petite base de back-end de mon côté, non pas pour remplacer le travail existant, mais pour reprendre progressivement les étapes et essayer de comprendre pourquoi chaque outil est nécessaire.

Pour cette première session, je ne voulais pas encore travailler sur Prisma, PostgreSQL, l'authentification ou Docker.

Mon objectif était simplement de repartir du début et de réussir à lancer un serveur Express.

---

## 1. Initialisation du projet avec npm

J'ai commencé dans mon dossier `backend` avec :

```bash
npm init
```

Je savais que cette commande servait à initialiser le projet avec npm, mais je ne me souvenais plus précisément de toutes les informations demandées pendant cette étape.

La commande m'a posé plusieurs questions :

- package name ;
- version ;
- description ;
- entry point ;
- test command ;
- git repository ;
- keywords ;
- author ;
- license.

À la fin, npm a créé le fichier `package.json`.

Cette étape m'a permis de revoir plus concrètement le rôle de ce fichier : il contient notamment des informations sur le projet, les scripts et les dépendances installées.

J'ai également revu qu'il existe une commande plus rapide :

```bash
npm init -y
```

qui accepte directement les valeurs par défaut.

Pour cette fois, utiliser `npm init` sans `-y` était intéressant, car cela m'a permis de revoir les différentes informations que npm propose de configurer lors de l'initialisation.

---

## 2. Installation d'Express

Pour pouvoir créer le serveur, j'ai ensuite installé Express :

```bash
npm i express
```

Après cette commande, Express est apparu dans les `dependencies` du `package.json`.

À ce stade, le principe est assez clair pour moi :

> J'ai besoin d'Express dans l'application pour créer mon serveur, donc je l'installe comme dépendance du projet.

C'est une partie que je retrouve plus facilement, car j'ai déjà utilisé Express dans les projets précédents.

---

## 3. Création d'un serveur minimal

J'ai créé un fichier :

```text
src/index.ts
```

avec un serveur volontairement très simple :

```ts
import express from "express";
import "dotenv/config";

const app = express();

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`🚀 Serveur lancé sur http://localhost:${PORT}`);
});
```

J'ai également installé :

```bash
npm i dotenv
```

pour pouvoir utiliser une variable d'environnement pour le port.

À ce moment-là, mon objectif n'était toujours pas de construire l'architecture complète du back-end.

Je voulais seulement arriver à lancer ce premier serveur avant d'ajouter d'autres éléments.

---

## 4. Les premiers problèmes liés à TypeScript

Comme mon fichier était en `.ts`, VS Code m'a signalé des problèmes liés aux types.

J'ai notamment installé :

```bash
npm i --save-dev @types/node
```

Je savais déjà que les packages `@types` étaient liés au typage et qu'ils permettaient à TypeScript d'obtenir des informations de types pour certaines bibliothèques.

En revanche, ce qui était beaucoup moins clair pour moi était la raison pour laquelle ce type de package est souvent installé avec :

```bash
--save-dev
```

Je ne comprenais pas réellement pourquoi npm distingue les `dependencies` des `devDependencies`, ni pourquoi on ne pouvait pas simplement installer tous les packages de la même manière avec `npm i`.

C'est donc surtout cette distinction que je dois reprendre.

### Points à reprendre

- Quelle est la différence entre `dependencies` et `devDependencies` ?
- Que fait exactement l'option `--save-dev` ?
- Pourquoi certains packages nécessaires pendant le développement sont-ils séparés des dépendances nécessaires au fonctionnement de l'application ?
- Dans quelle catégorie dois-je placer `express`, `dotenv`, `tsx` et les packages `@types`, et pourquoi ?

---

## 5. Mise en place du script `dev`

Pour lancer le serveur pendant le développement, je voulais utiliser :

```bash
npm run dev
```

Je savais déjà que `npm run dev` nécessite qu'un script `dev` soit défini dans le `package.json`.

En revanche, je ne me souvenais plus précisément de la commande à associer à ce script dans le cas d'un serveur écrit en TypeScript.

Mon premier essai n'était donc pas lié au fait d'ignorer qu'il fallait créer le script `dev`, mais plutôt au contenu à placer derrière ce script.

---

## 6. Confusion entre `tsc` et l'exécution du serveur

J'ai d'abord ajouté :

```json
"dev": "tsc src/index.ts"
```

Je savais que `tsc` était lié à TypeScript, mais je ne me souvenais plus précisément de son rôle par rapport à l'exécution directe de mon serveur.

En reprenant ce point, j'ai retrouvé la distinction suivante :

- `tsc` est le compilateur TypeScript. Il permet notamment de transformer du code TypeScript en JavaScript ;

  <details>
  <summary>🇰🇷 한국어 설명</summary>

  `tsc`는 **TypeScript 컴파일러**야.

  우리가 작성한 `.ts` 파일을 컴퓨터가 실행할 수 있도록 `.js` 파일로 변환해줘.

  예를 들어:

  ```text
  index.ts
     ↓ tsc
  index.js
  ```

  그래서 나중에 프로덕션에서는 보통:

  ```bash
  npm run build
  ```

  로 TypeScript를 JavaScript로 변환한 다음,

  ```bash
  node dist/index.js
  ```

  처럼 변환된 JavaScript를 실행할 수 있어.

  **기억할 것:**  
  `tsc` = TS → JS로 **변환(컴파일)**

  </details>

- `tsx` permet d'exécuter directement un fichier TypeScript pendant le développement, sans avoir à lancer séparément une étape de compilation avant de démarrer le serveur.

  <details>
  <summary>🇰🇷 한국어 설명</summary>

  `tsx`는 개발할 때 `.ts` 파일을 **바로 실행할 수 있게 해주는 도구**야.

  즉 개발할 때 매번:

  ```text
  TypeScript 작성
       ↓
  tsc로 JavaScript 변환
       ↓
  node로 JavaScript 실행
  ```

  이렇게 따로 할 필요 없이:

  ```bash
  tsx src/index.ts
  ```

  로 바로 실행할 수 있어.

  지금 GamerChallenges의:

  ```json
  "dev": "tsx src/index.ts"
  ```

  가 바로 이 용도야.

  그래서:

  ```bash
  npm run dev
  ```

  를 실행하면 `tsx`가 `src/index.ts`를 실행해서 개발 서버를 시작해.

  **기억할 것:**  
  `tsx` = 개발 중 TS를 **바로 실행**

  </details>

Dans mon cas, mon objectif était de lancer directement mon serveur Express écrit dans `src/index.ts`.

J'ai donc installé :

```bash
npm install --save-dev tsx
```

puis défini :

```json
"dev": "tsx src/index.ts"
```

Je commence à retrouver la différence entre `tsc` et `tsx`, mais je dois encore la pratiquer pour être capable de l'expliquer naturellement sans aide.

---

## 7. Une erreur JSON dans `package.json`

Pendant la modification manuelle du `package.json`, j'ai fait une erreur de syntaxe.

Lorsque j'ai lancé une commande npm, j'ai obtenu :

```text
EJSONPARSE
```

avec notamment le message :

```text
Expected double-quoted property name in JSON
```

J'avais laissé une virgule incorrecte dans le fichier.

Cette erreur a empêché aussi bien l'utilisation des scripts npm que l'installation de `tsx`.

Cela m'a permis de constater concrètement que npm doit d'abord pouvoir lire correctement le `package.json` avant d'utiliser les scripts ou d'y enregistrer de nouvelles dépendances.

Après avoir corrigé la syntaxe JSON, l'installation de `tsx` a fonctionné.

---

## 8. Premier résultat

J'ai finalement relancé :

```bash
npm run dev
```

Cette fois, npm a exécuté :

```text
> backend@1.0.0 dev
> tsx src/index.ts
```

et j'ai obtenu :

```text
🚀 Serveur lancé sur http://localhost:3000
```

Le serveur fonctionne donc.

---

## Ce que je retiens réellement de cette session

Je savais encore créer rapidement un petit serveur Express, surtout parce que j'avais déjà utilisé ce type de code dans d'autres projets.

En revanche, repartir d'un projet vide m'a montré que j'ai plus de difficulté à reconstruire toute la préparation de l'environnement autour de ce code lorsque rien n'est encore configuré.

J'ai repris progressivement le chemin suivant :

```text
npm init
    ↓
package.json
    ↓
installation des dépendances
    ↓
création de src/index.ts
    ↓
définition du script "dev"
    ↓
exécution avec tsx
    ↓
serveur Express lancé
```

Je savais déjà qu'il fallait définir un script `dev` dans le `package.json` pour utiliser `npm run dev`.

Ce que j'avais besoin de retrouver était surtout **la commande à associer à ce script** dans le contexte de mon serveur TypeScript.

C'est en essayant d'abord :

```text
tsc src/index.ts
```

puis en reprenant le rôle de `tsx` que j'ai pu préciser ce point et arriver finalement à :

```text
tsx src/index.ts
```

Plusieurs notions restent cependant encore à clarifier :

- la différence entre `dependencies` et `devDependencies` ;
- le rôle de l'option `--save-dev` et la raison pour laquelle certaines dépendances sont séparées ;
- la différence précise entre `tsc` et `tsx` ;
- ce qui relève réellement de TypeScript, de Node.js et de npm.

Je ne considère donc pas ces notions comme acquises.

Le but de cette reprise depuis zéro est justement de reconstruire progressivement les liens entre ces différents éléments.

---

## Ressenti / lien avec la formation

Cette étape m'a donné une impression assez différente du travail back-end que j'avais surtout retenu du Bloc B.

Dans le Bloc B, je pensais davantage à la logique de l'application :

```text
requête
   ↓
route
   ↓
controller
   ↓
données
   ↓
réponse
```

Aujourd'hui, je me suis surtout retrouvée face à la préparation et à l'exécution du projet :

```text
npm
 ↓
packages
 ↓
TypeScript
 ↓
scripts
 ↓
environnement d'exécution
```

Cela me rappelle davantage certaines problématiques rencontrées ensuite autour de l'environnement de développement et du déploiement.

C'est probablement aussi pour cette raison que j'ai eu l'impression d'avoir oublié beaucoup de choses : je ne travaillais pas encore sur la logique métier de GamerChallenges, mais sur ce qui permet au projet de fonctionner avant même d'arriver à cette logique.

---

## À reprendre à la prochaine session

Avant d'ajouter Prisma, PostgreSQL, les routes ou Docker, je veux être capable d'expliquer simplement, avec mes propres mots :

> Pourquoi ai-je installé `express`, `dotenv`, `@types/node` et `tsx` ?

Je veux également reprendre cette question :

> Quand je tape `npm run dev`, que se passe-t-il entre cette commande et l'affichage « Serveur lancé » ?

Je préfère clarifier ce petit morceau avant de continuer à reconstruire le back-end.