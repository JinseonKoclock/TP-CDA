# Carnet de bord – Compréhension du contrôleur d'authentification

> 16/07/2026  

> [!NOTE]
> **À propos de ce document**
>
> Ce carnet de bord a été rédigé à partir de mes séances de **Rubber Duck Debugging** avec mon assistant IA ChatGPT.
>
> Toutes les explications, les erreurs, les hésitations, les fausses pistes, les essais, les questions et le brainstorming présents dans ce document reflètent mon propre cheminement de réflexion pendant l'apprentissage.
>
> ChatGPT m'a aidé à structurer les idées, à reformuler certains passages et à rendre le document plus clair, mais chaque section a été relue, corrigée, complétée et validée par moi avant d'être intégrée à mon carnet de bord.
>
> L'objectif de ce document n'est pas uniquement de conserver le résultat final, mais également de garder une trace de ma manière de raisonner, de comprendre les concepts et de progresser au fil de ma formation.

---

Aujourd'hui, je n'ai pas cherché à développer une nouvelle fonctionnalité. J'ai préféré reprendre le code du formateur afin de comprendre précisément son fonctionnement.

En corrigeant l'exercice, je me suis rendu compte que le formateur avait choisi une approche différente de la mienne.

Dans l'énoncé, il n'était pas demandé de créer un middleware. J'ai donc implémenté toute la logique d'authentification directement dans le contrôleur.

La correction m'a permis de comprendre qu'il était plus pertinent d'isoler cette logique dans un middleware afin de pouvoir la réutiliser sur plusieurs routes protégées et d'alléger les contrôleurs.

J'ai ainsi mieux compris pourquoi `req.user` est ajouté dans le middleware avant d'être utilisé dans le contrôleur.

---

## Pourquoi créer un fichier `express.d.ts` ?

J'ai compris que, par défaut, Express ne possède pas la propriété `req.user`.

Le fichier **`express.d.ts`** permet donc d'étendre l'interface `Request` afin d'ajouter cette propriété.

Ainsi, TypeScript reconnaît correctement `req.user` dans tout le projet.

```ts
declare global {
  namespace Express {
    interface Request {
      user?: Omit<User, "password"> | null;
    }
  }
}
```

---

## Comprendre `Omit<User, "password">`

Je comprenais déjà le rôle de `Omit`, mais en essayant de l'expliquer avec mes propres mots pendant une séance de *Rubber Duck Debugging*, je me suis rendu compte que j'avais choisi une mauvaise formulation.

En réalité, `Omit<User, "password">` ne sélectionne pas le mot de passe : il fait exactement l'inverse en retirant cette propriété du type `User`.

Cette discussion m'a également permis de découvrir l'utilitaire TypeScript `Pick`, qui, lui, sert à sélectionner uniquement certaines propriétés d'un type.

Comparer `Omit` et `Pick` m'a aidé à mieux mémoriser leur différence :

- `Omit` : retire une ou plusieurs propriétés.
- `Pick` : conserve uniquement les propriétés sélectionnées.

---

## Les imports TypeScript

Avant, je voyais simplement :

```ts
import { Request, Response } from "express";
```

sans vraiment comprendre leur intérêt.

J'ai compris que ces imports servent uniquement au typage.

En JavaScript, on écrivait simplement :

```js
function loginUser(req, res) {}
```

En TypeScript, on précise que :

- `req` est un objet `Request` ;
- `res` est un objet `Response`.

Cela permet à TypeScript de proposer l'autocomplétion et de détecter les erreurs avant l'exécution.

---

## Comprendre `async` et `await`

Je savais que `await` permet d'attendre le résultat d'une opération qui prend du temps avant de poursuivre l'exécution du code.

En l'expliquant avec mes propres mots, j'ai d'abord parlé de l'attente d'une réponse du serveur. J'ai ensuite précisé que le terme le plus juste est **opération asynchrone**, car il peut s'agir de plusieurs types d'opérations :

