# DevLog — Authentification avec React Context

**Date : 22/09/2026**  
**Projet : GamerChallenges**  
**Partie : Front-end — Authentification**

---

## Objectif de la séance

Continuer le travail sur l'authentification de GamerChallenges.

La connexion et l'inscription étaient déjà reliées à l'API.

Le problème principal était le suivant :

- la connexion fonctionnait ;
- le cookie d'authentification était bien créé ;
- mais le Header ne savait pas immédiatement que l'utilisateur venait de se connecter ;
- il fallait actualiser la page pour voir apparaître le bouton `Déconnexion`.

L'objectif était donc de comprendre pourquoi l'interface React ne se mettait pas à jour et de trouver une manière de partager l'utilisateur connecté entre plusieurs composants.

---

# 1. Comprendre le problème

Au départ, le Header gérait lui-même l'utilisateur avec un état local.

Le problème est que `LoginPage` et `Header` sont deux composants différents.

Quand `LoginPage` effectue la connexion, le navigateur reçoit bien les cookies d'authentification.

Mais cela ne modifie pas automatiquement l'état React du Header.

J'ai donc distingué deux choses :

```text
Cookie
→ permet au navigateur et au back-end de gérer l'authentification.

État React "user"
→ permet au front-end de savoir quel utilisateur est connecté
  et d'adapter l'interface.
```

Le cookie et l'état React n'ont donc pas le même rôle.

---

# 2. Pourquoi utiliser React Context ?

J'avais besoin d'un état accessible depuis plusieurs composants.

Par exemple :

```text
LoginPage
   ↓
connexion
   ↓
mise à jour de l'utilisateur
   ↓
AuthContext
   ↓
Header
```

J'ai donc utilisé React Context pour centraliser l'état de l'utilisateur connecté.

L'idée générale est :

```text
AuthContext
├── user
├── login()
└── logout()
```

`LoginPage` peut utiliser `login()`.

`Header` peut utiliser `user` et `logout()`.

Ils utilisent ainsi la même information.

---

# 3. Création du type User

J'ai défini la structure d'un utilisateur connecté :

```tsx
type User = {
    id: number;
    email: string;
    username: string;
    role: string;
    avatar: string | null;
};
```

Ce type représente les informations utilisateur utilisées par le front-end.

---

# 4. Création du type AuthContextType

J'ai ensuite défini ce que le Context doit partager :

```tsx
type AuthContextType =
    | {
        user: User | null;
        login: (email: string, password: string) => Promise<void>;
        logout: () => Promise<void>;
    }
    | undefined;
```

J'ai compris la différence entre :

```text
user: null
```

et :

```text
AuthContext = undefined
```

`user === null` signifie :

> aucun utilisateur n'est actuellement connecté.

`undefined` signifie :

> le Context n'a pas encore reçu de valeur depuis son Provider.

Ce ne sont donc pas les mêmes situations.

---

# 5. Création du Context

Le Context est créé avec :

```tsx
export const AuthContext = createContext<AuthContextType>(undefined);
```

J'ai retenu la lecture suivante :

```text
createContext
→ crée un Context React

<AuthContextType>
→ définit la forme des données partagées

undefined
→ valeur par défaut en dehors du Provider
```

---

# 6. Création du AuthProvider

J'ai créé un `AuthProvider`.

Son rôle est de contenir l'état d'authentification et de le rendre accessible aux composants enfants.

L'état principal est :

```tsx
const [user, setUser] = useState<User | null>(null);
```

J'ai compris que `user` constitue ici la source principale pour savoir si quelqu'un est connecté.

Il n'est donc pas nécessaire de créer en plus :

```text
isLoggedIn
pseudo
```

Je peux simplement utiliser :

```text
user === null
→ utilisateur non connecté

user !== null
→ utilisateur connecté

user.username
→ nom de l'utilisateur
```

Cela évite d'avoir plusieurs états représentant la même information.

---

# 7. Comprendre `children`

Le Provider reçoit :

```tsx
{
    children,
}: {
    children: React.ReactNode;
}
```

Au début, cette notion n'était pas claire pour moi.

Avec :

```tsx
<AuthProvider>
    <App />
</AuthProvider>
```

`App` devient le `children` de `AuthProvider`.

Le Provider retourne ensuite :

```tsx
<AuthContext.Provider value={value}>
    {children}
</AuthContext.Provider>
```

Donc `children` signifie ici :

> les composants placés à l'intérieur de `AuthProvider`.

J'ai également compris que cela n'a rien à voir avec le rendu conditionnel.

Par exemple :

```tsx
condition ? composantA : composantB
```

est du rendu conditionnel.

`children` représente simplement le contenu placé entre les balises du composant.

---

# 8. Gestion de la connexion dans le Context

La fonction `login()` envoie la requête vers :

```text
POST /api/auth/login
```

avec :

```tsx
credentials: "include"
```

Cela permet au navigateur d'envoyer et de recevoir les cookies utilisés pour l'authentification.

