# DevLog — Du lancement du serveur à la première connexion Front / Back

**Date :** 17/09/2026  
**Projet :** GamerChallenge
## Contexte

Lors de la session précédente, j'avais réussi à lancer un serveur Express minimal avec :

```bash
npm run dev
```

mais j'avais volontairement laissé plusieurs points à reprendre avant de continuer avec Prisma, PostgreSQL, les routes ou Docker.

Je voulais notamment mieux distinguer le rôle de npm, de `tsx`, du fichier `index.ts` et d'Express, ainsi que clarifier la différence entre les dépendances utilisées par l'application et celles utilisées uniquement pendant le développement.

Aujourd'hui, je reprends donc le back-end à partir de ces questions avant d'ajouter de nouvelles briques au projet.

---

## 1. Reprendre le chemin de `npm run dev`

J'ai commencé par essayer de reconstruire moi-même ce qui se passe lorsque je lance :

```bash
npm run dev
```

Au départ, j'ai placé `index.ts` avant `tsx` dans mon raisonnement, en pensant notamment aux imports présents au début du fichier.

En reprenant l'ordre étape par étape, j'ai pu préciser le chemin :

```text
npm run dev
    ↓
npm cherche le script "dev" dans package.json
    ↓
"dev": "tsx src/index.ts"
    ↓
tsx exécute src/index.ts
    ↓
le code contenu dans index.ts est exécuté
    ↓
Express est utilisé par ce code
```

Le point que j'ai surtout clarifié est que `tsx` n'est pas exécuté à cause d'un `import` présent dans `index.ts`.

C'est le script `dev` du `package.json` qui demande d'exécuter :

```text
tsx src/index.ts
```

`tsx` intervient donc avant l'exécution du contenu de mon fichier.

Je peux maintenant mieux séparer les différents rôles :

- **npm** : trouve et lance la commande associée au script demandé ;
- **tsx** : permet d'exécuter mon fichier TypeScript ;
- **`index.ts`** : contient le code que j'ai écrit ;
- **Express** : est une bibliothèque utilisée par ce code pour créer le serveur.

Pour vérifier mon raisonnement, je me suis demandé ce qui se passerait si je supprimais :

```ts
import express from "express";
```

`tsx` serait quand même lancé, puisque son exécution dépend du script `dev` et non de l'import d'Express.

En revanche, si le reste du fichier continue à utiliser `express()`, l'exécution rencontrera ensuite une erreur puisque `express` ne sera plus défini.

---

## 2. Installation de `@types/express`

En reprenant mon fichier :

```ts
import express from "express";
```

VS Code signalait un problème sur Express.

J'ai installé :

```bash
npm i --save-dev @types/express
```

Je savais déjà que les packages `@types` étaient liés au typage avec TypeScript.

Ce qui reste à clarifier pour moi est surtout la raison pour laquelle ce type de package est installé dans les `devDependencies` avec `--save-dev`, plutôt que comme une dépendance classique avec simplement `npm i`.

Cette situation rejoint donc directement une question que j'avais laissée ouverte lors de la session précédente.

### À clarifier

- Quelle est la différence entre `dependencies` et `devDependencies` ?
- Que fait exactement `--save-dev` ?
- Pourquoi `express` et `@types/express` ne sont-ils pas placés dans la même catégorie ?
- Est-ce que toutes les dépendances utilisées avec TypeScript doivent être installées avec `--save-dev` ?

---

## 3. Clarifier `dependencies` et `devDependencies`

Je savais déjà que les packages `@types` étaient utilisés pour le typage avec TypeScript.

En revanche, je ne comprenais pas réellement pourquoi certains packages étaient installés avec :

```bash
npm i
```

et d'autres avec :

```bash
npm i --save-dev
```

Je me demandais notamment pourquoi ne pas simplement installer tous les packages de la même manière.

J'ai repris cette distinction à partir des packages que j'utilise réellement dans mon petit back-end :

```text
dependencies
├── express
└── dotenv

devDependencies
├── @types/node
├── @types/express
└── tsx
```

La distinction que je retiens pour le moment est la suivante :

- les `dependencies` correspondent aux dépendances nécessaires au fonctionnement de l'application ;
- les `devDependencies` regroupent les outils dont j'ai besoin principalement pendant le développement.

