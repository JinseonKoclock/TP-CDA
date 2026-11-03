# DevLog — Séparer les environnements et partager la configuration Docker
**29 octobre 2026 — Gamer Challenge**  
Suite du journal sur les erreurs HTTP 409 et 400.

## Avant le commit : vérifier les fichiers à partager
Je voulais enfin committer et pousser le travail Docker et les corrections du backend. Le git status montrait notamment les fichiers TypeScript modifiés, le Dockerfile, compose.yaml, .dockerignore, .env.docker et dist/.

Nous avons distingué le code et la configuration à versionner des fichiers propres à l’environnement et des résultats de compilation. Le dossier dist/ est produit par le build ; il n’a pas besoin d’être ajouté au dépôt.

Avant de poursuivre, j’ai soulevé un point : le formateur séparait les fichiers d’environnement locaux, Docker et de test. J’avais déjà créé mon fichier Docker en suivant cette organisation, alors que certaines valeurs Docker avaient aussi été ajoutées à mon .env pendant nos échanges.

## Relire les fichiers plutôt que conserver les doublons
J’ai partagé le contenu de .env.docker, de .env et de compose.yaml pour les mettre en cohérence. Le fichier de test, déjà configuré auparavant, n’a pas été repris dans cette séquence.

La répartition retenue :

| Fichier | Utilisation |
|---|---|
| .env | Serveur Node local : PORT, DATABASE_URL locale, JWT_SECRET |
| .env.docker | Variables utilisées par Compose : configuration PostgreSQL, URL DB Docker, port API publié, JWT_SECRET |
| Fichier env de test existant | Configuration des tests, conservée séparément |

Nous avons précisé que partager un même nom de variable entre environnements n’est pas un défaut. Chaque fichier doit contenir les valeurs nécessaires à son contexte.

### Incohérences repérées
- L’URL Docker utilisait api-db alors que le service PostgreSQL s’appelle db : l’hôte doit correspondre au nom de service.
- JWT_SECRET était présent dans le fichier local, mais absent du fichier Docker.
- POSTGRES_USER et POSTGRES_DB étaient écrits dans .env.docker, alors que Compose utilisait encore des valeurs fixes. Nous avons proposé de les lire avec des expressions de substitution.
- Le port DB 9999 et les variables CLIENT n’étaient pas utilisés dans le Compose partagé. Nous les avons retirés de cette configuration backend ; les besoins du frontend seront revus dans son propre travail Docker.

Aucun reset du volume n’était requis pour cette réorganisation avec les mêmes identifiants DB. Nous n’avons pas changé le schéma.

## Comprendre le rôle de --env-file
Le fichier .env.docker contient déjà des commentaires du formateur sur l’option --env-file. J’ai demandé à y regrouper les commandes utiles.

Deux rôles ont été distingués :
- --env-file .env.docker fournit les valeurs utilisées dans les expressions de Compose ;
- environment définit les variables transmises au conteneur.

Choisir un fichier pour Compose ne signifie donc pas transmettre automatiquement toutes ses variables à chaque conteneur.

## Changement de code ou changement de configuration ?
J’ai demandé s’il fallait « prévenir » les conteneurs après la modification des fichiers env.

Nous avons séparé deux cas :
- **Variables d’exécution ou configuration Compose modifiées** : up -d applique les changements et recrée les conteneurs concernés si leur configuration a changé.
- **Code backend modifié** : le code étant copié et compilé dans l’image, il faut reconstruire l’image avec --build, puis mettre à jour le conteneur.

Un simple restart ne charge pas les nouvelles valeurs d’environnement et ne remplace pas le code contenu dans l’image.

Cette distinction s’applique à notre montage actuel. Elle ne transforme pas les variables utilisées pendant un build frontend en variables modifiables à l’exécution.

## Partager la configuration avec Marie-Laure
J’ai fait remarquer que si .env.docker était ignoré par Git, Marie-Laure ne le recevrait pas après un pull : il faudrait bien lui fournir un point de départ.

Nous avons retenu un fichier .env.docker.example à versionner. Elle pourra le copier en .env.docker et utiliser les commentaires qui expliquent les commandes.

Une proposition avec des valeurs sensibles vides a d’abord été faite. J’ai précisé ma préférence actuelle : ce projet contient des données de démonstration et je souhaite partager les valeurs de développement pour simplifier l’installation. Le modèle peut donc contenir des valeurs de démonstration explicites ; les vrais fichiers d’environnement restent distincts.

Les exclusions proposées pour Git sont .env, .env.docker et dist/. Si une règle générale ignore .env.*, il faut autoriser explicitement .env.docker.example.