- une requête Prisma vers la base de données ;
- un hash avec Argon2 ;
- une validation asynchrone ;
- ou toute autre opération qui retourne une promesse.

J'ai retenu que `async` permet d'utiliser `await` dans une fonction, tandis que `await` suspend l'exécution de cette fonction jusqu'à ce que le résultat soit disponible.

Cela évite que le code suivant utilise une valeur qui n'a pas encore été retournée.

---

## Déstructuration d'objet

J'ai commencé à analyser cette ligne :

```ts
const { firstname, lastname, email, password, confirm } =
  await registerUserBodySchema.parseAsync(req.body);
```

Au départ, je pensais que la déstructuration servait à reconstruire un objet.

En réalité, elle permet simplement d'extraire directement plusieurs propriétés d'un objet afin de les stocker dans des variables.

Cette ligne réalise donc deux opérations :

1. validation du contenu de `req.body` grâce à Zod (`parseAsync`) ;
2. déstructuration de l'objet retourné afin de récupérer les différentes propriétés.


---

## Ce que je retiens

- Confirmer le rôle de `express.d.ts` dans l'extension de l'interface `Request`.
- Mieux distinguer les utilitaires TypeScript `Omit` et `Pick`.
- Comprendre plus précisément le rôle des imports `Request` et `Response` en TypeScript.
- Affiner ma manière d'expliquer le fonctionnement de `async` et `await` avec le vocabulaire approprié (*opération asynchrone*).
- Comprendre que la déstructuration permet d'extraire directement les propriétés d'un objet.
- Identifier qu'une seule ligne de code peut réaliser plusieurs opérations (validation avec Zod + déstructuration).
- Mieux distinguer ce qui relève du contrôleur et ce qui peut être délégué à un middleware selon le design de l'application.

---

## Validation des données avec `parseAsync()`

En analysant cette ligne :

```ts
const { firstname, lastname, email, password, confirm } =
  await registerUserBodySchema.parseAsync(req.body);
```

j'ai compris que `parseAsync(req.body)` compare les données envoyées par le client avec le schéma Zod prévu pour l'inscription.

Le déroulement est le suivant :

```text
req.body
      ↓
Validation avec le schéma Zod
      ↓
┌───────────────┬─────────────────────────┐
│ Données OK    │ Données invalides       │
│               │                         │
│ Objet validé  │ Zod lève une erreur     │
│ et typé       │                         │
└───────────────┴─────────────────────────┘
```

Une fois les données validées, la déstructuration extrait directement les propriétés de l'objet retourné.

J'ai également remarqué qu'une seule ligne réalise plusieurs opérations :

- validation des données ;
- création d'un objet typé ;
- déstructuration de cet objet.

---

## Une erreur de recherche qui m'a appris quelque chose

En cherchant la documentation de `parseAsync`, je suis tombé sur la documentation de **Valibot**.

Je ne savais pas encore que `parseAsync` provenait de Zod.

En relisant les imports du fichier, j'ai retrouvé :

```ts
import z from "zod";
```

J'ai compris que la première chose à faire lorsqu'une méthode est inconnue est d'identifier la bibliothèque dont elle provient avant de chercher sa documentation.

Par exemple :

```ts
z.object(...)
```

→ documentation Zod

```ts
prisma.user.findFirst(...)
```

→ documentation Prisma

Cette erreur de recherche m'a permis d'adopter une nouvelle habitude : toujours regarder les imports avant de consulter la documentation.

---

## Différence entre `throw` et `res.status().json()`

Jusqu'à présent, je savais qu'une réponse comme :

```ts
res.status(400).json(...)
```

envoyait une erreur au client.

En revanche, je ne m'étais jamais demandé ce qu'il se passait après.

J'ai compris que :

```ts
res.status(400).json(...)
```

envoie immédiatement la réponse HTTP depuis le contrôleur.

À l'inverse :

```ts
throw new BadRequestError(...)
```