L'option :

```bash
--save-dev
```

permet donc d'indiquer à npm que le package doit être enregistré dans les `devDependencies` du `package.json`.

Je pensais au départ que séparer les deux catégories permettait surtout d'éviter des conflits entre trop de packages.

J'ai précisé ce point : ce n'est pas la raison principale. La séparation permet surtout de distinguer ce qui est nécessaire à l'exécution de l'application de ce qui sert à son développement, et d'éviter notamment d'installer inutilement certains outils de développement dans un environnement qui n'en a pas besoin.

### Une confusion autour de `tsx`

Lorsque j'ai essayé de classer les packages, j'ai d'abord placé `tsx` dans les `dependencies`.

Mon raisonnement était que `tsx` servait à compiler le TypeScript et qu'il devait donc être nécessaire pour faire fonctionner le serveur.

Cette réponse m'a permis de repérer une confusion qui était encore présente dans mon raisonnement :

```text
tsc
→ compilateur TypeScript

tsx
→ outil pratique pour exécuter directement un fichier TypeScript
   pendant le développement
```

Dans mon environnement de développement, j'utilise actuellement :

```json
"dev": "tsx src/index.ts"
```

Mais si l'application est ensuite compilée :

```text
src/index.ts
    ↓
   tsc
    ↓
dist/index.js
```

le serveur peut exécuter directement le JavaScript obtenu :

```bash
node dist/index.js
```

Dans cette situation, `tsx` n'est plus nécessaire pour exécuter le fichier JavaScript.

C'est ce raisonnement qui m'a permis de mieux comprendre pourquoi `tsx` est placé dans les `devDependencies`.

---

## 4. Première tentative de connexion entre le front-end et mon back-end d'apprentissage

Après avoir réussi à lancer mon serveur Express, j'ai voulu vérifier concrètement comment établir une première communication entre le front-end existant de GamerChallenge et le petit back-end que je reconstruis séparément.

Je ne souhaite pas modifier directement le back-end développé par l'équipe pour cette expérimentation.

Mon organisation actuelle est donc la suivante :

```text
gamerChallenge/
├── back/       → back-end du projet de l'équipe
├── backend/    → mon back-end d'apprentissage construit depuis zéro
└── front/      → front-end réel du projet
```

L'objectif est d'utiliser le vrai front-end du projet avec mon back-end d'apprentissage afin de comprendre progressivement comment les deux communiquent.

### Protéger le front-end avant l'expérimentation

Avant de modifier le front-end, j'ai réalisé que je ne voulais pas tester directement sur la branche `develop`, notamment parce que cette expérimentation est personnelle et que les autres membres de l'équipe ne travaillent pas actuellement dessus.

J'ai donc créé une branche dédiée :

```bash
git switch -c feat/connection_B-F
```

Cela me permet de tester la connexion sans intégrer directement mes modifications dans `develop`.

Si l'expérimentation n'est finalement pas conservée, je pourrai revenir sur `develop` sans y intégrer ce travail.

---

## 5. Créer un premier point de communication avec Express

Pour commencer, je ne cherche pas encore à récupérer de véritables challenges depuis PostgreSQL.

Je veux uniquement vérifier le chemin suivant :

```text
Front-end
    ↓
requête HTTP
    ↓
Back-end Express
    ↓
réponse JSON
    ↓
Front-end
```

J'ai donc ajouté une route de test dans mon back-end :

```ts
app.get("/api/test", (req, res) => {
  res.json({ message: "Connexion avec le back réussie !" });
});
```

Cette petite route m'a également permis de reprendre plusieurs notions déjà rencontrées :

```text
app.get()
→ réception d'une requête HTTP GET

req
→ request / requête reçue

res
→ response / réponse envoyée

res.json()
→ envoi d'une réponse au format JSON
```

À ce stade, cette route ne représente aucune fonctionnalité métier de GamerChallenge. Elle sert uniquement à vérifier la communication entre les deux applications.

---

## 6. Envoyer une requête depuis `HomePage`

Dans le front-end, j'ai utilisé `useEffect` afin de déclencher une requête lors du chargement de `HomePage`.

J'ai ajouté :

