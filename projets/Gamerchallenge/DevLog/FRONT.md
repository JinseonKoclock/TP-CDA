# DevLog — Vérification du mot de passe actuel avant modification du profil

**Date :** 1er novembre 2026  
**Projet :** GamerChallenges

## Besoin et choix de fonctionnement

J’ai proposé d’ajouter une vérification du mot de passe avant d’enregistrer les modifications du profil. J’hésitais entre demander une nouvelle connexion à l’ouverture de la page et demander le mot de passe lors de l’enregistrement.

Le choix retenu est de laisser la page accessible à l’utilisateur connecté, puis de vérifier son mot de passe actuel avant toute modification du profil, y compris un changement de pseudo. Le formulaire distingue :

- `currentPassword` : vérification de l’identité avant l’enregistrement ;
- `password` : nouveau mot de passe, facultatif ;
- `confirm` : confirmation du nouveau mot de passe.

J’ai aussi précisé que le mot de passe saisi devait être masqué à l’écran. L’input utilise `type="password"`. Ce masquage concerne l’affichage ; la protection pendant le transport relève de HTTPS.

## Modifications du back-end

Dans `src/controllers/auth.controller.ts`, la fonction `updateMe` a été complétée :

1. Ajout de `currentPassword` obligatoire au schéma Zod, avec une longueur minimale de 1 caractère.
2. Récupération de l’utilisateur en base à partir de `req.user.id`.
3. Vérification avec `await argon2.verify(user.password, currentPassword)` avant les contrôles de doublons et la mise à jour.
4. En cas d’échec, interruption par `UnauthorizedError` avec le message : `Le mot de passe actuel est incorrect.`

```ts
const isPasswordValid = await argon2.verify(
  user.password,
  currentPassword
);

if (!isPasswordValid) {
  throw new UnauthorizedError(
    "Le mot de passe actuel est incorrect."
  );
}
```

Le mot de passe actuel sert uniquement à la vérification : il n’est pas ajouté aux données à enregistrer. Le nouveau mot de passe reste facultatif et est hashé avec Argon2 lorsqu’il est fourni. La réponse publique conserve uniquement `id`, `pseudo` et `email`.

## Modifications du front-end

Dans `src/pages/ProfilePage/EditProfilePage.tsx` :

- ajout du state `currentPassword` ;
- ajout d’un champ obligatoire « Mot de passe actuel », placé avant les actions du formulaire ;
- utilisation de `type="password"`, `autoComplete="current-password"` et d’un texte d’aide ;
- ajout de `currentPassword` au corps JSON envoyé à `PATCH /api/auth/me`, indépendamment du changement éventuel de mot de passe ;
- remise à zéro du state après une modification réussie, avec les autres champs de mot de passe.

## Essais manuels, doute et débogage

J’ai volontairement saisi un mauvais mot de passe actuel et j’ai eu l’impression que la modification était tout de même acceptée. J’ai ensuite signalé un changement de mot de passe dont je ne connaissais plus la valeur, ce qui a interrompu le déroulement des essais.

Le code du contrôleur partagé comportait pourtant la vérification attendue, avec `await` et un arrêt avant la mise à jour en cas d’échec. Il fallait donc distinguer le code affiché, le code réellement exécuté et les requêtes observées.

Dans l’onglet Réseau, les informations que je consultais concernaient des requêtes `GET /api/auth/me`, avec notamment un statut 304, et non le PATCH d’enregistrement. La capture montrait bien un GET sélectionné. L’absence de corps dans cette requête ne renseignait donc pas sur l’envoi de `currentPassword`.

La commande de développement affichée était :

```text
tsx src/index.ts
```

Cette commande n’active pas la surveillance des fichiers. Un redémarrage a été proposé pour appliquer les changements, sans que cela permette d’établir à lui seul la cause de l’incident.

Une récupération par remplacement du hash en base a été envisagée. Les captures montraient deux comptes différents : Jinseon (id 8), puis SpeedPaw (id 1), qui était le compte concerné par les essais. Aucun remplacement du hash n’a été confirmé dans cet échange. J’ai ensuite proposé de créer un utilisateur de test et exprimé le besoin de faire une pause.

