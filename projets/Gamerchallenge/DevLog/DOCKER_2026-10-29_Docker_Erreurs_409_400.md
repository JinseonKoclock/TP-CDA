# DevLog — Heureuse de voir des erreurs 409 et 400 !
**29 octobre 2026 — Gamer Challenge**

## Reprendre Docker : image, conteneur et volume
En relançant mon environnement, Adminer et PostgreSQL étaient actifs, mais pas l’API. J’ai demandé pourquoi.

La veille, j’avais construit l’image API et exécuté les migrations et le seed dans des conteneurs temporaires avec `run --rm`. Le conteneur destiné à laisser tourner le serveur n’avait pas encore été créé.

Après `docker compose up -d api-service`, les logs ont confirmé le lancement de `node dist/src/index.js` sur le port 3000.

J’ai alors repensé à la commande de la veille : fatiguée, je n’avais pas demandé pourquoi supprimer ce conteneur. Je me demandais aussi comment le démarrage pouvait être si rapide après une suppression.

Nous avons précisé la distinction :
- l’image reste disponible après le build ;
- un conteneur est créé depuis cette image pour exécuter une commande ;
- le volume PostgreSQL conserve les données.

`--rm` avait supprimé les conteneurs temporaires, pas l’image ni les données. Le démarrage n’exigeait donc pas de recommencer l’installation et la compilation.

## Ne pas confondre démarrage et vérification fonctionnelle
J’ai demandé si le backend Docker était terminé. Il fallait encore vérifier les requêtes réelles.

En ouvrant `localhost:3000`, puis `/api`, j’ai rencontré `Cannot GET`. Le serveur était accessible, mais ces chemins n’avaient pas de route GET. Le préfixe `/api` ne constitue pas automatiquement une page d’accueil.

Dans Swagger, sur `http://localhost:3000/api-docs`, j’ai confirmé l’inscription, la modification des informations utilisateur, la connexion et la consultation de l’utilisateur connecté.

J’ai pensé que le reste fonctionnerait probablement aussi. La création d’un challenge a pourtant révélé un cas encore mal traité.

## Premier 500 : le titre était déjà utilisé
Les logs ont identifié la cause :

```text
Unique constraint failed on the constraint: challenge_title_key
code: P2002
```

La DB refusait un doublon de titre. Ce n’était pas une panne de connexion Docker.

Le catch de mon contrôleur renvoyait 500 pour toute erreur, alors que mon projet possédait déjà une classe `ConflictError` et un middleware capable de répondre avec son statut 409.

Nous avons modifié le catch de `addChallenge` pour transformer P2002 en `ConflictError` avec le message :
**« Un challenge avec ce titre existe déjà. »**

Les autres erreurs sont relancées vers le middleware commun. Express 5 transmet les erreurs des fonctions async au gestionnaire d’erreurs.

## Pourquoi prisma et Prisma ?
Cette correction m’a contrariée : je ne voyais pas ces deux imports ensemble dans les fichiers du formateur. Je ne voulais pas ajouter de complexité sans en comprendre la raison.

| Nom | Rôle |
|---|---|
| `prisma` | Instance configurée qui exécute les requêtes DB |
| `Prisma`, renommé `PrismaErrors` | Accès notamment à la classe utilisée pour reconnaître l’erreur |

Le code généré et mon fichier de configuration n’ont pas la même responsabilité. Je ne maintiens pas manuellement deux clients : mon fichier centralise la connexion et partage une instance.

Une réexportation a été proposée pendant la discussion, mais n’était pas nécessaire. Nous avons poursuivi avec :

```ts
import { prisma } from "../database/prisma.ts";
import { Prisma as PrismaErrors } from "../../generated/prisma/client.ts";
```

J’ai aussi demandé si cette séparation pouvait être critiquée par le jury. L’explication à retenir porte sur la centralisation de la configuration, pas sur un nombre idéal de fichiers.

### Comparer avec le formateur
J’ai partagé sa fonction `createQuestion`. Elle vérifiait les droits, les données et l’existence d’un niveau, mais son catch renvoyait simplement 500. Elle ne distinguait pas les doublons.

L’absence de l’import d’erreur dans cet extrait ne prouvait donc pas une autre solution : ce traitement spécifique n’y figurait pas. Sans son schéma, nous ne pouvions pas conclure quelles contraintes uniques existaient sur les questions.

## Les petits blocages pendant la correction
J’ai rencontré des imports manquants, une confusion entre `prisma` et l’accès aux classes d’erreur, puis une erreur sur `err` de type `unknown`.

J’ai aussi oublié `&&` entre les conditions. Le message « ')' attendu » ne venait pas d’une parenthèse mal fermée :

```ts
if (
  err instanceof PrismaErrors.PrismaClientKnownRequestError &&
  err.code === "P2002"
) {
  throw new ConflictError(
    "Un challenge avec ce titre existe déjà.",
  );
}
```

Le premier contrôle précise le type de l’erreur ; le second vérifie son code.

J’ai ensuite signalé un contrôle TypeScript sans erreur.

## Test de non-régression et mise à jour Docker
J’ai lu mon fichier de tests et constaté l’absence d’un cas de titre en doublon.

Nous avons préparé un test AAA :
1. Créer un utilisateur authentifié et un challenge.
2. Envoyer une création avec le même titre.
3. Vérifier le statut 409 et le message.
4. Vérifier qu’une seule ligne porte ce titre en DB.

Le code du test a été fourni pour ajout. Son résultat d’exécution n’a pas été communiqué ; je ne le présente pas comme passé.

Je savais déjà que mes modifications locales ne seraient pas automatiquement visibles dans le conteneur. J’ai demandé s’il fallait supprimer l’image avant de la reconstruire. La suppression manuelle n’était pas nécessaire :

