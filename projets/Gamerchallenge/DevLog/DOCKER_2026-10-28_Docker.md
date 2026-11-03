# DevLog — Préparer le backend GamerChallenge pour Docker
Date : 28 octobre 2026

## Objectif de la séance

Le MVP backend me semble largement avancé. Le frontend n’étant pas encore terminé, je reporte les tests de l’application complète et je profite de cette attente pour préparer Docker. Avant de continuer à copier le code du cours, je veux comprendre les étapes et les adapter à mon projet.

## 1. Distinguer environnement et données

J’ai commencé par expliquer l’image comme une « boîte contenant une recette » et le conteneur comme l’exécution de cette boîte. L’échange m’a permis de préciser la distinction :
- Dockerfile : les instructions pour construire l’image ;
- image : le code et l’environnement préparés ;
- conteneur : une instance créée à partir de cette image.

Ma question suivante concernait les données : faut-il une autre image pour les conserver, ou peut-on utiliser la base locale ? J’ai distingué l’image du logiciel de base de données, le conteneur qui exécute ce logiciel et le volume qui conserve les données indépendamment du conteneur.

J’ai ensuite demandé si le volume était stocké localement, puis déduit que deux ordinateurs peuvent exécuter la même application avec des données différentes. La précision importante est que cela dépend de la base à laquelle l’application se connecte : deux applications reliées à la même base distante utilisent les mêmes données.

Une parenthèse sur mon application de stage, reliée à une base distante interne à l’entreprise, m’a aidée à faire cette distinction. Pour GamerChallenge, nous envisageons un backend et une base PostgreSQL dans des conteneurs séparés, avec un volume pour les données. Une nouvelle base ne récupérera pas automatiquement les données de ma base locale existante.

## 2. Partir du Dockerfile du cours et vérifier les commandes

J’ai copié le Dockerfile du formateur. Il contient notamment :

```dockerfile
RUN npm run db:generate
RUN npm run build
CMD ["npm", "run", "docker:start"]
```

En comparant ces lignes avec mon package.json, nous avons constaté que db:generate existait déjà, mais pas build ni docker:start.

Le principe RUN / CMD commence à prendre sens :
- RUN exécute une commande pendant la construction de l’image ;
- CMD définit la commande exécutée par défaut au démarrage du conteneur.

Ce n’est donc pas RUN qui construit l’image : docker build lit le Dockerfile et exécute ses étapes.

## 3. Revenir sur TypeScript : mes questions et les ambiguïtés

La préparation de Docker a fait ressortir plusieurs questions :
- Que représente le dossier dist ?
- Que signifie exactement noEmit ?
- Puisque TypeScript ressemble à JavaScript avec des types, pourquoi faut-il une transformation ?
- Si nous retirons npm run build, où cette transformation a-t-elle lieu ?

Je me souvenais surtout d’avoir utilisé tsx pendant les exercices, beaucoup moins tsc. Mes scripts expliquent cette impression : dev et db:seed utilisent tsx, tandis que typecheck utilise tsc --noEmit.

Les distinctions clarifiées pendant la discussion sont :

| Outil ou option | Rôle |
| --- | --- |
| tsc — TypeScript Compiler | Vérifier les types et, selon la configuration, générer des fichiers JavaScript |
| --noEmit / noEmit: true | Ne pas produire de fichiers de sortie |
| tsx — TypeScript Execute | Transformer le code TypeScript pour l’exécuter, sans effectuer de vérification des types |
| dist | Nom conventionnel du dossier contenant les résultats du build |

Un point a nécessité plusieurs échanges : une explication de l’IA mélangeait « ne pas transformer » et « ne pas enregistrer de fichiers JavaScript dans dist ». J’ai relevé la contradiction en demandant où se faisait la compilation avec tsx. La formulation a été corrigée : tsx effectue la transformation nécessaire à l’exécution, mais ne prépare pas un dossier dist comme notre build avec tsc.

Je ne considère pas encore toutes ces notions comme automatisées. Les relier aux commandes réelles de mon projet m’aide davantage que de mémoriser seulement leurs définitions.

## 4. Choisir de compiler pendant la construction de l’image

Une première proposition consistait à conserver tsx pour démarrer l’application dans le conteneur. J’ai préféré suivre le principe du formateur :
1. générer le client Prisma ;
2. vérifier les types et générer le JavaScript pendant le build ;
3. exécuter ensuite le JavaScript avec Node.

Ce qui m’intéresse est de détecter les problèmes de compilation pendant la construction de l’image. Cela ne prouve pas encore que l’application fonctionnera correctement avec sa base ou son frontend.

Les scripts retenus sont :

