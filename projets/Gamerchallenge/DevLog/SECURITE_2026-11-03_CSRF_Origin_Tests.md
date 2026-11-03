# DevLog — GamerChallenge — 3 novembre 2026

## Objectif de la séance

Ajouter une vérification de l’origine des requêtes dans le backend, dans le cadre du travail sur la protection CSRF, puis adapter les tests d’intégration.

La méthode travaillée aujourd’hui repose sur l’en-tête `Origin`. Aucun token CSRF n’a été ajouté pendant cette séance.

## Reprendre le problème dans le bon ordre

Au début, l’ajout du contrôle d’origine a provoqué plusieurs échecs dans les tests des challenges. Les indications initiales de l’IA m’orientaient vers des requêtes d’inscription et de connexion dans `createAuthenticatedUser()`. Pourtant, le code partagé montrait que cette fonction crée directement un utilisateur avec Prisma et génère un JWT. Elle ne passe pas par une requête HTTP.

La fonction `loginTestUser()`, elle, envoie réellement une requête `POST` à la route de connexion. Ajouter `Origin` à cette requête ne l’ajoute pas automatiquement aux requêtes suivantes des tests.

Pour isoler le problème, j’ai utilisé Ctrl+Z dans `app.ts` et `auth.ts`, puis relancé les tests. Les six tests existants des challenges passaient de nouveau. Seul le test de refus d’une suppression depuis une origine non autorisée échouait.

J’ai alors demandé à reprendre l’objectif et l’ordre des étapes. Je voulais d’abord implémenter le comportement dans le backend, puis adapter les tests à ce comportement. J’ai également demandé à avancer une étape à la fois.

Ce retour en arrière a permis de distinguer les fonctionnalités déjà testées du nouveau contrôle que nous voulions ajouter.

## Implémentation effectuée

### Une liste d’origines partagée

Dans `src/app.ts`, les adresses déjà utilisées pour CORS ont été regroupées dans une variable.

```ts
const allowedOrigins = [
  "http://localhost:5173",
  "http://localhost:8000",
];
```

La configuration CORS utilise maintenant `origin: allowedOrigins`. La même liste est transmise au middleware de vérification d’origine.

### Un middleware dans le dossier existant

J’ai fait remarquer que le projet possède déjà un dossier `middlewares` et demandé à y placer le code. Le fichier créé est `src/middlewares/verify-origin.middleware.ts`.

```ts
import type { RequestHandler } from "express";

export function verifyOrigin(allowedOrigins: string[]): RequestHandler {
  return (req, res, next) => {
    if (["GET", "HEAD", "OPTIONS"].includes(req.method)) {
      next();
      return;
    }

    const origin = req.get("Origin");

    if (!origin || !allowedOrigins.includes(origin)) {
      res.status(403).json({
        message: "Origine de la requête non autorisée.",
      });
      return;
    }

    next();
  };
}
```

Le comportement implémenté est le suivant.

- `GET`, `HEAD` et `OPTIONS` passent à la suite du traitement.
- Pour les autres méthodes, le middleware lit `Origin`.
- Une origine absente ou non autorisée entraîne une réponse `403`.
- Une origine autorisée permet de continuer avec `next()`.

### Branchement avant les routes

Après l’import du middleware dans `app.ts`, le branchement des routes devient le suivant.

```ts
app.use("/api", verifyOrigin(allowedOrigins), router);
```

Le middleware est ainsi exécuté avant les routes de l’API. Son enregistrement avec `app.use` le rend actif lorsque cette application traite les requêtes.

## Adaptation des tests et résultat

Une fois le middleware connecté, j’ai anticipé que les requêtes de modification envoyées sans `Origin` seraient refusées. Les tests devaient donc représenter explicitement une origine autorisée pour atteindre le comportement qu’ils cherchaient à vérifier.

J’ai ajouté l’en-tête suivant aux requêtes concernées dans les fichiers de test.

```ts
Origin: "http://localhost:5173",
```

La clarification portait sur l’emplacement de cet en-tête. Il est ajouté aux requêtes `fetch` des tests, dans leurs `headers`. Le contrôle de l’origine reste centralisé dans le middleware.

J’ai également exécuté le scénario suivant.

```ts
it("should reject deletion from an untrusted origin even with the author's cookie", ...);
```

J’ai signalé que les tests des challenges, y compris ce scénario, passaient, puis que tous les fichiers de test passaient après les adaptations.

Un point de vigilance a été discuté pendant le débogage. Un test qui attend simplement `403` pour une action interdite à un autre utilisateur peut recevoir ce même statut dès le contrôle d’origine. Sa requête doit donc avoir une origine autorisée pour réellement atteindre la vérification des droits.

Le scénario d’origine non autorisée doit conserver une origine refusée pour la requête qu’il teste. Les éventuels scénarios d’origine absente doivent conserver cette absence. La version finale complète des fichiers de test n’a pas été relue par l’IA pendant cet échange. Le résultat consigné ici correspond à mes exécutions locales annoncées.

## Bilan de la séance

Le contrôle d’origine a été séparé dans un middleware, branché avant le routeur et pris en compte dans les tests. Le travail a avancé après avoir clarifié la fonction des helpers de test et repris les étapes dans l’ordre que j’avais demandé.

Cette séance couvre l’implémentation du contrôle `Origin` et l’adaptation des tests. La revue globale de la sécurité du backend reste à poursuivre.

## À reprendre demain — 4 novembre 2026

- [ ] Examiner les appels `res.cookie(...)` et les options des cookies d’authentification, notamment `SameSite`, `HttpOnly` et `Secure`.
- [ ] Examiner les routes `GET` pour vérifier si certaines modifient l’état de l’application, puisque le middleware les laisse passer.

Ces deux vérifications sont reportées à la prochaine séance.
