# SC03 - Authentification et sécurité Web

> 15/07/2026

---

## 🎯 Objectif

Comprendre les principaux mécanismes d'authentification modernes ainsi que les notions de sécurité associées.

L'objectif n'était pas seulement de savoir utiliser une bibliothèque, mais de comprendre pourquoi ces mécanismes existent et quels problèmes ils résolvent.

---

## 🔑 Mots-clés étudiés

| Acronyme | Nom complet | Signification |
|-----------|-------------|---------------|
| HTTP | HyperText Transfer Protocol | Protocole de communication entre un navigateur et un serveur. |
| HTTPS | HyperText Transfer Protocol Secure | Version sécurisée de HTTP utilisant TLS. |
| TLS | Transport Layer Security | Protocole qui chiffre les communications entre le client et le serveur. |
| JWT | JSON Web Token | Jeton utilisé pour authentifier un utilisateur. |
| API | Application Programming Interface | Interface permettant à deux applications de communiquer. |
| REST | Representational State Transfer | Style d'architecture utilisé pour concevoir des API Web. |
| CRUD | Create, Read, Update, Delete | Les quatre opérations principales sur les données. |
| SPA | Single Page Application | Application Web fonctionnant sur une seule page HTML. |
| SaaS | Software as a Service | Logiciel accessible directement via Internet sans installation locale. |
| OAuth | Open Authorization | Protocole permettant de déléguer l'autorisation et servant de base à l'authentification via des fournisseurs comme Google ou GitHub. |
| OIDC | OpenID Connect | Couche d'authentification construite au-dessus d'OAuth 2.0. |
| XSS | Cross-Site Scripting | Attaque consistant à injecter du JavaScript malveillant dans une page Web. |
| CSRF | Cross-Site Request Forgery | Attaque qui force un utilisateur authentifié à exécuter une action à son insu. |
| SQL | Structured Query Language | Langage permettant de manipuler une base de données relationnelle. |
| ORM | Object-Relational Mapping | Technique permettant de manipuler une base de données avec des objets. |
| CDN | Content Delivery Network | Réseau de serveurs distribuant les ressources statiques au plus près des utilisateurs. |
| DNS | Domain Name System | Système qui traduit un nom de domaine en adresse IP. |
| CORS | Cross-Origin Resource Sharing | Mécanisme contrôlant les requêtes entre différentes origines. |
| CPU | Central Processing Unit | Processeur chargé d'exécuter les instructions d'un programme. |
| CSS | Cascading Style Sheets | Langage de mise en forme des pages Web. |
| HTML | HyperText Markup Language | Langage de structure des pages Web. |
| JSON | JavaScript Object Notation | Format léger d'échange de données entre applications. |
| DOM | Document Object Model | Représentation d'une page HTML manipulable en JavaScript. |
| MVC | Model-View-Controller | Architecture séparant les données, l'affichage et la logique métier. |
| POO | Programmation Orientée Objet (Object-Oriented Programming) | Paradigme de programmation basé sur les objets et les classes. |
| UI | User Interface | Interface utilisateur visible et manipulée par l'utilisateur. |
| UX | User Experience | Expérience globale ressentie par l'utilisateur lors de l'utilisation d'une application. |
| DB | Database | Base de données. |
| RDBMS | Relational Database Management System | Système de gestion de bases de données relationnelles (PostgreSQL, MySQL...). |
| SQLi | SQL Injection | Attaque consistant à injecter du code SQL malveillant dans une requête. |
| MFA | Multi-Factor Authentication | Authentification nécessitant plusieurs facteurs (mot de passe + code, par exemple). |
| SSO | Single Sign-On | Authentification unique permettant d'accéder à plusieurs services avec une seule connexion. |
| CLI | Command Line Interface | Interface en ligne de commande (Terminal). |
| GUI | Graphical User Interface | Interface graphique permettant d'interagir avec une application. |
| IDE | Integrated Development Environment | Environnement de développement intégré (VS Code, IntelliJ...). |
| CI | Continuous Integration | Intégration continue : automatisation des tests et de la validation du code. |
| CD | Continuous Deployment / Continuous Delivery | Déploiement continu ou livraison continue selon le contexte. |
| YAML | YAML Ain't Markup Language | Format de fichier utilisé notamment pour GitHub Actions et Docker Compose. |
| SSH | Secure Shell | Protocole permettant de se connecter à un serveur de manière sécurisée. |


---

## HTTP vs HTTPS

### HTTP

Les données circulent sans chiffrement.

