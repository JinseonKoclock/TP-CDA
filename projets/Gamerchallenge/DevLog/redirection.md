# DevLog — Redirection après connexion

**Date : 22/09/2026**  
**Projet : GamerChallenges**  
**Branche : `feat/login`**

---

## Objectif

Après une connexion réussie, rediriger automatiquement l'utilisateur vers la page d'accueil.

Le comportement souhaité est :

```text
Connexion réussie
↓
mise à jour de l'utilisateur
↓
redirection vers "/"
↓
affichage de la page d'accueil
```

Il fallait cependant éviter de rediriger l'utilisateur lorsque la connexion échoue.

---

## 1. Problème avec la fonction `login()`

Dans `AuthContext`, la fonction `login()` était définie avec :

    login: (email: string, password: string) => Promise<void>;

`Promise<void>` signifie que la fonction effectue une opération asynchrone mais ne retourne pas de résultat utilisable par `LoginPage`.

`LoginPage` ne pouvait donc pas savoir si la connexion avait réussi ou échoué.

Faire directement :

    await login(email, password);
    navigate("/");

aurait posé un problème :

```text
connexion réussie → redirection
connexion échouée → redirection également ❌
```

---

## 2. Retourner un boolean

J'ai modifié le type de `login()` :

    login: (email: string, password: string) => Promise<boolean>;

La fonction peut maintenant indiquer son résultat :

```text
true
→ connexion réussie

false
→ connexion échouée
```

Dans `login()` :

    if (responseLogin.ok) {
        const data = await responseLogin.json();
        setUser(data);

        return true;
    } else {
        console.error("Erreur lors de la connexion.");

        return false;
    }

J'ai compris que la définition dans `AuthContextType` représente le contrat de la fonction.

Si j'écris :

    Promise<boolean>

la fonction doit réellement retourner un boolean.

---

## 3. Utilisation de `useNavigate`

Dans `LoginPage`, j'ai importé `useNavigate` depuis React Router.

    import { Link, useNavigate } from "react-router";

Puis :

    // Permet de naviguer vers une autre page depuis le code React.
    const navigate = useNavigate();

`navigate()` permet de déclencher une navigation depuis le code React.

Par exemple :

    navigate("/");

redirige vers la page d'accueil.

---

## 4. Vérification du résultat avant la redirection

Dans `handleSubmit`, je récupère maintenant le résultat de `login()` :

    // Attend le résultat de la connexion.
    // `success` vaut true si la connexion a réussi, sinon false.
    const success = await login(email, password);

Puis je vérifie ce résultat :

    // Redirige vers la page d'accueil uniquement si la connexion a réussi.
    if (success) {
        // Réinitialise les champs du formulaire.
        form.reset();

        // Redirige l'utilisateur vers la page d'accueil.
        navigate("/");
    }

Le fonctionnement final est donc :

```text
LoginPage
↓
login(email, password)
↓
AuthContext
↓
requête POST /api/auth/login
↓
responseLogin.ok ?
│
├── OUI
│   ↓
│   setUser(data)
│   ↓
│   return true
│   ↓
│   success = true
│   ↓
│   navigate("/")
│   ↓
│   page d'accueil
│
└── NON
    ↓
    return false
    ↓
    success = false
    ↓
    pas de redirection
```

---

## Ce que j'ai compris

- `Promise<void>` signifie que je n'attends aucune valeur de retour utilisable.
- `Promise<boolean>` permet à une fonction asynchrone de retourner `true` ou `false`.
- Modifier le type d'une fonction ne suffit pas : son implémentation doit respecter ce type.
- `LoginPage` ne doit pas décider elle-même si l'API a accepté la connexion.
- `AuthContext.login()` effectue la connexion et retourne son résultat.
- `LoginPage` utilise ensuite ce résultat pour décider s'il faut naviguer.
- `useNavigate()` permet de changer de route depuis le code React.
- La redirection doit être effectuée uniquement après une connexion réussie.

---

## Formule à retenir

```text
ACTION
login()

↓ retourne un résultat

RÉSULTAT
true / false

↓ permet une décision

NAVIGATION
true → navigate("/")
false → rester sur LoginPage
```

---

## Git

Les modifications ont été enregistrées sur la branche :

    feat/login

Les fichiers concernés par cette évolution sont principalement :

    src/context/AuthContext.tsx
    src/pages/LoginPage/LoginPage.tsx

La fonctionnalité a été testée : après une connexion réussie, l'utilisateur arrive automatiquement sur la page d'accueil.