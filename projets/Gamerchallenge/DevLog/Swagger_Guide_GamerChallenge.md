# Swagger — Guide d’utilisation

**02/10/2026**

Swagger permet de consulter la documentation de l’API et d’envoyer des requêtes depuis le navigateur. Le backend traite les requêtes ; nous vérifions les réponses.

## 1. Installer les packages

Dans le dossier **backend** :

```bash
npm install swagger-ui-express swagger-jsdoc
npm install -D @types/swagger-ui-express @types/swagger-jsdoc
```

| Package | Rôle |
| --- | --- |
| `swagger-ui-express` | Afficher la documentation et les boutons de test |
| `swagger-jsdoc` | Construire la spécification OpenAPI à partir des annotations |
| `@types/...` | Fournir les types pour TypeScript |

## 2. Créer la configuration

Créer **`src/swagger.ts`** :

```ts
import swaggerJsdoc from "swagger-jsdoc";

export const swaggerSpec = swaggerJsdoc({
  definition: {
    // Version de la spécification OpenAPI.
    openapi: "3.0.0",
    info: {
      title: "GamerChallenge API",
      version: "1.0.0",
      description: "Documentation de l’API GamerChallenge",
    },
    // Préfixe de l’API sur le même serveur que Swagger.
    servers: [{ url: "/api" }],
  },
  // Chemin relatif au dossier depuis lequel on lance npm.
  apis: ["./src/routers/*.ts"],
});
```

**À distinguer :** `servers.url` indique où envoyer les requêtes ; `apis` indique où lire les commentaires de documentation.

## 3. Connecter Swagger à Express

Dans **`src/index.ts`**, ajouter les imports :

```ts
import swaggerUi from "swagger-ui-express";
import { swaggerSpec } from "./swagger";
```

Après la création de `app`, avant le branchement du router et `app.listen` :

```ts
// Affiche la documentation et les outils de test.
app.use("/api-docs", swaggerUi.serve, swaggerUi.setup(swaggerSpec));

// Branche les routes de l’API.
app.use("/api", router);
```

## 4. Documenter une route

Dans **`src/routers/challenge.router.ts`**, placer cette annotation au-dessus de la route :

```ts
/**
 * @openapi
 * /challenges:
 *   get:
 *     summary: Récupérer la liste des challenges
 *     tags:
 *       - Challenges
 *     responses:
 *       '200':
 *         description: Liste des challenges récupérée avec succès.
 */
router.get("/challenges", getAll);
```

Le préfixe `/api` est déjà défini dans la configuration : le test appelle donc **`GET /api/challenges`**. Respecter l’indentation du commentaire.

Les annotations décrivent l’API ; elles ne créent ni la route ni le contrôleur.

## 5. Tester dans le navigateur

1. Lancer le backend depuis sa racine avec `npm run dev`.
2. Ouvrir **http://localhost:3000/api-docs** (adapter le port si nécessaire).
3. Cliquer sur la ligne **GET /challenges**.
4. Cliquer sur **Try it out**, puis **Execute**.
5. Vérifier **Server response** : le code reçu et le **Response body**.

Pour la liste : vérifier un code **200** et les données attendues du seed. Le code annoncé dans la documentation n’est pas une preuve de réussite : regarder la réponse réelle du serveur.

## 6. Si quelque chose ne fonctionne pas

| Symptôme | Vérification |
| --- | --- |
| `No operations defined in spec!` | Ajouter une annotation, vérifier le chemin `apis`, puis redémarrer le backend |
| Une nouvelle annotation n’apparaît pas | Redémarrer le backend puis actualiser la page Swagger |
| La page Swagger est inaccessible | Vérifier que le backend tourne et que le port est correct |
| Réponse 404 | Comparer le chemin documenté avec la route Express et son préfixe |
| Réponse 500 | Consulter l’erreur dans le terminal du backend |

**Pour les prochaines routes :** ajouter leurs annotations, redémarrer le serveur si nécessaire, puis tester les réponses réelles.
