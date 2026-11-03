# DevLog — 1er novembre 2026
## GamerChallenges : déconnexion et lecture du flux React

### Point de départ

La déconnexion échouait avec une requête `GET /api/auth/logout` et une réponse `404`. Je voulais aussi revenir à l’accueil après la déconnexion. En reprenant du code écrit environ un mois auparavant, je ne retrouvais plus clairement le lien entre le bouton, le Context et les fonctions appelées.

Le débogage a donc servi à remettre le fonctionnement en ordre, puis à identifier ce qui restait confus dans ma lecture du code.

### 1. Déconnexion : retrouver le bon endroit à modifier

Le backend attendait :

```ts
router.post("/logout", verifyAuth, logout);
```

Le frontend envoyait encore une requête `GET`. J’ai d’abord travaillé sur une requête dans le Header, puis supprimé ce code puisque la déconnexion était déjà gérée par AuthContext. Cependant, la requête du Context utilisait toujours `GET` : modifier une fonction qui n’est pas celle appelée par le bouton ne corrige pas le parcours exécuté.

J’ai aussi supprimé l’état local `user` du Header, qui faisait doublon avec celui récupéré par `useAuth()`.

En voulant ajouter la redirection, j’ai placé `useNavigate()` dans AuthProvider. L’application a alors cessé de s’afficher, avec une erreur concernant ce composant. L’explication envisagée était un Provider situé hors du Router ; l’ordre des composants et le message d’erreur détaillé n’ont pas été vérifiés pendant cet échange.

La correction proposée, après laquelle j’ai confirmé que cela fonctionnait, répartit les actions ainsi :

| Emplacement | Responsabilité |
| --- | --- |
| AuthContext : `logout` | Envoyer la requête `POST`, vérifier la réponse et exécuter `setUser(null)` en cas de succès. Signaler un échec en levant une erreur. |
| Header : `handleLogout` | Attendre `logout()`, fermer le menu et revenir à l’accueil. Intercepter une éventuelle erreur. |

```tsx
const handleLogout = async () => {
  try {
    await logout();
    closeMenu();
    navigate("/");
  } catch (error) {
    console.error(error);
  }
};
```

J’avais déjà identifié que si `user` restait renseigné côté React, l’interface continuerait à afficher l’état connecté. Ce point n’était donc pas à reprendre depuis zéro. La difficulté était surtout de retrouver quelle fonction modifie cet état et comment le clic y mène.

### 2. Événements : définir, connecter et appeler une fonction

Ma question était : « Est-ce que je comprends mal la fonction utilisée avec `onClick` et son application au bouton ? »

La distinction travaillée est la suivante :

| Code | Rôle |
| --- | --- |
| `const handleLogout = async () => { ... }` | Définir les actions à effectuer. |
| `onClick={handleLogout}` | Donner à React la fonction à exécuter lors du clic. |
| `await logout()` | Appeler effectivement une autre fonction et attendre sa fin. |

`onClick={logout}` était également une connexion valide. Le passage par `handleLogout` permet d’ajouter des actions après la déconnexion, notamment la navigation.

J’ai également demandé pourquoi le login utilisait `onSubmit` alors que le logout utilisait `onClick`. Le premier répond à la soumission d’un formulaire, y compris avec Entrée ; le second répond au clic sur le bouton. Le choix de `onClick` n’était pas la cause de l’échec de déconnexion.

### 3. Menu mobile : où sont définis « ouvert » et « fermé » ?

En relisant le Header, je ne retrouvais pas la fonction associée à cet état :

```tsx
const [isMenuOpen, setIsMenuOpen] = useState(false);
```

Puis j’ai demandé si ce code était une fonction fournie par React :

```tsx
const toggleMenu = () => {
  setIsMenuOpen((currentValue) => !currentValue);
};
```

Nous avons séparé les éléments : `toggleMenu` est une fonction définie dans le projet ; `setIsMenuOpen` est la fonction de mise à jour obtenue avec `useState`. La fonction `(currentValue) => !currentValue` est écrite dans notre code : React l’appelle avec l’état précédent pour calculer le suivant.

Ma question suivante était : « Mais où décide-t-on que cela ouvre ou ferme le menu ? Dans le CSS ? »

La liaison se trouve dans le JSX :

```tsx
className={
  isMenuOpen
    ? "site-header__menu site-header__menu--open"
    : "site-header__menu"
}
```

J’ai d’abord eu du mal à retrouver la classe avec la recherche, puis trouvé cette règle dans `Header.css` :

```css
.site-header__menu--open {
  display: block;
}
```

Le lien à retenir est donc : clic sur le bouton → `toggleMenu` → changement d’état → ajout ou retrait de la classe → application du CSS. `true` ne signifie pas automatiquement « ouvert » : c’est notre code qui lui associe cette classe. Le style de base et les éventuelles media queries déterminent l’affichage lorsque cette classe est absente ; ils n’ont pas été examinés ici.

### Bilan et point à consolider

La déconnexion fonctionne après les modifications. J’ai aussi retrouvé la règle CSS utilisée pour afficher le menu mobile.

Le point à consolider est la lecture d’un parcours complet : partir d’un événement dans le JSX, retrouver la fonction appelée, puis suivre les changements d’état et leur effet sur l’affichage. Les explications ci-dessus constituent des repères de révision ; elles ne signifient pas que tous ces mécanismes sont déjà maîtrisés.

Ce document a été rédigé avec une démarche d’apprentissage actif, en utilisant ChatGPT comme support de réflexion et de structuration. Les questions et essais viennent de mon travail ; la validation finale du contenu et des acquis reste la mienne.