Après une connexion réussie :

```tsx
const data = await responseLogin.json();
setUser(data);
```

C'est `setUser(data)` qui permet de mettre immédiatement à jour l'état React.

Le Header peut alors se mettre à jour sans actualisation de la page.

J'ai également compris la différence entre :

```text
login
```

et :

```text
responseLogin
```

`login` est une action / fonction.

`responseLogin` représente la réponse HTTP reçue depuis l'API.

C'est donc sur `responseLogin` que je peux vérifier :

```tsx
responseLogin.ok
```

---

# 9. Récupérer l'utilisateur après une actualisation

Un autre problème apparaît lors d'un `F5`.

L'état React est recréé et `user` revient initialement à `null`.

Mais le cookie d'authentification peut toujours être présent dans le navigateur.

J'ai donc déplacé la vérification de l'utilisateur dans `AuthProvider` avec un `useEffect`.

Le Context appelle :

```text
GET /api/auth/me
```

avec :

```tsx
credentials: "include"
```

Si le cookie est valide, le back-end renvoie l'utilisateur.

Le Context exécute ensuite :

```tsx
setUser(data);
```

Le fonctionnement devient :

```text
F5
↓
état React recréé
↓
user = null
↓
GET /api/auth/me
↓
le back-end vérifie le cookie
↓
utilisateur récupéré
↓
setUser(data)
↓
interface mise à jour
```

---

# 10. Gestion de la déconnexion

La fonction `logout()` appelle :

```text
GET /api/auth/logout
```

Le back-end supprime les cookies d'authentification.

Mais supprimer le cookie ne suffit pas pour mettre immédiatement l'interface à jour.

Il faut également faire :

```tsx
setUser(null);
```

Le fonctionnement est donc :

```text
logout()
↓
requête vers le back-end
↓
suppression des cookies
↓
setUser(null)
↓
mise à jour du Header
```

J'ai retenu qu'il faut gérer les deux côtés :

```text
Back-end
→ supprimer l'authentification

Front-end
→ mettre à jour l'état React
```

---

# 11. Valeurs partagées par le Context

Le Provider partage :

```tsx
const value = {
    user,
    login,
    logout,
};
```

Je ne partage pas directement `setUser`.

Les composants utilisent les actions prévues par le Context plutôt que de modifier directement son état.

---

# 12. Création du hook useAuth

J'ai ajouté :

```tsx
export function useAuth() {
    const context = useContext(AuthContext);

    if (!context) {
        throw new Error("useAuth doit être utilisé dans un AuthProvider");
    }

    return context;
}
```

Cela permet d'utiliser plus facilement le Context dans les composants.

Par exemple :

```tsx
const { user, logout } = useAuth();
```

dans le Header.

Ou :

```tsx
const { login } = useAuth();
```

dans `LoginPage`.

---

# 13. Connexion du Provider à l'application

Dans `main.tsx`, j'ai entouré `App` avec `AuthProvider`.

La structure devient :

```text
StrictMode
└── BrowserRouter
    └── AuthProvider
        └── App
            ├── Header
            ├── LoginPage
            └── autres composants
```

Les composants situés sous `AuthProvider` peuvent ainsi utiliser `useAuth()`.

---

# 14. Modification de LoginPage

Le formulaire récupère :

```tsx
const { login } = useAuth();
```

Après récupération des valeurs avec `FormData`, je vérifie que l'email et le mot de passe sont bien des chaînes de caractères :

```tsx
if (typeof email !== "string" || typeof password !== "string") {
    return;
}
```

Puis :

```tsx
await login(email, password);
```

Ainsi, `LoginPage` n'a plus besoin de gérer lui-même toute la logique de connexion.

---

# 15. Modification du Header

Le Header récupère maintenant :

```tsx
const { user, logout } = useAuth();
```

L'affichage dépend directement de `user`.

Principe :

```text
user === null
→ Connexion
→ Inscription

user !== null
→ Déconnexion
```

Cela permet au Header de réagir immédiatement lorsque `AuthContext` modifie l'utilisateur.

---

# 16. Bug rencontré pendant l'intégration

Pendant la modification du Header, j'ai rencontré l'erreur :

```text
ReferenceError: useState is not defined
```

Le Header utilisait toujours :

```tsx
const [isMenuOpen, setIsMenuOpen] = useState(false);
```

mais l'import de `useState` avait été retiré pendant les modifications.

Le menu mobile reste un état local du Header.

Il ne doit pas être placé dans `AuthContext`, car les autres composants n'en ont pas besoin.

J'ai donc retenu cette distinction :

```text
isMenuOpen
→ état local
→ seulement utile au Header

user
→ état partagé
→ utile à plusieurs composants
→ AuthContext
```

---

# 17. Avertissement Fast Refresh

Pendant le développement, Vite a également affiché un avertissement concernant `AuthContext.tsx` :

```text
Could not Fast Refresh ("AuthContext" export is incompatible)
```

