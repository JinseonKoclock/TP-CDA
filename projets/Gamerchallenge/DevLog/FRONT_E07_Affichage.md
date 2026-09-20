# DevLog E07 — Affichage dynamique des challenges depuis l'API

## Objectif

L'objectif de cette étape était de remplacer progressivement les données statiques du front-end par les données réelles provenant de la base PostgreSQL.

Le premier objectif concret était d'afficher les trois premiers challenges de la base de données dans les composants `ChallengeCard`.

---

## 1. Récupération des challenges depuis l'API

Le back-end possède actuellement la route :

```text
GET /api/challenges
```

Le controller exécute une requête SQL sur PostgreSQL :

```sql
SELECT * FROM challenge LIMIT 3;
```

Cette route retourne donc les trois premiers challenges enregistrés dans la base de données.

Les données retournées contiennent notamment :

- `id`
- `title`
- `game_name`
- `url_video`
- `description`
- `rules`
- `user_id`
- `created_at`

---

## 2. Connexion du front-end à l'API

Dans le front-end, l'adresse du back-end est définie dans `.env` :

```env
VITE_API_URL=http://localhost:3000
```

La HomePage récupère ensuite les données avec `fetch()` :

```ts
const response = await fetch(
    `${import.meta.env.VITE_API_URL}/api/challenges`
);

const data = await response.json();

setChallenges(data);
```

Le state a été typé avec l'interface `IChallenge` :

```ts
const [challenges, setChallenges] = useState<IChallenge[]>([]);
```

Cela signifie que `challenges` contient un tableau de challenges.

---

## 3. Création de l'interface IChallenge

Une interface TypeScript a été créée pour représenter la structure d'un challenge reçu depuis l'API.

```ts
export default interface IChallenge {
    id: number;
    title: string;
    game_name: string;
    url_video: string;
    description: string;
    rules: string;
    user_id: number;
    created_at: string;
}
```

Cette interface permet de représenter côté front-end la structure des données provenant du back-end.

---

## 4. Utilisation de `map()`

Les challenges reçus depuis l'API sont stockés dans :

```ts
challenges
```

Comme il s'agit d'un tableau, `map()` permet de créer un composant `ChallengeCard` pour chaque challenge.

```tsx
{challenges.map((challenge) => (
    <ChallengeCard_Back
        key={challenge.id}
        id={challenge.id}
        title={challenge.title}
        game_name={challenge.game_name}
        url_video={challenge.url_video}
        description={challenge.description}
        created_at={challenge.created_at}
    />
))}
```

### Ce que j'ai compris

Le composant qui possède le tableau utilise `map()`.

`HomePage` possède :

```text
challenges[]
```

Elle crée donc plusieurs composants :

```text
ChallengeCard
```

Le fonctionnement est donc :

```text
HomePage
   │
   └── challenges[]
          │
          └── map()
               │
               ├── ChallengeCard 1
               ├── ChallengeCard 2
               └── ChallengeCard 3
```

Chaque `ChallengeCard` reçoit ensuite les informations d'un seul challenge grâce aux props.

---

## 5. Props du composant ChallengeCard

Le composant `ChallengeCard` ne reçoit actuellement que les informations nécessaires à son affichage.

```ts
interface ChallengeCardProps {
    id: number;
    title: string;
    game_name: string;
    url_video: string;
    description: string;
    created_at: string;
}
```

Cela permet de distinguer :

- la structure complète d'un challenge provenant de l'API avec `IChallenge` ;
- les données réellement nécessaires au composant avec `ChallengeCardProps`.

---

## 6. Gestion de la vidéo

Au départ, j'ai essayé d'intégrer directement les vidéos avec une `iframe`.

Cette solution posait plusieurs problèmes :

- les URL YouTube classiques ne sont pas directement des URL `embed` ;
- plusieurs formats d'URL YouTube existent ;
- le cahier des charges ne précise pas que les vidéos doivent obligatoirement provenir de YouTube ;
- d'autres plateformes vidéo pourraient être utilisées.

J'ai donc choisi une solution plus simple pour le MVP.

