# SC03E01 - Authentification JWT : Première tentative

> 15/07/2026

---

## 🎯 Objectif

Implémenter la route `GET /auth/me` permettant de récupérer les informations de l'utilisateur authentifié à partir d'un JWT, puis commencer l'écriture des tests automatisés associés.

---

## 🛠 Technologies

- TypeScript
- Express
- Prisma ORM
- PostgreSQL
- JWT (jsonwebtoken)
- Argon2
- Node Test
- Fetch API

---

## 📌 Ce que j'ai réalisé

- Implémentation de la route `GET /auth/me`.
- Vérification de la présence du header `Authorization`.
- Extraction de l'access token.
- Vérification du JWT avec `jsonwebtoken`.
- Récupération du `userId` depuis le payload du token.
- Recherche de l'utilisateur avec Prisma.
- Exclusion du mot de passe de la réponse grâce à `omit`.
- Début de l'écriture des tests automatisés.

---

## Difficultés rencontrées

J'ai réussi à implémenter la route `GET /auth/me`, mais j'ai rencontré des difficultés lors de l'écriture du test automatisé.

Au début, je pensais devoir appeler la route `/auth/login` afin de récupérer un JWT avant de tester `/auth/me`.

En analysant le fonctionnement de mon projet, j'ai compris que le token était stocké dans un cookie HTTP Only lors du login, alors que ma route `/auth/me` attendait un token envoyé dans le header `Authorization`.

Cette différence m'a obligé à réfléchir à une autre stratégie de test.

J'ai alors envisagé de créer directement un utilisateur dans la base de données, de générer un JWT avec `generateAuthenticationTokens()` puis de l'envoyer dans le header `Authorization`.

Je n'ai cependant pas réussi à terminer complètement ce test avant la fin de la séance.

---

## Ce que j'ai appris

Aujourd'hui, j'ai surtout compris qu'écrire un test ne consiste pas uniquement à écrire du code.

Il faut d'abord comprendre précisément comment circule l'authentification dans l'application (headers, cookies, JWT, base de données...) avant de pouvoir reproduire ce comportement dans un test.