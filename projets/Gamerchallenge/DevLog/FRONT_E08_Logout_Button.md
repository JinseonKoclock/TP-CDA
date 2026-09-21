# DevLog - Header et état de connexion

## Objectif

Dans le Header de GamerChallenge, je veux afficher des boutons différents selon l'état de connexion de l'utilisateur.

### Utilisateur non connecté

```text
Connexion | Inscription
```

### Utilisateur connecté

```text
Déconnexion
```

---

## Structure à retenir dans un composant React

Pour organiser le composant, je peux séparer mentalement le code en trois parties :

```tsx
const Header = () => {

  // 1. ÉTAT
  // Les informations dont le composant a besoin.
  // Exemple : savoir si l'utilisateur est connecté.


  // 2. ACTIONS / FONCTIONS
  // Ce que le composant doit pouvoir faire.
  // Exemple : déconnecter l'utilisateur.


  return (

    // 3. AFFICHAGE
    // Ce que l'utilisateur voit.
    // Exemple : afficher Connexion ou Déconnexion
    // selon l'état de connexion.

  );
};
```

### Formule mentale

```text
ÉTAT
  ↓
ACTION
  ↓
AFFICHAGE
```

Ou, dans mon cas :

```text
Est-ce que l'utilisateur est connecté ?
                ↓
        Que doit faire le bouton ?
                ↓
        Quel bouton afficher ?
```

---

## Important : le rôle du `return`

Le `return` ne sert pas à déterminer lui-même si l'utilisateur est connecté.

Il utilise une information obtenue auparavant pour décider quoi afficher.

Par exemple :

```tsx
{isLoggedIn ? (
  // Afficher Déconnexion
) : (
  // Afficher Connexion + Inscription
)}
```

C'est du **rendu conditionnel**.

Le principe est :

```text
condition ? résultat si vrai : résultat si faux
```

Dans GamerChallenge :

```text
isLoggedIn ?

OUI  → Déconnexion
NON  → Connexion + Inscription
```

---

## Mais d'où vient `isLoggedIn` ?

Je ne dois pas simplement écrire :

```tsx
const isLoggedIn = true;
```

Cela ne représenterait pas réellement l'état d'authentification de l'utilisateur.

Le backend de GamerChallenge possède déjà une route :

```text
GET /api/auth/me
```

Elle permet de vérifier l'utilisateur actuellement authentifié.

Le Header devra donc utiliser le système d'authentification existant du backend pour connaître l'état réel de connexion.

---

## Rôle de `credentials: "include"`

Lors de la connexion, le backend crée les tokens d'authentification et utilise des cookies.

Dans le `fetch` du frontend :

```tsx
credentials: "include"
```

permet au navigateur d'envoyer et de recevoir les cookies d'authentification lors des échanges avec l'API.

À ne pas confondre :

```text
Backend
   ↓
crée le JWT / gère le cookie

Frontend
   ↓
credentials: "include"
permet au navigateur de transmettre les cookies
```

`credentials: "include"` ne crée donc pas le token.

---

## Flux prévu pour le Header

```text
Chargement du Header
        ↓
GET /api/auth/me
        ↓
L'utilisateur est-il authentifié ?
        ↓
   ┌────┴────┐
   │         │
  OUI       NON
   │         │
   ↓         ↓
Déconnexion  Connexion
             Inscription
```

Puis lors du clic sur Déconnexion :

```text
Clic sur Déconnexion
        ↓
GET /api/auth/logout
        ↓
Backend supprime/invalide l'authentification
        ↓
Mise à jour de l'état du frontend
        ↓
Le Header affiche de nouveau
Connexion + Inscription
```

---

## Ce que je retiens

Quand je construis un composant React avec un comportement dynamique, je peux me poser trois questions :

```text
1. ÉTAT
De quelle information ai-je besoin ?

2. ACTION
Qu'est-ce que l'utilisateur ou le composant peut faire ?

3. AFFICHAGE
Que dois-je afficher en fonction de cet état ?
```

Pour le Header de GamerChallenge :

```text
ÉTAT
→ utilisateur connecté ou non

ACTION
→ connexion / déconnexion

AFFICHAGE
→ Connexion + Inscription
  OU
→ Déconnexion
```

Cette structure m'aide à comprendre où placer chaque partie du code au lieu d'ajouter du code directement dans le `return`.