```ts
useEffect(() => {
  // Test de connexion avec le back-end.
  fetch("http://localhost:3000/api/test")
    .then((response) => response.json())
    .then((data) => console.log(data));
}, []);
```

Mon back-end d'apprentissage utilisant le port `3000`, j'ai volontairement utilisé cette adresse dans le `fetch`.

Ce test m'a permis de mieux visualiser le rôle de `fetch` dans la communication entre les deux applications :

```text
HomePage
    ↓
fetch("http://localhost:3000/api/test")
    ↓
Express
    ↓
app.get("/api/test")
    ↓
res.json(...)
    ↓
réponse reçue par le front
```

Je comprends donc mieux que l'on ne « connecte » pas chaque ligne du front-end au back-end.

Le front-end effectue des requêtes HTTP vers les points d'entrée de l'API lorsqu'il a besoin de données ou d'une action du serveur.

### Reprendre le rôle de `useEffect`

En ajoutant cette requête, je me suis rendu compte que le rôle précis de `useEffect` était devenu un peu flou dans ma mémoire.

Je l'associais à l'idée d'un changement entre l'état initial et ce qui se passe après le chargement du composant, mais cette représentation n'était pas suffisamment précise.

`useEffect` permet d'exécuter un traitement qui ne fait pas directement partie du rendu du composant.

Dans mon cas, le traitement concerné est la requête HTTP vers le back-end :

```ts
useEffect(() => {
  fetch("http://localhost:3000/api/test")
    .then((response) => response.json())
    .then((data) => console.log(data));
}, []);
```

Je peux représenter simplement son fonctionnement actuel comme ceci :

```text
HomePage est rendu
        ↓
useEffect est exécuté
        ↓
fetch() envoie la requête HTTP
        ↓
le back-end répond
```

Le tableau vide :

```ts
[]
```

est le tableau des dépendances du `useEffect`.

Dans ce cas, aucune dépendance n'est surveillée. Pour le moment, je retiens donc que cet effet est utilisé pour déclencher cette opération lors de la première apparition du composant, plutôt que de relancer directement le `fetch` à chaque rendu.

J'avais également tendance à mélanger légèrement `useEffect` et `useState`.

Je garde pour l'instant cette distinction simple :

```text
useEffect
→ déclencher un traitement lié au composant
→ ici : demander des données au back-end

useState
→ conserver une donnée dans l'état du composant
→ une modification du state peut provoquer un nouveau rendu
```

Pour mon test actuel, je n'utilise pas encore `useState`, puisque je veux seulement vérifier la connexion et afficher la réponse dans la console avec :

```ts
console.log(data);
```

Lorsque je récupérerai réellement les challenges pour les afficher dans `HomePage`, je pourrai reprendre `useEffect` et `useState` ensemble afin de mieux comprendre leur complémentarité.

### Une confusion avec `async` / `await`

En regardant le fonctionnement de `useEffect`, j'ai eu l'impression qu'il remplissait un rôle proche de `async` et `await`, puisqu'une requête est lancée puis traitée lorsque la réponse arrive.

J'ai clarifié que les responsabilités sont différentes :

```text
useEffect
→ détermine quand déclencher le traitement

fetch + .then() ou async/await
→ gèrent le traitement asynchrone
```

Dans mon exemple, ce n'est donc pas `useEffect` qui gère l'asynchronisme : il déclenche un traitement qui contient lui-même une opération asynchrone.

---

## 7. Première erreur rencontrée : CORS

Lors du premier test depuis le navigateur, la requête n'a pas abouti.

La console du navigateur affichait notamment :

```text
Blocage d'une requête multiorigines (Cross-Origin Request)
```

et indiquait que l'en-tête :

```text
Access-Control-Allow-Origin
```

était absent.

Le front-end fonctionne actuellement sur :

```text
http://localhost:5173
```

alors que mon back-end fonctionne sur :

```text
http://localhost:3000
```

Cette erreur m'a permis de retrouver concrètement l'utilité de CORS, que j'avais déjà vu dans le back-end du projet sans encore l'avoir rencontré moi-même dans cette reconstruction.

J'ai donc commencé par installer le package :

```bash
npm i cors
```

Puis j'ai ajouté l'import :

