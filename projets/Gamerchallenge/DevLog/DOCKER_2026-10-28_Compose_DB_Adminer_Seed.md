# DevLog — Docker Compose et première base conteneurisée
**Date : 28 octobre 2026 — Projet Gamer Challenge**

## Point de départ
La compilation TypeScript locale avait réussi. Le Dockerfile prévoyait déjà l’installation des dépendances, la génération du client Prisma, puis la compilation avant le démarrage de l’API. La suite du travail consistait à relier les services avec Compose et à vérifier chaque étape.

Ce journal reprend nos échanges et mes vérifications. L’IA m’aide à les structurer ; la relecture et la validation de ce qui correspond réellement à mon apprentissage restent de mon côté.

## 1. Vérifier ce qui entre dans l’image
J’ai présenté mon Dockerfile et vérifié la présence de `.dockerignore`. Son contenu exclut notamment `node_modules`, `dist`, `generated` et les fichiers `.env`, avec une exception pour `.env.exemple`.

Les dossiers `generated` et `dist` sont reconstruits dans l’image par `db:generate` et `build`. La lecture successive de deux fichiers avec `cat` avait collé la dernière ligne de `.dockerignore` au commentaire du fichier Prisma : il manquait un saut de ligne à la fin du premier fichier.

Nous avons aussi examiné `prisma7.config.ts`, qui lit `process.env["DATABASE_URL"]`. La réussite ultérieure du build a confirmé que la génération du client fonctionnait dans cette image, sans copier mon fichier `.env`.

## 2. Construire Compose à partir du projet du formateur
J’ai voulu suivre la structure du formateur et comprendre les différences plutôt que reprendre une configuration sans les examiner.

La base locale utilise PostgreSQL **17.11**. Nous avons retenu `postgres:17` pour rester sur le même numéro majeur, sans fixer exactement la version 17.11.

Le service `db` définit le compte, le mot de passe et le nom de la base à initialiser. Le volume `postgres_data` est monté dans `/var/lib/postgresql/data` pour conserver les données indépendamment du conteneur.

J’ai ensuite retrouvé trois paramètres du projet du formateur que je souhaitais ajouter :

- `restart: unless-stopped` ;
- un réseau explicite ;
- un `healthcheck` avec `pg_isready`.

### Préciser ma compréhension
Je me souvenais que le réseau permettait de relier la DB et l’API. Cette idée était déjà présente : nous l’avons précisée avec un réseau commun, `gamerchallenge-network`, et le nom de service `db` comme adresse de la base.

En revanche, je pensais que le healthcheck vérifiait la connexion entre l’API et la DB. Nous avons corrigé ce point : **ce healthcheck vérifie que PostgreSQL est prêt à accepter des connexions depuis son propre conteneur**. Il ne valide ni la connexion réelle de l’API, ni la présence des tables.

Un conteneur `unhealthy` n’est pas automatiquement redémarré par `restart` : l’état de santé et l’arrêt du processus sont deux choses différentes.

Les variables du healthcheck sont écrites avec `$$` pour que le shell du conteneur lise ses propres variables `POSTGRES_USER` et `POSTGRES_DB`.

## 3. Adapter le service API à mon arborescence
J’ai partagé le service API du formateur. Nous avons adapté les points qui dépendent de mon projet :

| Élément | Choix pour Gamer Challenge | Pourquoi |
|---|---|---|
| Contexte de build | `.` | Compose et Dockerfile sont dans le dossier backend |
| Port de l’application | `3000` | Cohérence avec le port interne publié |
| Dépendance | `db` | C’est le nom réel de mon service PostgreSQL |
| Adresse de la DB | `API_DATABASE_URL` transmise comme `DATABASE_URL` | Utiliser une adresse Docker distincte de l’adresse locale |
| Réseau | `gamerchallenge-network` | API et DB doivent pouvoir communiquer |

Dans l’URL Docker, l’hôte est `db`, pas `localhost`. Depuis le conteneur API, `localhost` désignerait le conteneur API lui-même.

La condition `service_healthy` attend la disponibilité de PostgreSQL ; **elle ne crée pas les tables**.

## 4. Tester progressivement au lieu d’attendre la fin
J’ai demandé si c’était trop tôt pour tester. Nous avons séparé les vérifications : configuration Compose, construction de l’image, démarrage de la DB, migrations, puis données de démonstration.

Pendant le build, npm a affiché un avertissement de dépréciation concernant `glob@11.1.0`. Nous n’avons pas modifié les dépendances à ce stade. La recherche de la dépendance responsable avec `npm explain glob` reste à faire.

Je me souvenais que la première construction Docker pouvait être longue. Nous avons relié cela au téléchargement de l’image et à l’installation initiale des dépendances, puis au rôle du cache lors des constructions suivantes.

