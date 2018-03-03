---
title: Introduction
---
Bienvenue dans notre voyage dans le monde de Kubernetes !

Dans cette série, nous allons essayer de démontrer les avantages offerts par des clusters basés sur Kubernetes. Pour cela, nous allons mettre en place une application minimale, déployée dans un cluster local, qui permettra de mettre en place tous les éléments nécessaires à une démonstration des possibilités offertes par Kubernetes. Autant vous prévenir tout de suite, ce ne sera pas une partie de plaisir. En fait, pour une installation aussi minimale que celle que nous allons mettre en place, ce n'est peut être même pas la solution la plus adaptée. Adapter son application à Kubernetes reste encore aujourd'hui complexe, mais une fois tous les éléments mis en place, il devient alors très facile d'adapter son infrastructure aux besoins en capacité de calcul, de mémoire, de stockage, et de bande passante. Alors, prêts? Partez !

# Application

Commençons par décrire notre application, et les différents éléments qui la composent. Nous allons simuler une application exposant une API pour être utilisée, et réalisant un travail long par le biais d'un *worker*. On pourra imaginer par exemple un algorithme de traitement d'image. Dans notre cas, et pour rester minimaliste, nous allons nous contenter d'une recherche de mots dans un texte. Pour simuler l'utilisation de plusieurs machines avec des temps processeurs distincts, nous mettrons le programme en sommeil une seconde.

Le principal objectif de notre installation sera de pouvoir démultiplier les workers, afin de répartir la charge de travail sur plusieurs machines. Il faut donc disposer d'un côté d'une API, et de l'autre, d'un Worker auquel elle transmettra des instructions. Pour pouvoir communiquer efficacement dans un environnement distribué, nous allons utiliser un Broker, qui est chargé de transmettre les messages. Cela permet de répartir les messages entre toutes les instances connectées, ainsi qu'à assurer certaines mesures de *Quality of Service*. Nous choisissons ici RabbitMQ, qui peut fonctionner en mode distribué. Enfin, notre API aura aussi besoin de stocker des données, et nous utiliserons ici MongoDB, qui peut aussi être déployé sur un cluster.

# Défis techniques

La mise en place et la configuration de ces différents outils présente déjà une certaine difficulté. Mais leur utilisation dans un cluster Kubernetes entraine aussi d'autres types de problèmes. Il faut donc trouver un compromis entre le besoin en passage à l'échelle nécessaire au produit, et la complexité technique de l'environnement de production. En particulier, nous traiterons dans cette série des problèmes suivants:

* le stockage distribué, permanent, et partagé de fichiers
* la gestion des environnements de développement et de production
* l'agrégation des logs
* la gestion d'un changement de version

# Docker

Cette présentation sera centrée sur Kubernetes, qui est essentiellement un outil de gestion de déploiement dans un cluster. Mais il nous faut tout de même des images à déployer ! Nous allons donc effectuer un bref rappel sur Docker, qui est la technologie de gestion des containers que nous allons utiliser. 

À la différence des machines virtuelles, lourdes et gourmandes en ressources, les containers n'utilisent pas la virtualisation. Le prix à payer est alors une isolation moins parfaite, même si celle-ci reste tout de même globalement assurée. On peut choisir parmi un grand nombre d'images préexistantes (python, node, etc), et simplement y installer notre application. Cela rend donc la tâche de configuration moins importante, du moins en théorie. Une fois notre image personnalisée construite, il n'y a plus qu'à la lancer avec Docker, et notre application est désormais en production !

# Kubernetes

Et Kubernetes dans tout ça? Et bien il s'agit de l'orchestrateur. Mais qu'est-ce qu'un orchestrateur? Nous allons garder cela pour un prochain article !
