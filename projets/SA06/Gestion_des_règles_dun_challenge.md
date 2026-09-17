# GamerChallenges — Gestion des règles d'un challenge

## 1. Choix de conception

Pour le MVP de GamerChallenges, les règles d'un challenge ne seront pas gérées comme des données indépendantes.

Elles resteront une information appartenant directement au `CHALLENGE`.

Dans le MCD, le principe envisagé est donc :

```text
CHALLENGE
────────────────
Titre
URL de la vidéo
Description
Règles
```

L'objectif est de permettre à l'utilisateur de saisir plusieurs règles dans un seul `textarea`.

Exemple :

```text
Pas de glitch
Une seule tentative
Vidéo obligatoire
```

Ces règles pourront être enregistrées dans la base de données sous la forme d'une seule valeur textuelle.

---

## 2. Projet utilisé comme référence

Cette réflexion s'appuie sur un exercice réalisé précédemment pendant la formation O'clock :

**`SA06-challenge-orecipes-largenty`**

Ce projet utilisait **Svelte** côté front-end.

Dans ce projet, une valeur saisie dans un `textarea` était conservée dans l'état du composant :

```svelte
let description = $state("");
```

Le `textarea` était lié à cette variable :

```svelte
<textarea
  id="description"
  bind:value={description}
>
</textarea>
```

Lors de la soumission du formulaire, cette valeur était ajoutée dans un objet JavaScript :

```js
let newRecipe = {
  id: nextId.toString(),
  title: title,
  image: imageSrc,
  description: description,
};
```

Puis l'objet était envoyé au back-end sous forme de JSON :

```js
body: JSON.stringify(newRecipe)
```

Le principe observé était donc :

```text
textarea
   ↓
état du composant
   ↓
objet JavaScript
   ↓
JSON
   ↓
requête POST
   ↓
Back-end
   ↓
BDD
```

Même si GamerChallenges utilise React et non Svelte, le principe de circulation des données reste similaire.

---

# 3. Application avec React

## Formulaire de création d'un challenge

### Fichier envisagé

```text
src/pages/CreateChallengePage/CreateChallengePage.tsx
```

### Gestion du state

```tsx
import { useState } from "react";

function CreateChallengePage() {
  // Contient l'ensemble des règles saisies dans le textarea.
  // Les retours à la ligne font partie de la chaîne de caractères.
  const [rules, setRules] = useState("");

  return (
    <main>
      <h1>Proposer un challenge</h1>

      <form>
        <div>
          <label htmlFor="rules">Règles</label>

          {/*
            L'utilisateur peut saisir plusieurs règles
            dans un seul textarea.

            Exemple :
            Pas de glitch
            Une seule tentative
            Vidéo obligatoire
          */}
          <textarea
            id="rules"
            name="rules"
            value={rules}
            onChange={(event) => setRules(event.target.value)}
            placeholder={`Pas de glitch
Une seule tentative
Vidéo obligatoire`}
            rows={6}
          />
        </div>
      </form>
    </main>
  );
}

export default CreateChallengePage;
```

Si l'utilisateur saisit :

```text
Pas de glitch
Une seule tentative
Vidéo obligatoire
```

la valeur contenue dans `rules` correspond conceptuellement à :

```ts
"Pas de glitch\nUne seule tentative\nVidéo obligatoire"
```

Le caractère `\n` représente ici un retour à la ligne.

---

# 4. Envoi des règles au back-end

### Fichier envisagé

```text
src/pages/CreateChallengePage/CreateChallengePage.tsx
```

La fonction de soumission pourra récupérer la valeur de `rules` et l'intégrer à l'objet représentant le challenge.