Après de nouveaux essais de connexion, j’ai indiqué que cela fonctionnait et qu’il pouvait y avoir eu une confusion de ma part. **La cause exacte des observations initiales n’a pas été démontrée : elle ne peut pas être attribuée avec certitude à Firefox, au serveur non redémarré ou à une erreur de saisie.**

## Correction Swagger effectuée pendant la séance

Dans la documentation de `POST /auth/login`, le `description` de la réponse `'500'` était mal indenté. Il a été replacé sous cette réponse :

```yaml
      '500':
        description: Erreur interne du serveur
```

J’ai confirmé que Swagger fonctionnait à nouveau après la correction. Cette erreur de documentation est distincte de la vérification du mot de passe actuel.

## Tests d’intégration

Pour sortir de l’incertitude des essais manuels, j’ai demandé un fichier de tests consacré à `currentPassword`.

Le fichier `src/controllers/auth-current-password.spec.test.ts` a été rédigé avec l’aide de l’IA à partir de la structure existante : `node:test`, `assert`, Prisma, Argon2, `setupIntegrationTests` et `createAuthenticatedUser`.

Il crée des utilisateurs de test avec un mot de passe connu. Il réutilise la vérification qui impose la base `gamerchallenge_test`, ainsi que le nettoyage et le démarrage du serveur prévus par la configuration existante.

| Scénario | Vérifications |
| --- | --- |
| Mot de passe actuel incorrect | Réponse 401, message attendu, pseudo/email/hash inchangés ; ancien mot de passe encore valide |
| Mot de passe actuel absent | Erreur de validation, aucune modification du profil |
| Mot de passe actuel vide | Erreur de validation, aucune modification du profil |
| Bon mot de passe, modification du profil | Réponse 200, pseudo/email enregistrés, hash inchangé, aucun mot de passe dans la réponse |
| Bon mot de passe, changement de mot de passe | Nouveau hash valide ; connexion refusée avec l’ancien mot de passe et acceptée avec le nouveau |
| Mot de passe d’un autre compte | Réponse 401, aucun des deux comptes modifié |

Pour les champs absents ou vides, le test accepte 400 ou 422 : le gestionnaire du ZIP de référence renvoyait 400, tandis que des réponses de validation plus récentes observées sur le projet utilisaient 422.

Commande utilisée pour lancer ce fichier seul :

```bash
node --env-file=src/tests/config/.env.test --import tsx --test src/controllers/auth-current-password.spec.test.ts
```

**Résultat : j’ai confirmé que les six tests étaient passés.** L’exécution a été réalisée dans mon environnement local ; l’IA n’a pas exécuté les tests contre ma base.

Ces tests vérifient le comportement de l’API et les données en base. Ils ne reproduisent pas l’autocomplétion du navigateur ni les interactions du formulaire React.

## Ce que cette démarche a permis de préciser

- Le champ masqué et obligatoire dans le front doit être complété par une vérification réelle dans le back-end.
- Pour vérifier qu’un refus protège les données, le statut HTTP seul ne suffit pas : les tests comparent aussi les valeurs en base avant et après la requête.
- Les essais automatisés avec des utilisateurs isolés et des mots de passe connus permettent de vérifier le comportement sans dépendre des valeurs mémorisées pendant plusieurs essais manuels.

Ces éléments décrivent les vérifications et les précisions de cette séance, sans présenter tous les mécanismes abordés comme nouvellement appris ou entièrement maîtrisés.

## Sauvegarde Git

Après la réussite des tests, j’ai demandé à préparer le commit et le push des changements du back-end et du front-end. Leur réussite reste à confirmer par le résultat des commandes Git.

---

Ce document a été rédigé avec une démarche d’apprentissage actif, en utilisant ChatGPT comme support de réflexion et de structuration.

Mes questions, mes essais, mes hésitations et les résultats partagés constituent la base de ce carnet. L’IA aide à structurer et à préciser la formulation ; la relecture et la validation finale restent de ma responsabilité.