### Solution retenue

Pour une URL YouTube :

1. récupérer l'identifiant de la vidéo ;
2. construire l'URL de sa miniature ;
3. afficher cette miniature dans la carte ;
4. lorsque l'utilisateur clique sur la miniature, ouvrir l'URL originale de la vidéo.

L'URL originale reste utilisée pour le lien :

```tsx
<a
    href={url_video}
    target="_blank"
    rel="noopener noreferrer"
>
```

Ainsi, le lien vers la vidéo n'est pas limité à YouTube.

Si l'URL appartient à une autre plateforme, une image par défaut peut être utilisée comme miniature tout en conservant le lien original.

---

## 7. Récupération des miniatures YouTube

Les données de test contenaient plusieurs formats d'URL YouTube :

```text
youtube.com/live/...
youtu.be/...
```

Une fonction permet donc d'extraire l'identifiant de la vidéo selon le format de l'URL.

L'identifiant permet ensuite de construire l'adresse de la miniature :

```text
https://img.youtube.com/vi/VIDEO_ID/hqdefault.jpg
```

Cette miniature est affichée dans la carte avec une icône de lecture.

L'objectif de cette fonction n'est pas de lire la vidéo directement dans l'application.

Elle sert uniquement à récupérer une miniature adaptée lorsque l'URL correspond à une vidéo YouTube.

Le clic utilise toujours l'URL originale enregistrée dans la base de données.

---

## 8. Problèmes rencontrés

### 8.1 Les anciennes données restaient affichées

Au début, les cartes affichaient toujours les anciennes données statiques.

Pour comprendre le problème, j'ai vérifié progressivement le parcours des données :

```text
PostgreSQL
    ↓
API
    ↓
fetch()
    ↓
state challenges
    ↓
map()
    ↓
ChallengeCard
```

Les logs de la console ont permis de confirmer que l'API envoyait correctement les nouvelles données.

---

### 8.2 `url_video` apparaissait comme `undefined`

Pendant le débogage, j'ai utilisé :

```ts
console.log("challenge :", challenge);
console.log("challenge.url_video :", challenge.url_video);
```

Cela m'a permis de vérifier directement le contenu des objets présents dans le tableau `challenges`.

J'ai finalement confirmé que `url_video` était bien présent dans les données reçues depuis l'API.

Cette vérification m'a permis de ne pas modifier inutilement la base de données ou le back-end alors que les données arrivaient correctement jusqu'au front-end.

---

### 8.3 Mauvaise utilisation du `return`

Pendant les tests, le `return` principal du composant avait été commenté.

Le `map()` était exécuté, mais le JSX généré n'était plus retourné par le composant.

J'ai compris qu'un composant React doit retourner le JSX qui doit être affiché :

```tsx
const Component = () => {
    // Traitements

    return (
        // JSX affiché
    );
};
```

Récupérer correctement les données ne suffit donc pas.

Il faut également retourner les éléments React qui utilisent ces données pour qu'ils soient affichés dans le navigateur.

---

### 8.4 Ancienne image toujours affichée

Une ancienne image continuait à apparaître dans la zone média de la carte.

Elle provenait du CSS :

```css
background-image: url("../../assets/images/Card_Background.jpg");
```

Elle était définie directement dans `.challenge-card__media`.

Le problème ne venait donc pas du cache du navigateur.

Cette image de fond a été retirée afin de permettre l'affichage dynamique de la miniature correspondant à la vidéo.

---

### 8.5 Vérification des fichiers utilisés

Pendant les tests, plusieurs versions des fichiers étaient conservées afin de ne pas écraser immédiatement le travail existant.

Par exemple :

```text
HomePage
HomePage_Back

ChallengeCard
ChallengeCard_Back
```

Cela a demandé de vérifier attentivement :

- les imports ;
- les noms de fichiers ;
- les fichiers CSS associés ;
- le composant réellement utilisé par le router.

Cette étape m'a montré qu'un problème d'affichage ne vient pas forcément de la logique de récupération des données.

