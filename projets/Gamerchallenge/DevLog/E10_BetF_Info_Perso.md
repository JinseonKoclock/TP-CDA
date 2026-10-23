# DevLog – 25/09/2026 – Dynamisation de la modification du profil utilisateur

## Objectif

L'objectif de cette session était de dynamiser la page **« Mes informations personnelles »** afin de permettre à un utilisateur connecté de consulter et modifier ses informations.

Les informations modifiables dans le MVP sont :

- le nom d'utilisateur ;
- l'adresse e-mail ;
- le mot de passe.

La gestion de l'avatar a finalement été exclue du MVP afin de rester concentrés sur les fonctionnalités principales.

---

## 1. Création de la route de modification du profil

Une nouvelle route a été ajoutée dans le backend :

```http
PATCH /api/auth/me
```

Cette route est protégée par le middleware `verifyAuth`.

```ts
router.patch("/auth/me", verifyAuth, authController.updateLoggedUser);
```

Le middleware permet d'identifier l'utilisateur connecté grâce au token d'authentification et de récupérer son identifiant avec :

```ts
req.user.id
```

Ainsi, l'identifiant de l'utilisateur à modifier n'est pas envoyé par le frontend.

---

## 2. Validation des données avec Zod

Les données pouvant être modifiées sont validées avec Zod.

Les champs sont optionnels afin de permettre une modification partielle du profil.

```ts
username: z.string().min(1).optional(),
email: z.email().optional(),
password: z.string().min(8).max(100).optional(),
confirm: z.string().optional()
```

Pour le mot de passe, les mêmes contraintes principales que lors de l'inscription sont conservées.

La confirmation du nouveau mot de passe est également vérifiée avant la modification.

---

## 3. Vérification de l'adresse e-mail

Avant de modifier l'adresse e-mail, une vérification est effectuée afin de s'assurer qu'elle n'est pas déjà utilisée par un autre utilisateur.

L'utilisateur actuellement connecté est exclu de cette recherche :

```ts
where: {
  email,
  NOT: {
    id: req.user!.id
  }
}
```

Cela permet à l'utilisateur de conserver son adresse e-mail actuelle sans provoquer de conflit.

---

## 4. Modification du mot de passe

Le nouveau mot de passe n'est traité que lorsqu'il est renseigné.

Avant son enregistrement en base de données, il est hashé avec Argon2 :

```ts
if (password) {
  hashedPassword = await argon2.hash(password);
}
```

Le mot de passe en clair n'est donc pas enregistré dans la base de données.

---

## 5. Tests de l'API

La route a été testée avec Thunder Client.

Tests effectués :

- récupération de l'utilisateur connecté ;
- modification du username uniquement ;
- conservation de l'adresse e-mail actuelle ;
- tentative d'utilisation de l'adresse e-mail d'un autre utilisateur ;
- vérification de la confirmation du mot de passe ;
- modification correcte du mot de passe.

Une erreur de structure dans le contrôleur a également été identifiée pendant les tests : la mise à jour de l'utilisateur se trouvait accidentellement dans la condition vérifiant l'e-mail.

Cela provoquait une requête qui restait en attente lorsque seul le username était modifié.

La structure du contrôleur a été corrigée et les différents cas ont ensuite été testés avec succès.

---

## 6. Dynamisation de la page frontend

La page `EditProfilePage` récupère maintenant les informations de l'utilisateur connecté avec :

```http
GET /api/auth/me
```

Les informations récupérées sont placées dans les states React :

```ts
const [username, setUsername] = useState("");
const [email, setEmail] = useState("");
const [password, setPassword] = useState("");
const [confirm, setConfirm] = useState("");
```

Les champs du formulaire sont maintenant des champs contrôlés avec `value` et `onChange`.

---

## 7. Envoi des modifications au backend

Lors de la validation du formulaire, les nouvelles informations sont envoyées avec :

```http
PATCH /api/auth/me
```

Les cookies d'authentification sont transmis avec :

```ts
credentials: "include"
```

Le mot de passe et sa confirmation ne sont ajoutés à la requête que si l'utilisateur renseigne un nouveau mot de passe :

```ts
const updatedUser = {
  username,
  email,
  ...(password && {
    password,
    confirm,
  }),
};
```

Cela permet de modifier uniquement le username ou l'e-mail sans avoir à saisir de nouveau mot de passe.

---

## 8. Notifications utilisateur

Des notifications ont été ajoutées afin d'informer l'utilisateur du résultat de la modification.

Deux cas sont gérés :

- modification réussie ;
- erreur retournée par l'API.

Les notifications disparaissent automatiquement après trois secondes grâce à `setTimeout`.

---

## 9. Protection des pages privées

Pendant les tests, j'ai constaté qu'un utilisateur non connecté pouvait encore afficher directement certaines pages privées depuis leur URL.

Le backend protégeait bien les API avec `verifyAuth`, mais le frontend ne protégeait pas encore les routes.

Un composant `ProtectedRoute` a donc été créé.

Son fonctionnement est le suivant :

1. vérifier l'état d'authentification ;
2. attendre la fin de cette vérification avec `isLoading` ;
3. afficher la page si un utilisateur est connecté ;
4. rediriger vers `/login` dans le cas contraire.

```tsx
if (isLoading) {
  return null;
}

if (!user) {
  return <Navigate to="/login" replace />;
}

return children;
```

Les pages concernées peuvent maintenant être entourées par `ProtectedRoute`.

---

## 10. Ajout de `isLoading` dans le contexte d'authentification

Un état `isLoading` a été ajouté dans `AuthProvider`.

```ts
const [isLoading, setIsLoading] = useState(true);
```

Il permet de différencier deux situations :

- l'application n'a pas encore terminé de vérifier l'authentification ;
- la vérification est terminée et aucun utilisateur n'est connecté.

Sans cet état, `user` vaut initialement `null` et le frontend pourrait rediriger l'utilisateur vers `/login` avant même que la requête `/api/auth/me` soit terminée.

`isLoading` a également été ajouté au `AuthContext` afin d'être accessible depuis `ProtectedRoute`.

---

## 11. Validation finale

Après récupération des dernières modifications de la branche `develop`, le projet a été compilé avec :

```bash
npm run build
```

Résultat :

```text
✓ built
```

La compilation TypeScript et le build Vite se terminent correctement.

---

## Ce que j'ai compris

Cette session m'a permis de mieux comprendre la différence entre la protection côté frontend et la sécurité côté backend.

`ProtectedRoute` améliore la navigation et empêche l'affichage d'une page privée dans l'interface lorsqu'un utilisateur n'est pas connecté.

Cependant, la véritable protection des données reste assurée par le backend avec `verifyAuth`.

J'ai également mieux compris le rôle de `AuthProvider` et de `AuthContext` : ils permettent de partager l'état de l'utilisateur connecté entre plusieurs composants sans devoir refaire cette gestion indépendamment dans chaque page.

Enfin, `isLoading` est nécessaire car `user === null` ne signifie pas forcément que l'utilisateur est déconnecté : au démarrage de l'application, cela peut simplement signifier que la vérification de l'authentification n'est pas encore terminée.