**Résultat observé :**

```text
[+] Building 1/1
 ✔ api-service  Built
```

Cela confirme la construction de l’image, y compris les étapes Prisma et TypeScript. Cela ne confirme pas encore le fonctionnement du serveur HTTP.

## 5. Démarrer PostgreSQL et appliquer les migrations
J’ai démarré uniquement le service DB, puis consulté son état :

```text
backend-db-1   postgres:17   ...   db   ...   Up 49 seconds (healthy)   5432/tcp
```

Nous avons ensuite utilisé un conteneur API temporaire pour appliquer les migrations existantes à cette nouvelle base.

**Résultat observé :**

```text
All migrations have been successfully applied.
```

La commande remplace le démarrage habituel du serveur par la tâche de migration. Avec `--rm`, le conteneur temporaire est supprimé à la fin, mais les données restent dans le volume PostgreSQL.

## 6. Ajouter Adminer pour voir la base
J’ai proposé d’ajouter un outil d’administration afin de consulter plus facilement la DB. Nous avons ajouté un service Adminer sur le même réseau, avec `db` comme serveur par défaut et le port 8080 pour l’interface.

J’ai confirmé que j’accédais correctement à Adminer depuis mon environnement VM. L’éventuelle différence entre la machine du navigateur et celle de Docker ne bloquait donc pas mon accès.

## 7. Remplir la base avec le seed existant
Après la connexion à Adminer, j’ai demandé comment lancer le seeding.

Le projet possède déjà le script :

```json
"db:seed": "tsx prisma/seed.ts"
```

Nous avons repris le même principe que pour les migrations : lancer une tâche ponctuelle dans un conteneur API temporaire, avec la configuration Compose pour atteindre la DB Docker.

Point concret vérifié dans le fichier fourni : **ce seed supprime les données existantes des utilisateurs, challenges, participations et votes avant de recréer les données de démonstration**. Ce n’est donc pas un simple ajout de lignes.

À ce stade, j’ai indiqué que les étapes s’enchaînaient avec succès et demandé ce journal. La sortie détaillée du seed n’a pas été collée dans la conversation ; je pourrai compléter la vérification avec le message de fin et les données visibles dans Adminer.

## Ressenti et façon de travailler
« C’est amusant quand tout réussit à la suite. »

Le travail a avancé par petits contrôles. J’ai comparé la configuration du formateur avec la mienne, proposé les options que je reconnaissais, puis ajusté une compréhension précise du healthcheck. Les questions et la lecture des résultats ont fait partie du travail autant que l’écriture du YAML.

## État à la fin de cette session
| Étape | État dans nos échanges |
|---|---|
| Compilation TypeScript locale | Réussite confirmée avant cette séquence |
| Image API | Build réussi, sortie partagée |
| PostgreSQL Docker | État healthy, sortie partagée |
| Migrations | Réussite, sortie partagée |
| Adminer | Accès confirmé dans la VM |
| Seed | Commande donnée ; progression réussie indiquée, sortie détaillée non partagée |
| API HTTP en fonctionnement | Pas encore vérifiée dans cette séquence |
| Avertissement glob | À examiner |

## Commandes — ordre du parcours
Les commandes ci-dessous sont regroupées pour retrouver le déroulement. Les commandes de contrôle proposées sans sortie partagée sont identifiées ; je ne les présente pas comme des résultats vérifiés.

### A. Vérifications avant Compose
Depuis le dossier backend :

```bash
find . -name '.dockerignore' -not -path './node_modules/*'
cat .dockerignore prisma7.config.ts
```

Dans la session SQL de la base locale :

```sql
SELECT version();
```

### B. Vérifier la configuration et construire l’image
Contrôle de configuration proposé, sans sortie partagée :

```bash
docker compose config --quiet
```

Construction de l’image, réussite confirmée :

```bash
docker compose build api-service
```

### C. Démarrer la DB et regarder son état
```bash
docker compose up -d db
docker compose ps
```

### D. Appliquer les migrations
```bash
docker compose run --rm api-service npx prisma migrate deploy --config=prisma7.config.ts
```

### E. Démarrer Adminer
Commande donnée, accès à l’interface confirmé ensuite :

```bash
docker compose up -d adminer
```

### F. Lancer le seed
Commande donnée pour remplir la nouvelle base de démonstration ; attention, elle remplace les données existantes :

```bash
docker compose run --rm api-service npm run db:seed
```

### À part : diagnostic proposé, pas encore effectué dans nos échanges
```bash
npm explain glob
```

## Prochaine étape
Vérifier les données du seed dans Adminer, puis démarrer le service API et contrôler ses logs et une route HTTP. Ces vérifications restent distinctes de la réussite du build et des migrations.

