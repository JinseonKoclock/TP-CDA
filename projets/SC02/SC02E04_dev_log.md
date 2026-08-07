# SC02E04 Dev Log

> 09/07/2026

## Préparation avant le challenge GitHub Actions

Avant de commencer le challenge sur GitHub Actions, j'ai préféré reprendre tranquillement les manipulations réalisées pendant le cours. Pendant la séance, je n'avais pas été suffisamment concentré et je sentais que je risquais de copier les commandes sans réellement comprendre ce que je faisais.

J'ai commencé par créer une nouvelle branche dédiée au challenge.

```bash
git switch -c Challenge_SC02E04
```

Je pensais pouvoir continuer rapidement, mais le premier `git commit` a immédiatement été bloqué par Husky.

Le premier message d'erreur ne venait finalement pas de Husky lui-même, mais d'un `package.json` invalide. Une simple erreur de syntaxe empêchait NPM de lire le fichier et le script `pre-commit` échouait automatiquement.

Après avoir corrigé ce problème, un second blocage est apparu.

Cette fois, ESLint indiquait qu'aucun fichier `eslint.config.js` n'était trouvé. J'ai vérifié son existence avec :

```bash
find -name eslint.config
```

Le fichier n'était effectivement pas présent.

Une fois la configuration ajoutée, Husky a enfin pu exécuter les différentes commandes prévues avant le commit.

Les tests d'intégration se sont tous exécutés avec succès :

* 7 tests exécutés
* 7 réussites
* aucune erreur

J'ai ensuite découvert que Husky ne lançait pas uniquement les tests. Après leur réussite, il exécutait automatiquement ESLint, qui a détecté 247 erreurs de qualité de code.

Sur le moment, je pensais que Husky était capable de corriger ces erreurs. J'ai donc essayé plusieurs commandes :

```bash
npx husky --fix
```

puis

```bash
npm husky --fix
```

Avant de comprendre que Husky ne corrige absolument rien.

Son rôle est uniquement de lancer automatiquement les commandes configurées dans les hooks Git.

C'est donc ESLint qu'il fallait réellement exécuter :

```bash
npx eslint --fix
```

La quasi-totalité des erreurs ont alors été corrigées automatiquement.

Il ne restait finalement qu'une seule erreur :

```ts
const body = await httpResponse.json();
```

La variable `body` n'était jamais utilisée. Il suffisait simplement de supprimer cette variable inutile :

```ts
await httpResponse.json();
```

Je pensais que cette préparation était enfin terminée... mais une nouvelle erreur est apparue lors du `git push`.

```text
sh: 0: Illegal option --
```

Au début, je pensais encore à un problème lié à Husky.

En vérifiant la configuration Git, j'ai finalement découvert que le chemin des hooks Git avait été complètement modifié par erreur :

```bash
git config core.hooksPath
```

Le résultat était :

```text
--fix/_
```

au lieu de :

```text
.husky/_
```

Cette erreur provenait de mes différentes tentatives avec la commande `husky --fix`.

Après avoir corrigé ce paramètre, j'ai enfin retrouvé une configuration cohérente.

Même si cette préparation m'a pris beaucoup plus de temps que prévu, elle m'a permis de mieux comprendre le rôle de chaque outil.

Je confondais encore Husky et ESLint au début.

Je commence à comprendre maintenant que :

* Husky ne vérifie pas la qualité du code ; il exécute automatiquement des commandes lors des événements Git (`pre-commit`, `pre-push`, etc.).
* ESLint analyse le code et signale les erreurs de qualité ou de style.
* Les tests vérifient le comportement de l'application, alors qu'ESLint vérifie la qualité du code.
* Git peut également être bloqué par une mauvaise configuration des hooks, ce qui explique pourquoi il est important de lire attentivement les messages d'erreur plutôt que de relancer les mêmes commandes au hasard.

Finalement, je n'ai même pas encore commencé le challenge GitHub Actions, mais cette préparation m'a déjà permis de mieux comprendre tout l'écosystème qui intervient avant même qu'un simple `git push` n'arrive sur GitHub.

---

### Petite aventure avec Git... avant même de commencer le challenge 😅

Je voulais simplement retrouver mon ancien dossier `docs_Jinseon`.

