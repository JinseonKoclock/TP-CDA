# DevLog — Configuration de Prisma et première migration

**Projet : GamerChallenges **

> Ce document structure les essais, les questions et les explications de L'IA. Il reste à relire et à compléter avec mes propres mots pour noter ce que je peux désormais expliquer seule.


## Objectif

Créer les tables PostgreSQL à partir du fichier `prisma/schema.prisma`. Le schéma Prisma est prêt ; cette séance porte sur la remise à zéro de la base et les erreurs rencontrées lors de la première migration.

## 1. Remise à zéro de la base

Dans psql, connecté à la base `gamerchallenge`, j’ai exécuté les commandes proposées pendant l’échange :

```sql
BEGIN;
DROP SCHEMA public CASCADE;
CREATE SCHEMA public AUTHORIZATION gamerchallenge;
COMMIT;
```

Ces commandes suppriment le schéma `public` et ses objets, puis recréent un schéma vide appartenant au rôle `gamerchallenge`. Cette opération efface les tables et leurs données : elle correspond ici à mon choix de repartir sur une base vide.

Vérification :

```sql
\dt
```

Résultat obtenu : `N'a pas trouvé de tables.`

Ma question était ensuite : faut-il sortir de psql et se reconnecter avec `sudo` ?

La réponse de l’AI était : non, pas à cette étape. Le schéma `public` avait déjà été recréé avec `gamerchallenge` comme propriétaire. Il suffisait de quitter psql avec `\q`, puis de lancer la commande Prisma depuis le terminal du dossier `backend`.

Plus tard, l’erreur P3014 a révélé un autre besoin : le rôle `gamerchallenge` n’avait pas le droit de créer la shadow database. C’est à ce moment-là qu’une connexion administrateur avec `sudo -u postgres psql` a été proposée pour lui accorder le droit `CREATEDB`.


## 2. Configuration utilisée par Prisma

Le fichier `prisma7.config.ts` contient :

```ts
import "dotenv/config";
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: process.env["DATABASE_URL"],
  },
});
```

Point clarifié pendant le dépannage : cette configuration lit précisément `DATABASE_URL`. Une autre variable appelée `DB_URL` n’est pas utilisée automatiquement à sa place.

## 3. Première tentative : erreur P1000

Depuis le dossier `backend` :

```bash
npx prisma migrate dev --name init --config prisma7.config.ts
```

La sortie indiquait une connexion à `mydb`, puis une erreur d’authentification `P1000`.

En regardant `.env`, nous avons trouvé deux URL : `DB_URL` pointait vers `gamerchallenge`, mais `DATABASE_URL` contenait encore l’exemple créé par `prisma init`, avec `johndoe`, `randompassword` et `mydb`.

J’ai corrigé `DATABASE_URL` pour utiliser mon compte et ma base. Exemple à adapter, sans conserver le mot de passe réel dans ce DevLog :

```env
DATABASE_URL="postgresql://gamerchallenge:MOT_DE_PASSE@localhost:5432/gamerchallenge?schema=public"
```

Après une nouvelle tentative, Prisma affichait bien la base `gamerchallenge`. L’erreur suivante concernait les permissions : la correction de l’URL avait donc permis de franchir l’étape d’authentification.

## 4. Deuxième tentative : erreur P3014

Message obtenu :

```text
Prisma Migrate could not create the shadow database.
ERREUR: droit refusé pour créer une base de données
```

L’explication apportée pendant l’échange : `migrate dev` utilise une base temporaire, appelée *shadow database*, pour vérifier les migrations. Le rôle `gamerchallenge` ne pouvait pas créer cette base.

La solution proposée pour mon environnement local de développement est :

```bash
sudo -u postgres psql
```

Puis, dans psql :

```sql
ALTER ROLE gamerchallenge CREATEDB;
\q
```

Le droit `CREATEDB` autorise la création de bases ; il ne transforme pas le rôle en superutilisateur.

**État au moment de la rédaction :** ces dernières commandes ont été proposées, mais leur exécution et le succès de la migration n’ont pas encore été confirmés dans l’échange.

## 5. Commandes à distinguer

| Commande | Rôle |
| --- | --- |
| `prisma generate` | Générer le client utilisé par le code pour accéder aux données. |
| `prisma migrate dev` | Créer et appliquer les migrations en développement pour faire évoluer les tables. |

Générer le client ne suffit donc pas à créer les tables. Dans cette séance, la commande utilisée pour la migration précise aussi le fichier de configuration :

```bash
npx prisma migrate dev --name init --config prisma7.config.ts
```

## 6. Suite à vérifier

- [x] Remettre le schéma `public` à zéro et constater l’absence de tables.
- [x] Corriger la variable `DATABASE_URL` utilisée par Prisma.
- [x] Identifier le blocage lié à la création de la shadow database.
- [ ] Exécuter la modification du rôle et vérifier son résultat.
- [ ] Relancer la migration depuis le dossier `backend`.
- [ ] Vérifier les tables dans psql avec `\dt` et examiner le fichier SQL créé dans `prisma/migrations`.

## Retour sur la démarche

Le dépannage s’est appuyé sur les sorties du terminal : d’abord le nom de la base (`mydb`), puis le message de droits insuffisants. Ces deux erreurs correspondent à deux étapes différentes : se connecter à la bonne base, puis disposer des droits nécessaires à la migration de développement.