```tsx
const handleSubmit = async (event: React.FormEvent<HTMLFormElement>) => {
  // Empêche le navigateur de recharger la page
  // lors de la soumission du formulaire.
  event.preventDefault();

  // Création de l'objet qui sera envoyé au back-end.
  // Les autres propriétés du challenge devront être ajoutées
  // selon la structure réelle du formulaire.
  const newChallenge = {
    rules: rules,
  };

  // Envoie les données au back-end.
  const response = await fetch("http://localhost:3000/api/challenges", {
    method: "POST",

    // Indique au serveur que les données envoyées sont du JSON.
    headers: {
      "Content-Type": "application/json",
    },

    // Transforme l'objet JavaScript en JSON.
    body: JSON.stringify(newChallenge),
  });

  // Vérifie si la requête a échoué.
  if (!response.ok) {
    console.error("Erreur lors de la création du challenge");
    return;
  }

  // Transforme la réponse JSON du serveur
  // en objet JavaScript.
  const data = await response.json();

  console.log("Challenge créé :", data);

  // Vide le textarea après la création du challenge.
  setRules("");
};
```

Le formulaire devra alors appeler cette fonction :

```tsx
<form onSubmit={handleSubmit}>
```

Le flux devient :

```text
textarea
   ↓
rules
   ↓
newChallenge.rules
   ↓
JSON.stringify()
   ↓
POST /api/challenges
   ↓
Express
   ↓
BDD
```

---

# 5. Stockage envisagé dans la BDD

Les règles peuvent rester une seule valeur textuelle associée au challenge.

Exemple conceptuel :

```text
CHALLENGE

title       = "Super Mario Bros. en moins de 10 minutes !"

description = "Terminez le jeu le plus rapidement possible."

rules       = "Pas de glitch
               Une seule tentative
               Vidéo obligatoire"
```

Il n'est donc pas nécessaire, pour le MVP envisagé, de créer une entité `RÈGLE` uniquement pour pouvoir afficher plusieurs règles dans l'interface.

---

# 6. Récupération des règles

Lorsque le front-end récupère un challenge avec une requête GET, le back-end pourrait retourner un objet de ce type :

```json
{
  "title": "Super Mario Bros. en moins de 10 minutes !",
  "description": "Terminez le jeu le plus rapidement possible.",
  "rules": "Pas de glitch\nUne seule tentative\nVidéo obligatoire"
}
```

Dans React :

```tsx
challenge.rules
```

correspond toujours à une seule chaîne de caractères.

---

# 7. Transformer le texte en tableau dans React

Pour afficher chaque règle séparément, le texte peut être découpé au niveau des retours à la ligne avec `split()`.

### Fichier envisagé

```text
src/pages/ChallengePage/ChallengePage.tsx
```

```tsx
// Découpe le texte à chaque retour à la ligne.
//
// Exemple :
// "Pas de glitch\nUne seule tentative"
//
// devient :
// ["Pas de glitch", "Une seule tentative"]
const rulesList = challenge.rules.split("\n");
```

Le résultat est alors un tableau :

```ts
[
  "Pas de glitch",
  "Une seule tentative",
  "Vidéo obligatoire",
];
```

---

# 8. Supprimer les lignes vides

Un utilisateur peut éventuellement laisser une ligne vide dans le `textarea`.

Exemple :

```text
Pas de glitch

Une seule tentative

Vidéo obligatoire
```

Après `split("\n")`, certaines valeurs du tableau seraient donc vides.

On peut les supprimer avec `filter()` :

```tsx
// Supprime les lignes qui ne contiennent aucun texte.
const filteredRules = rulesList.filter(
  (rule) => rule.trim() !== "",
);
```

On obtient alors :

```ts
[
  "Pas de glitch",
  "Une seule tentative",
  "Vidéo obligatoire",
];
```

---

# 9. Afficher les règles avec map()

Une fois le tableau obtenu, React peut parcourir les règles avec `map()`.

```tsx
<section>
  <h2>Règles</h2>

  <ul>
    {/*
      map() parcourt le tableau.

      Pour chaque règle trouvée,
      React crée un élément <li>.
    */}
    {filteredRules.map((rule, index) => (
      <li key={index}>{rule}</li>
    ))}
  </ul>
</section>
```

Le navigateur affichera alors :

```text
Règles

• Pas de glitch
• Une seule tentative
• Vidéo obligatoire
```

---

# 10. Flux complet