## Une documentation directement utile
J’ai demandé des commentaires en français à coller dans le fichier env Docker. Nous avons regroupé :
- contrôle de la configuration ;
- lancement des services ;
- rebuild après modification du code ;
- état des conteneurs et logs ;
- migrations et seed ;
- arrêt sans suppression et redémarrage ;
- différence entre stop et down.

J’ai ensuite vérifié que les deux cas « code modifié » et « env modifié » figuraient bien dans cette aide. Une section dédiée aux modifications d’environnement a été ajoutée à la proposition de commentaires.

Le même bloc doit figurer dans l’exemple versionné pour que Marie-Laure bénéficie aussi de cette documentation.

## Commandes de référence
Les commandes suivantes ont été proposées pendant cette séquence. Leur résultat d’exécution n’a pas été partagé ici ; elles ne constituent pas une preuve supplémentaire de validation.

Depuis le dossier backend :

### Vérifier puis appliquer une modification des variables d’exécution
```bash
docker compose --env-file .env.docker config --quiet
docker compose --env-file .env.docker up -d
```

### Après une modification du code backend
```bash
npm run typecheck
docker compose --env-file .env.docker up -d --build api-service
```

### Contrôler le résultat
```bash
docker compose --env-file .env.docker ps -a
docker compose --env-file .env.docker logs --tail=80 api-service
```

Pour suivre les logs en direct ; Ctrl+C quitte seulement le suivi :

```bash
docker compose --env-file .env.docker logs -f api-service
```

### Exécuter les tâches DB ponctuelles
```bash
docker compose --env-file .env.docker run --rm api-service npx prisma migrate deploy --config=prisma7.config.ts
docker compose --env-file .env.docker run --rm api-service npm run db:seed
```

Notre seed supprime puis recrée les données de démonstration. Une modification d’env ou du traitement d’erreurs ne nécessite pas de le relancer.

### Arrêter avant de fermer la VM, puis reprendre
```bash
docker compose --env-file .env.docker stop
docker compose --env-file .env.docker start
```

start redémarre les conteneurs existants ; il ne les crée pas et ne reconstruit pas les images.

### Supprimer les conteneurs et le réseau du projet
```bash
docker compose --env-file .env.docker down
```

Sans -v, le volume PostgreSQL nommé est conservé. down -v supprimerait aussi les volumes du projet et leurs données. Pour un simple arrêt avant de fermer la VM, utiliser stop.

### Installation à partir de l’exemple partagé
```bash
cp .env.docker.example .env.docker
```

### Commit proposé
Après préparation des fichiers et du modèle :

```bash
git add .gitignore .dockerignore .env.docker.example Dockerfile compose.yaml package.json tsconfig.json src/controllers/challenges.controller.ts src/controllers/challenges.controller.spec.test.ts src/middlewares/error.handler.ts
git diff --cached --stat
git commit -m "Configurer Docker et améliorer les réponses d'erreur du backend" -m "Ajouter les services API, PostgreSQL et Adminer, la compilation TypeScript et un exemple de configuration Docker. Traiter les titres en doublon avec une réponse 409 et les requêtes JSON mal formées avec une réponse 400. Ajouter un test de création avec un titre déjà utilisé."
git push
```

La réussite du commit et du push n’a pas été confirmée dans cette conversation.

## Préparer la suite frontend sans attendre sa finalisation
J’ai demandé s’il était possible de préparer Docker pendant que Marie-Laure code encore. Le périmètre retenu est de travailler sur la version frontend actuellement disponible, déjà reliée au backend selon mon retour, puis de vérifier sa construction et ses requêtes réelles.

La configuration Docker et la validation de la version finale du frontend restent deux étapes distinctes. Après intégration des changements de Marie-Laure, il faudra reconstruire et vérifier les parcours concernés. Le travail frontend doit continuer dans une nouvelle conversation.

## Fin de session
Aujourd’hui, je ne cherchais pas seulement une commande pour pousser : j’ai interrompu le processus pour vérifier que nos fichiers d’environnement étaient réellement séparés, utilisables par Compose et partageables avec ma coéquipière.

La documentation a été construite à partir de mes questions pratiques : comment transmettre les réglages, comment appliquer un changement et comment retrouver les commandes sans tout redemander.

L’IA a aidé à comparer et structurer les propositions. Les fichiers appliqués et les résultats de commandes restent à valider dans mon environnement ; je ne les note pas comme terminés sans confirmation.

**Fin de l’étude pour aujourd’hui.**

