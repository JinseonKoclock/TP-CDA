네가 이번 Sprint에서 작업하면서 가장 설명하고 싶은 **인증 상태 관리(AuthContext)**

environ 3 min

De mon côté, cette semaine, j'ai principalement travaillé sur la gestion de l'utilisateur connecté côté front.

J'ai d'abord travaillé sur la page des informations personnelles.
L'utilisateur connecté peut consulter ses informations, puis modifier son nom d'utilisateur, son email et, s'il le souhaite, son mot de passe.

Pour envoyer les modifications au back, j'utilise une requête PATCH vers /api/auth/me. J'ai également ajouté des notifications pour indiquer à l'utilisateur si la modification a réussi ou si une erreur a été retournée par l'API.

Mais le travail que je voudrais surtout présenter concerne l'AuthContext.

Nous avions besoin de connaître l'utilisateur connecté à plusieurs endroits de l'application. Par exemple, le Header doit adapter son affichage, et certaines pages comme le profil doivent être accessibles uniquement à un utilisateur connecté.

Pour éviter de gérer séparément l'utilisateur dans chaque composant, nous utilisons donc un AuthProvider. Il conserve l'utilisateur dans un state et partage user, login, logout et isLoading grâce à l'AuthContext.

Au démarrage de l'application, l'AuthProvider envoie une requête GET /api/auth/me avec credentials: "include". Le back vérifie alors le cookie d'authentification et, si l'utilisateur est authentifié, renvoie ses informations. Le front peut ensuite les enregistrer avec setUser.

J'ai également ajouté isLoading, parce qu'au démarrage user vaut d'abord null, même si l'utilisateur est déjà connecté. Il faut donc attendre la réponse de l'API avant de décider s'il est réellement connecté ou non.

Enfin, j'ai créé un ProtectedRoute qui utilise cet état. Tant que la vérification est en cours, il attend. Si aucun utilisateur n'est connecté, il redirige vers /login. Sinon, il affiche la page protégée.

Cela permet donc d'avoir une gestion centralisée de l'utilisateur connecté et de contrôler l'accès aux pages privées côté front.

Enfin, j'ai également préparé la page de modification d'un challenge. La partie front est commencée, mais la connexion complète avec le back reste encore à terminer.