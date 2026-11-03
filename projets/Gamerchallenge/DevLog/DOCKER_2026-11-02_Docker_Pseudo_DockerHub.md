# DevLog — 2 novembre 2026

## GamerChallenges : inscription dans Docker et publication des images

### Objectif de la séance

Vérifier que les dernières corrections du frontend fonctionnent dans l’application exécutée avec Docker, sauvegarder les modifications sur GitHub et publier les images frontend et backend sur Docker Hub.

## 1. Le blocage : le code local et la requête envoyée ne correspondaient pas

J’avais repéré un champ `username` dans le frontend et l’avais remplacé par `pseudo`, le nom attendu par le backend. L’inscription fonctionnait en local, mais échouait encore depuis l’application Docker sur `http://localhost:8000/register`.

Le backend retournait une erreur de validation :

```text
path: ["pseudo"]
Invalid input: expected string, received undefined
```

Dans Network, la requête d’inscription envoyait toujours une propriété `username`. Le problème observable était donc précis : le backend attendait `pseudo`, mais ce champ n’était pas présent dans les données reçues.

Ma question était devenue : « Pourquoi Docker envoie-t-il encore `username` alors que j’ai déjà corrigé le code et relancé le build ? »

J’ai aussi redémarré la VM, sans résoudre immédiatement le problème. Cette tentative n’a pas permis d’identifier la cause.

## 2. Vérifier le code et les chemins avant de reconstruire encore

Depuis le dossier backend, j’ai recherché les deux noms dans les sources frontend :

```bash
grep -RnE 'username|pseudo' ../frontend/src
```

La recherche a montré que le corps de la requête d’inscription utilisait déjà le bon champ :

```tsx
body: JSON.stringify({ pseudo, email, password, confirm })
```

Deux références à `username` restaient toutefois dans le contexte d’authentification et dans l’affichage du profil :

- `src/context/AuthContext.ts` : propriété du type utilisateur ;
- `src/pages/ProfilePage/ProfilePage.tsx` : affichage de `user.username`.

Ces références ont été harmonisées avec `pseudo`. Elles concernaient le profil et son typage ; leur présence ne suffisait pas à expliquer pourquoi la requête d’inscription observée utilisait encore `username`.

Le terminal affichait parfois `~/Desktop/html/gamerChallenge/backend`, alors que je travaillais aussi avec `/var/www/html/gamerChallenge/backend`. Nous avons vérifié s’il s’agissait de deux copies :

```bash
realpath ~/Desktop/html/gamerChallenge/backend
realpath /var/www/html/gamerChallenge/backend
```

Les deux commandes ont donné `/var/www/html/gamerChallenge/backend`. L’hypothèse de deux dossiers différents a donc été écartée pour ces chemins.

## 3. Vérifier chaque étape : build, conteneur, navigateur

### Construction de l’image

```bash
docker compose --env-file .env.docker up -d --build front-service
```

La sortie a confirmé que la copie du code, `npm run build` et la copie de `dist/` dans l’image Nginx avaient bien été exécutées. Le build s’est terminé avec succès.

L’avertissement concernant l’absence de `buildx` n’a pas empêché ce build d’aboutir.

### Exécution du nouveau conteneur

```bash
docker compose --env-file .env.docker ps front-service
```

Le résultat indiquait un conteneur créé quelques secondes auparavant, à l’état `Up`, avec le port `8000` de la machine relié au port `80` du conteneur.

Cela confirmait le démarrage d’un nouveau conteneur, mais pas encore les fichiers réellement utilisés par le navigateur.

### Contrôle des fichiers JavaScript

Une recherche de `username` dans les fichiers JavaScript du conteneur n’a affiché aucun résultat. Nous avons ensuite listé les fichiers servis par Nginx :

```bash
docker compose --env-file .env.docker exec front-service ls /usr/share/nginx/html/assets
```

Le fichier JavaScript présent était :

```text
index-CgeNRDAI.js
```

Je ne savais pas où retrouver ce nom dans les outils du navigateur. J’ai donc été guidée pour ouvrir Network, recharger la page et filtrer les requêtes avec `index`.

La capture a montré une réponse HTTP `200` pour `index-CgeNRDAI.js` depuis `localhost:8000`. Le nom correspondait à celui du fichier dans le conteneur.

Après ce contrôle et un nouvel essai depuis la page rechargée, l’inscription d’un nouvel utilisateur et la connexion ont fonctionné.

### Ce que les vérifications permettent de conclure

Le frontend reconstruit et le conteneur lancé ont finalement permis une inscription et une connexion réussies. Le navigateur demandait alors un fichier JavaScript portant le même nom que celui du conteneur.

En revanche, les éléments recueillis ne permettent pas d’attribuer avec certitude toute la panne initiale au cache du navigateur, à un ancien build ou à la consultation d’une ancienne requête dans Network. Plusieurs vérifications et rechargements ont eu lieu. Je conserve cette incertitude plutôt que d’enregistrer une cause non démontrée.

