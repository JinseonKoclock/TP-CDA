# GamerChallenges — Implémentation React des règles

> ⚠️ Ces exemples sont des propositions d'implémentation.
> Ils devront être adaptés à la structure réelle des composants et des routes de GamerChallenges.

## Flux prévu

```text
CreateChallengePage
textarea → rules (string) → POST

ChallengePage
GET → rules (string) → split("\n") → map() → <li>
```

---

## 1. Saisie des règles

### Fichier envisagé

`src/pages/CreateChallengePage/CreateChallengePage.tsx`

Dans le formulaire de création d'un challenge, les règles peuvent être saisies dans un seul `textarea`.

```tsx
import { useState } from "react";

function CreateChallengePage() {
  // Le state "rules" contient tout le texte saisi dans le textarea.
  // Les différentes règles sont séparées par des retours à la ligne.
  const [rules, setRules] = useState("");

  // Fonction appelée lors de l'envoi du formulaire.
  const handleSubmit = async (event: React.FormEvent<HTMLFormElement>) => {
    // Empêche le rechargement automatique de la page.
    event.preventDefault();

    // Pour le moment, on ne montre ici que la propriété "rules".
    // Les autres propriétés du challenge pourront être ajoutées
    // selon la structure réelle du formulaire.
    const newChallenge = {
      rules: rules,
    };

    // Envoie le nouveau challenge au back-end.
    const response = await fetch("http://localhost:3000/api/challenges", {
      method: "POST",

      // Indique au serveur que le contenu envoyé est du JSON.
      headers: {
        "Content-Type": "application/json",
      },

      // Transforme l'objet JavaScript en JSON avant son envoi.
      body: JSON.stringify(newChallenge),
    });

    // Vérifie si la requête a échoué.
    if (!response.ok) {
      console.error("Erreur lors de la création du challenge");
      return;
    }

    // Récupère la réponse JSON envoyée par le back-end.
    const data = await response.json();

    console.log("Challenge créé :", data);

    // Vide le textarea après la création du challenge.
    setRules("");
  };

  return (
    <main>
      <h1>Proposer un challenge</h1>

      <form onSubmit={handleSubmit}>
        <div>
          <label htmlFor="rules">Règles</label>

          {/*
            Le textarea permet d'écrire plusieurs règles.
            Chaque retour à la ligne sera conservé
            dans la chaîne de caractères.
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

        <button type="submit">Créer le challenge</button>
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

le state `rules` contient en réalité une chaîne de caractères correspondant à :

```ts
"Pas de glitch\nUne seule tentative\nVidéo obligatoire"
```

C'est cette chaîne de caractères qui est envoyée au back-end.

Le flux est donc :

```text
textarea
    ↓
useState
    ↓
rules
    ↓
newChallenge.rules
    ↓
JSON.stringify()
    ↓
POST /api/challenges
    ↓
Back-end
    ↓
BDD
```

---

## 2. Récupération et affichage des règles

### Fichier envisagé

`src/pages/ChallengePage/ChallengePage.tsx`

Supposons que le challenge récupéré depuis le back-end possède une propriété :

```ts
challenge.rules
```

et que celle-ci contienne :

```ts
"Pas de glitch\nUne seule tentative\nVidéo obligatoire"
```

Une première possibilité consiste à effectuer directement les transformations dans le JSX.

```tsx
<section>
  <h2>Règles</h2>

  <ul>
    {/*
      split("\n") découpe le texte
      à chaque retour à la ligne.

      Exemple :
      "Pas de glitch\nUne seule tentative"

      devient :
      ["Pas de glitch", "Une seule tentative"]
    */}
    {challenge.rules
      .split("\n")

      // Supprime les lignes vides
      // éventuellement saisies par l'utilisateur.
      .filter((rule) => rule.trim() !== "")

      // Transforme chaque règle
      // en élément <li>.
      .map((rule, index) => (
        <li key={index}>{rule}</li>
      ))}
  </ul>
</section>
```

Le résultat affiché serait :

```text
Règles

