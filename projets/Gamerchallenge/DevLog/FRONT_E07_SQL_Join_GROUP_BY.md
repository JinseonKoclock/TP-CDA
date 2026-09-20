# DevLog E08 — Enrichissement de l'API : auteur et nombre de likes

## Objectif

Aujourd'hui, j'ai continué à connecter les données PostgreSQL à l'interface React.

Les cartes des challenges affichaient déjà les informations provenant de la table `challenge`, mais il manquait encore deux informations :

- le pseudo de l'auteur du challenge ;
- le nombre de likes reçus par le challenge.

Ces informations ne se trouvent pas directement dans la table `challenge`.

L'objectif était donc de comprendre comment récupérer des données provenant de plusieurs tables avec SQL, puis de les transmettre jusqu'au composant React `ChallengeCardBack`.

---

## 1. Récupération du pseudo de l'auteur

La table `challenge` contient une clé étrangère :

`challenge.user_id`

Cette clé fait référence à la clé primaire :

`user.id`

J'ai compris que la relation entre les deux tables pouvait donc être utilisée avec un `JOIN` :

```sql
JOIN "user"
  ON challenge.user_id = "user".id
```

Une fois les deux tables reliées, je peux récupérer le pseudo de l'utilisateur :

```sql
"user".pseudo
```

### Ce que j'ai compris

Le `JOIN` sert à relier les lignes de deux tables grâce à une relation entre leurs clés.

Dans mon cas :

- `challenge.user_id` → clé étrangère ;
- `user.id` → clé primaire.

Le `JOIN` permet ensuite d'accéder aux informations de l'utilisateur correspondant, notamment son `pseudo`.

---

## 2. Récupération du nombre de likes

Les likes des challenges sont enregistrés dans la table `vote_challenge`.

Cette table contient notamment :

- `user_id` ;
- `challenge_id`.

Pour savoir combien de likes possède un challenge, il faut relier :

`challenge.id`

avec :

`vote_challenge.challenge_id`

Puis compter le nombre de lignes correspondantes.

```sql
COUNT(vote_challenge.challenge_id) AS likes
```

Le résultat du `COUNT()` est renommé `likes` afin d'obtenir une propriété facilement exploitable dans l'API.

---

## 3. Utilisation de LEFT JOIN

Pour les votes, j'ai utilisé :

```sql
LEFT JOIN vote_challenge
  ON challenge.id = vote_challenge.challenge_id
```

J'ai compris pourquoi un `LEFT JOIN` est intéressant ici.

Un challenge peut exister sans avoir encore reçu de vote.

Avec le `LEFT JOIN`, le challenge reste présent dans le résultat même lorsqu'aucune ligne correspondante n'existe dans `vote_challenge`.

Cela permet par exemple d'afficher :

`0 ❤️`

au lieu de faire disparaître complètement le challenge de la requête.

J'ai pu vérifier ce comportement directement dans l'interface : un challenge avec `0` like reste bien affiché.

---

## 4. Requête SQL obtenue

```sql
SELECT
  challenge.*,
  "user".pseudo,
  COUNT(vote_challenge.challenge_id) AS likes
FROM challenge
JOIN "user"
  ON challenge.user_id = "user".id
LEFT JOIN vote_challenge
  ON challenge.id = vote_challenge.challenge_id
GROUP BY challenge.id, "user".pseudo
LIMIT 3;
```

### À revoir

J'ai utilisé `GROUP BY` pour permettre le regroupement des votes par challenge avec `COUNT()`.

Je n'ai pas encore suffisamment étudié son fonctionnement pour considérer cette notion comme acquise.

Je dois revenir dessus afin de comprendre précisément pourquoi l'utilisation d'une fonction d'agrégation comme `COUNT()` nécessite ici un regroupement des résultats.

---

## 5. Transmission des nouvelles données vers React

Après avoir enrichi la réponse de l'API, j'ai ajouté les nouvelles propriétés au type `IChallenge`.

Notamment :

```ts
pseudo: string;
likes: number;
```

J'ai également ajouté ces propriétés dans les props du composant `ChallengeCardBack`.

Dans `HomePageBack`, les données doivent être explicitement transmises au composant :

```tsx
pseudo={challenge.pseudo}
likes={challenge.likes}
```

J'avais d'abord ajouté `pseudo` dans les props du composant sans le transmettre depuis `HomePageBack`.

Le texte `Par` apparaissait donc dans la carte, mais pas le pseudo.

Cela m'a permis de mieux comprendre qu'une propriété déclarée dans une interface TypeScript ne transmet aucune donnée automatiquement.

Le chemin réel est :

```text
PostgreSQL
    ↓
requête SQL
    ↓
API Express
    ↓
fetch()
    ↓
objet challenge
    ↓
props React
    ↓
ChallengeCardBack
    ↓
affichage
```

---

## 6. Affichage dans ChallengeCardBack

La carte affiche maintenant :

