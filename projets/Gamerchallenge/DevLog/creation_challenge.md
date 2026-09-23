# DevLog – 23/09/2026

## Objectif du jour

Continuer la dynamisation de la création d'un challenge et commencer la mise en place de la route POST côté back-end.

---

## 1. Vérification des routes existantes

J'ai commencé par vérifier les routes déjà présentes dans le back-end.

Deux fichiers concernant les challenges étaient présents :

- `challenge.route.ts`
- `challenge.router.ts`

J'ai vérifié lequel était réellement utilisé dans `index.router.ts`.

```ts
import { router as challengeRouter } from "./challenge.route.ts";

router.use("/challenges", challengeRouter);
```

Le fichier réellement utilisé est donc `challenge.route.ts`.

La route GET permettant de récupérer les challenges était déjà présente :

```ts
router.get("/", getAllChallenges);
```

Le controller `getAllChallenges` existait également déjà.

---

## 2. Création d'une branche pour le POST Challenge

J'ai créé une branche dédiée à la création d'un challenge :

```bash
git switch -c feat/POSTchallenge
```

L'objectif est de travailler sur :

```text
POST /api/challenges
```

sans modifier directement la branche `developp`.

---

## 3. Vérification du middleware d'authentification

J'ai étudié le middleware `verifyAuth`.

Son fonctionnement est le suivant :

1. récupération du cookie `accessToken` ;
2. vérification du JWT ;
3. récupération de l'utilisateur dans la BDD ;
4. ajout de l'utilisateur dans `req.user`.

```ts
req.user = user;
```

Cela permet de récupérer l'auteur du challenge directement depuis l'utilisateur connecté.

Il n'est donc pas nécessaire d'envoyer `author_id` depuis le front.

Dans le controller, je peux utiliser :

```ts
req.user!.id
```

pour renseigner `author_id`.

---

## 4. Validation des données avec Zod

J'ai défini les données attendues pour la création d'un challenge.

```ts
const challengeBodySchema = z.object({
  title: z.string().min(3).max(100),
  videoUrl: z.url(),
  description: z.string().min(10).max(200),
  rules: z.array(z.string().min(10).max(200)),
  jeuId: z.number().int().positive(),
});
```

### Validation de l'URL

La syntaxe :

```ts
z.string().url()
```

était indiquée comme dépréciée dans la version de Zod utilisée.

J'utilise donc :

```ts
z.url()
```

### Validation des règles

Le formulaire permet d'ajouter plusieurs règles.

Le front envoie donc un tableau :

```ts
string[]
```

La validation doit utiliser :

```ts
z.array(z.string())
```

et non :

```ts
z.string()
```

---

## 5. Différence entre les données de l'API et les données Prisma

J'ai compris que les noms utilisés par le front ne sont pas obligés d'être identiques aux noms utilisés dans Prisma.

Exemple :

```text
API / Front        Prisma / BDD

videoUrl     →     url_video
jeuId        →     jeu_id
authorId     →     author_id
```

Il faut donc effectuer cette transformation dans le controller au lieu d'envoyer directement toutes les données avec :

```ts
...data
```

---

## 6. Préparation de `challengeInclude`

J'ai préparé les relations utiles pour récupérer les informations liées au challenge.

```ts
const challengeInclude = {
  user: {
    select: {
      id: true,
      username: true,
      avatar: true,
    },
  },

  jeu: {
    select: {
      id: true,
      title: true,
    },
  },
} as const;
```

Cela permet notamment d'obtenir :

- l'utilisateur qui a créé le challenge ;
- le jeu associé au challenge.

---

## 7. Préparation de `formatChallenge`

J'ai commencé une fonction permettant de transformer les données Prisma vers un format plus adapté à l'API.

Exemples :

```text
url_video        → videoUrl
jeu_id           → jeuId
user.username    → author
jeu.title        → game
```

Cette fonction permet de séparer le format interne de la BDD du format retourné au front.

Le typage du champ `rules` reste à améliorer car le champ est défini comme `Json` dans Prisma.

---

## 8. Création du controller `addChallenge`

J'ai commencé le controller permettant de créer un challenge.

Le fonctionnement prévu est :

