# Dev Log — MLD GamerChallenges

## Date

18/09/2026

## Objectif de la session

Comprendre la démarche Merise appliquée à GamerChallenges, finaliser le MCD puis le transformer en MLD afin de comprendre le passage du modèle conceptuel au modèle logique.

---

## Travail réalisé

### Passage du MCD au MLD

J'ai travaillé sur la transformation de mon MCD en MLD.

J'ai compris que le MCD reste centré sur les données et les règles métier, alors que le MLD commence à représenter la structure relationnelle de la future base de données.

Dans le MLD, j'ai donc défini :

- les tables ;
- les clés primaires ;
- les clés étrangères ;
- les tables de jointure ;
- les clés primaires composées.

---

### Ajout des clés primaires `id`

Dans mon MCD, j'avais choisi les déterminants suivants :

- `Pseudo` pour `USER` ;
- `Titre` pour `CHALLENGE` ;
- `URL de la vidéo` pour `PARTICIPATION`.

En travaillant sur le MLD et en comparant ma structure avec les exemples vus en cours, j'ai compris que je pouvais utiliser une clé primaire `id` pour les tables principales.

J'ai donc choisi :

- `USER.id`
- `CHALLENGE.id`
- `PARTICIPATION.id`

Les relations entre les tables peuvent ensuite utiliser ces clés primaires comme références.

---

### Transformation des relations 1:N

J'ai compris que les associations 1:N du MCD sont représentées dans le MLD par des clés étrangères.

Par exemple :

`USER — PROPOSER — CHALLENGE`

devient une référence vers `USER(id)` dans `CHALLENGE`.

De la même manière, `PARTICIPATION` contient une référence vers `USER(id)` et une référence vers `CHALLENGE(id)`.

---

### Transformation des relations N:N

Les deux relations de vote de mon MCD sont de type N:N :

- un utilisateur peut voter pour plusieurs challenges et un challenge peut recevoir plusieurs votes ;
- un utilisateur peut voter pour plusieurs participations et une participation peut recevoir plusieurs votes.

J'ai compris qu'une relation N:N nécessite une table de jointure dans le MLD.

J'ai donc obtenu :

- `VOTE_CHALLENGE`
- `VOTE_PARTICIPATION`

Ces tables utilisent leurs deux clés étrangères comme clé primaire composée.

Cela permet également d'empêcher un utilisateur de voter plusieurs fois pour le même élément.

---

## Réflexion sur les participations

Au départ, j'avais défini une règle indiquant qu'un utilisateur ne pouvait soumettre qu'une seule participation pour un même challenge.

En construisant le MLD, je me suis demandé si cette limitation était réellement nécessaire.

Comme GamerChallenges est une plateforme basée sur les défis, j'ai finalement décidé qu'un utilisateur pouvait tenter plusieurs fois le même challenge et soumettre plusieurs vidéos.

J'ai donc modifié cette règle métier.

Un utilisateur peut maintenant soumettre plusieurs participations pour un même challenge.

Chaque participation correspond à une soumission distincte.

Cette décision permet également de conserver une structure `PARTICIPATION` simple sans ajouter de contrainte d'unicité sur la combinaison utilisateur/challenge.

---

## MLD obtenu

<pre>
USER (<u>id</u>, pseudo, email, mot_de_passe)

CHALLENGE (<u>id</u>, titre, nom_du_jeu, url_video, description, regles, #USER(id))

PARTICIPATION (<u>id</u>, url_video, #USER(id), #CHALLENGE(id))

VOTE_CHALLENGE (<u>#USER(id)</u>, <u>#CHALLENGE(id)</u>)

VOTE_PARTICIPATION (<u>#USER(id)</u>, <u>#PARTICIPATION(id)</u>)
</pre>

---

## Ce que j'ai compris

À la fin de cette session, je retiens principalement que :

- le MCD représente les données et les relations du point de vue métier ;
- le MLD transforme cette conception en structure relationnelle ;
- une relation 1:N peut être représentée par une clé étrangère ;
- une relation N:N nécessite une table de jointure ;
- une clé primaire permet d'identifier une ligne ;
- une clé étrangère permet de relier des tables ;
- plusieurs clés étrangères peuvent former ensemble une clé primaire composée ;
- une règle métier doit être justifiée par le fonctionnement réel de l'application et ne doit pas être ajoutée inutilement.

J'ai également compris qu'il est important de distinguer le vocabulaire utilisé selon le niveau de modélisation :

- dans le MCD, je parle notamment de **déterminants** ;
- dans le MLD, je parle de **clés primaires** et de **clés étrangères**.

---

## Prochaine étape

Passer du MLD au MPD afin de préparer l'implémentation de la base de données avec PostgreSQL.