• Pas de glitch
• Une seule tentative
• Vidéo obligatoire
```

---

## 3. Variante plus facile à comprendre et à déboguer

Pour mieux visualiser les différentes transformations, il est possible de préparer les données avant le `return` du composant.

### Étape 1 — Transformer la string en tableau

```tsx
// Découpe le texte des règles
// à chaque retour à la ligne.
const rulesList = challenge.rules.split("\n");
```

Avec :

```ts
challenge.rules =
  "Pas de glitch\nUne seule tentative\nVidéo obligatoire";
```

`rulesList` devient :

```ts
[
  "Pas de glitch",
  "Une seule tentative",
  "Vidéo obligatoire",
];
```

### Étape 2 — Supprimer les lignes vides

```tsx
// Supprime les éventuelles lignes vides.
const filteredRules = rulesList.filter(
  (rule) => rule.trim() !== "",
);
```

Cette étape est utile si l'utilisateur saisit par exemple :

```text
Pas de glitch

Une seule tentative

Vidéo obligatoire
```

Les lignes vides ne seront alors pas affichées comme des règles.

### Étape 3 — Afficher le tableau

Dans le JSX :

```tsx
<section>
  <h2>Règles</h2>

  <ul>
    {/*
      Parcourt le tableau des règles
      et crée un <li> pour chaque règle.
    */}
    {filteredRules.map((rule, index) => (
      <li key={index}>{rule}</li>
    ))}
  </ul>
</section>
```

Le chemin de transformation devient très visible :

```text
challenge.rules
      │
      │ split("\n")
      ▼
rulesList
["règle 1", "règle 2", "règle 3"]
      │
      │ filter()
      ▼
filteredRules
["règle 1", "règle 2", "règle 3"]
      │
      │ map()
      ▼
<li>règle 1</li>
<li>règle 2</li>
<li>règle 3</li>
```

Cette écriture permet de distinguer clairement le rôle de chaque méthode :

- `split()` : transforme la chaîne de caractères en tableau ;
- `filter()` : retire les éléments vides ;
- `map()` : parcourt le tableau pour produire les éléments affichés par React.

---

## 4. Comparaison avec SA06-challenge-orecipes-largenty

Cette approche reprend un principe déjà rencontré dans le projet O'clock :

**`SA06-challenge-orecipes-largenty`**

Dans ce projet Svelte, un tableau pouvait être parcouru avec :

```svelte
{#each recipe.steps as step}
  <li>{step}</li>
{/each}
```

Dans React, un principe comparable peut être obtenu avec `map()` :

```tsx
{filteredRules.map((rule, index) => (
  <li key={index}>{rule}</li>
))}
```

On peut donc faire le rapprochement suivant :

```text
Svelte
{#each tableau as element}
        ↓
parcours du tableau
        ↓
affichage

React
tableau.map(...)
        ↓
parcours du tableau
        ↓
affichage
```

La syntaxe change entre Svelte et React, mais le raisonnement reste similaire.

---

## 5. Vue d'ensemble

```text
CRÉATION DU CHALLENGE

Utilisateur
    ↓
<textarea>
    ↓
useState
    ↓
rules = string
    ↓
JSON.stringify()
    ↓
POST
    ↓
Back-end
    ↓
BDD


AFFICHAGE DU CHALLENGE

BDD
    ↓
Back-end
    ↓
GET
    ↓
challenge.rules
    ↓
split("\n")
    ↓
rulesList
    ↓
filter()
    ↓
filteredRules
    ↓
map()
    ↓
<li>Règle</li>
```

---

## 6. Point à retenir

> **Les règles peuvent être stockées comme une seule valeur textuelle dans le challenge tout en étant affichées comme plusieurs éléments distincts dans React.**

Le principe envisagé est donc :

```text
MCD
CHALLENGE.Règles
        ↓
BDD
rules = TEXT
        ↓
React Form
<textarea>
        ↓
React Detail
split("\n")
        ↓
filter()
        ↓
map()
        ↓
<li>...</li>
```

Cette proposition devra être adaptée aux fichiers réels de GamerChallenges, notamment :

- au nom réel de la propriété des règles ;
- au type TypeScript du challenge ;
- à la route réelle de l'API ;
- au formulaire existant ;
- à la structure actuelle de `ChallengePage.tsx`.

Avant l'intégration définitive, il faudra donc vérifier les fichiers existants afin de ne pas adapter le projet à une structure supposée.