```text
UTILISATEUR
    │
    │ saisit plusieurs lignes
    ▼
<textarea>
    │
    ▼
useState
    │
    │ rules = string
    ▼
JSON.stringify()
    │
    ▼
POST
    │
    ▼
BACK-END
    │
    ▼
BDD
rules = TEXT
    │
    ▼
GET
    │
    ▼
REACT
challenge.rules
    │
    │ split("\n")
    ▼
TABLEAU
    │
    │ filter()
    ▼
TABLEAU SANS LIGNES VIDES
    │
    │ map()
    ▼
RENDU HTML
    │
    ▼
<li>Règle 1</li>
<li>Règle 2</li>
<li>Règle 3</li>
```

---

# 11. Lien avec SA06-challenge-orecipes-largenty

Dans le projet :

**`SA06-challenge-orecipes-largenty`**

un mécanisme similaire avait été rencontré avec Svelte.

La donnée était récupérée depuis le back-end :

```svelte
const getRecipes = async () => {
  const response = await fetch("http://localhost:3000/recipes");
  const data = await response.json();

  recipes = data;
};
```

Puis les recettes étaient parcourues avec :

```svelte
{#each recipes as recipe}
```

Le code contenait également un exemple de parcours d'un tableau de `steps` :

```svelte
{#each recipe.steps as step}
  <li>{step}</li>
{/each}
```

Dans React, le principe équivalent pour parcourir un tableau est notamment l'utilisation de `map()` :

```tsx
{filteredRules.map((rule, index) => (
  <li key={index}>{rule}</li>
))}
```

Le framework et la syntaxe changent, mais le principe reste similaire :

```text
Svelte
{#each tableau as element}

React
tableau.map(...)
```

---

# 12. Justification du choix

Pour le MVP de GamerChallenges, je n'ai pas identifié de fonctionnalité nécessitant de manipuler individuellement chaque règle dans la base de données.

Je n'ai notamment pas besoin, à ce stade, de :

- rechercher une règle individuellement ;
- relier une règle à une autre entité ;
- gérer une catégorie par règle ;
- modifier une règle indépendamment du challenge ;
- supprimer une règle directement dans la BDD indépendamment du challenge.

Je préfère donc conserver les règles comme une information textuelle du challenge.

La transformation nécessaire pour leur présentation peut être effectuée côté front-end.

> **La manière dont une donnée est stockée et la manière dont elle est présentée dans l'interface sont deux problématiques différentes.**

Dans GamerChallenges :

```text
STOCKAGE
rules = TEXT
        ↓
TRANSFORMATION REACT
split("\n")
        ↓
TABLEAU
        ↓
RENDU
map()
        ↓
<li>...</li>
```

---

# 13. Limite de cette solution

Ce choix correspond aux besoins actuellement identifiés pour le MVP.

Si les besoins fonctionnels évoluent et qu'il devient nécessaire de gérer chaque règle indépendamment, le modèle de données devra être réévalué.

Par exemple, une autre modélisation pourrait devenir pertinente si l'application devait permettre de gérer individuellement chaque règle.

Le choix actuel n'est donc pas basé sur l'idée qu'une valeur `TEXT` est toujours préférable.

Il est basé sur les besoins fonctionnels identifiés pour le MVP de GamerChallenges.

---

# 14. Explication possible devant le jury

> Pour le MVP, nous n'avons pas besoin de gérer chaque règle comme une donnée indépendante dans la base de données.
>
> J'ai donc choisi de conserver les règles comme une information textuelle appartenant au challenge.
>
> L'utilisateur peut saisir plusieurs règles dans un `textarea`. Les retours à la ligne sont conservés dans la chaîne de caractères.
>
> Lorsque le challenge est récupéré côté React, je peux utiliser `split()` pour transformer cette chaîne en tableau, supprimer les éventuelles lignes vides avec `filter()`, puis utiliser `map()` pour afficher chaque règle dans un élément `li`.
>
> J'avais déjà rencontré un principe similaire pendant ma formation dans le projet `SA06-challenge-orecipes-largenty`, réalisé avec Svelte. Cela m'a aidée à comprendre que la structure utilisée pour stocker une donnée et sa représentation dans l'interface ne sont pas obligatoirement identiques.
>
> Si les besoins évoluent et nécessitent plus tard de gérer chaque règle indépendamment, il faudra alors réévaluer cette modélisation.