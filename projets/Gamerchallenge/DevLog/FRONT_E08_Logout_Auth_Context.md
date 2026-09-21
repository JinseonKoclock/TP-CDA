# DevLog — Authentification Front / Logout / React Context

**Date : 21/09/2026**  
**Projet : GamerChallenges**  
**Travail en cours : Déconnexion et gestion de l'état d'authentification**

---

## 1. Objectif de la séance

Continuer la partie authentification :

- Connexion : terminée
- Inscription : terminée
- Déconnexion : en cours

Objectifs :

1. connecter le bouton `Déconnexion` à l'API ;
2. vérifier le fonctionnement du logout ;
3. savoir si un utilisateur est connecté ;
4. afficher les bons boutons dans le Header ;
5. comprendre pourquoi l'état d'authentification doit être partagé entre plusieurs composants React.

---

# 2. Vérification du logout

Le backend possède la route :

    GET /api/auth/logout

Cette route utilise le middleware `verifyAuth`.

Flux :

    GET /api/auth/logout
            ↓
        verifyAuth
            ↓
    lecture du cookie accessToken
            ↓
    vérification du JWT
            ↓
        req.user
            ↓
        logoutUser()
            ↓
    suppression du refresh token
            ↓
    suppression des cookies

Dans `Header.tsx`, j'ai ajouté :

    const handleLogout = async () => {
      const logoutResponse = await fetch(
        `${import.meta.env.VITE_API_URL}/api/auth/logout`,
        {
          method: "GET",

          // Envoie les cookies d'authentification avec la requête.
          credentials: "include",
        }
      );
    };

Puis un bouton temporaire :

    <button
      className="site-header__login"
      type="button"
      onClick={handleLogout}
    >
      Déconnexion
    </button>

---

# 3. Résultat du test du logout

Les logs du backend ont montré :

    POST /auth/login  → 200
    GET /auth/logout → 200

Le logout fonctionne donc correctement.

J'ai ensuite obtenu :

    UnauthorizedError: Vous n'êtes pas connecté.

Au début, j'ai pensé qu'il y avait un problème avec le logout.

En réalité, j'avais simplement cliqué plusieurs fois sur le bouton `Déconnexion`.

Après le premier clic :

    accessToken existe
            ↓
    GET /auth/logout
            ↓
    200
            ↓
    cookies supprimés

Au clic suivant :

    accessToken n'existe plus
            ↓
    verifyAuth
            ↓
    401 Unauthorized

Ce comportement est donc normal.

---

# 4. Modèle mental utilisé pour le Header

Pour comprendre l'organisation du composant React :

    ÉTAT
      ↓
    ACTION
      ↓
    AFFICHAGE

Dans le Header :

    ÉTAT
    utilisateur connecté ou non
            ↓
    ACTION
    login / logout
            ↓
    AFFICHAGE
    Connexion + Inscription
    OU
    Déconnexion

---

# 5. `user` plutôt que `isLoggedIn`

Au départ, j'avais pensé utiliser :

    const [isLoggedIn, setIsLoggedIn] = useState(false);

Cela semble logique pour faire :

    isLoggedIn
        ?
    Déconnexion
        :
    Connexion + Inscription

Mais nous avons également besoin des informations de l'utilisateur connecté.

Nous avons donc choisi un state `user`.

Type utilisé :

    type User = {
      id: number;
      email: string;
      username: string;
      role: string;
      avatar: string | null;
    };

State :

    const [user, setUser] = useState<User | null>(null);

Le même state permet de savoir si quelqu'un est connecté :

    user === null
    → utilisateur non connecté

    user !== null
    → utilisateur connecté

Il n'est donc pas nécessaire d'avoir en même temps :

    user
    +
    isLoggedIn

Sinon, les deux states pourraient se contredire.

Exemple :

    user = null
    isLoggedIn = true

Dans ce cas, quelle information représente réellement l'état de l'application ?

Avec seulement `user` :

    user
     ├── null
     │    → non connecté
     │
     └── User
          → connecté

`user` devient donc la seule source de vérité pour l'état d'authentification côté React.

Notion importante :

    SINGLE SOURCE OF TRUTH

---

# 6. Vérification de l'utilisateur avec `/auth/me`

Dans `Header.tsx`, j'ai ajouté `useEffect`.

