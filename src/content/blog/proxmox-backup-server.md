---
title: "Proxmox Backup Server : la solution complète pour gérer vos sauvegardes efficacement"
description: "Un article sur mon aventure avec proxmox"
pubDate: 10/21/2025
heroImage: "/blog/pbs/cover.png"
---

Alors, aujourd’hui, on parle de backup. Ouais, ce truc qui fait briller les yeux des sys-admin, tech leads et DSI… et qui donne des sueurs froides à tout le monde quand ça plante. Bref, le sujet qui peut transformer une journée tranquille en apocalypse de disques durs.

Imagine un monde où chaque bit, chaque octet, chaque disque dur fait ce qu’on lui demande, exactement comme tu veux. Un monde parfait, où tu ne te réveilles jamais en mode « oh non, mon serveur a crashé ». Beau rêve, non ? Spoiler : ça n’existe pas… sauf peut-être dans la tête des chefs cyber-sécu qui passent leur temps à te hurler : « T’as pas fermé ta session en allant déjeuner ! ». Ouais… et toi tu réponds dans ta tête : « Et toi t’as pas backupé mon serveur, hein ? ».

#### Retour a la réalite

Mais quand on me dit qu’il n’y a pas de backup et que je vois le loustic en pleine croisade pour sauver le disque dur perdu après un magnifique CTRL+A+SUPPR qui a vidé tout le NFS… là je me dis : ok, je comprend ton délire. Vasi, prends mes disques, fais-moi un RAID 20 avec mirrors, réplique ça sur trois sites différents, et ajoute un petit backup hors de la planète.

Parlons backups. La résilience des données et leur conservation… c’est un vrai univers à lui tout seul, surtout quand on se demande comment le disque dur a décider de planter juste avant la deadline. Beaucoup trop vaste pour en faire le tour dans un petit article. Du coup, on va se concentrer sur ce que j’ai testé et expérimenté : PBS, alias Proxmox Backup Server.

#### Mon PBS

##### Bon deja c'est quoi ? 

C’est une solution de backup développée par Proxmox, donc pas un petit script bricolé à la va-vite dans un coin sombre du serveur. Simple, mais efficace. En gros, c’est comme si tes VM avaient leur propre garde du corps.

Et le mieux, c’est que tout se passe proprement, sans rituels bizarres ni incantations secrètes devant ton disque dur. Compression, vérification, sécurité… PBS s’occupe de tout, pendant que toi tu peux aller prendre un café ou te demander pourquoi ton serveur a toujours l’air de te regarder avec reproche.

##### Pourquoi j’ai choisi PBS

- Tout mon infra est déjà sous Proxmox, donc la compatibilité est naturelle.
- La mise en place est rapide, pas besoin de sortir la boîte à outils de 300 pages.
- Il y a un client Proxmox Backup pour sauvegarder des fichiers locaux.
- Compression et sécurité des backups intégrées.

##### Installation

Ducoup, je l’ai installé sur mon node-2. Sa mission : pouvoir backup mon premier node-1 ainsi qu’un serveur Git hébergé sur le free tier d’Oracle. 

![installation](/blog/pbs/installation.png)


##### L'integration avec Proxmox


L’intégration est parfaite : on sent que la fonction principale de PBS est de sauvegarder des VM entières. Une fois l’installation terminée, il ne reste plus qu’à l’ajouter dans l’interface Proxmox en tant que stockage.

![installation](/blog/pbs/add-proxmox-disk.png)



Il ne reste plus qu’à définir des règles automatiques pour les backups en choisissant le nouveau stockage. Ici, je vais sauvegarder ma VM principale, qui héberge mes services les plus importants, ainsi que ma VM firewall.

![installation](/blog/pbs/backup.png)

Et voilà, je peux lancer un backup à tout moment, sans stress.

Au départ, j’avais quelques aprioris : je pensais que lancer un backup allait freezer complètement la VM… mais en fait, je peux l’utiliser normalement pendant le processus.

Je croyais aussi que PBS allait copier tous mes disques à chaque fois, mais grâce à lvm-thin, seules les données réellement présentes sont sauvegardées. Cerise sur le gâteau : PBS compare les fichiers entre les backups et ne copie que ce qui a été ajouté ou modifié. Résultat, ça économise un max de place.

![installation](/blog/pbs/pbs-backup-vm.png)


Résultat : j’ai de la marge avant de saturer. 14 ans, tranquille…

![installation](/blog/pbs/pbs-backup-time.png)

##### Backup de mon serveur git

Actuellement, mon serveur Git est sur un autre serveur, dans un autre réseau. Comme le disait un vieux sage fou : « Exposer un service sur Internet, même sécurisé, c’est dangereux ». Du coup, je ne voulais pas forwarder PBS, même si je ne doute pas de sa sécurité.

Le support direct dans Proxmox est béton, mais le client de synchronisation est une autre histoire. Il existe un client officiel, mais c’est un vrai nid à problèmes.


###### Sur Linux, tu fonctionnes
Ma surprise : le client n’est dispo que sur Linux. Pas de version Windows officielle. Il existe un projet en Go fait par la communauté, qui fonctionne bien, mais c’aurait été pratique que le client de base soit exécutable sur Windows… adieu mon idée de sauvegarder mon ordinateur portable. 

###### Le CLI sera ton arme
Oui, je sais, CLI > GUI. Mais une interface, même en mode TUI, aurait été pratique pour créer rapidement des backups sans taper des commandes.

