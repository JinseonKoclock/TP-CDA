# DevLog – 24/09/2026

## Objectif du jour

Finaliser la dynamisation de la page `AddChallengePage` afin de permettre à un utilisateur connecté de proposer un challenge et de l'enregistrer réellement dans la BDD.

Le principal problème identifié hier était la relation obligatoire entre `Challenge` et `Jeu`.

---

## 1. Compréhension du problème avec `jeu_id`

Dans le modèle Prisma, un challenge doit obligatoirement être associé à un jeu :

```prisma
jeu    Jeu @relation(fields: [jeu_id], references: [id], onDelete: Cascade)
jeu_id Int
```

Le formulaire `AddChallengePage` ne permettait cependant pas de choisir un jeu.

Pour créer un challenge, le back-end doit donc recevoir l'identifiant du jeu sélectionné :

```ts
jeuId: number
```

Le fonctionnement recherché est :

```text
BDD : table Jeu
        ↓
GET /api/jeux
        ↓
Liste des jeux dans AddChallengePage
        ↓
L'utilisateur sélectionne un jeu
        ↓
jeuId envoyé dans le POST
        ↓
POST /api/challenges
        ↓
jeu_id enregistré en BDD
```

Les entités et relations existantes avec `Categorie` dans la BDD n'ont pas été modifiées.

---

## 2. Création du controller pour récupérer les jeux

J'ai créé :

```text
src/controllers/jeu.controller.ts
```

Le controller récupère tous les jeux présents dans la BDD avec Prisma :

```ts
import type { Request, Response } from "express";

import { prisma } from "../models/index.ts";

export async function getJeux(
  // La requête n'est pas utilisée ici.
  // Le "_" indique simplement que ce paramètre est volontairement inutilisé.
  _req: Request,
  res: Response
) {
  const jeux = await prisma.jeu.findMany();

  res.status(200).json(jeux);
}
```

J'ai également compris la différence entre :

```ts
req
```

et :

```ts
_req
```

Le `_` n'a aucun comportement particulier pour Express.

Il indique simplement que le paramètre est nécessaire dans la signature du controller mais qu'il n'est volontairement pas utilisé dans la fonction.

---

## 3. Création de la route des jeux

J'ai créé :

```text
src/routes/jeu.route.ts
```

avec :

```ts
import { Router } from "express";

import { getJeux } from "../controllers/jeu.controller.ts";

export const router = Router();

router.get("/", getJeux);
```

Puis j'ai connecté ce router dans `index.router.ts` :

```ts
import { router as jeuxRouter } from "./jeu.route.ts";

router.use("/jeux", jeuxRouter);
```

Le router principal étant déjà monté sous `/api`, l'endpoint final est :

```text
GET /api/jeux
```

---

## 4. Test de `GET /api/jeux`

J'ai testé directement :

```text
http://localhost:3000/api/jeux
```

L'API retourne correctement les jeux présents dans la BDD :

```text
1 → The Legend of Zelda: Breath of the Wild
2 → Hollow Knight
3 → Stardew Valley
4 → Baldur's Gate 3
5 → Minecraft
6 → Valorant
```

Le fonctionnement suivant est donc validé :

```text
GET /api/jeux
      ↓
jeu.route.ts
      ↓
getJeux()
      ↓
prisma.jeu.findMany()
      ↓
PostgreSQL
      ↓
JSON
```

---

## 5. Ajout de la liste des jeux dans `AddChallengePage`

J'ai ensuite modifié le front afin de récupérer les jeux depuis l'API.

Une interface permet de typer les données :

```ts
interface Jeu {
  id: number;
  title: string;
}
```

Un state conserve la liste :

```ts
const [jeux, setJeux] = useState<Jeu[]>([]);
```

La liste est récupérée depuis :

```text
GET /api/jeux
```

au chargement de la page.

Elle est ensuite utilisée pour générer les options d'un `<select>`.

Le formulaire permet maintenant à l'utilisateur de sélectionner le jeu concerné par son challenge.

---

## 6. Ajout de `jeuId` dans les données du formulaire

Dans `handleSubmit`, je récupère maintenant le jeu sélectionné :

```ts
const jeuId = formData.get("jeuId");
```

Je vérifie également son type :

```ts
typeof jeuId !== "string"
```

Puis je l'ajoute aux données envoyées :

```ts
const challengeData = {
  title,
  videoUrl,
  description,
  rules,
  jeuId: Number(jeuId),
};
```