Le fichier exporte actuellement plusieurs éléments :

- le Context ;
- le Provider ;
- le hook `useAuth`.

Cet avertissement n'était pas la même chose que l'erreur `useState is not defined`.

La fonctionnalité d'authentification a été prioritaire pour cette séance.

La structure pourra être nettoyée plus tard si nécessaire.

---

# 18. Résultat obtenu

Après les modifications, le fonctionnement attendu a pu être obtenu :

```text
CONNEXION

LoginPage
↓
login()
↓
POST /api/auth/login
↓
cookie créé
↓
setUser(data)
↓
AuthContext mis à jour
↓
Header mis à jour
↓
Déconnexion affichée sans F5
```

Après actualisation :

```text
F5
↓
état React recréé
↓
GET /api/auth/me
↓
cookie vérifié
↓
setUser(data)
↓
session affichée à nouveau
```

Déconnexion :

```text
Header
↓
logout()
↓
GET /api/auth/logout
↓
cookies supprimés
↓
setUser(null)
↓
Connexion + Inscription affichées
```

---

# 19. Différence avec l'exemple du cours

Je me suis basée sur l'exemple `Orecipes` du cours pour comprendre React Context.

Mais je n'ai pas copié exactement sa gestion de l'authentification.

Dans l'exemple du cours, le front-end conserve notamment un token dans un état React.

Dans GamerChallenges, le back-end utilise des cookies `HttpOnly`.

Le front-end n'a donc pas besoin de stocker directement le JWT.

Pour GamerChallenges :

```text
Cookie HttpOnly
→ gestion du token d'authentification

AuthContext
→ gestion de l'utilisateur pour l'interface React
```

Cette différence m'a permis de comprendre qu'un exemple de cours doit être adapté à l'architecture réelle du projet.

---

# 20. Formule à retenir

La structure qui m'aide à comprendre le fonctionnement est :

```text
ÉTAT
↓
ACTION
↓
AFFICHAGE
```

Dans ce cas :

```text
ÉTAT
user

ACTION
login()
logout()

AFFICHAGE
Header
```

Et plus globalement :

```text
Cookie
→ authentification navigateur / back-end

user
→ état utilisé par React

Context
→ partage de cet état entre les composants
```

---

# 21. Présentation du jour

La fonctionnalité a été terminée juste avant la présentation.

Je n'ai donc pratiquement pas eu le temps de préparer ou relire mon texte de présentation.

J'ai finalement réalisé la démonstration directement dans l'application plutôt que d'utiliser une vidéo enregistrée.

La démonstration portait sur les fonctionnalités développées autour de l'authentification.

Cette situation m'a montré qu'il est important de pouvoir expliquer le fonctionnement du projet avec mes propres mots, même sans avoir un texte préparé devant moi.

---

# 22. PostgreSQL — commande à retenir

Pour se connecter à la base GamerChallenges :

    psql -U gamerchallenge -d gamerchallenge

Pour afficher les utilisateurs :

    SELECT id, email, username FROM "User";

Pour afficher les tables :

    \dt

Pour quitter PostgreSQL :

    \q

Attention :

```text
Commande correcte : psql
Pas : postgre
```

---

# Ce que j'ai réellement compris aujourd'hui

- Un cookie d'authentification et un état React n'ont pas le même rôle.
- Une modification du cookie ne met pas automatiquement l'interface React à jour.
- Un état local appartient à un composant.
- Un Context permet de partager un état entre plusieurs composants.
- `user` peut servir de source unique pour connaître l'état de connexion.
- `user === null` représente un utilisateur non connecté.
- `undefined` dans `createContext` représente l'absence de valeur fournie par le Provider.
- `children` représente les composants placés dans le Provider.
- `login()` est une action alors que `responseLogin` est une réponse HTTP.
- `setUser(data)` permet au Header de réagir immédiatement après la connexion.
- `/auth/me` permet de restaurer l'utilisateur après un rafraîchissement de la page.
- Une déconnexion nécessite à la fois la suppression de l'authentification côté back-end et la mise à jour de l'état côté front-end.
- Tous les états n'ont pas besoin d'être placés dans un Context.
- Un exemple du cours doit être compris puis adapté à l'architecture réelle du projet.

---

# Points à reprendre lors de la prochaine séance

- Vérifier l'état Git de la branche `feat/logout`.
- Vérifier les modifications de `package.json` avant de les conserver.
- Nettoyer progressivement l'ancien code commenté devenu inutile.
- Vérifier les commentaires devenus obsolètes dans `LoginPage`.
- Examiner l'avertissement Fast Refresh lié à `AuthContext`.
- Faire un dernier test complet connexion → F5 → déconnexion → F5.
- Vérifier la réponse du back-end lors du login afin de ne pas renvoyer d'informations sensibles inutiles, notamment le hash du mot de passe.
- Préparer ensuite le commit de la fonctionnalité une fois le code nettoyé et testé.