Import :

    import { useEffect, useState } from "react";

Puis :

    useEffect(() => {
      const checkAuth = async () => {
        const response = await fetch(
          `${import.meta.env.VITE_API_URL}/api/auth/me`,
          {
            credentials: "include",
          }
        );

        if (response.ok) {
          const data = await response.json();
          setUser(data);
        } else {
          setUser(null);
        }
      };

      checkAuth();
    }, []);

Fonctionnement :

    Header chargé
         ↓
    useEffect
         ↓
    GET /api/auth/me
         ↓
    envoi du cookie accessToken
         ↓

    si 200
         ↓
    setUser(data)

    si 401
         ↓
    setUser(null)

Le tableau vide :

    []

indique ici que l'effet est exécuté lors du montage du composant et n'est pas relancé à chaque rendu.

---

# 7. Affichage conditionnel du Header

Objectif :

    user existe
        ↓
    Déconnexion

    user est null
        ↓
    Connexion + Inscription

Code :

    {user ? (
      <button
        className="site-header__login"
        type="button"
        onClick={handleLogout}
      >
        Déconnexion
      </button>
    ) : (
      <>
        <Link
          className="site-header__login"
          to="/login"
          onClick={closeMenu}
        >
          Connexion
        </Link>

        <Link
          className="site-header__register"
          to="/register"
          onClick={closeMenu}
        >
          Inscription
        </Link>
      </>
    )}

---

# 8. Fragment React

J'avais d'abord écrit deux `<Link>` directement dans la deuxième partie du ternaire.

Cela provoquait des erreurs JSX.

Les deux éléments doivent être regroupés :

    <>
      <Link>...</Link>
      <Link>...</Link>
    </>

`<>...</>` est un Fragment React.

Il permet de regrouper plusieurs éléments JSX sans ajouter un élément HTML supplémentaire dans le DOM.

---

# 9. Problème découvert pendant le test

Après avoir effectué la connexion :

    Connexion réussie !

mais le Header continuait d'afficher :

    Connexion
    Inscription

Le bouton `Déconnexion` n'apparaissait pas.

Par contre, après avoir appuyé sur :

    F5

le bouton `Déconnexion` apparaissait.

C'est ce comportement qui a permis d'identifier le vrai problème.

---

# 10. Pourquoi le bouton n'apparaissait pas immédiatement

Au premier chargement :

    Header créé
        ↓
    user = null
        ↓
    useEffect
        ↓
    GET /auth/me
        ↓
    utilisateur pas encore connecté
        ↓
    401
        ↓
    user = null

Ensuite je me connecte :

    LoginPage
        ↓
    POST /api/auth/login
        ↓
    connexion réussie
        ↓
    backend crée les tokens
        ↓
    navigateur reçoit les cookies HttpOnly

Le login fonctionne donc bien.

Mais le Header possède toujours :

    user = null

Le backend a créé le cookie, mais React ne modifie pas automatiquement son state `user`.

---

# 11. Cookie et state React : deux rôles différents

C'est un point important à retenir.

Le cookie existe déjà correctement dans le navigateur.

Le problème n'est PAS :

    "Comment transmettre le cookie au Header ?"

Le navigateur gère déjà le cookie.

Le problème est :

    "Comment informer React que l'utilisateur vient de se connecter ?"

Rôle du cookie :

    Cookie accessToken
            ↓
    authentifier les requêtes HTTP
            ↓
    backend / verifyAuth

Rôle du state React :

    user
      ↓
    déterminer l'état de l'interface
      ↓
    afficher les bons composants

Donc :

    COOKIE
    → authentification navigateur/backend

    USER STATE
    → état de l'interface React

---

# 12. Pourquoi F5 faisait apparaître Déconnexion

Lorsque je fais F5 :

    F5
     ↓
    application React rechargée
     ↓
    Header recréé
     ↓
    useEffect exécuté
     ↓
    GET /api/auth/me

Mais cette fois le navigateur possède déjà :

    accessToken

Donc :

    /auth/me
        ↓
    verifyAuth
        ↓
    JWT valide
        ↓
    utilisateur récupéré
        ↓
    réponse 200
        ↓
    setUser(data)
        ↓
    user !== null
        ↓
    Déconnexion

Cela confirme que :

