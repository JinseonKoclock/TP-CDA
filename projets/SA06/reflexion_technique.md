# Réflexion technique — Gestion des règles d'un challenge

## Projet de référence

Cette réflexion s'appuie notamment sur un exercice réalisé précédemment pendant la formation O'clock :

**`SA06-challenge-orecipes-largenty`**

Ce projet utilisait **Svelte** côté front-end.

Même si GamerChallenges utilise React, l'analyse de cet ancien projet m'a permis de retrouver un principe de circulation des données que je peux réutiliser dans mon projet actuel.

---

## 1. Fonctionnement observé dans le projet O'recipes

Dans `SA06-challenge-orecipes-largenty`, la description d'une recette était saisie dans un `textarea`.

```svelte
let description = $state("");
```

Le champ était lié à cette variable :

```svelte
<textarea
  id="description"
  bind:value={description}
>
</textarea>
```

Lors de la soumission du formulaire, cette valeur était intégrée dans l'objet représentant la nouvelle recette :

```js
let newRecipe = {
  id: nextId.toString(),
  title: title,
  image: imageSrc,
  description: description,
};
```

L'objet était ensuite envoyé au back-end sous forme de JSON :

```js
body: JSON.stringify(newRecipe)
```

Le principe général était donc :

```text
textarea
   ↓
variable
   ↓
objet JavaScript
   ↓
JSON
   ↓
requête POST
   ↓
Back-end / BDD
```

Pour récupérer les données, le front-end effectuait ensuite une requête GET :

```js
const response = await fetch("http://localhost:3000/recipes");
const data = await response.json();

recipes = data;
```

Puis Svelte parcourait les recettes :

```svelte
{#each recipes as recipe}
```

et affichait notamment leur description :

```svelte
<p>{recipe.description}</p>
```

Le flux complet pouvait donc être représenté ainsi :

```text
Formulaire
   ↓
Front-end
   ↓
POST
   ↓
Back-end
   ↓
BDD
   ↓
GET
   ↓
Front-end
   ↓
Rendu dans l'interface
```

---

## 2. Application envisagée dans GamerChallenges

Pour GamerChallenges, je souhaite appliquer un principe similaire à la gestion des règles d'un challenge.

Dans le MCD, les règles restent une information appartenant au challenge :

```text
CHALLENGE
────────────────
Titre
URL de la vidéo
Description
Règles
```

Je ne souhaite donc pas créer une entité `RÈGLE` indépendante uniquement pour stocker chaque ligne du règlement.

Dans le formulaire de création d'un challenge, l'utilisateur pourrait saisir l'ensemble des règles dans un seul `textarea`.

Exemple :

```text
Pas de glitch
Une seule tentative
Vidéo obligatoire
```

Ces informations pourraient être enregistrées sous la forme d'une seule valeur textuelle associée au challenge.

---

## 3. Séparer le stockage de la présentation

Le fait de stocker les règles dans une seule valeur ne signifie pas qu'elles doivent obligatoirement être affichées sous la forme d'un seul paragraphe.

Par exemple, le texte :

```text
Pas de glitch
Une seule tentative
Vidéo obligatoire
```

peut être séparé côté front-end à partir des retours à la ligne.

En JavaScript / TypeScript, le principe pourrait être :

```ts
challenge.rules.split("\n");
```

Ce traitement produirait conceptuellement :

```js
[
  "Pas de glitch",
  "Une seule tentative",
  "Vidéo obligatoire"
]
```

React pourrait ensuite parcourir ce tableau avec `map()` afin d'afficher chaque règle dans un élément `<li>`.

Le principe serait donc :

```text
BDD
│
│ rules = texte
│
▼
Front-end
│
│ split("\n")
▼
tableau temporaire
│
│ map()
▼
<li>Règle 1</li>
<li>Règle 2</li>
<li>Règle 3</li>
```

Le tableau utilisé pour l'affichage n'a donc pas besoin d'être identique à la manière dont la donnée est stockée dans la base de données.

---

## 4. Pourquoi ce choix ?

Pour le MVP de GamerChallenges, les règles servent principalement à décrire les conditions d'un challenge.

À ce stade, je n'ai pas identifié de besoin métier nécessitant de rechercher, modifier, supprimer ou relier individuellement chaque règle à d'autres données de l'application.

Créer une entité et une structure de données supplémentaires uniquement pour permettre l'affichage de plusieurs règles rendrait donc le modèle plus complexe sans répondre à un besoin fonctionnel identifié dans le MVP.

Je préfère conserver une donnée simple dans le modèle et effectuer la transformation nécessaire au moment de l'affichage.

---

## 5. Point important : données et affichage

Cette réflexion m'a permis de mieux distinguer deux problématiques :

> **La manière dont une donnée est stockée ne détermine pas obligatoirement la manière dont elle doit être affichée.**

Dans ce cas :

```text
Stockage
Règles = une valeur textuelle
          ↓
Transformation dans le front
split("\n")
          ↓
Affichage
une liste de plusieurs règles
```

L'interface peut donc présenter plusieurs éléments visuels sans qu'il soit nécessaire de créer une entité différente dans le modèle de données pour chacun de ces éléments.

---

## 6. Lien avec mon apprentissage

Cette solution ne vient pas uniquement d'une nouvelle syntaxe découverte pendant GamerChallenges.

Elle s'appuie sur un mécanisme déjà rencontré dans le projet :

**`SA06-challenge-orecipes-largenty`**

En reprenant ce projet et en analysant le trajet de la donnée entre le formulaire, l'objet JavaScript, la requête HTTP et le rendu, j'ai pu réutiliser le même raisonnement dans un autre contexte.

Le framework est différent :

```text
O'recipes       → Svelte
GamerChallenges → React
```

mais le principe reste comparable :

```text
saisie utilisateur
      ↓
état du front-end
      ↓
requête HTTP
      ↓
back-end / BDD
      ↓
récupération
      ↓
transformation éventuelle
      ↓
rendu
```

Cela me permet de réutiliser un concept appris précédemment plutôt que de reproduire une solution sans comprendre son fonctionnement.

---

> Et pour le jury, je retiendrais surtout cette explication courte, parce qu'elle correspond vraiment à ton raisonnement :

« Pour le MVP, je n'avais pas besoin de gérer chaque règle comme une donnée indépendante. J'ai donc choisi de conserver les règles comme une information du challenge. L'utilisateur peut les saisir dans un textarea, puis côté front je peux séparer le texte sur les retours à la ligne et utiliser map() pour afficher chaque règle dans un li. J'avais déjà rencontré un flux similaire dans le projet O'clock SA06-challenge-orecipes-largenty, réalisé avec Svelte. Même si GamerChallenges utilise React, j'ai pu réutiliser le principe de circulation et de transformation des données. »

Il y a surtout une phrase que je veux que tu retiennes, parce qu'elle montre une vraie compréhension de conception et pas seulement de syntaxe :

Le stockage d'une donnée et sa représentation dans l'interface sont deux problématiques différentes.

Ça, si le jury te demande « Pourquoi ne pas avoir créé une table rule ? », c'est le début d'une réponse solide. Ensuite tu rattaches la décision au besoin réel du MVP : aucune fonctionnalité ne nécessite actuellement de manipuler individuellement les règles en BDD.

Et je garde bien le nom exact comme référence pour la suite : SA06-challenge-orecipes-largenty → Svelte → textarea → POST/GET → rendu → source d'inspiration pour Règles dans GamerChallenges.

---

