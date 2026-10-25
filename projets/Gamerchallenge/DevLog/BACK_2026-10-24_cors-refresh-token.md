# 24/10/2026 — GamerChallenges : débogage CORS et renouvellement des tokens

## Contexte
Après le travail sur l’authentification du backend, j’ai essayé de me connecter depuis le client React. Le navigateur affichait une erreur CORS avec un statut `(null)`, puis une erreur `NetworkError`.

## Débogage : remonter de l’erreur à la configuration
Au lieu de modifier directement la configuration, nous avons d’abord vérifié si le backend répondait :

```bash
curl -i http://localhost:3000/api/auth/login
```

Le serveur Express a répondu `404 — Cannot GET /api/auth/login`. Cette réponse confirmait que le backend était joignable depuis le terminal. Le test utilisait GET, alors que la connexion attendait POST : ce 404 ne suffisait donc pas à conclure que la route de connexion était incorrecte.

L’en-tête de réponse autorisait déjà `http://localhost:5173`, qui correspondait à l’adresse du frontend. La comparaison des deux adresses n’a donc pas révélé de décalage d’origine.

Pour retrouver l’appel de connexion dans le frontend, j’ai demandé où chercher le code. L’examen du `fetch` a montré `credentials: "include"`. Côté backend, la configuration CORS autorisait l’origine, mais ne contenait pas `credentials: true`.

Modification effectuée dans `src/index.ts` :

```ts
app.use(
  cors({
    // Autorise le front-end et les cookies d’authentification.
    origin: "http://localhost:5173",
    credentials: true,
  }),
);
```

Le point clarifié pendant cet échange : autoriser l’origine du frontend ne suffit pas pour une requête utilisant des cookies ; la configuration doit également autoriser les credentials.

## Refresh token : relire le code et raccorder la route
J’ai demandé si le renouvellement devait être placé dans le contrôleur d’authentification. Le code partagé contenait déjà `refreshAccessToken` : nous avons relu son fonctionnement avant de raccorder la route.

Le traitement présenté :
- lit le refresh token dans le cookie ;
- calcule son hash pour rechercher l’enregistrement en base ;
- vérifie son expiration ;
- génère un nouvel access token et un nouveau refresh token ;
- remplace les anciens refresh tokens de l’utilisateur en base et met à jour les cookies HttpOnly.

Le refresh token utilisé ici est une chaîne aléatoire dont le hash est conservé en base. Sa validation repose sur la recherche en base et la date d’expiration.

J’ai précisé ma convention de routage avec une référence directe à la fonction :

```ts
router.post("/auth/refresh", refreshAccessToken);
```

L’orthographe `refesh` a été corrigée en `refresh`. La documentation Swagger a été préparée pour `POST /api/auth/refresh`, avec les réponses 200 et 401. J’ai ensuite confirmé que le test fonctionnait.

## Point relevé pendant la relecture
Le login renvoyait `.json(user)`, ce qui pouvait inclure le hash du mot de passe. Une correction a été proposée pour ne retourner que `id`, `pseudo` et `email`. Son application n’a pas été confirmée dans cet échange.

## État en fin de séance
- Ajout de `credentials: true` visible dans le diff Git.
- Fonctionnement du refresh confirmé après le raccordement et la documentation Swagger.
- Commandes de commit et de push préparées pour les deux étapes ; leur exécution finale n’a pas été montrée.
- La connexion depuis le frontend après correction CORS et les scénarios refresh absent, invalide ou expiré restent à confirmer explicitement.

*Note rédigée avec l’aide de l’IA à partir de cet échange, à relire et à valider par moi.*

