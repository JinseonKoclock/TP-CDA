# Carnet de bord – Migration de mon environnement Teleporter vers un nouveau PC

## Contexte

Pendant les vacances, j'ai souhaité préparer mon nouvel ordinateur afin d'être prêt pour la reprise des cours.

Mon objectif n'était pas seulement de copier des fichiers, mais de retrouver exactement le même environnement de développement que celui utilisé pendant toute la formation O'clock.

## Démarche

Avant toute manipulation, j'ai souhaité réduire l'espace occupé dans la machine virtuelle afin de faciliter sa sauvegarde et son transfert.

### 1. Nettoyage de Docker

J'ai commencé par vérifier l'espace utilisé par Docker :

```bash
docker system df
```

J'ai constaté que plusieurs gigaoctets étaient récupérables au niveau des images, des volumes et du cache de build.

J'ai ensuite arrêté les conteneurs encore actifs puis supprimé les ressources Docker inutilisées :

```bash
docker stop $(docker ps -q)
docker system prune -a --volumes
```

Cette opération m'a permis de récupérer environ **11,69 Go** dans la machine virtuelle.

### 2. Préparation de l'espace libre du disque virtuel

Après le nettoyage de Docker, la taille du fichier VDI visible depuis Windows restait toutefois inchangée.

J'ai donc rempli temporairement l'espace libre de la machine virtuelle avec des zéros :

```bash
sudo dd if=/dev/zero of=/EMPTY bs=1M status=progress
```

La commande s'est poursuivie jusqu'à ce que l'espace disponible soit rempli.

J'ai ensuite supprimé le fichier temporaire :

```bash
sudo rm -f /EMPTY
sync
```

Puis j'ai arrêté proprement la machine virtuelle :

```bash
sudo poweroff
```

Cette opération préparait le disque à un éventuel compactage, mais je n'ai finalement pas effectué le compactage avec `VBoxManage`, car j'ai obtenu entre-temps un disque dur externe de **1 To**. La contrainte de taille qui m'avait poussée à envisager cette opération n'était donc plus nécessaire.

### 3. Sauvegarde de la machine virtuelle

J'ai copié le dossier complet :

```text
VirtualBox VMs
└── Téléporteur v9.1.0
```

depuis mon ancien ordinateur vers le disque dur externe.

J'ai choisi de conserver cette copie sur le disque externe comme sauvegarde complète de mon environnement.

### 4. Installation de VirtualBox sur le nouvel ordinateur

Sur le nouveau PC, j'ai installé Oracle VirtualBox.

L'installation a d'abord été bloquée car **Microsoft Visual C++ Redistributable** était manquant.

J'ai installé la version x64 demandée, puis j'ai relancé l'installation de VirtualBox.

L'installateur m'a également signalé que les dépendances nécessaires aux bindings Python n'étaient pas présentes. Ces bindings n'étant pas nécessaires pour mon utilisation de la machine virtuelle, j'ai poursuivi l'installation sans eux.

### 5. Copie de la VM sur le SSD

Après avoir vérifié que la sauvegarde était présente sur le disque dur externe, j'ai copié le dossier :

```text
Téléporteur v9.1.0
```

dans :

```text
C:\Users\Home\VirtualBox VMs\
```

sur le SSD du nouvel ordinateur.

J'ai ensuite ouvert le fichier :

```text
Téléporteur v9.1.0.vbox
```

avec VirtualBox afin d'enregistrer la machine virtuelle sur le nouvel ordinateur.

Les problèmes rencontrés lors de l'enregistrement et du premier démarrage sont détaillés dans la section suivante.

---

## Difficultés rencontrées

Au premier démarrage sur le nouvel ordinateur, la machine virtuelle refusait de démarrer correctement.

Lors de l'installation de VirtualBox, l'assistant m'a d'abord demandé d'installer **Microsoft Visual C++ Redistributable** avant de pouvoir terminer l'installation.

Une fois VirtualBox installé, j'ai copié le dossier complet **Téléporteur v9.1.0** sur le SSD puis j'ai ouvert le fichier :

```text
Téléporteur v9.1.0.vbox
```

À ce moment-là, VirtualBox affichait le message :

> *The machine ... has the same UUID as an existing virtual machine.*

J'avais en réalité enregistré deux fois la même machine virtuelle : une fois depuis le disque dur externe, puis une seconde fois depuis le SSD.

Pour corriger ce problème, je suis passé par l'interface graphique de VirtualBox :

- clic droit sur **Téléporteur v9.1.0** ;
- **Supprimer...** ;
- laisser **décochée** l'option **Delete the virtual machine files and virtual hard disks** ;
- cliquer sur **Supprimer** afin de retirer uniquement l'enregistrement de la machine virtuelle.

Ensuite, j'ai rouvert le fichier :

```text
C:\Users\Home\VirtualBox VMs\Téléporteur v9.1.0\Téléporteur v9.1.0.vbox
```

La machine virtuelle s'est alors enregistrée correctement et Ubuntu a démarré.

### Problème de souris

Après le premier démarrage, Ubuntu fonctionnait mais la souris présentait plusieurs dysfonctionnements :

- le curseur disparaissait ;
- la souris restait capturée dans la VM ;
- la touche **Ctrl droite** ne permettait pas toujours de libérer le curseur.

Depuis le menu de VirtualBox, j'ai ouvert :

```text
Périphériques
→ Upgrade Guest Additions...
```

VirtualBox m'a indiqué que :

- Guest Additions dans la VM : **7.1.4**
- VirtualBox installé sur Windows : **7.2.14**

J'ai donc lancé la mise à jour des Guest Additions directement depuis VirtualBox.

Pendant l'installation, le curseur est progressivement redevenu visible et les interactions avec la souris sont redevenues normales.

Après la mise à jour, j'ai vérifié que :

- le curseur était visible ;
- la souris circulait correctement dans Ubuntu ;
- les fenêtres pouvaient être ouvertes normalement (Terminal, VS Code, etc.).

### Snapshot

Une fois tous les tests terminés, j'ai créé un snapshot afin de conserver un état propre de la machine virtuelle.

Nom du snapshot :

```text
Téléporteur propre - 07/08/2026
```

Ainsi, si un futur problème apparaît (mise à jour, mauvaise manipulation, installation...), je pourrai revenir rapidement à cet état fonctionnel.

## Vérifications

Après le démarrage, j'ai vérifié que mon environnement de développement était entièrement opérationnel :

- tous les dépôts Git étaient présents ;
- les dépôts conservaient leur remote GitHub ;
- Docker fonctionnait correctement ;
- Node.js était disponible ;
- les projets O'clock étaient identiques à ceux de mon ancien ordinateur.

J'ai également créé un snapshot intitulé :

> Téléporteur propre - 07/08/2026

afin de pouvoir revenir rapidement à cet état si nécessaire.

## Ce que j'ai appris

Cette migration m'a permis de mieux comprendre le fonctionnement d'une machine virtuelle VirtualBox.

J'ai également découvert la différence entre :

- une sauvegarde complète de la VM ;
- un snapshot ;
- l'enregistrement d'une machine virtuelle dans VirtualBox.

Je me sens désormais capable de déplacer mon environnement de développement vers un autre ordinateur sans repartir de zéro.