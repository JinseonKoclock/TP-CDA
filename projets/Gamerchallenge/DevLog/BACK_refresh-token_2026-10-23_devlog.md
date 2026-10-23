# DevLog — Reprise de l’authentification et gestion du refresh token

**Date : 23/10/2026**  
**Projet : GamerChallenges — Backend**

## Contexte

Après environ trois semaines sans travailler sur GamerChallenges, j’ai repris le backend. Le projet ne démarrait plus et la commande `npm run typecheck` a signalé 16 erreurs dans `auth.controller.ts`.

Ces erreurs ne correspondaient pas à 16 problèmes indépendants. Elles provenaient surtout d’un décalage entre le contrôleur d’authentification et le schéma Prisma actuel.

## Retour sur la conception

Lors de la conception initiale du MCD, je n’avais pas prévu la gestion des refresh tokens.

En étudiant ensuite le code d’authentification du formateur, j’ai compris le rôle du refresh token : il permet de renouveler un access token expiré sans demander à l’utilisateur de se reconnecter immédiatement. J’ai donc ajouté a posteriori une entité `RefreshToken`, distincte de `User`, avec une relation entre les deux.

Cette modification a été faite tardivement et le contrôleur est resté partiellement basé sur l’ancienne structure du code du formateur. C’est la raison principale des erreurs TypeScript retrouvées aujourd’hui.

## Problème identifié

Le contrôleur utilise encore certains anciens noms de champs :

- `token` au lieu de `tokenHash` ;
- `expires_at` au lieu de `expiresAt` ;
- `user_id` au lieu de `userId` ;
- `username` au lieu de `pseudo`.

D’autres erreurs concernent le typage de `req.user` et des propriétés qui n’existent pas dans le modèle `User`, comme `role` et `updatedAt`.

## Ce que j’ai compris aujourd’hui

Le schéma Prisma prévoit volontairement de ne jamais enregistrer le refresh token original dans la base :

- le navigateur conserve le token original dans un cookie HttpOnly ;
- la base de données conserve uniquement son empreinte dans `tokenHash` ;
- lors d’un renouvellement, le token reçu doit être transformé avec la même fonction de hachage avant la recherche en base.

Le module Node.js `crypto` sert ici à deux opérations différentes :

- produire un refresh token aléatoire avec `randomBytes()` ;
- produire son empreinte avec SHA-256.

Je connaissais déjà le principe du hachage des mots de passe avec Argon2. La nouveauté est d’avoir compris pourquoi le refresh token original et son hash doivent également être séparés.

## Décision prise

Je conserve la structure actuelle avec `tokenHash`, car elle évite de stocker le refresh token original en base de données.

Cette amélioration de sécurité n’avait pas été planifiée dès le MCD. Elle est apparue en ajoutant tardivement la gestion des refresh tokens, puis en analysant l’incompatibilité entre le contrôleur et le schéma Prisma.

## État actuel et prochaines étapes

La correction n’est pas encore terminée. Les prochaines étapes sont :

1. ajouter une fonction `hashRefreshToken()` dans `src/lib/token.ts` ;
2. l’utiliser lors de l’enregistrement et de la recherche du refresh token ;
3. adapter le contrôleur aux champs `tokenHash`, `expiresAt` et `userId` ;
4. déclarer correctement le type de `req.user` ;
5. remplacer `username` par `pseudo` et retirer les propriétés absentes du modèle ;
6. vérifier que les réponses de l’API ne renvoient jamais le hash du mot de passe ;
7. relancer `npm run typecheck`, puis tester les routes d’authentification.