- le titre du challenge ;
- le nom du jeu ;
- la description ;
- la miniature de la vidéo ;
- le pseudo de l'auteur ;
- la date formatée ;
- le nombre de likes ;
- le lien vers le détail du challenge.

Pour l'auteur et la date :

```tsx
<div className="challenge-card__author">
  <span>
    <FaUser aria-hidden="true" />
    Par {pseudo}
  </span>

  <span>{formattedDate}</span>
</div>
```

Pour les likes :

```tsx
<div className="challenge-card__stats">
  <span title="Nombre de likes">
    <FaHeart aria-hidden="true" />
    {likes}
  </span>
</div>
```

---

## 7. Debug rencontré

Après l'ajout du pseudo, celui-ci n'apparaissait pas dans l'interface.

En inspectant les données dans la console du navigateur, j'ai constaté que `pseudo` n'était pas présent dans l'objet reçu par React.

Cela m'a permis de chercher le problème plus haut dans le flux de données au lieu de modifier directement le composant React.

Après redémarrage du serveur backend, la nouvelle requête SQL a bien été prise en compte et `pseudo` est apparu dans la réponse de l'API.

Cette erreur m'a rappelé qu'après une modification du backend, il faut vérifier que le serveur exécute bien la nouvelle version du code.

---

## 8. Résultat

La connexion complète fonctionne maintenant :

```text
PostgreSQL
    ↓
Express / SQL
    ↓
API REST
    ↓
React
    ↓
ChallengeCardBack
```

Les cartes affichent désormais dynamiquement le pseudo de leur auteur ainsi que leur nombre de likes.

J'ai également pu vérifier qu'un challenge sans vote reste affiché avec `0` like grâce au `LEFT JOIN`.

---

## Ce que j'ai réellement appris aujourd'hui

- Relier une clé étrangère à une clé primaire avec un `JOIN`.
- Récupérer une information située dans une autre table.
- Comprendre l'intérêt d'un `LEFT JOIN` lorsqu'une relation peut ne pas exister.
- Utiliser `COUNT()` pour compter les votes associés à un challenge.
- Donner un nom au résultat calculé avec `AS likes`.
- Suivre une donnée depuis PostgreSQL jusqu'à son affichage dans React.
- Comprendre qu'une interface TypeScript décrit les données attendues mais ne transmet pas les données.
- Utiliser la console du navigateur pour déterminer à quel niveau du flux une donnée est absente.

---

---

## 9. Tri des challenges les plus populaires

Après avoir réussi à récupérer le nombre de likes, j'ai remarqué que la section de la page d'accueil s'appelle :

`Challenges à la une`

Cependant, la requête utilisait uniquement :

```sql
LIMIT 3;
```

Cela permettait de récupérer trois challenges, mais sans garantir qu'il s'agissait réellement des challenges les plus populaires.

J'ai donc ajouté un tri sur le nombre de likes :

```sql
ORDER BY likes DESC
LIMIT 3;
```

`DESC` permet de classer les résultats par ordre décroissant.

La logique devient donc :

```text
Calcul du nombre de likes
        ↓
Classement du plus grand au plus petit
        ↓
Sélection des 3 premiers résultats
```

Ainsi, les trois challenges affichés dans la section `Challenges à la une` correspondent maintenant aux challenges ayant reçu le plus de likes.

---

## 10. Séparation entre la liste complète et les challenges à la une

En réfléchissant à l'utilisation de cette requête, j'ai constaté un problème de conception.

La route :

```text
GET /api/challenges
```

ne devrait pas forcément retourner uniquement les trois challenges les plus populaires.

Cette route pourra notamment être utilisée par la page affichant la liste des challenges.

J'ai donc commencé à séparer les responsabilités entre deux controllers :

```text
getAll()
→ récupérer une liste de challenges

getFeatured()
→ récupérer les 3 challenges les plus populaires
```

L'objectif est également d'utiliser deux routes ayant des responsabilités différentes :

```text
GET /api/challenges
→ liste des challenges

GET /api/challenges/featured
→ challenges à la une de la page d'accueil
```

Cette séparation permet d'éviter de modifier une même requête SQL en fonction de la page qui l'utilise.

Elle permet également de donner aux fonctions un nom correspondant à la donnée qu'elles retournent plutôt qu'au composant React qui les utilise.

---

## 11. Bug rencontré dans le controller getAll

Lors du test de `getAll`, la requête semblait extrêmement lente.

La requête SQL était pourtant exécutée et les résultats apparaissaient dans le terminal grâce à :

```ts
console.log(challenges.rows);
```

Le problème ne venait finalement pas de PostgreSQL.

J'avais oublié d'envoyer la réponse HTTP :

```ts
res.status(200).json(challenges.rows);
```

Mon controller s'arrêtait donc après :

```ts
const challenges = await Client.query(query);
console.log(challenges.rows);
```

La base de données avait terminé son travail, mais Express ne répondait jamais au client.