Il peut également venir d'un mauvais fichier importé ou d'une mauvaise association entre un composant et son fichier CSS.

---

## 9. Résultat obtenu

Les trois premiers challenges de PostgreSQL sont maintenant affichés dynamiquement dans le front-end.

Les informations suivantes proviennent réellement de l'API :

```text
title
game_name
description
created_at
url_video
```

Les miniatures correspondant aux vidéos YouTube sont également affichées.

Le flux complet fonctionne maintenant :

```text
PostgreSQL
      ↓
pg
      ↓
db_client.ts
      ↓
Controller
      ↓
Router
      ↓
Express API
      ↓
fetch()
      ↓
useState
      ↓
map()
      ↓
Props
      ↓
ChallengeCard
      ↓
Affichage dans le navigateur
```

Cette étape permet donc de confirmer que le front-end et le back-end communiquent correctement avec les données provenant de PostgreSQL.

---

## 10. Point à améliorer : affichage de la date

La date est actuellement reçue sous son format ISO.

Exemple :

```text
2025-01-31T23:00:00.000Z
```

Ce format est adapté à l'échange de données entre le back-end et le front-end, mais il n'est pas adapté à l'affichage utilisateur.

La prochaine amélioration consistera à transformer cette valeur côté front-end avec `Date`.

Par exemple :

```ts
const formattedDate = new Date(created_at).toLocaleDateString("fr-FR");
```

L'objectif est d'obtenir un affichage du type :

```text
01/02/2025
```

Je préfère conserver la date complète envoyée par l'API et gérer son format d'affichage dans le front-end.

Le back-end fournit ainsi la donnée, tandis que le front-end décide de sa présentation à l'utilisateur.

---

## 11. Données encore manquantes

La carte n'est pas encore complète.

Deux informations doivent encore être récupérées depuis la base de données.

### Auteur du challenge

L'API retourne actuellement :

```text
user_id
```

Mais l'interface doit afficher le nom de l'utilisateur.

Il faudra donc relier :

```text
challenge.user_id
        ↓
user.id
        ↓
username
```

Cette étape nécessitera une jointure SQL.

---

### Nombre de likes

La carte doit également afficher le nombre total de votes reçus par chaque challenge.

Il faudra relier :

```text
challenge
    ↓
vote_challenge
    ↓
nombre de votes
```

Cette étape permettra notamment de travailler avec :

```text
JOIN
COUNT()
GROUP BY
```

---

## 12. Prochaine étape

Compléter les données retournées par :

```text
GET /api/challenges
```

Ordre prévu :

1. récupérer le `username` de l'auteur avec une jointure SQL ;
2. vérifier le résultat retourné par l'API ;
3. récupérer le nombre de votes de chaque challenge ;
4. afficher `username` et le nombre de likes dans `ChallengeCard` ;
5. formater `created_at` pour l'affichage ;
6. réintégrer les challenges dynamiques dans la HomePage complète sans supprimer les autres sections existantes.

---

## Ce que je retiens

- `fetch()` permet au front-end d'interroger mon API.
- `response.json()` permet de récupérer les données JSON de la réponse HTTP.
- `useState<IChallenge[]>([])` stocke la liste des challenges dans le composant.
- `map()` permet de créer un composant pour chaque élément du tableau.
- Les props permettent de transmettre les données d'un challenge au composant `ChallengeCard`.
- Une donnée présente dans PostgreSQL traverse plusieurs couches avant d'être affichée dans React.
- Une API peut fonctionner correctement alors que l'affichage React ne fonctionne pas.
- Les `console.log()` permettent de vérifier à quelle étape une donnée est présente ou disparaît.
- Un composant React doit retourner son JSX pour que celui-ci soit affiché.
- Les noms des imports et des fichiers utilisés doivent être vérifiés pendant le débogage.
- Les données brutes et leur présentation à l'utilisateur ont des responsabilités différentes.
- Il est préférable de vérifier chaque étape du flux avant d'ajouter une nouvelle fonctionnalité.
- La connexion complète PostgreSQL → API → React fonctionne maintenant pour les challenges.