###### Des binaires ARM64, jamais tu ne compilera
Pas de binaires ARM64 officiels. Mon seul serveur à sauvegarder est en ARM64, j’ai donc dû me débrouiller pour trouver des binaires. Heureusement, un apôtre m’a montré la voie : il existe un repo avec des binaires ARM64 déjà compilés, merci beaucoup !

https://github.com/wofferl/proxmox-backup-arm64


##### Go Scripting

Du coup, j’ai choisi une solution simple : mon serveur se connecte à un VPN WireGuard, puis pousse tous les fichiers vers PBS. Et tout ça est orchestré par… le saint cron job, fidèle compagnon de toutes mes galères d’automatisation.

```sh
#!/bin/bash
set -euo pipefail

echo "=== $(date) : Début backup Forgejo ===" >> "$LOGFILE"

# Monter le VPN en mode silencieux
if ! wg show wg0 >/dev/null 2>&1; then
    echo "Connexion WireGuard..." >> "$LOGFILE"
    wg-quick up wg0 >/dev/null 2>&1 || { echo "Erreur WireGuard" >> "$LOGFILE"; exit 1; }
    sleep 3
fi

# Backup Forgejo
echo "Lancement backup Forgejo vers PBS..." >> "$LOGFILE"
export PBS_PASSWORD=12345
proxmox-backup-client backup forgejo.pxar:/home/ubuntu/forgejo/forgejo/ --repository oracle@pbs@192.168.1.256:datastore


# Démonter le VPN seulement si on l’a monté
if wg show wg0 >/dev/null 2>&1; then
    echo "Déconnexion WireGuard..." >> "$LOGFILE"
    wg-quick down wg0 >/dev/null 2>&1 || echo "Erreur déconnexion VPN" >> "$LOGFILE"
fi

echo "=== $(date) : Backup terminé ===" >> "$LOGFILE"

```

Ce que j’aime avec le bash, c’est que ça reste simple et puissant. Du coup, je partage mon script, parce que j’ai un peu galéré à trouver sur Internet des exemples qui faisaient exactement ce que je voulais.

Au final, après plusieurs relectures de la doc, le client PBS devient assez instinctif. Certes, on ressent un petit pincement de frustration à ne pas être guidé pas à pas, mais une fois qu’on a compris le truc, tout roule plutôt bien.


##### Mon avis 

En soi, PBS fait très bien ce pour quoi il a été conçu : sauvegarder des VM Proxmox. Le client pourrait bénéficier d’un meilleur support, mais il fonctionne déjà correctement, ce qui est déjà un bon point.

J’ai aussi découvert qu’il est possible d’ajouter un endpoint S3 pour stocker une copie des backups dans le cloud. Et pour les aventuriers, il y a une API pour automatiser tout ça. Pour moi, PBS, c’est un projet solide, avec un support fort côté Proxmox, mais qui pourrait devenir beaucoup plus général. Après, reste à voir si Proxmox décidera de pousser ces fonctionnalités.

Côté stockage, il gère RAID, ZFS, LVM et tout ce que Proxmox propose nativement.

C’est brut, simple, mais suffisant. L’installation et la configuration m’ont pris une soirée, et après ça, tout roule.



##### Le débat éternel du backup

Je vais être honnête : au début, j’étais pas hyper chaud pour sauvegarder mes VM entières.
Je voyais déjà le disque se remplir à la vitesse de la lumière, les sauvegardes de 200 Go qui s’enchaînent, et moi qui regarde la barre de progression comme un idiot en me demandant si c’était bien utile.

Mon idée de base, c’était plutôt “à l’ancienne” : un bon vieux point de montage NFS, un OpenMediaVault bien configuré, et des sauvegardes service par service.
Simple, propre, maîtrisé.

Mais avec Proxmox Backup Server, j’ai fini par me poser la vraie question : quelle est la meilleure façon de faire des sauvegardes aujourd’hui ?

Les deux écoles

En creusant un peu, je me suis rendu compte qu’il y a deux camps bien distincts dans le monde merveilleux du backup.

École n°1 : les anciens.
Ceux qui ont connu l’époque où on écrivait des scripts shell à la main, qui faisaient un pg_dump toutes les dix minutes, puis qui compressaient ça dans un coin obscur du serveur en priant pour que le cron ne plante pas.
L’approche est granulaire, efficace, et tu peux revenir à n’importe quel instant précis.

École n°2 : les modernes.
Ceux qui ne jurent que par les snapshots, Veeam, PBS, et qui dorment paisiblement parce qu’ils savent qu’ils peuvent rollback une VM entière en deux clics.
Le confort absolu, tant qu’on n’a pas besoin de revenir à “l’état d’il y a 10 minutes” mais plutôt “celui d’avant-hier”.

Mon avis de (faux) sage

Je pense que les deux approches ont leur place.
Pour les systèmes critiques — base de données, CI/CD, services sensibles — rien ne vaut une sauvegarde ciblée et fréquente.
Mais pour tout le reste (VM, lab, serveurs persos, trucs qu’on a juste pas envie de reconfigurer à la main), les backups “gros volume” à la Proxmox font parfaitement le taf.

En fait, PBS m’a surtout fait comprendre que le débat ne devrait pas être “quelle méthode est la meilleure”, mais plutôt “comment combiner les deux intelligemment”.

Et vous, vous êtes plutôt de quel avis ?
Venez en débattre sur LinkedIn, qu’on rigole un peu de nos configs “temporaires” qui durent depuis 2018. 😏