# 31/10/2026 — Du build frontend bloqué à l’affichage des données avec Docker

## Point de départ

Le build de l’image frontend échouait pendant `tsc -b`, avant même le build Vite. Mon objectif était de résoudre les erreurs en confrontant la réponse réelle de l’API, le type TypeScript et le code des pages, plutôt que de modifier tous les fichiers à l’aveugle.

Je travaille dans ma copie personnelle `frontend`, reliée au dépôt Frontend_Jinseon. Le dossier `frontendMLM` reste une copie de l’ancien travail commun ; j’ai décidé de ne pas fusionner l’ancienne branche `fix/challenge-cards`.

## Comparer les données avant de remplacer les propriétés

J’ai commencé par consulter une réponse de `GET /api/challenges`, puis le formateur du backend. Le frontend utilisait encore des noms et des structures provenant de l’ancienne API.

| Ancienne utilisation | Donnée réellement envoyée |
| --- | --- |
| `videoUrl` ou `url_video` | `urlVideo` |
| `game` ou `jeu.title` | `gameName` |
| `authorId` ou `user.id` | `userId` |
| `user.username` | `author`, directement une chaîne |
| `voter.length` | `likes`, directement un nombre |

Devant les nombreuses occurrences, j’ai envisagé les remplacements groupés dans VS Code. Mais certaines adaptations changent la structure utilisée, pas seulement le nom : un nombre de likes ne se manipule pas comme un tableau de votes.

J’ai aussi momentanément confondu le code de ChallengeCard avec celui de ChallengesPage en reprenant le frontend. Revenir au nom du fichier et à son export a permis de retrouver mes repères.

## Dates, valeurs nulles et composants

Dans ChallengesPage, j’ai conservé le classement prévu : les jours les plus récents d’abord, puis les challenges les plus aimés au sein d’une même journée.

Une première proposition de correction transformait les dates en nombres trop tôt, alors que la suite utilisait encore `getFullYear()`, `getMonth()` et `getDate()`. La correction finale conserve des objets Date jusqu’au calcul du jour.

Pour les champs encore typés comme pouvant être nuls, j’ai ajouté un traitement adapté à leur usage : description vide pour la carte et date affichée uniquement si elle existe. Une seule réponse JSON avec des valeurs renseignées ne suffit pas à prouver que ces champs ne peuvent jamais être nuls.

Le composant garde sa prop `game` ; la page lui transmet `challenge.gameName`. Le nom d’une prop d’affichage n’a pas besoin d’être identique au nom du champ de l’API.

## Comprendre le classement avant de le modifier

J’ai demandé à lire la logique de LeaderboardPage avant de poursuivre les corrections. La boucle prépare les données avant le rendu : elle regroupe les challenges par identifiant d’auteur et additionne les likes reçus sur leurs challenges.

Le backend envoie déjà le nombre de votes de chaque challenge avec `likes: challenge._count.votes`. Le frontend calcule ensuite le total par auteur.

J’ai conservé cette logique en utilisant `userId` comme clé de la Map, `author` comme nom et `likes` comme score à additionner. Après une modification, une accolade supplémentaire a fermé le composant trop tôt : les classements ne retrouvaient plus `challenges` ni `playersById`. Comparer le code modifié au bloc d’origine a permis de localiser cette erreur.

### Réflexion pour une évolution du projet

La discussion sur les totaux m’a amenée à questionner l’utilité de la pagination pour une page qui affiche seulement un Top 10.

La distinction utile est celle entre les données récupérées et celles affichées : actuellement, le frontend reçoit tous les challenges, calcule les classements et n’affiche que dix résultats. Un Top 10 n’a pas besoin de pagination pour autant.

En fin de projet, j’ai pris conscience que limiter l’affichage à dix résultats ne limite pas la quantité de données transférées, ce qui m’a amenée à envisager une API dédiée au classement.

**Évolution envisagée :** déplacer l’agrégation et le tri vers le backend pour ne renvoyer que le Top 10 lorsque le volume augmentera. Cette évolution reste à réaliser ; le calcul côté frontend est conservé pour le MVP.

## Du build réussi au problème de connexion

Les vérifications ont ensuite passé :

- `npx tsc -b` : aucune erreur.
- `npm run build` : build Vite réussi, annoncé en 886 ms.
- Build Docker de `front-service` : image créée.

Une déclaration en double de `front-service` dans Compose a également été corrigée avant de pouvoir construire l’image.

Une fois les conteneurs démarrés, la page affichait pourtant « Impossible de récupérer les challenges ». J’ai d’abord constaté un problème côté API : aucun port publié n’apparaissait pour son conteneur, malgré la configuration présente dans Compose.

Un serveur backend local tournait également ; j’ai envisagé un conflit sur le port 3000, sans confirmer que c’était la cause initiale. Après sa fermeture et la recréation du conteneur API, le mapping `3000:3000` était bien visible.

Le navigateur a ensuite révélé un autre problème, cette fois explicite : le backend autorisait seulement l’origine `http://localhost:5173`, alors que le frontend Docker était ouvert sur `http://localhost:8000`.

J’ai modifié la configuration CORS dans `src/app.ts` pour autoriser les deux origines, puis reconstruit l’image backend. Après actualisation, les données se sont affichées.

## Résultat et suite

Le frontend est servi par Nginx sur le port 8000 et la récupération des challenges fonctionne avec le backend conteneurisé. Le succès du build ne validait donc pas à lui seul la communication depuis le navigateur : les ports publiés et CORS ont dû être vérifiés séparément.

Les messages de commit ont été préparés pour les adaptations frontend et pour Compose/CORS côté backend. Il reste à terminer la sauvegarde Git, puis à vérifier les parcours fonctionnels, notamment l’authentification et les actions sur les challenges. L’affichage des données fonctionne ; tous les parcours ne sont pas encore validés.

*Ce journal reprend mes questions, mes essais et les résultats observés pendant la séance. L’IA a aidé à structurer le récit et à préciser les formulations ; la relecture finale me revient.*

