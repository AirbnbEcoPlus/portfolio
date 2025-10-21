---
title: "Pourquoi j’ai tout migré en VM : mon aventure avec Proxmox"
description: "Le cluster proxmox"
pubDate: 10/12/2025
heroImage: "/blog/proxmox/cover.png"
---


### L'Avant

Il y a plus d'un an et demi j'ai monté un ordinateur avec un seul objectif, une machine de guerre capable de faire tourner n’importe quelle application gourmande, jeux vidéo compris eh eh. avec un budget recrac de 700 euros, après pas mal de galère, quelques négociations dignes d’un marchand de tapis sur Leboncoin et surtout beaucoup de récup, Le PC etait née.

Les composants : 
- i3-12100f
- 32gb de ram ddr4
- AMD RX 6700 XT
- 3 disque hdd qui font 2to de hdd
- 2 disque de 500gb de ssd et 1 de 256gb

Simple mais suffisant pour faire tourner tout ce que je lui balance dessus, sans compter le nombre de cycles (Les utilisateurs gentoo on vous voit...).

Sauf qu’avec le temps, j’ai commencé à détourner la bête : lancer des simulations, expérimenter des modèles d’IA, compiler des projets… tout ce qui fait transpirer la carte graphique ! 

Vous voyez le hic : la plupart de ces outils marchent bien mieux sous Linux, alors que mon install de base était sous Windows, oui je sais meme avec du matos AMD avec un peu de bidouille pour ROCM on s'en sort sur linux.

### Pourquoi pas un dual boot ? 

Je l’ai (comme beaucoup) envisagé… jusqu’à ce que je réalise :   

- Gérer plusieurs systèmes sur plusieurs partitions, c’est vite prise de tête.  
- Si un OS manque de place et que l’autre a plein d’espace libre, bon courage.  
- Et si un jour je veux passer d’Arch à NixOS juste pour tester ou builder un projet, c’est tout un mic-mac…
- Bref, c’est galère à gérer et trop rigide pour mes besoins qui changent tout le temps.
     

Autre souci : je voulais pouvoir, parfois, lancer un build tout en jouant. Le dual boot t’oblige à choisir ton camp à chaque démarrage : frustrant. 

### Pourquoi pas tout sous Linux ? 

J’ai tenté ! Franchement ça marche plutôt bien, le support gaming sous Linux a beaucoup progressé merci a Steam, dev du projet Wine et bidouilleurs. Beaucoup de jeux (même ceux protégés par des anti-cheats) passent sous Wine ou Proton, à force de tweaks. 

Mais…   

- Il y a encore des applis dont j’ai besoin qui ne tournent que sous Windows (coucou Delphi).  
- Et puis parfois, j’ai envie d’une vraie machine de test Windows pour checker la compatibilité de mes projets avec si possible un gpu.
     

Au fond, ce que je cherchais vraiment… 

…c’est de la liberté. Pouvoir changer de distribution sur un coup de tête, cumuler plusieurs systèmes en même temps (Linux, Windows…), passer de l’un à l’autre facilement sans tout installer, partitionner ou rebooter trois fois par jour. 



### Proxmox 

Du coup, la solution qui cochait toutes mes cases, c’était Proxmox. 

#### Pour les érudits qui ne connaitrait pas encore proxmox

Pour ceux qui ne connaissent pas : Proxmox, c’est en gros un chef d’orchestre pour ton ordinateur. Tu l’installes sur ta machine, et tu peux faire tourner plein de “petits ordis virtuels” à l’intérieur : un pour le dev, un pour le gaming, un autre pour tester des trucs…  

Pas besoin d’être une grosse boîte : n’importe qui peut l’installer pour jouer au chef d’orchestre de ses machines à la maison.

Pour gérer notre serveur proxmox il faut se connecter au panel d'administration (un site web) dans lequel on pourra gerer toutes nos machines virtuelles.

Déjà, j’utilisais l’outil pour une partie de mon infra perso, alors ça ne me dépayse pas trop. Avec Proxmox, je peux facilement ajouter ou déplacer du stockage, migrer mes VM à la volée, organiser mes datas en « pools » selon mes besoins… C’est plus modulable que mes excuses pour repousser les updates Windows.