n'envoie aucune réponse.

`throw` interrompt immédiatement l'exécution du contrôleur et délègue le traitement de l'erreur.

Au début, je pensais que `throw` faisait simplement « changer de fichier ».

En réalité, il transmet l'erreur à la partie de l'application chargée de la gérer.

Cette réflexion m'a permis de commencer à voir le déroulement complet d'une requête plutôt que de regarder uniquement une ligne de code.

---

## Découverte du dossier `lib`

En recherchant l'origine de `BadRequestError`, j'ai trouvé :

```text
src/lib/errors.ts
```

J'ai découvert que `lib` est l'abréviation de **library**.

Dans ce projet, ce dossier contient des outils réutilisables dans plusieurs parties de l'application.

Par exemple :

```text
src/lib/
├── errors.ts
├── token.ts
├── validators.ts
└── validators.unit.test.ts
```

J'ai compris que ce dossier n'est pas spécifique à Express ou à TypeScript.

Il correspond simplement à un choix d'organisation du projet.

Cette discussion m'a également amené à commencer à observer davantage la structure générale du projet plutôt que de me concentrer uniquement sur le contenu d'un fichier.

---

## Réactivation des notions de classes

En ouvrant `errors.ts`, j'ai retrouvé des notions étudiées plusieurs semaines auparavant.

Je pensais avoir oublié le fonctionnement des classes.

En réalité, les souvenirs sont revenus progressivement en relisant le code et en échangeant dessus.

### Le rôle du `constructor`

J'ai revu que le `constructor` est exécuté automatiquement lorsqu'un objet est créé avec `new`.

Son rôle est d'initialiser les propriétés du nouvel objet.

### Le rôle de `super()`

J'ai également retrouvé le fonctionnement de `super()`.

Dans cette classe :

```ts
export class BadRequestError extends HttpClientError {
  constructor(message: string) {
    super(message, { status: 400 });
  }
}
```

`super()` appelle le constructeur de la classe parente.

L'exécution remonte donc dans la hiérarchie des classes avant de revenir dans la classe enfant.

Cette représentation m'a aidé à mieux visualiser le déroulement :

```text
BadRequestError
        ↓
HttpClientError
        ↓
Error
        ↑
HttpClientError
        ↑
BadRequestError
```

### Le rôle de `this`

J'ai également revu le rôle de `this`.

`this` représente l'objet actuellement créé.

Par exemple :

```ts
this.status = status;
```

signifie que la valeur reçue par le constructeur est enregistrée dans la propriété `status` du nouvel objet.

---

## Comprendre les erreurs HTTP personnalisées

J'ai compris que la classe native `Error` ne possède pas de propriété `status`.

C'est la classe :

```ts
HttpClientError
```

qui ajoute cette propriété.

Les classes enfants définissent ensuite automatiquement le code HTTP correspondant :

```ts
BadRequestError
→ 400
```

```ts
UnauthorizedError
→ 401
```

```ts
ForbiddenError
→ 403
```

```ts
ConflictError
→ 409
```

Cela permet de créer facilement une erreur possédant un message et un statut HTTP adapté.

---

## Ce que je retiens

- `parseAsync(req.body)` valide les données envoyées avec le schéma Zod prévu.
- Une validation réussie retourne un objet validé et typé.
- Une validation échouée lève une erreur et interrompt le contrôleur.
- Avant de chercher une méthode, je dois identifier la bibliothèque grâce aux imports.
- `res.status(...).json(...)` répond directement au client.
- `throw` interrompt l'exécution et délègue le traitement de l'erreur.
- Le dossier `lib` contient des outils communs réutilisables dans plusieurs parties du projet.
- `constructor` initialise un objet lors de sa création.
- `super()` appelle le constructeur de la classe parente.
- `this` représente l'objet en cours de création.
- La relecture d'un cas concret m'a permis de réactiver progressivement des notions de programmation orientée objet déjà étudiées.