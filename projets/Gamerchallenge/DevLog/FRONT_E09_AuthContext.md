# DevLog — Mise en place de l'AuthContext

## Objectif

Centraliser l'état d'authentification de GamerChallenges afin que plusieurs composants puissent connaître l'utilisateur connecté.

Avant la mise en place du Context :

- la connexion fonctionnait ;
- le backend créait correctement les cookies d'authentification ;
- `/api/auth/me` permettait de récupérer l'utilisateur connecté ;
- la déconnexion fonctionnait côté API ;
- mais le `Header` ne détectait pas immédiatement une nouvelle connexion.

Après une connexion, il fallait actualiser la page avec `F5` pour que le bouton `Déconnexion` apparaisse.

---

## 1. Problème identifié

Le cookie et l'état React ne jouent pas le même rôle.

    Cookie
    → information d'authentification échangée entre le navigateur et le backend

    user
    → état React utilisé pour mettre à jour l'interface

Le backend pouvait donc considérer l'utilisateur comme connecté grâce au cookie, alors que le `Header` possédait encore :

    user = null

Le cookie était bien créé, mais React ne savait pas automatiquement que l'état de l'utilisateur avait changé.

Il fallait donc partager l'état `user` entre les différents composants.

---

## 2. Pourquoi utiliser un Context ?

Le `LoginPage` doit pouvoir modifier l'utilisateur connecté après une connexion réussie.

Le `Header` doit pouvoir lire ce même utilisateur afin d'afficher :

    user !== null
        → Déconnexion

    user === null
        → Connexion + Inscription

L'objectif est donc d'obtenir :

    LoginPage
        ↓
      login()
        ↓
    AuthContext
        ↓
       user
        ↓
      Header

Le Context sert ici de zone commune permettant de partager l'état d'authentification.

---

## 3. Tous les états ne doivent pas aller dans le Context

Dans le `Header`, j'avais notamment :

    const [isMenuOpen, setIsMenuOpen] = useState(false);

    const [user, setUser] = useState<User | null>(null);

Il ne faut pas déplacer automatiquement tous les états dans le Context.

### `isMenuOpen`

Il sert uniquement à savoir si le menu mobile du `Header` est ouvert.

    isMenuOpen
        ↓
    utilisé uniquement par Header
        ↓
    reste dans Header

### `user`

Il représente l'utilisateur actuellement connecté.

Cette information doit être utilisée par plusieurs composants.

    user
        ↓
    LoginPage
    Header
    autres composants si nécessaire
        ↓
    AuthContext

Règle retenue :

    État utilisé par un seul composant
        → état local du composant

    État devant être partagé entre plusieurs composants
        → Context

---

## 4. Définition de l'utilisateur

J'ai commencé par définir la structure d'un utilisateur connecté :

    type User = {
        id: number;
        email: string;
        username: string;
        role: string;
        avatar: string | null;
    };

`User` représente donc la forme des données concernant un utilisateur.

---

## 5. Définition du contenu du Context

Le Context doit partager trois éléments :

    type AuthContextType =
        | {
            user: User | null;
            login: (email: string, password: string) => Promise<void>;
            logout: () => Promise<void>;
        }
        | undefined;

Le Context contient donc :

    AuthContext
    ├── user
    ├── login()
    └── logout()

### Pourquoi `User | null` ?

`user` peut contenir :

    User
    → un utilisateur est connecté

ou :

    null
    → aucun utilisateur n'est connecté

Cela évite de créer un deuxième état `isLoggedIn`.

La présence ou non de `user` suffit pour déterminer l'état de connexion.

---

## 6. Création du Context

    export const AuthContext = createContext<AuthContextType>(undefined);

Décomposition :

    AuthContext
    → nom du contexte

    createContext
    → crée le contexte

    <AuthContextType>
    → définit la forme des données partagées

    undefined
    → valeur par défaut tant qu'aucun Provider ne fournit les données

