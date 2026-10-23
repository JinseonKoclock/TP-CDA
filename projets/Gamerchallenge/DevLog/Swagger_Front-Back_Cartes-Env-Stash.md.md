# DevLog — 02/10/2026

## Swagger et cartes invisibles : mes essais et vérifications

---

### Swagger : je cherchais un élément qui n’apparaissait pas

Après avoir ajouté la documentation de la route, je ne comprenais pas ce que signifiait « ouvrir GET /challenges » : cette ligne n’apparaissait pas à l’écran.

J’ai arrêté puis redémarré le serveur. L’opération est alors apparue et les instructions sont devenues claires.

En lançant la requête, j’ai trouvé ça vraiment amusant : je pouvais appeler mon API sans écrire un code de requête supplémentaire et voir les données du seed dans la réponse.

---

### Les cartes ne s’affichaient pas : plusieurs problèmes se superposaient

J’ai cloné le frontend de Marie-Laure pour vérifier son fonctionnement avec mon nouveau backend. Les cartes ne s’affichaient pas.

Avec l’aide de l’IA, j’ai comparé le code des cartes avec un objet réellement renvoyé par Swagger. Le frontend utilisait encore les anciens champs : `jeu.title`, `user.username`, `voter.length` et `url_video`.

Un autre problème se trouvait avant même le rendu : le filtre sur `challenge.status` éliminait tous les challenges, car ce champ était absent de la nouvelle réponse.

J’ai adapté les champs, le type `Challenge` et le tri. Mais une erreur de lecture JSON est ensuite apparue : les corrections d’affichage ne suffisaient donc pas à tout résoudre.

---

### Le fichier `.env` oublié

En regardant la fonction `fetchChallenges`, je me suis rendu compte que je n’avais pas créé le `.env` du frontend après le clonage.

J’ai ajouté l’adresse du backend et redémarré le frontend. Cet oubli expliquait le problème de configuration de l’adresse de l’API. Nous n’avons toutefois pas inspecté le contenu exact de la réponse qui avait provoqué l’erreur JSON.

---

### « Et si le `.env` suffisait finalement ? »

Même après avoir repéré les différences de structure, je voulais vérifier si le frontend d’origine pouvait fonctionner simplement avec le bon `.env`.

J’ai donc mis mes modifications de côté avec stash pour tester le code original.

Cette manipulation a elle aussi demandé une vérification : le premier stash avait été créé, mais des modifications restaient staged. Un second stash a permis de retrouver un état propre. Le `.env`, déjà ignoré par Git, est resté en place.

Le frontend original affichait toujours « Aucun challenge disponible pour le moment ». J’ai ensuite restauré mes modifications sur ma branche : les cartes s’affichaient correctement.

Cette comparaison répondait à ma question : dans cet essai, ajouter le `.env` ne suffisait pas ; l’adaptation du frontend à la nouvelle réponse était également nécessaire.

---

### Ce que je retiens

Une carte invisible ne signifie pas forcément que l’API ne fonctionne pas. Aujourd’hui, j’ai dû regarder trois endroits : l’adresse de la requête, la structure des données reçues et le filtre appliqué avant l’affichage.

Le moment le plus utile a été de revenir volontairement au code original pour vérifier mon hypothèse, puis de restaurer mes corrections. Je ne voulais pas seulement supposer que cela fonctionnerait : je voulais voir la différence.

L’IA m’a aidée à comparer les structures et à proposer les corrections. De mon côté, j’ai exécuté les manipulations, testé les deux versions et confirmé l’affichage dans le navigateur.