- le login fonctionne ;
- le cookie fonctionne ;
- `/auth/me` fonctionne ;
- le Header sait afficher `Déconnexion` quand `user` existe.

Le problème restant est la mise à jour du state React immédiatement après le login.

---

# 13. Le vrai problème React

Actuellement :

    LoginPage
    └── effectue le login

et :

    Header
    └── possède user / setUser

Le problème :

    LoginPage

ne peut pas directement utiliser :

    setUser

qui appartient au Header.

Schéma actuel :

    Header
      └── user / setUser

    LoginPage
      └── login

Il manque un endroit commun pour partager l'état d'authentification.

---

# 14. Pourquoi React Context devient nécessaire

Nous avons besoin d'une structure commune :

                    AuthContext
                    user / setUser
                       /     \
                      /       \
                     ↓         ↓
                LoginPage    Header

Ainsi :

    LoginPage
        ↓
    login réussi
        ↓
    modification de user
        ↓
    AuthContext
        ↓
    Header reçoit le nouvel état
        ↓
    Déconnexion apparaît

Et pour le logout :

    Header
        ↓
    logout réussi
        ↓
    user = null
        ↓
    AuthContext
        ↓
    Header se met à jour
        ↓
    Connexion + Inscription apparaissent

Objectif final :

    AUCUN F5 NÉCESSAIRE

---

# 15. Projet O'clock de référence

Projet à utiliser demain :

    /var/www/html/oClock/SC07/E03-Orecipes-context-largenty

Fichier particulièrement important :

    frontend/src/context/AuthContext.tsx

Commande :

    cd /var/www/html/oClock/SC07/E03-Orecipes-context-largenty

    cat frontend/src/context/AuthContext.tsx

IMPORTANT :

`cat` sert ici à afficher le contenu du fichier.

Ne pas taper directement :

    ./frontend/src/context/AuthContext.tsx

car Bash essaierait d'exécuter le fichier.

---

# 16. Structure du AuthContext du cours

Le fichier du formateur est organisé en trois grandes parties.

## 1 — createContext

Commentaire du cours :

    // 1. on définit un contexte

Code :

    export const AuthContext =
      createContext<AuthContextType>(undefined);

Idée :

    createContext
        ↓
    crée la "boîte" commune

---

## 2 — AuthProvider

Commentaire du cours :

    // 2. on crée un provider pour diffuser la valeur

Le Provider contient les states et fonctions qui doivent être accessibles aux composants.

Idée :

    AuthProvider
        ↓
    contient les informations
        ↓
    les diffuse aux composants enfants

---

## 3 — useAuth

Commentaire du cours :

    // 3. on crée un hook personnalisé pour utiliser le contexte

Le hook permet aux composants d'accéder plus facilement au contexte.

Idée :

    useAuth()
        ↓
    accéder facilement à AuthContext

---

# 17. Modèle mental du Context

À retenir :

    createContext
        ↓
    crée la boîte

    Provider
        ↓
    remplit et partage la boîte

    useAuth
        ↓
    permet aux composants d'utiliser la boîte

Dans GamerChallenges :

                    AuthContext
                        │
                       user
                      /    \
                     /      \
                    ↓        ↓
              LoginPage    Header

---

# 18. Attention : ne pas copier le AuthContext du cours tel quel

Le projet Orecipes utilise notamment :

    isLoggedIn
    token
    pseudo
    error

Le fonctionnement du token n'est pas identique à GamerChallenges.

Dans le code du cours :

    login
      ↓
    réponse API
      ↓
    récupération du token
      ↓
    token stocké dans le state React

Dans GamerChallenges :

    login
      ↓
    backend génère JWT
      ↓
    backend envoie Set-Cookie
      ↓
    navigateur stocke accessToken HttpOnly

Donc le frontend GamerChallenges n'a pas besoin de gérer directement le JWT.

Notre AuthContext devrait plutôt tendre vers quelque chose comme :

    AuthContext
    │
    ├── user
    ├── login()
    └── logout()

Le projet Orecipes doit servir de modèle pour comprendre React Context, pas de code à copier aveuglément.

---

# 19. À FAIRE DEMAIN

## Étape 1 — Vérifier Git avant de coder

Se placer dans le frontend :

    cd /var/www/html/gamerChallenge/front

