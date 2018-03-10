---
title: Conclusions
order: 9
---

Maintenant que ce large tour d'horizon des possibilités offertes par Kubernetes est terminé, il est temps d'en tirer quelques conclusions. Pour pouvoir vraiment parler des avantages et des inconvénients de Kubernetes, il faut le comparer à ses concurrents. Le principal à l'heure actuelle est Docker-Swarm. Une autre alternative serait aussi tout simplement de ne rien utiliser, et de continuer à utiliser ses machines de production comme avant. 

# Complexité d'utilisation
Nous l'avons démontré tout au long de cette série, l'utilisation de Kubernetes est assez complexe. Il faut du temps pour vraiment maitriser tous les outils mis à disposition, et il n'est pas toujours facile de savoir quel type d'objet utiliser.

Au contraire, Docker-Swarm est sensiblement plus simple à manipuler. Cela est notamment du au fait que nous utilisons des images Docker, mais cela veut aussi dire que nous sommes limités par.. L'API de Docker. 

Finalement, la complexité additionnelle se traduit par des fonctionnalités plus avancées. Kubernetes est certes plus compliqué à mettre en place, mais permet par exemple de gérer l'auto-scaling : si des ressources de calcul viennent à manquer, le cluster peut s'adapter et provisionner de nouveaux Pods.

Il est intéressant de noter que s'il est plus simple de déployer son application de manière traditionnelle qu'avec Kubernetes, ce n'est pas forcément vrai s'il l'on compare à Docker-Swarm. En effet, il est à peu près équivalent d'installer une application sur une machine virtuelle, et de lister toutes ces commandes dans un Dockerfile. Docker permet par contre ensuite de réitérer l'opération, sans risque de différences de configuration.

# Passage à l'échelle
Pour ce qui est du passage à l'échelle, nous n'aborderons bien sûr pas la méthode traditionnelle, puisque celle-ci est dans ce cas complètement dépassée.

L'utilisation de Pods dans Kubernetes, là ou Docker-Swarm se limite à des images, est un réel atout pour le passage à l'échelle. Dans notre série, nous les utilisions de manière quasi-équivalente. Mais pour une entreprise, coordonner le déploiement de plusieurs images au sein d'un même Pod peut être une fonctionnalité nécessaire.

Sans entrer dans ces considérations, Kubernetes permet tout de même de contrôler des clusters plus étendus, même si Docker-Swarm tolère quand même des clusters de l'ordre du millier de nœuds.

# Gestion des états
Pour ce qui est de la gestion des états, ce sont cette fois Docker-Swarm et Kubernetes qui ne peuvent pas se comparer à l'installation traditionnelle. Nous l'avons vu, que ce soit pour gérer une base de données, ou bien le stockage, l'utilisation de containers reste encore une vraie difficulté dès que l'application n'est plus *Stateless*. Bien sûr, ce choix se fait par contre au dépend de la flexibilité.

Il n'y a par contre pas de réels différences entre les deux outils de gestion de cluster, pour lesquels il est toujours un peu compliqué de retenir un état. Kubernetes propose tout de même de belles avancées dans ce domaine, avec l'apparition des Statefulsets dans la version stable. C'est dans ce domaine particulier que tous les outils d'orchestration de containers doivent encore se développer.

# Conclusion sur les alternatives
Finalement, il apparait assez évident que l'on ne peut même pas réellement parler d'alternative, tant les cas d'utilisation sont différents.

Des installations traditionnelles n'ont aujourd'hui plus beaucoup d'avenir, excepté pour les petites entreprises, qui n'en sont qu'à leurs débuts. Même dans ces situations, utiliser des images Docker, même sans recourir à Docker-Swarm, peut être intéressant. Cela nécessite par contre déjà une certaine expérience.

Pour des besoins un peu plus avancés, Docker-Swarm commence à devenir intéressant. Il est un peu plus compliqué à utiliser, mais reste assez accessible. Swarm est par exemple utilisé par la compagnie finlandaise de transport ferroviaire, mais cela reste assez marginal dans de très grandes entreprises.

Enfin, Kubernetes a été conçu pour assurer un maximum de flexibilité à ses utilisateurs, et cela vient bien sûr avec une complexité bien supérieure. Bien qu'objectivement supérieur à Docker-Swarm, il n'en reste pas moins pas forcément adapté à tous les cas d'usages, et notamment à celui des moyennes entreprises. Kubernetes est donc actuellement supporté par Microsoft Azure, Google Compute Engine, ou encore OpenShift. Il est de plus soutenu par Google, et utilisé par de grands groupes à l'échelle mondiale: Amadeus, eBay, ou encore BlaBlaCar. Il s'agit pour le moment de l'outil le plus utilisé, et de très loin, pour l'orchestration de container en production.