```bash
docker compose up -d --build api-service
```

Compose reconstruit l’image et remplace le conteneur API, sans effacer le volume DB.

Après mise à jour, j’ai confirmé dans Swagger : **409 pour un titre déjà utilisé**.

## Deuxième 500 : j’ai soupçonné la longueur du titre
Une modification de challenge a ensuite renvoyé 500. J’ai pensé que le titre était trop long et demandé pourquoi Zod ne renvoyait pas son message.

Nous avons relu le code : `.max(100)` était présent, et `.partial()` rend les champs facultatifs sans enlever leurs contraintes. Le contrôleur renvoyait déjà 422 en cas de validation échouée.

Les logs ont révélé une autre cause :

```text
SyntaxError: Expected double-quoted property name in JSON
type: entity.parse.failed
status: 400
```

J’avais laissé une virgule après le dernier champ :

```text
{
  "title": "Minecraft Nomadic Survival - Docker TEST",
}
```

Le JSON était invalide. Sa lecture échouait avant le contrôleur, donc avant Zod.

J’ai d’abord appelé cela une erreur de syntaxe Swagger. Nous avons précisé que c’est une règle JSON, indépendante de Swagger. La virgule expliquait le refus de la requête ; la réponse 500 révélait quand même un défaut du middleware.

## Zod faisait bien son travail
J’ai envoyé volontairement un titre de plus de 100 caractères dans un JSON valide, en plaisantant dans le titre sur la réaction de Zod.

La réponse contenait bien :

```json
{
  "field": "title",
  "message": "Too big: expected string to have <=100 characters"
}
```

La validation de longueur fonctionnait. Nous n’avions pas besoin de la réécrire pour résoudre l’erreur de virgule.

La distinction utile :
**lecture du JSON → validation Zod → requête DB.**

## Un 400 utile pour le JSON invalide
J’ai demandé comment rendre l’erreur immédiatement compréhensible et s’il suffisait de conserver son numéro.

Nous avons ajouté dans le middleware, après `headersSent` et avant les autres traitements :

```ts
// Signaler un JSON mal formé avant la validation Zod.
if (
  error instanceof SyntaxError &&
  "type" in error &&
  error.type === "entity.parse.failed"
) {
  return res.status(400).json({
    message:
      "JSON invalide. Vérifiez les guillemets et les virgules.",
  });
}
```

Le marqueur `entity.parse.failed` cible l’échec de lecture du JSON : il ne transforme pas toutes les SyntaxError en erreurs client. Le message reste utile sans exposer le body reçu ni les détails internes.

Après mise à jour Docker, j’ai confirmé le 400 :
> « C’est la première fois que je suis contente de voir une erreur 400 ! »

## Résultats et limites
| Cas | État dans nos échanges |
|---|---|
| API Docker | Conteneur actif, démarrage confirmé par les logs |
| Inscription, modification utilisateur, connexion, informations utilisateur | Fonctionnement confirmé dans Swagger |
| Création avec titre déjà présent | 409 confirmé |
| Titre dépassant 100 caractères | Message Zod constaté |
| JSON avec virgule finale | 400 confirmé après correction |
| Test automatisé du doublon | Code préparé ; résultat non communiqué |
| Création avec titre inédit après correction | À confirmer explicitement |
| Modification vers un titre existant | Traitement 409 non ajouté dans cette séquence |

## Ressenti
J’ai eu l’impression de découvrir un bug chaque jour et me suis demandé si le travail finirait. Aujourd’hui, les essais ont surtout transformé des 500 vagues en réponses correspondant aux causes réelles.

J’ai comparé le code du formateur, questionné les imports, proposé une hypothèse sur Zod et utilisé les logs pour la vérifier. Les doutes, les petits essais et la discussion ont fait partie de l’apprentissage.

L’IA a aidé à analyser, proposer du code et structurer ce journal. Les questions, les essais dans mon environnement et la validation finale restent de mon côté. Ce texte reste à relire pour vérifier qu’il reflète exactement ma compréhension.

## Commandes — dans l’ordre du parcours
Depuis le dossier backend.

### 1. Démarrer l’API et vérifier son état
```bash
docker compose up -d api-service
docker compose ps -a
docker compose logs --tail=50 api-service
```

### 2. Diagnostiquer le premier 500
```bash
docker compose logs --tail=80 api-service
```

### 3. Relire la structure des erreurs
Commande proposée ; le contenu a ensuite été partagé :

```bash
cat src/middlewares/error.handler.ts src/lib/errors.ts
```

### 4. Contrôler les types et mettre à jour Docker
```bash
npm run typecheck
docker compose up -d --build api-service
```

Le contrôle des types a été signalé sans erreur pendant la correction de la création. Le retour 409 a ensuite été confirmé dans Swagger.

### 5. Diagnostiquer le 500 en modification
```bash
docker compose logs --tail=60 api-service
```

### 6. Après la correction du middleware JSON
Séquence proposée, suivie du retour 400 confirmé :

```bash
npm run typecheck
docker compose up -d --build api-service
```

### Tests automatisés — commande proposée, résultat non partagé
```bash
npm run test:integration
```

Cette commande lance la suite configurée. Ne pas noter le nouveau test comme réussi avant d’avoir lu son résultat.

## À reprendre
- Confirmer le résultat du test de création en doublon.
- Vérifier une création avec titre inédit après correction.
- Traiter le conflit de titre lors d’une modification.
- Ajouter un test du JSON mal formé si nous poursuivons la couverture du middleware.

Ces points restent distincts des réponses 409 et 400 déjà confirmées manuellement.