```json
"build": "tsc",
"docker:start": "node dist/src/index.js"
```

Le chemin contient dist/src, car rootDir est la racine du projet : la structure des dossiers est conservée dans dist. Le projet utilise aussi config.ts à la racine et le client Prisma dans generated.

## 5. Erreur de configuration : comprendre extends

L’IA avait proposé un fichier tsconfig.build.json séparé pour hériter du tsconfig.json de développement. J’ai placé ce contenu dans tsconfig.json, pensant modifier le fichier existant.

Résultat :

```text
Circularity detected while resolving configuration: .../tsconfig.json
```

La ligne suivante demandait au fichier de se charger lui-même :

```json
"extends": "./tsconfig.json"
```

J’ai demandé pourquoi ajouter un second fichier alors que le projet du formateur n’en utilisait qu’un. Nous avons retenu une configuration unique. La séparation développement / build était une possibilité d’organisation, pas une nécessité pour Docker.

Une autre erreur est apparue ensuite :

```text
error TS5058: The specified path does not exist: 'tsconfig.build.json'.
```

Le script build pointait encore vers ce fichier absent. Je l’ai remplacé par tsc -p tsconfig.json et le build a réussi. Le script final retenu est tsc : depuis la racine du projet, il retrouve ce même fichier de configuration.

## 6. Comparer les options plutôt que copier sans comprendre

J’ai demandé les différences entre les configurations proposées et celle du formateur.

### Sélection des fichiers

include: ["./**/*.ts"] prend comme point de départ les fichiers TypeScript de l’ensemble du projet, alors qu’une liste de chemins sélectionne certaines zones. Les exclusions réduisent cette sélection initiale. Un fichier importé peut toutefois être pris en compte même s’il n’est pas sélectionné directement.

J’ai choisi de suivre le modèle du formateur :

```json
"include": ["./**/*.ts"],
"exclude": ["node_modules", "dist", "**/*.test.ts"]
```

Le motif **/*.test.ts exclut aussi les fichiers qui finissent par .spec.test.ts ou .unit.test.ts. En revanche, il n’exclut pas tous les fichiers auxiliaires du dossier src/tests.

### Résolution des modules

Nous avons remplacé Bundler par NodeNext pour correspondre à l’exécution du JavaScript généré directement avec Node. Les deux options sont associées :

```json
"module": "NodeNext",
"moduleResolution": "NodeNext"
```

rewriteRelativeImportExtensions permet de conserver les imports relatifs en .ts dans le code source tout en les transformant en .js dans les fichiers générés.

### Configuration finale partagée pendant la séance

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmit": false,
    "noEmitOnError": true,
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["node"],
    "rootDir": ".",
    "outDir": "./dist"
  },
  "include": ["./**/*.ts"],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts"
  ]
}
```

noEmitOnError empêche la génération de nouveaux résultats en présence d’erreurs de compilation. noEmit: false autorise la génération ; cette valeur pourrait aussi être omise puisqu’elle est la valeur par défaut.

sourceMap et forceConsistentCasingInFileNames ont été proposés en complément, mais leur ajout n’a pas été confirmé dans la suite de l’échange. declaration et declarationMap ont été discutés : ils ne sont pas nécessaires pour exécuter ce backend.

## 7. Résultat vérifié et suite

J’ai exécuté le build sur ma machine de développement et confirmé qu’il s’était terminé sans erreur.

À ce stade :
- le build TypeScript local a réussi ;
- la construction de l’image Docker n’a pas encore été validée ;
- le démarrage du JavaScript compilé n’a pas encore été confirmé ;
- le fichier Compose n’a pas encore été rédigé.

Avant Compose, l’IA a proposé de créer un .dockerignore pour éviter de copier node_modules, dist, generated, Git et les fichiers d’environnement locaux dans l’image. generated et dist seront reconstruits pendant le build Docker. La création effective de ce fichier reste à confirmer.

Prochaines étapes : relire le Dockerfile réellement enregistré, préparer .dockerignore, puis configurer le backend, PostgreSQL et le volume dans compose.yaml. La disponibilité des variables attendues par la configuration Prisma pendant le build reste également à vérifier.

## Regard sur ma manière d’apprendre

Cette séance a surtout été un travail de questions, de comparaison et de débogage : j’ai interrompu la copie pour demander pourquoi chaque étape existait. Le choix d’un seul tsconfig.json vient aussi de cette démarche : je voulais comprendre la différence avec le cours avant d’ajouter de la structure.

L’IA a servi à expliquer, comparer et organiser les modifications ; certaines explications ont dû être corrigées. Ce journal restitue ces échanges et reste à relire par moi pour confirmer qu’il représente fidèlement mon apprentissage.