```text
POST /api/challenges
        ↓
verifyAuth
        ↓
validation Zod
        ↓
récupération de req.user.id
        ↓
transformation des données
        ↓
prisma.challenge.create()
        ↓
réponse HTTP 201
```

Les données sont préparées de cette manière :

```ts
data: {
  title: data.title,
  url_video: data.videoUrl,
  description: data.description,
  rules: data.rules,
  jeu_id: data.jeuId,
  author_id: req.user!.id,
  status: false,
}
```

`author_id` est récupéré grâce à l'utilisateur authentifié et non depuis les données envoyées par le navigateur.

Le `status` est actuellement positionné à `false` pour représenter un challenge proposé mais pas encore actif.

Ce choix devra être confirmé avec le fonctionnement définitif du projet.

---

## 9. Route POST

La route POST doit être protégée par `verifyAuth`.

La route correcte est :

```ts
router.post("/", verifyAuth, addChallenge);
```

Comme `challengeRouter` est déjà monté dans `index.router.ts` avec :

```ts
router.use("/challenges", challengeRouter);
```

les routes finales sont :

```text
GET  /api/challenges
POST /api/challenges
```

Il ne faut donc pas écrire :

```ts
router.post("/challenges/add", verifyAuth, addChallenge);
```

car `/challenges` est déjà défini dans le router parent.

---

## 10. Problème découvert : relation avec `Jeu`

En étudiant le modèle Prisma, j'ai remarqué qu'un challenge doit obligatoirement être associé à un jeu :

```prisma
jeu    Jeu @relation(fields: [jeu_id], references: [id], onDelete: Cascade)
jeu_id Int
```

Cependant, le formulaire `AddChallengePage` ne permet actuellement pas de sélectionner un jeu.

La BDD contient déjà plusieurs jeux grâce au seeding :

- Zelda ;
- Hollow Knight ;
- Stardew Valley ;
- Baldur's Gate ;
- Minecraft ;
- Valorant.

Il faudra donc ajouter la gestion du jeu dans le formulaire avant de pouvoir terminer complètement la création d'un challenge.

---

## À faire à la prochaine session

- [ ] Vérifier que `router.post("/", verifyAuth, addChallenge)` est correctement enregistré.
- [ ] Vérifier les éventuelles erreurs TypeScript du controller.
- [ ] Améliorer le typage du champ Prisma `rules`.
- [ ] Prévoir la récupération de la liste des jeux.
- [ ] Ajouter la sélection du jeu dans `AddChallengePage`.
- [ ] Ajouter `jeuId` aux données envoyées par le front.
- [ ] Tester `POST /api/challenges`.
- [ ] Vérifier que `author_id` correspond à l'utilisateur connecté.
- [ ] Vérifier la création du challenge dans PostgreSQL.
- [ ] Confirmer la signification et la valeur initiale de `status`.
- [ ] Connecter ensuite la liste des challenges au GET `/api/challenges`.

---

## Ce que j'ai compris aujourd'hui

- Une route Express peut utiliser plusieurs middlewares avant d'arriver au controller.
- `verifyAuth` permet de récupérer l'utilisateur connecté dans `req.user`.
- L'identifiant de l'auteur ne doit pas être fourni par le front lorsqu'il peut être déterminé grâce à l'authentification.
- Zod permet de valider les données reçues avant de les envoyer à Prisma.
- Un tableau de règles doit être validé comme `string[]` et non comme une simple `string`.
- Les noms utilisés par l'API peuvent être différents des noms utilisés dans la BDD.
- Le controller peut assurer la transformation entre le format de l'API et le format Prisma.
- `include` permet de récupérer les relations associées à une entité avec Prisma.
- Une relation obligatoire dans la BDD (`jeu_id`) a des conséquences sur le formulaire front-end.

---

## Point de reprise

La prochaine session commencera par la gestion de `jeuId`.

Le back-end attend maintenant :

```ts
jeuId: z.number().int().positive()
```

mais `AddChallengePage` ne permet pas encore de sélectionner un jeu ni d'envoyer cet identifiant.

C'est donc le prochain point à traiter avant de tester complètement :

```text
AddChallengePage
        ↓
jeuId
        ↓
POST /api/challenges
        ↓
addChallenge
        ↓
Prisma
        ↓
PostgreSQL
```