Côté performances, c’est franchement bluffant : la carte graphique n’est pas bridée grâce au passage direct en PCIe, donc côté gaming ou calculs lourds, aucune perte. Pour le processeur et les disques, il peut y avoir une toute petite latence par rapport à du bare metal mais sincèrement, c’est optimisé depuis longtemps et je ne suis pas à quelques millisecondes près pour ce que je fais. 

Petit bonus, je peux démarrer la machine à distance avec le Wake-on-LAN : pratique pour lancer une VM sans être physiquement sur place. Et l’accès au panneau d’admin (et donc à mes VM) à distance, c’est parfait pour bricoler depuis n’importe où, que ce soit chez moi ou ailleurs. 

Bien sûr, il reste quelques inconvénients : 

- J’ai qu’une seule carte graphique, donc impossible de la partager entre plusieurs VM : il faut choisir laquelle aura la “patate graphique” à un moment donné.
- Avoir plusieurs VM, ça finit par prendre de la place côté stockage (et il faut un minimum d’organisation si on veut s’y retrouver).
     

Pour pallier tout ça, j’ai décidé de ne pas simplement faire une VM gaming, une VM dev, etc., mais plutôt de séparer mes services par VM selon leur besoin en matériel : par exemple, une VM juste pour le dev CPU, une autre avec l’accès GPU dédiée. Ça me permet d’optimiser les ressources sans avoir des machines qui dorment pour rien. 




## Mon plan 

Ducoup j'ai sortie ma meilleur clé usb ventoy (10 balles, un investissement stratégique) pour télécharger la dernière version de proxmox et hop après l'installation, un nouveau monde m'etait ouvert.


### 1. La creation du super-cluster 

J’avais déjà une machine sous Proxmox : le bon vieux serveur maison qui héberge mes trucs persos (Nextcloud, Jenkins, Git, Keycloak, un VPN pour faire sérieux).
Alors forcément, l’idée de tout réunir dans un “cluster” me démangeait : deux machines connectées, c’est déjà le pouvoir du bouton magique WakeOnLan, et la fierté d’avoir un “vrai” cluster à la maison (même si à deux nœuds, l’effet est surtout psychologique).
La vraie utilité ? Pouvoir éteindre “la machine de guerre” (850W, ça ne fait pas rire EDF) et la rallumer à distance pour jouer.

![Le cluster en question](/blog/proxmox/cluster.png)

Quel cluster de folie ! OK, c’est pas la NASA… Mais avouez, dans mon salon, ça a de la gueule. Un jour, les GAFAM me demanderont des tips. On y croit.

### 2. Le formatage des disques 

Là, c’était festival du stockage :   

- NVMe rapides (pour le système et les applis qui causent vite)
- vieux HDD récup d’un autre âge (pour les data qui ont moins peur du temps que moi)     

Ouais je sais du sata c'est vieux et pour du gaming ça coince souvent mais ça coute pas cher et c'est souvent de la récup. Rien ne vaut un bon disque mécanique qui perdra toutes ses données une fois qu'il sera tombé.

J’ai rangé tout ce bazar sans RAID “parano”, en me disant que le “NO MIRROR AD VITAM ETERNAM”, c’est un sport un peu risqué mais ça fait gagner de la place.


1. Le système proxmox sur un SSD NVME 256GB
2. Un pool LVM de 1TO avec deux SSD NVME 500GB
3. Un pool de 2TO avec tout les disques SATA


##### Les VM
On passe au plus interressant, LES MACHINES VIRTUELLES.

###### Gaming

![artemis](/blog/proxmox/artemis.png)

Pour la VM de gaming, j'essaie un nouveau concept qui marche super chez les GAFAM : le cloud gaming. En gros, j'ai mon serveur à distance qui dispose d'un environnement Windows, et je peux m'y connecter avec n'importe quel PC pour jouer avec une bonne latence. Cette magie est possible grâce à deux choses : déjà, la fibre (merci aux opérateurs !), et Artemis (forké de Sunshine, qui lui-même est inspiré de GeForce Experience). Bref, un petit logiciel qui permet, comme une session à distance ou TSE, de capturer l'écran de votre PC et de l'envoyer sur n'importe quel autre ordinateur. Sauf que la différence avec du RDP, c'est qu'il est boosté aux hormones et permet d'envoyer du 500 Mbps de bitrate à la seconde, de quoi augmenter exponentiellement la facture de ton abonnement 5G 200 Go en quelques heures. 