Puis :

    git branch
    git status

Vérifier que le travail est bien sur :

    feat/logout

Ne rien modifier avant cette vérification.

---

## Étape 2 — Relire le Context du cours

Commande :

    cat /var/www/html/oClock/SC07/E03-Orecipes-context-largenty/frontend/src/context/AuthContext.tsx

Identifier :

    1. createContext
    2. AuthProvider
    3. useAuth

Ne pas chercher à tout mémoriser.

Objectif :

    comprendre le rôle de chaque partie.

---

## Étape 3 — Vérifier si GamerChallenges possède déjà un dossier context

Dans :

    /var/www/html/gamerChallenge/front

faire :

    ls src

Si le dossier `context` n'existe pas :

    mkdir src/context

Puis créer :

    touch src/context/AuthContext.tsx

---

## Étape 4 — Construire AuthContext progressivement

Ne PAS copier tout le fichier du formateur.

Ordre prévu :

    1. type User

    2. type AuthContextType

    3. createContext

    4. AuthProvider

    5. state user

    6. vérification de l'utilisateur avec /auth/me

    7. gestion du login

    8. gestion du logout

    9. value du Provider

    10. hook useAuth

Méthode :

    comprendre
        ↓
    coder
        ↓
    vérifier TypeScript
        ↓
    tester

UNE ÉTAPE À LA FOIS.

---

## Étape 5 — Déplacer le state `user`

Actuellement :

    Header
      └── user

Objectif :

    AuthContext
      └── user
           ├── Header
           └── LoginPage

Le Header ne doit plus être le seul propriétaire de l'état d'authentification.

---

## Étape 6 — Connecter LoginPage au Context

Résultat attendu :

    POST /api/auth/login
            ↓
          200
            ↓
    user mis à jour
            ↓
    Header rerender
            ↓
    Déconnexion

Sans :

    F5

---

## Étape 7 — Connecter le logout au Context

Résultat attendu :

    clic Déconnexion
            ↓
    GET /api/auth/logout
            ↓
          200
            ↓
      user = null
            ↓
    Header rerender
            ↓
    Connexion + Inscription

Sans :

    F5

---

# 20. Tests à effectuer une fois le Context terminé

## Test 1

Ouvrir l'application sans être connecté.

Résultat attendu :

    Connexion
    Inscription

---

## Test 2

Se connecter.

Résultat attendu immédiatement :

    Déconnexion

Pas besoin de F5.

---

## Test 3

Faire F5 après connexion.

Résultat attendu :

    Déconnexion

`/auth/me` doit retrouver l'utilisateur grâce au cookie.

---

## Test 4

Cliquer sur Déconnexion.

Résultat attendu immédiatement :

    Connexion
    Inscription

Pas besoin de F5.

---

## Test 5

Faire F5 après déconnexion.

Résultat attendu :

    Connexion
    Inscription

---

# 21. Point exact de reprise demain matin

État actuel :

    Login backend           → OK
    Inscription             → OK
    Logout backend          → OK
    Cookies HttpOnly        → OK
    /auth/me                → OK
    Condition JSX           → OK

Problème restant :

    LoginPage modifie l'authentification
    MAIS
    Header possède son propre state user

Donc :

    le Header ne connaît pas immédiatement
    le résultat du login

Solution prévue :

    React Context

Première commande demain :

    cd /var/www/html/gamerChallenge/front
    git branch
    git status

Puis ouvrir le code de référence :

    cat /var/www/html/oClock/SC07/E03-Orecipes-context-largenty/frontend/src/context/AuthContext.tsx

Puis reprendre par :

    createContext
        ↓
    AuthProvider
        ↓
    useAuth

en adaptant progressivement le code à GamerChallenges.

---

# MÉMO ULTRA COURT

    Cookie
      ↓
    authentification HTTP

    user
      ↓
    état React de l'utilisateur

    Context
      ↓
    partage de user entre les composants

Et toujours garder en tête :

    ÉTAT
      ↓
    ACTION
      ↓
    AFFICHAGE

Pour GamerChallenges :

    AuthContext.user
           ↓
      login / logout
           ↓
         Header
           ↓
      user existe ?
        /       \
      oui       non
       ↓         ↓
    Déconnexion  Connexion
                 Inscription