`FormData` retourne la valeur du `<select>` sous forme de chaîne de caractères.

Par exemple :

```text
"5"
```

Mais le back-end attend :

```ts
jeuId: z.number().int().positive()
```

Il faut donc convertir la valeur :

```ts
Number(jeuId)
```

pour obtenir :

```text
5
```

---

## 7. Premier test du POST : erreur 422

Lors du premier test :

```text
POST /api/challenges
```

l'API a retourné :

```text
422 Unprocessable Entity
```

J'ai d'abord vérifié les données réellement envoyées par le navigateur.

Le `jeuId` était correctement présent :

```text
jeuId: 5
```

J'ai ensuite consulté la réponse du back-end afin d'identifier précisément l'erreur Zod.

La réponse indiquait :

```text
path: ["description"]
code: "too_big"
maximum: 200
```

Le problème venait donc de cette validation :

```ts
description: z.string().min(10).max(200)
```

La description utilisée pour le test dépassait 200 caractères.

Cette erreur m'a permis de vérifier que la validation Zod fonctionnait correctement.

Elle m'a également rappelé qu'il vaut mieux consulter la réponse précise de l'API avant de modifier le code en supposant l'origine d'une erreur.

---

## 8. Validation complète de la création d'un challenge

Après correction du problème de validation, la création du challenge a fonctionné.

Le challenge de test a été enregistré dans PostgreSQL avec notamment :

```text
title      → Catch me if you can
author_id  → 1
jeu_id     → 5
status     → false
url_video  → URL YouTube
rules      → tableau JSON
```

Le jeu sélectionné était `Minecraft`.

Dans la table `Jeu` :

```text
Minecraft → id 5
```

La BDD contient bien :

```text
jeu_id → 5
```

La relation entre le challenge et le jeu sélectionné fonctionne donc correctement.

---

## 9. Auteur du challenge

L'identifiant de l'auteur n'est pas envoyé par le formulaire.

La route utilise le middleware :

```ts
verifyAuth
```

qui récupère l'utilisateur connecté et le place dans :

```ts
req.user
```

Le controller peut ensuite utiliser :

```ts
author_id: req.user!.id
```

Cela évite de faire confiance à un `author_id` envoyé par le navigateur.

Lors du test, l'auteur a bien été enregistré dans la BDD.

---

## Résultat final

Le flux complet de création d'un challenge fonctionne maintenant :

```text
Utilisateur connecté
        ↓
AddChallengePage
        ↓
GET /api/jeux
        ↓
Sélection du jeu
        ↓
Remplissage du formulaire
        ↓
FormData
        ↓
jeuId converti en number
        ↓
POST /api/challenges
        ↓
verifyAuth
        ↓
Validation Zod
        ↓
addChallenge()
        ↓
Prisma
        ↓
PostgreSQL
        ↓
Challenge créé ✅
```

---

## Ce que j'ai compris aujourd'hui

- `findMany()` permet de récupérer plusieurs enregistrements avec Prisma.
- `findUnique()` sert à rechercher un enregistrement unique selon une valeur unique.
- Une route doit être reliée au router principal pour devenir accessible depuis l'API.
- Les chemins des routers Express se combinent entre eux.
- `useEffect` permet notamment de déclencher la récupération des jeux au chargement du composant.
- `useState` permet de conserver les jeux récupérés afin de les utiliser dans le rendu.
- Un `<select>` peut être généré dynamiquement à partir des données provenant de la BDD.
- `FormData` retourne la valeur d'un `<select>` sous forme de chaîne de caractères.
- `Number()` permet ici de convertir l'identifiant avant de l'envoyer à l'API.
- Une erreur HTTP `422` signifie ici que la requête a bien atteint l'API mais que les données n'ont pas passé la validation.
- Il faut lire le détail de l'erreur Zod avant de supposer quelle donnée pose problème.
- `verifyAuth` permet d'utiliser l'identité de l'utilisateur connecté pour définir l'auteur du challenge.
- Une relation BDD comme `jeu_id` a un impact direct sur le formulaire et sur le contrat de l'API.

---

## Git

### Back-end

Travail effectué sur :

```text
feat/POSTchallenge
```

Ajout notamment de :

```text
GET /api/jeux
POST /api/challenges
```

Le travail a été poussé et fusionné sans conflit.

### Front-end

Travail poursuivi sur :

```text
feat/ProposerUnChallenge
```

Commit :

```bash
git commit -m "feat: Ajouter la sélection du jeu à la création d'un challenge"
```

---