C'est super stable, on ne ressent même pas la latence et j’ai moi-même été super étonné — j'avais l'impression de jouer directement sur mon écran de PC. 

Voici les specs que j'ai mises pour cette VM : 

- 8 Coeurs
- 24 GB de ram
- 500GB SSD
- 1TO HDD

De quoi pouvoir jouer sans compromis, j'ai même automatisé la création de backup et téléchargé Playnite pour avoir une interface comme sur Xbox ou PlayStation.

![artemis2](/blog/proxmox/artemis2.png)

Je peux maintenant jouer chez moi sur mon écran de travail et à tout moment décider d’aller jouer sur la télé, ou même chez un ami ! Et sans perdre ma partie en cours.

###### Dev

Ça fait un moment que je vois passer partout le concept de remote coding : en gros, coder sur une machine puissante à distance, et garder son propre poste super léger. L’idée m’a vite séduit : pourquoi encombrer mon laptop du boulot ou mon PC principal de tonnes d’outils, alors que je peux tout centraliser dans une seule VM et déporter les calculs, les builds, et même l’intelligence pas très artificielle de VSCode ? 

J’ai sauté le pas : création d’une VM ArchLinux dédiée au développement, taillée sur mesure : 

- 8 cœurs, parce qu’il n’y a jamais trop de threads,
- 20 Go de RAM (Chrome, c’est pour toi),
- 100 Go de SSD pour la réactivité,
- 500 Go de HDD pour le stockage des projets et le « au cas où ».
     

L’installation ? Un bon vieux SSH bien configuré, un port ouvert, et voilà : je connecte mon VSCode à distance en deux clics, peu importe où je me trouve ou sur quelle machine locale. Plus aucun souci de compatibilité, plus d’environnement pourri à chaque nouveau projet, plus rien à casser chez moi. 

Et ça marche vraiment du tonnerre : 

- Mes dossiers et codes sont centralisés, jamais éparpillés entre trois ordis,
- Aucun build qui cale ma machine principale quand je veux faire autre chose à côté,
- Grâce à la montée en puissance du web, la plupart des apps que je développe restent légères côté client. Donc zéro regret d’avoir tout mis « à distance » !
- Et bonus, vu que j'ai choisis gnome a tout moment je peux me connecter en RDP pour coder des application graphique linux.
     

Bon, je ne vais pas mentir : pour des applis graphiques lourdes, le remote coding montre vite ses limites (oui, Figma over SSH, on repassera). Mais vu la tendance actuelle, difficile de justifier autrement : tout ce qui ne se gère pas dans le browser ou en client léger, ce n’est plus trop mon quotidien. 

Fun fact : c’est dans cette VM que je rédige cet article.

![dev](/blog/proxmox/dev.png)


###### Ai

Celle-là, c’est presque la jumelle de ma VM de dev… sauf que je lui ai offert le GPU ! L’objectif ? Héberger une instance OLLAMA et un OpenWebUI, pour pouvoir expérimenter toutes sortes de modèles d’IA à la maison. J’y prévois aussi un petit Jupyter Notebook et ComfyUI, histoire de tester, bidouiller, et jouer avec les modèles, sans polluer mon environnement principal. 

En gros, c’est la VM qui sert à tous les calculs lourds et aux expériences IA un peu folles. Si j’ai besoin de faire tourner un modèle, de benchmarker une démo IA ou juste de flinguer mon GPU le temps d’un week-end, c’est ici que ça se passe. 


- 8 Coeurs
- 16GB de ram
- 100 GB SSD
- 500 GB HDD

![ai](/blog/proxmox/ai.png)

Il ne manque plus qu'une interface qui me félicite avec des badges à chaque plantage de notebook… mais chaque chose en son temps.

### Conclusion

Bon ben voilà, j’ai présenté ma petite fierté du moment… sachant très bien que dans quatre mois, je serai reparti pour tout effacer/reconstruire dans une quête sans fin de “plus de perfs” ou “encore plus souple”. On se rassure comme on peut. 

Cet article a été rédigé à quatre mains avec GPT-4, ce qui m’a coûté la fortune de 0,6 centime. À ce rythme, la prochaine fois, je tente la sueur, le courage, et zéro IA. (À suivre…)

Si tu lis ça dans six mois, parie sur le fait que j’ai déjà tout modifié trois fois. Bref, à bientôt pour le compte-rendu du prochain démontage.