```ts
import cors from "cors";
```

---

## 8. Une erreur TypeScript qui réutilise immédiatement les notions précédentes

Après l'installation de `cors`, TypeScript m'a signalé :

```text
Le fichier de déclaration du module 'cors' est introuvable.
'.../node_modules/cors/lib/index.js' a implicitement un type 'any'.

Essayez 'npm i --save-dev @types/cors'
```

Cette fois, j'ai pu identifier directement ce qui manquait.

Le package `cors` est nécessaire au fonctionnement du serveur :

```text
cors
→ dependency
```

alors que les déclarations de types sont utilisées par TypeScript pendant le développement :

```text
@types/cors
→ devDependency
```

J'ai donc installé :

```bash
npm i --save-dev @types/cors
```

Cette situation m'a permis de réutiliser immédiatement la distinction que je venais de travailler entre `dependencies` et `devDependencies`.

Ce n'était plus seulement une distinction observée dans `package.json` : j'ai rencontré un cas concret où j'ai dû déterminer moi-même comment installer les deux packages.

---

## 9. Validation de la première connexion Front / Back

Après avoir configuré CORS pour autoriser les requêtes provenant du front-end sur `http://localhost:5173`, j'ai relancé le test.

Cette fois, la console du navigateur a affiché :

`Object { message: "Connexion avec le back réussie !" }`

La communication fonctionne donc dans les deux sens :

Front-end → requête HTTP → Express → réponse JSON → Front-end.

Ce premier test reste volontairement très simple : aucune base de données ni donnée métier n'est encore utilisée.

Il m'a cependant permis de vérifier concrètement le rôle de `fetch`, d'une route Express et de CORS dans la communication entre mes deux applications.

---

## 10. Afficher réellement une donnée du back-end dans `HomePage`

Après avoir vérifié la réponse du back-end avec `console.log`, j'ai voulu faire un test encore très simple avant de travailler avec les véritables données des challenges.

Mon objectif était uniquement d'afficher directement dans la page le message envoyé par le back-end :

```text
Connexion avec le back réussie !
```

Jusqu'ici, la réponse était bien reçue par le front-end, mais elle était seulement affichée dans la console.

Pour pouvoir utiliser cette donnée dans le rendu React, j'ai ajouté `useState` :

```ts
const [message, setMessage] = useState("");
```

Puis j'ai remplacé le `console.log` par :

```ts
.then((data) => {
  setMessage(data.message);
});
```

Enfin, j'ai temporairement ajouté dans le JSX :

```tsx
<p>{message}</p>
```

Le chemin de la donnée devient donc :

```text
Back-end
    ↓
res.json({ message: "..." })
    ↓
fetch()
    ↓
data.message
    ↓
setMessage(data.message)
    ↓
message
    ↓
<p>{message}</p>
```

Ce petit test m'a permis de voir concrètement la complémentarité entre les éléments que je venais de reprendre :

```text
useEffect
→ déclenche la requête vers le back-end

fetch
→ effectue la requête HTTP

useState
→ conserve la donnée reçue

JSX
→ utilise cette donnée pour l'afficher
```

### Une erreur sur l'emplacement de `useState`

Lors de ma première tentative, j'ai placé `useState` au mauvais endroit.

React m'a alors signalé :

```text
React Hook "useState" cannot be called inside a callback.
React Hooks must be called in a React function component
or a custom React Hook function.
```

La page ne pouvait plus être rendue correctement.

En relisant mon code, j'ai constaté que le problème venait simplement de l'emplacement du Hook. Après avoir replacé `useState` directement dans le composant `HomePage`, le rendu a de nouveau fonctionné.

### Résultat

Le message provenant de mon back-end s'affiche maintenant directement dans la page :

```text
Connexion avec le back réussie !
```

J'ai donc réussi à aller un peu plus loin que le simple test dans la console :

```text
Express
   ↓
HTTP / JSON
   ↓
React
   ↓
state
   ↓
affichage dans HomePage
```

Pour le moment, je préfère conserver ce test très simple.

La prochaine étape pourra consister à remplacer progressivement ce message de test par une véritable donnée liée à GamerChallenge, sans complexifier immédiatement le test avec plusieurs challenges.