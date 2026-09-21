# DevLog - Créer manuellement une Pull Request sur GitHub

## Situation

Après avoir terminé une fonctionnalité :

1. J'ai créé un commit sur ma branche.
2. J'ai envoyé la branche sur GitHub avec `git push`.
3. La branche existait bien sur GitHub, mais GitHub ne proposait pas automatiquement le bouton pour créer une Pull Request.

---

## 1. Vérifier que la branche existe sur le dépôt distant

```bash
git branch -r
```

Si la liste est longue, je peux rechercher directement ma branche :

```bash
git branch -r | grep inscription
```

Exemple :

```text
origin/feat/inscription
```

Cela confirme que la branche a bien été envoyée sur GitHub.

---

## 2. Créer la Pull Request manuellement

Sur GitHub :

1. Aller dans **Pull requests**.
2. Cliquer sur **New pull request**.
3. Sélectionner les branches :

```text
base: develop
compare: feat/inscription
```

- `base` = branche dans laquelle je veux intégrer mon travail.
- `compare` = branche contenant ma nouvelle fonctionnalité.

4. Vérifier les commits et les fichiers modifiés.
5. Cliquer sur **Create pull request**.
6. Vérifier la Pull Request.
7. Effectuer le merge dans `develop`.

---

## 3. Mettre à jour le dépôt local après le merge

Le merge réalisé sur GitHub modifie le `develop` distant.

Mon `develop` local n'est pas automatiquement mis à jour.

Je retourne donc sur `develop` :

```bash
git switch develop
```

Puis je récupère les dernières modifications :

```bash
git pull
```

---

## Exemple réalisé sur GamerChallenge

Branche de travail :

```text
feat/inscription
```

Pull Request :

```text
feat/inscription → develop
```

Après le merge de la Pull Request, j'ai exécuté :

```bash
git switch develop
git pull
```

Mon `develop` local a alors récupéré les modifications qui avaient été mergées sur GitHub.

---

## À retenir

`git push`, Pull Request, merge et `git pull` correspondent à des étapes différentes :

```text
Branche locale
     |
     | git push
     v
Branche distante sur GitHub
     |
     | Pull Request
     v
Proposition d'intégration dans develop
     |
     | Merge
     v
develop distant mis à jour
     |
     | git pull
     v
develop local mis à jour
```

- `git push` : envoie mes commits sur GitHub.
- **Pull Request** : propose d'intégrer ma branche dans une autre branche.
- **Merge** : intègre réellement les modifications dans la branche cible.
- `git pull` : récupère les modifications du dépôt distant dans mon dépôt local.

### Important

Le fait d'avoir fait `git push` ne signifie pas que ma fonctionnalité est déjà intégrée dans `develop`.

Il faut encore créer et merger la Pull Request.