Important :

    user === null
    → utilisateur non connecté

    AuthContext === undefined
    → aucun Provider ne fournit encore le contexte

Ce sont deux situations différentes.

---

## 7. Comprendre `children`

Dans le Provider apparaît :

    export default function AuthProvider({
        children,
    }: {
        children: React.ReactNode;
    }) {

Au départ, j'ai confondu `children` avec le rendu conditionnel.

Mais ce sont deux notions différentes.

### `children`

`children` représente ce qui est placé entre les balises d'un composant.

Par exemple :

    <AuthProvider>
        <App />
    </AuthProvider>

Ici :

    <App />

est le `children` de `AuthProvider`.

On peut visualiser la structure ainsi :

    <AuthProvider>
          ↓
        <App />     ← children
          ↓
    </AuthProvider>

`React.ReactNode` indique que `children` peut être un élément que React sait afficher.

### À ne pas confondre avec une condition

    children
    → « Qu'est-ce qui se trouve à l'intérieur de mon composant ? »

    condition ? A : B
    → « Qu'est-ce que j'affiche selon une condition ? »

---

## 8. État partagé de l'utilisateur

Dans le Provider :

    const [user, setUser] = useState<User | null>(null);

Au démarrage :

    user = null

Puis :

    user
    → permet de lire l'utilisateur actuel

    setUser(...)
    → permet de modifier l'utilisateur actuel

---

## 9. Fonction `login`

La requête de connexion ne doit pas être exécutée directement dans `AuthProvider`.

Elle doit être exécutée uniquement lorsque l'utilisateur demande à se connecter.

J'ai donc créé une fonction :

    const login = async (email: string, password: string) => {
        ...
    };

Structure :

    AuthProvider
        │
        └── login(email, password)
                ↓
              fetch()
                ↓
          responseLogin

J'ai également appris à distinguer :

    login
    → fonction représentant l'action de connexion

    responseLogin
    → réponse HTTP reçue du serveur

J'avais écrit par erreur :

    if (login.ok)

Mais `.ok` appartient à la réponse HTTP.

La bonne logique est donc :

    if (responseLogin.ok)

---

## 10. Mise à jour de `user` après la connexion

Lorsque la connexion réussit :

    if (responseLogin.ok) {
        const data = await responseLogin.json();
        setUser(data);
    }

Le fonctionnement devient :

    utilisateur envoie le formulaire
        ↓
    login(email, password)
        ↓
    POST /api/auth/login
        ↓
    backend valide les identifiants
        ↓
    cookies d'authentification créés
        ↓
    données utilisateur retournées
        ↓
    setUser(data)
        ↓
    user est mis à jour
        ↓
    les composants utilisant le Context sont mis à jour

C'est notamment `setUser(data)` qui permettra au `Header` de réagir sans avoir besoin d'un `F5`.

---

## 11. Différence avec le projet du formateur

Projet de référence :

    E03-Orecipes-context-largenty/frontend/src/context/AuthContext.tsx

Dans le projet du formateur, React conservait notamment :

    isLoggedIn
    token
    pseudo
    error

Le logout pouvait donc remettre directement ces états à zéro :

    setIsLoggedIn(false);
    setToken(null);
    setPseudo(null);
    setError(null);

GamerChallenges fonctionne différemment.

Le JWT est stocké dans des cookies HttpOnly.

Le frontend ne doit donc pas récupérer et stocker directement le token dans un state React.

Dans GamerChallenges :

    Cookie HttpOnly
    → authentification HTTP

    user
    → état React utilisé par l'interface

---

## 12. Fonction `logout`

Pour GamerChallenges, faire uniquement :

    setUser(null);

ne suffit pas.

Cela modifierait l'interface React, mais les cookies d'authentification seraient toujours présents.

Il faut donc effectuer deux actions :

    logout()
        ↓
    GET /api/auth/logout
        ↓
    backend supprime les cookies
        ↓
    setUser(null)
        ↓
    React supprime l'utilisateur connecté

La fonction `logout` est donc asynchrone :

    const logout = async () => {
        const responseLogout = await fetch(
            `${import.meta.env.VITE_API_URL}/api/auth/logout`,
            {
                method: "GET",
                credentials: "include",
            }
        );

        if (responseLogout.ok) {
            setUser(null);
        }
    };

Son type est donc :

    logout: () => Promise<void>;

---

## 13. Création de `value`

Le Context ne doit pas partager directement tous les outils internes du Provider.

Par exemple :

    setUser

reste utilisé à l'intérieur du Provider.

Les autres composants doivent utiliser les actions prévues :

    login()
    logout()

Les valeurs exposées sont donc :

    const value = {
        user,
        login,
        logout,
    };

On peut distinguer :

    Interne au Provider
    ├── setUser
    └── gestion des requêtes

    Partagé avec les composants
    ├── user
    ├── login()
    └── logout()

---

## 14. Le Provider et `children`

Le Provider transmet ensuite `value` à ses composants enfants :

    return (
        <AuthContext.Provider value={value}>
            {children}
        </AuthContext.Provider>
    );

La structure peut être représentée ainsi :

    value
    ├── user
    ├── login()
    └── logout()
         ↓
    AuthContext.Provider
         ↓
      children
         ↓
       <App />
         ↓
    Header / LoginPage / autres composants

Les composants placés à l'intérieur de `AuthProvider` pourront donc accéder aux valeurs partagées par le Context.

---

## 15. Formule à retenir

    ÉTAT
      ↓
    ACTION
      ↓
    AFFICHAGE

Dans GamerChallenges :

    ÉTAT
    user

      ↓

    ACTIONS
    login()
    logout()

      ↓

    AFFICHAGE
    user !== null
        → Déconnexion

    user === null
        → Connexion + Inscription

---

## 16. Ce que j'ai réellement compris aujourd'hui

- Un cookie d'authentification et un état React `user` ne sont pas la même chose.
- La modification d'un cookie ne provoque pas automatiquement la mise à jour de l'interface React.
- `Context` permet de partager un état entre plusieurs composants.
- Tous les states ne doivent pas être déplacés dans un Context.
- `children` représente les éléments placés à l'intérieur d'un composant.
- `children` n'est pas un mécanisme de rendu conditionnel.
- `user` permet de lire l'état actuel.
- `setUser` permet de modifier cet état.
- `login` est une fonction, alors que `responseLogin` est une réponse HTTP.
- Après une connexion réussie, `setUser(data)` permet de synchroniser l'interface avec l'utilisateur connecté.
- Après une déconnexion réussie, `setUser(null)` remet l'interface dans l'état déconnecté.
- Dans GamerChallenges, le logout doit également appeler l'API afin que le backend supprime les cookies HttpOnly.
- Le code du formateur sert de modèle de structure, mais il doit être adapté à l'architecture réelle de GamerChallenges.

---

## 17. État actuel

Structure construite dans `AuthContext.tsx` :

    AuthProvider
    │
    ├── ÉTAT
    │   └── user / setUser
    │
    ├── ACTION
    │   ├── login()
    │   │   ├── POST /api/auth/login
    │   │   └── setUser(data)
    │   │
    │   └── logout()
    │       ├── GET /api/auth/logout
    │       └── setUser(null)
    │
    └── PARTAGE
        └── value
            ├── user
            ├── login
            └── logout

Prochaine étape :

- terminer le branchement de `AuthProvider` dans l'application ;
- créer/utiliser le hook `useAuth()` ;
- remplacer l'état `user` local du `Header` par le `user` du Context ;
- utiliser `login()` du Context dans `LoginPage` ;
- tester que `Connexion` devient immédiatement `Déconnexion` sans actualiser la page.