Je me suis souvenu que je l'avais créé pour séparer mes notes personnelles des fichiers de documentation du projet. Comme il avait disparu après plusieurs mises à jour du dépôt du formateur, je me suis dit qu'il suffisait probablement de fusionner mes anciennes branches pour le récupérer.

Mauvaise idée...

J'ai commencé un `merge` sur la branche `master`, puis je me suis rendu compte que je n'avais absolument pas besoin de faire cette fusion pour commencer le challenge.

Lorsque j'ai voulu revenir sur ma branche de travail, Git m'a répondu :

```text
fatal: cannot switch branch while merging
```

J'avais complètement oublié qu'un merge commencé doit être terminé (ou annulé) avant de pouvoir changer de branche.

En regardant le résultat de `git status`, j'ai découvert que plusieurs fichiers étaient en conflit (`levels.controller.ts`, `levels.router.ts`, etc.). Je me suis retrouvé bloqué... simplement parce que je voulais retrouver un dossier de notes.

Finalement, la solution était beaucoup plus simple :

```bash
git merge --abort
```

Cette commande a annulé la fusion en cours et m'a permis de revenir immédiatement sur ma branche `Challenge_SC02E04`.

Au passage, j'ai même créé un fichier nommé `witch master` par erreur en tapant une mauvaise commande dans le terminal... 😅

Moralité : avant de lancer un `merge`, il vaut mieux se demander si c'est réellement nécessaire. Aujourd'hui, j'ai perdu un peu de temps, mais j'ai aussi découvert comment Git protège le dépôt lorsqu'une fusion est en cours et comment revenir proprement à l'état précédent.


## Merge Conflict : ma première vraie galère avec Git

Après avoir terminé le challenge SC02E04, je pensais qu'il ne restait plus qu'à fusionner ma branche avec `master`.

Je ne m'attendais absolument pas à rencontrer un conflit de fusion.

Au début, j'ai vu apparaître plusieurs conflits (`.gitignore`, `package.json` et `package-lock.json`) et j'ai immédiatement cru que j'avais cassé le projet.

En voyant disparaître certains fichiers et apparaître un dossier `node_modules`, j'ai commencé à paniquer.

J'ai d'abord essayé de comprendre ce qui s'était réellement passé avant de modifier les fichiers.

En comparant mon dépôt avec celui fourni par O'clock, j'ai finalement compris que :

- `node_modules` ne devait jamais être versionné ;
- `.gitignore` était simplement en conflit entre les deux branches ;
- `package.json` et `package-lock.json` devaient être restaurés correctement avant de terminer la fusion.

Le conflit lui-même était finalement beaucoup moins compliqué que ce que j'imaginais.

J'ai ensuite découvert une autre étape que je ne connaissais pas : le **merge commit**.

Nano s'est ouvert automatiquement afin de confirmer le message :

```text
Merge branch 'Challenge_SC02E04'
```

Je ne savais même pas que Git ouvrait un éditeur pour terminer une fusion.

Après avoir enregistré le message et quitté Nano, la fusion locale était enfin terminée.

C'est seulement à ce moment-là que j'ai réellement compris la différence entre :

- `git merge` : fusionner les branches **en local** ;
- `git push` : envoyer cette fusion sur GitHub.

Jusqu'à présent, je pensais inconsciemment que la fusion était terminée dès le `git merge`.

En voyant le message :

```text
Your branch is ahead of 'origin/master' by 8 commits.
(use "git push" to publish your local commits)
```

j'ai enfin compris que GitHub ne connaissait pas encore ma fusion.

Cette mésaventure m'a également appris une autre chose importante : Git protège beaucoup plus qu'on ne l'imagine.

À plusieurs reprises, `git status` m'a permis de vérifier précisément la situation avant de faire une erreur.

Avec un peu de recul, cette journée m'a probablement appris davantage sur Git qu'une fusion qui se serait déroulée sans aucun conflit.

---

### Petite anecdote 😅

À un moment, j'étais tellement frustré que j'ai simplement refermé le capot de mon ordinateur portable... sans même l'éteindre.

J'ai laissé le projet de côté jusqu'au lendemain.

Finalement, revenir avec la tête reposée a été la meilleure décision.

J'ai repris les conflits calmement, un par un, et j'ai réussi à terminer la fusion sans casser le projet.

Cette expérience m'a confirmé qu'en cas de doute, il vaut mieux s'arrêter quelques heures plutôt que de modifier des fichiers au hasard.