Le navigateur continuait donc à attendre la réponse.

### Ce que j'ai compris

Il faut distinguer :

```ts
console.log(challenges.rows);
```

qui affiche les données uniquement dans le terminal du serveur,

et :

```ts
res.status(200).json(challenges.rows);
```

qui envoie réellement une réponse HTTP au frontend.

Le fonctionnement complet est donc :

```text
Requête HTTP
    ↓
Controller Express
    ↓
Client.query()
    ↓
PostgreSQL
    ↓
challenges.rows
    ↓
res.status(200).json(...)
    ↓
Réponse HTTP
    ↓
Frontend
```

Cette erreur m'a également permis de comprendre qu'une requête qui semble rester longtemps en attente ne signifie pas forcément que la base de données est lente.

Il faut vérifier à quel endroit le traitement s'arrête.

---

## 12. Limitation temporaire de la liste des challenges

Même si la base de données contient actuellement peu de données, j'ai décidé de ne pas retourner un nombre illimité de challenges.

Pour le moment, la requête `getAll` sera limitée à 12 résultats :

```sql
ORDER BY likes DESC
LIMIT 12;
```

Cette solution reste volontairement simple pour le MVP.

Une pagination pourrait être ajoutée plus tard si le nombre de challenges devient important.

Je n'ai pas encore implémenté cette pagination afin de ne pas ajouter une nouvelle fonctionnalité avant d'avoir terminé les fonctionnalités principales.

### Remarque

Le nom `getAll` devient moins précis avec un `LIMIT 12`, puisqu'il ne retourne plus réellement tous les challenges.

Pour le moment, je conserve cette organisation pendant le développement, mais ce point pourra être revu lorsque la pagination sera mise en place.

---

## 13. Réflexion sur la réutilisation du fetch dans React

Je me suis demandé s'il serait préférable de déplacer le `fetch()` directement dans le composant `ChallengeCardBack` afin d'éviter de répéter le code dans plusieurs pages.

J'ai finalement compris que cela mélangerait deux responsabilités différentes.

Actuellement :

```text
HomePageBack
    ↓
récupère une liste de challenges
    ↓
map()
    ↓
ChallengeCardBack
    ↓
affiche un challenge
```

`ChallengeCardBack` reçoit donc les informations d'un seul challenge grâce aux props.

Il n'a pas besoin de récupérer lui-même toute la liste depuis l'API.

Si plusieurs pages utilisent plus tard exactement la même logique de récupération des données, il sera possible de factoriser cette partie dans une fonction dédiée ou dans un custom hook.

Pour le moment, je conserve la récupération des données au niveau de la page.

---

## 14. Réflexion sur l'URL de la page détail

J'ai également commencé à réfléchir à la future page de détail d'un challenge.

Actuellement, le lien utilise l'identifiant :

```tsx
<Link to={`/challenges/${id}`}>
```

Ce qui donnera par exemple :

```text
/challenges/3
```

J'ai envisagé d'utiliser le titre du challenge afin d'obtenir une URL plus lisible.

Une autre possibilité serait d'utiliser plus tard un `slug`, par exemple :

```text
/challenges/3/chateau-en-1-heure
```

Dans ce cas :

- `3` resterait l'identifiant technique permettant de retrouver précisément le challenge ;
- `chateau-en-1-heure` rendrait l'URL plus lisible.

Je conserve cependant pour le moment la solution simple basée sur l'identifiant.

La gestion de la route dynamique et de la page `ChallengeDetail` sera réalisée dans une prochaine étape.

---

## Bilan complémentaire

Cette partie du travail m'a permis de comprendre plusieurs éléments qui dépassent l'affichage des données.

J'ai notamment commencé à mieux distinguer les responsabilités entre :

```text
SQL
→ sélectionner et organiser les données

Controller
→ exécuter la requête et envoyer la réponse HTTP

Route API
→ définir l'URL permettant d'accéder à la ressource

Page React
→ demander et gérer les données nécessaires à la page

Composant ChallengeCard
→ afficher un challenge reçu via ses props
```

J'ai également rencontré une erreur importante avec `getAll` : la requête SQL fonctionnait, mais l'absence de `res.json()` empêchait la requête HTTP de se terminer.

Cela m'a permis de mieux comprendre qu'un problème visible dans le frontend peut provenir de différentes étapes du flux et qu'il faut vérifier chaque étape avant de conclure que la base de données ou React est responsable.

---

## Prochaine étape

La prochaine étape sera la création de la page de détail d'un challenge.

Objectif prévu :

```text
ChallengeCard
    ↓
clic sur "Voir les détails"
    ↓
/challenges/:id
    ↓
récupération de l'id dynamique
    ↓
appel de l'API
    ↓
récupération d'un seul challenge
    ↓
affichage dans ChallengeDetailPage
```

La question d'une URL utilisant un `slug` pourra être étudiée après avoir compris et fait fonctionner cette première version avec `:id`.