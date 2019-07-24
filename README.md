# chatavion
Système de messagerie sur DNS - Messaging system over DNS

ATTENTION ! CECI EST UN PROJET EXPÉRIMENTAL EXTRÊMEMENT INSTABLE. IL DOIT ÊTRE UTILISÉ UNIQUEMENT SUR MACHINE VIRTUELLE DÉDIÉE.
LES CONTRIBUTIONS SONT LES BIENVENUES. 

WARNING! THIS IS AN EXPERIMENTAL, HIGHLY UNSTABLE PROJECT. IT SHOULD BE USED ONLY ON A DEDICATED VIRTUAL MACHINE. CONTRIBUTIONS ARE WELCOME.

For more information in English, please see README.en.md

Chatavion est un système de messagerie fonctionnant uniquement par DNS. 
Il peut ainsi être utilisé sur tous les réseaux publics sans authentification ni paiement.
Il a été créé pour discuter depuis les réseaux Wi-Fi d'avion sans payer et a rempli ce rôle plusieurs fois.

Conçu initialement comme une preuve de concept d'envoi de message d'urgence en avion, aucun soin n'a été apporté à la conception ou au code (ça se voit très vite). C'est du bricolage et de l'assemblage de code dégueulasse. Mais ça marche (à peu près) !

Chatavion se décompose en 4 parties : 
 - Un client, utilisable en ligne de commande sous Android, conçu pour fonctionner avec Termux
 - Un serveur d'envoi (SEND) qui nécessite bash, un compilateur (gcc), le programme base32 et un serveur web (Apache)
 - Un serveur de réception (RECV) qui nécessite bash, cron (facultatif) et un serveur DNS (bind)
 - Une partie configuration DNS, qui peut être déléguée à un fournisseur comme freedns.afraid.org 
 