Un attaquant présent sur le réseau peut potentiellement intercepter les informations transmises.

### HTTPS

Les échanges entre le navigateur et le serveur sont chiffrés grâce à TLS.

Cela protège les identifiants, mots de passe, cookies et toutes les données échangées.

⚠️ HTTPS protège les données **pendant leur transport**, mais ne remplace pas le hash des mots de passe dans la base de données.

---

## Stateful vs Stateless

### Stateful

Le serveur mémorise l'utilisateur grâce à une session.

Le navigateur envoie uniquement un identifiant de session (cookie).

Le serveur retrouve ensuite les informations associées.

```
Navigateur
    ↓
Cookie (Session ID)
    ↓
Serveur
    ↓
Session utilisateur
```

---

### Stateless

Le serveur ne mémorise aucune session.

Chaque requête contient un JWT permettant d'identifier l'utilisateur.

Le serveur vérifie simplement la validité du token.

```
Navigateur
    ↓
JWT
    ↓
Serveur
    ↓
Vérification
```

---

## OAuth2 / OpenID Connect

Authentification réalisée par un fournisseur externe.

Exemples :

- Google
- GitHub
- Discord
- Facebook

Le site ne connaît jamais le mot de passe Google de l'utilisateur.

Il fait confiance au fournisseur d'identité.

---

## SPA (Single Page Application)

Une SPA charge une seule page HTML.

Les changements d'écran sont réalisés en JavaScript sans recharger toute la page.

Les données sont récupérées via des appels API (JSON).

Frameworks courants :

- React
- Vue
- Svelte
- Angular

---

## SaaS (Software as a Service)

Logiciel accessible directement via Internet sans installation locale.

Exemples :

- ChatGPT
- Gmail
- Notion
- Canva
- GitHub

---

# 🔒 Cookies HttpOnly

Un cookie HttpOnly n'est pas accessible via JavaScript.

Cela empêche un script malveillant de récupérer directement le JWT.

Le navigateur continue cependant d'envoyer automatiquement le cookie au serveur.

---

# XSS (Cross Site Scripting)

Injection de JavaScript malveillant dans une page.

Exemple dangereux :

```javascript
element.innerHTML = userInput;
```

Si l'utilisateur saisit :

```html
<script>alert("XSS")</script>
```

Le navigateur exécute le script.

Méthode plus sûre :

```javascript
element.textContent = userInput;
```

Le contenu est affiché comme du texte sans être exécuté.

---

# LocalStorage vs HttpOnly

## LocalStorage

Accessible depuis JavaScript.

En cas de XSS :

```javascript
localStorage.getItem("access_token")
```

Le token peut être volé.

---

## HttpOnly

JavaScript ne peut pas accéder au cookie.

Le navigateur continue de l'envoyer automatiquement lors des requêtes.

Réduit fortement le risque de vol de JWT via XSS.

---

# Zod

## safeParse()

Validation synchrone.

Utilisé pour vérifier :

- email
- longueur
- type
- format

---

## safeParseAsync()

Validation asynchrone.

Permet d'effectuer des vérifications nécessitant :

- une requête Prisma
- une base de données
- une API externe

---

# pgAdmin

Outil graphique d'administration PostgreSQL.

Permet notamment :

- visualiser les tables
- consulter les données
- exécuter du SQL
- vérifier les insertions réalisées par Prisma

---

# Git Stash

Commande Git permettant de sauvegarder temporairement des modifications non terminées.

Commandes principales :

```bash
git stash
git stash list
git stash pop
git stash apply
```

Très utile lorsqu'il faut changer rapidement de branche sans effectuer un commit incomplet.

---

# throw new Error()

Permet de créer puis lancer une erreur.

Exemple :

```typescript
throw new Error("User not found");
```

L'exécution s'arrête immédiatement.

L'erreur peut ensuite être interceptée dans un bloc `try/catch` ou par un middleware Express.

---

# Ce que j'ai retenu

Cette séance m'a permis de comprendre que la sécurité d'une application ne repose pas sur une seule technologie.

Chaque mécanisme répond à un problème précis :

- HTTPS protège les données pendant leur transport.
- Argon2 protège les mots de passe enregistrés en base.
- HttpOnly protège les cookies contre JavaScript.
- `textContent` limite les risques de XSS.
- OAuth délègue l'authentification à un fournisseur de confiance.
- JWT permet une authentification Stateless.

Ces notions fonctionnent ensemble pour construire un système d'authentification moderne et sécurisé.