## 4. Clarifier les commandes Docker

J’ai demandé la différence entre reconstruire l’image et recréer le conteneur, puis demandé des commentaires en français pour garder une référence pratique.

| Commande / option | Rôle |
| --- | --- |
| `build --no-cache front-service` | Reconstruire l’image sans réutiliser le cache de build ; ne remplace pas le conteneur en cours d’exécution |
| `up -d --no-deps --force-recreate front-service` | Recréer le conteneur avec l’image disponible, sans reconstruire l’image |
| `up -d --build --no-deps front-service` | Construire l’image, puis appliquer les changements au conteneur ; commande habituelle après une modification du frontend |

Commande à conserver pour le travail courant, depuis `backend/` :

```bash
docker compose --env-file .env.docker up -d --build --no-deps front-service
```

`--no-cache` et `--force-recreate` ne sont pas nécessaires systématiquement.

J’ai également lancé `docker compose --env-file .env.docker up` sans `-d` et demandé comment récupérer le terminal : cette forme reste attachée aux logs. `Ctrl + C` arrête les services démarrés par cette commande. Avec `up -d`, ils tournent en arrière-plan et le terminal reste disponible.

## 5. Sauvegarde sur GitHub

Le statut du frontend indiquait des modifications dans :

- `Dockerfile` ;
- `src/context/AuthContext.ts` ;
- `src/pages/InscriptionPage/InscriptionPage.tsx` ;
- `src/pages/NoticePage/NoticePage.tsx` ;
- `src/pages/ProfilePage/ProfilePage.tsx`.

Le push frontend a été confirmé par la sortie :

```text
6c9d740..53c571d  main -> main
```

J’ai demandé pourquoi les commandes proposées précisaient toujours `origin main`, alors que j’utilise habituellement `git push`. La branche suivait déjà `origin/main` : dans cette configuration, `git push` suffit.

Côté backend, `.env.docker.example` et `README.md` étaient modifiés. Les commandes de commit et de push ont été préparées, mais leur résultat n’a pas été partagé dans cet échange ; je ne les note donc pas comme confirmées.

## 6. Publication sur Docker Hub

Après vérification des images utilisées par Compose et authentification sous le compte `jinseonk`, les images locales ont reçu leurs noms Docker Hub :

```bash
docker tag backend-api-service:latest jinseonk/gamerchallenge-backend:latest
docker tag backend-front-service:latest jinseonk/gamerchallenge-frontend:latest

docker push jinseonk/gamerchallenge-backend:latest
docker push jinseonk/gamerchallenge-frontend:latest
```

Les premières lignes `Pushed` confirmaient l’envoi de certaines couches. La publication complète a ensuite été confirmée par les lignes finales contenant les digests :

| Image publiée | Digest |
| --- | --- |
| `jinseonk/gamerchallenge-frontend:latest` | `sha256:081996a4d9618d7deb765b3684ff35449a63c6d0aeb970c8ef3700d1f8c0d8fc` |
| `jinseonk/gamerchallenge-backend:latest` | `sha256:6ea6f1af7be1446eda7d30feb8b4b7a8d82e452745bb886ebcc4ee195fa65a89` |

La distinction a été précisée pendant la séance : GitHub reçoit les fichiers source, dont les Dockerfiles ; Docker Hub reçoit les images construites. Publier les images ne déploie pas automatiquement l’application sur un serveur.

## 7. Bilan et suite

### Résultats confirmés

- Inscription d’un nouvel utilisateur et connexion réussies dans l’application exécutée avec Docker.
- Reconstruction du frontend et démarrage de son nouveau conteneur confirmés.
- Comparaison du nom du fichier JavaScript entre le conteneur et Network.
- Push Git du frontend confirmé.
- Publication des deux images sur Docker Hub confirmée.

### Points clarifiés pendant le débogage

- Lire séparément le corps de la requête et la réponse du serveur permet de localiser un désaccord de champs.
- Une image construite, un conteneur démarré et un navigateur utilisant les fichiers attendus sont trois vérifications distinctes.
- Les options Docker ont été comparées pour conserver une procédure quotidienne plus simple.

### À poursuivre

- Confirmer la sauvegarde Git des modifications du backend si elle n’a pas encore été effectuée.
- Pour utiliser les images publiées sur une autre machine, préparer une configuration Compose qui les référence et fournit les variables nécessaires.
- Vérifier séparément toute future configuration de serveur : l’image frontend publiée a été construite pour une API accessible à `http://localhost:3000`.

## Démarche

Ce document a été rédigé avec une démarche d’apprentissage actif, en utilisant ChatGPT comme support de réflexion et de structuration.

Il restitue mes questions, les essais effectués et les résultats observables. L’IA aide à organiser le récit ; la relecture finale et la validation de ce qui correspond à mon apprentissage restent les miennes.