Ce fonctionnement est schématisé sur [ce forum](https://zestedesavoir.com/forums/sujet/12757/chatavion-une-messagerie-passe-partout/#p206189). 
 
1. Configuration DNS préalable

Chacun des serveurs doit disposer d'une adresse IPv4 fixe. Il est nécessaire d'attribuer un nom de domaine à chacun de ces serveurs. Exemple au hasard :

chatrecv.ca    66.66.166.166

chatsend.ca    33.33.133.133

Ensuite, deux autres domaines doivent renvoyer les requêtes reçues à ces serveurs, qui tâcheront de les interpréter. 
Pour cela, on définit des entrées de type NS. Exemple :

getmmsg.xx.yy   NS   chatrecv.ca

emgt.xx.yy      NS   chatsend.ca

2. Serveur d'envoi

Le serveur d'envoi est celui qui a l'adresse 33.33.133.133 et le nom de domaine chatsend.ca. 
Il est le nameserver du domaine emgt.xx.yy. 
Ainsi, lorsqu'une requête DNS demandant "ohyeah.emgt.xx.yy" est émise sur Internet, le serveur d'envoi la reçoit et l'interprète. 

Le système fonctionne avec un assemblage de plusieurs programmes. Il nécessite notamment apache et base32.

Chatavion étant un prototype extrêmement instable, le serveur utilisé peut planter sans prévenir. 
Il ne doit pas être utilisé pour autre chose. Tous les fichiers téléchargés depuis ce dépôt doivent être placés dans un même répertoire avec autorisation d'écriture. Toutes les commandes doivent être lancées en root.
On considère que le répertoire racine pour apache est /var/www/html/.

Le programme "sousdom" doit être compilé sur le serveur SEND. Le fichier source est sousdom.c. Exemple :
```gcc -o sousdom sousdom.c```

Le programme "rd2" doit aussi être compilé sur le serveur SEND. Le fichier source est recvdns.c. Exemple :
```gcc -o rd2 recvdns.c```

Une fois ces préparations effectuées, il n'y a plus qu'à lancer chat.sh en tâche de fond. Exemple :
```nohup bash chat.sh &```

chat.sh appelle rd2, qui attend une requête DNS. Quand une requête du type "ohyeah.emgt.xx.yy" est interceptée, rd2 enregistre le nom demandé dans un fichier "req". rd2 devait initialement renvoyer une réponse, mais je n'ai jamais réussi à forger un datagramme correct, alors j'ai laissé tomber cette partie, d'où ce bloc de code dégueulasse dans le fichier source.

chat.sh appelle ensuite sousdom, qui lit le fichier req et en extrait la première partie, celle qui vient avant le premier point (dans notre exemple, "ohyeah"). Ce programme en C est un vestige d'un autre projet très ancien et sa fonction pourrait être incluse directement dans le script. 

Pour des raisons techniques, les messages sont transmis en base32. La partie extraite est donc décodée avec base32.
Le programme ajoute la date et le préfixe "Avion : " au message, puis l'ajoute au bout d'un fichier (miaou.txt) directement dans le répertoire public du serveur apache. Ce préfixe permet de préciser que le message est émis depuis un réseau potentiellement instable, par opposition au préfixe "Terre" utilisé par un client web normal (non présent sur ce dépôt).

L'idée est que ce fichier texte sera récupéré par le serveur de réception pour être redistribué sous forme de DNS.
Le programme se bloque pendant 35 secondes pour éviter de récupérer de nouvelles tentatives d'envoi du même message, puis reprend du début.

3. Serveur de réception

Le serveur de réception est celui qui a l'adresse 66.66.166.166 et le nom de domaine chatrecv.ca. 
Il est le nameserver du domaine getmmsg.xx.yy. 
Ainsi, lorsqu'une requête DNS demandant "ohyeah.getmmsg.xx.yy" est émise sur Internet, le serveur d'envoi la reçoit et l'interprète. 

Comme pour le serveur d'envoi, il s'agit d'un prototype extrêmement instable, le serveur utilisé peut planter sans prévenir. 
Il ne doit pas être utilisé pour autre chose. Tous les fichiers téléchargés depuis ce dépôt doivent être placés dans un même répertoire avec autorisation d'écriture. Toutes les commandes doivent être lancées en root.

Le serveur RECV fonctionne selon un assemblage de plusieurs programmes, notamment le serveur DNS bind. Bind doit être installé avec sa configuration par défaut. Le fichier /etc/bind/named.conf.local doit être remplacé par le fichier named.conf.local présent sur ce dépôt. Il est nécessaire de remplacer getmmsg.xx.yy par le nom NS qui renvoit vers le serveur de réception.

Les fichiers nécessaires au fonctionnement du RECV sont avionfile.vierge, ip.sh, ip6.sh, dnavion.sh et cron.sh (facultatif si le serveur permet de définir des cron). avionfile.vierge doit être modifié en remplaçant getmmsg.xx.yy par le nom qui convient, comme pour named.conf.local. L'adresse IP 66.66.166.166 doit aussi être remplacée par celle du serveur de réception.

Dans dnavion.sh, même chose pour getmmsg.xx.yy et pour chatsend.ca, qui doit être remplacé par l'adresse du serveur d'envoi.

Pour faire fonctionner le serveur, bind doit d'abord être démarré, et dnavion.sh doit être lancé à intervalles réguliers, 
soit en programmant un cron qui le lance toutes les minutes par exemple, soit en lançant cron.sh en tâche de fond. Exemple :

```nohup bash cron.sh &```

dnavion.sh récupère miaou.txt (le log de conversation) depuis le serveur SEND. Il supprime tous les guillemets pour éviter des plantages.

Il réalise une copie d'un template vierge (avionfile.vierge). Ce fichier de configuration est rempli avec les messages du log selon le pattern suivant :
 - mX est un enregistrement de type texte (TXT) qui contient le message brut contenu à la ligne X
 - mX.nY est un enregistrement de type adresse IP (A) qui contient 4 caractères du message X avec l'offset Y transformés en valeurs numériques selon l'encodage ASCII (cette transformation est opérée par ip.sh)
 - mX.oY est un enregistrement de type adresse IPv6 (AAAA) qui contient 16 caractères du message X avec l'offset Y transformés en valeurs hexadécimales selon l'encodage ASCII (cette transformation est opérée par ip6.sh)
 
Ainsi, une requête DNS de type TXT sur m1.getmmsg.xx.yy renverra le premier message du log de conversation, m2.getmmsg.xx.yy, le deuxième, etc.

Les requêtes DNS de type TXT fonctionnent sur certains réseaux, comme le Wi-Fi du TGV, mais pas sur d'autres, comme le Wi-Fi de la compagnie aérienne ANA (c'est du vécu). 
Alternativement, il est possible de récupérer les messages sous forme de nombres, stockés dans des adresses IP. 
Ainsi, une requête DNS de type A (adresse IPv4) sur m1.n1.getmmsg.xx.yy renverra les 4 premiers caractères du premier message. m1.n2.getmmsg.xx.yy renverra les caractères 5 à 8 du premier message. m2.n1.getmmsg.xx.yy, les 4 premiers du deuxième message, etc. Selon le même principe, une requête DNS de type AAAA (adresse IPv6) sur m1.o1.getmmsg.xx.yy renverra les 16 premiers caractères du premier message.

4. Client

Un client a été conçu pour Android, pour fonctionner avec l'émulateur de terminal Termux. Il est composé des fichiers send.sh, recepauto.sh, ip2ascii.sh et ip6ascii.sh. Pour tout installer d'un coup, avec les paquets nécessaires, il est possible de télécharger et exécuter install.sh. Il est nécessaire de se placer dans un répertoire vide avec droits d'écriture. 

```wget https://raw.githubusercontent.com/vincesafe/chatavion/master/client/install.sh```

```bash install.sh```

send.sh permet l'envoi de messages. Dans ce fichier, il convient de remplacer emgt.xx.yy par le nom NS qui renvoit vers le serveur d'envoi.

Le réseau peut filtrer les requêtes DNS pour n'autoriser l'utilisation que d'un seul serveur, généralement celui fourni en DHCP. 
Dans ce cas, une application comme Network Info 2 permet de récupérer l'adresse de ce serveur. Il faut la renseigner dans le fichier "dnserv", précédée d'un @, comme ceci :

```echo "@1.2.3.4" > dnserv```

Le programme d'envoi s'utilise avec la commande suivante :

```./send.sh "voici un message"```

send.sh convertit le message en base32 et émet une requête vers (messageBase32).emgt.xx.yy. 
Si tout se passe bien, cette requête est interceptée par SEND et le message est enregistré. En cas d'échec du décodage base32, le message est ignoré. 
Comme expliqué plus haut, n'ayant jamais réussi à faire émettre une réponse correcte par rd2, il n'est pas possible d'avoir un retour immédiat du succès (ou non) de l'envoi. Une prochaine version, utilisant Node.js sur SEND, permettra de savoir si le message a été reçu.
Pour le moment, il faut utiliser un programme de réception pour voir la conversation.
Compte tenu de l'instabilité du système, éviter les caractères spéciaux augmente les chances de succès.

Le programme du serveur SEND bloque pendant 35 secondes l'envoi de messages après réception d'une requête valide. Il faut donc patienter au moins ce temps avant d'envoyer un nouveau message.

recepauto.sh émet tout simplement des requêtes DNS (m1.getmmsg.xx.yy, ...) pour réceptionner les messages du log. Au préalable, il faut remplacer getmmsg.xx.yy dans ce fichier par le nom NS qui renvoit vers le serveur de réception.
Il peut prendre 2 paramètres facultatifs : le serveur DNS à utiliser et l'offset. 
Il est conseillé d'utiliser le même serveur DNS que pour l'envoi. 
L'offset correspond à un numéro de ligne. On peut choisir de ne recevoir que les messages à partir du 5ème, par exemple. Cela évite de recharger les messages déjà lus, pratique si la discussion est longue et/ou le réseau lent. Exemple : 

```./recepauto.sh 1.2.3.4 5```

Lorsqu'il détecte qu'il n'y a plus de message à charger (échec de la requête DNS), le programme indique quel est l'offset suivant, pour s'économiser du temps lors de la prochaine réception.

recepauto.sh teste 3 modes de réception différents par ordre de rapidité. Le premier est le mode texte : les messages sont directement transmis sous forme de texte dans la réponse DNS. Ce mode est filtré sur certains hotspots. En cas d'échec, un message d'erreur s'affiche. Le deuxième mode utilise des adresses IPv6. Elles sont 128 bits, soit 16 octets, on peut donc y caser 16 caractères ASCII. Ce mode utilise ip6ascii pour convertir une adresse IPv6 en chaine de caractères et l'afficher. Le troisième mode utilise des adresses IPv4. Elles ne font que 4 octets, les caractères doivent donc être transmis 4 par 4. C'est le mode le plus lent.

La synchronisation du log de conversation (miaou.txt) se fait en fonction du cron (ou du script cron.sh) sur le serveur RECV. Il est normal d'avoir un délai d'une minute.
