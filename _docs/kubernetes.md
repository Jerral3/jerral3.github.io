---
title: Kubernetes
order: 1
---
Entrons maintenant dans le monde de Kubernetes. Pour ensuite faire passer notre application à l'échelle, nous ne pouvons pas nous contenter de lancer à la main de multiples instances de notre nouveau container. Nous avons besoin d'un orchestrateur, à la fois pour s'assurer que nous disposons du nombre souhaité d'instances de notre application, mais aussi pour gérer la communication entre les différents composants. Pour fonctionner efficacement, la mise à l'échelle doit en effet se faire à partir d'une architecture de *micro-service*. Gérer les multiples instances de ces multiples services va alors être le rôle de l'orchestrateur.

Si docker-swarm a un temps semblé prometteur, une majorité des applications fortement distribuées s'appuient désormais sur Kubernetes. La configuration est alors un peu plus complexe à mettre en place, mais nous allons maintenant en détailler les principaux éléments.

# Pré-requis
Pour des informations plus détaillées d'installation, veuillez vous reporter à la documentation de votre distribution.

Commençons par installer Kubernetes. Pour simuler un cluster sur notre machine, nous allons utiliser *minikube*. Pour cela, il va commencer par télécharger une image VirtualBox, pour créer une machine virtuelle qui représentera notre cluster. La version de minikube utilisée pour réaliser ce tutoriel est la v0.25.0, mais des versions antérieures ne devraient pas apporter de changements trop importants.
Pour utiliser ce cluster, nous allons aussi avoir besoin de *kubectl*, qui permet d'interagir avec l'API de Kubernetes. Ma version est actuellement la v1.9.3. Avec une version antérieure à la v1.9.0, quelques changements sont à  prévoir dans les fichiers de configuration qui suivront. En particulier, les *Statefulsets* n'étaient alors disponibles que dans la version béta, et le champ *version* des fichiers de configuration devra donc être adapté. Si vous rencontrez d'autres difficultés avec des versions plus anciennes, souvenez vous de cela.

Enfin, nous allons installer Docker, pour construire nos images. J'utilise ici la version 18.02.

# Démarrage du cluster

Pour démarrer le cluster minikube, il suffit de lancer la commande suivante :

```bash
$ minikube start
```

Attention, la première fois, cela risque d'être un peu long, car minikube doit télécharger divers fichiers iso pour la machine virtuelle.

# Interactions avec le cluster

Maintenant que notre cluster est démarré, nous allons pouvoir commencer à y ajouter des éléments. On utilise pour cela *kubectl*. Tous les objets Kubernetes peuvent être créés grâce à cette commande. Cependant, cela devient vite complexe, et ne favorise pas la réutilisabilité. Il existe donc un deuxième moyen, en utilisant des fichiers de configuration au format yaml. On peut alors ajouter un élément à notre cluster :

```bash
$ kubectl create -f file.yml
```

Attention avec votre éditeur de texte favori: le yaml est un format particulier, où il ne faut pas mélanger espaces et tabulations.
D'autres commandes peuvent être particulièrement utiles pour le debugging :

```bash
$ kubectl get all                          # Décris tous les éléments de votre cluster
$ kubectl describe elementType elementName # Décris un élément en particulier
$ kubectl delete elementType elementName   # Supprime un élément du cluster
$ kubectl logs podName                     # Permet d'accéder aux logs d'un pod
```

Mais au fait, je ne vous ai même pas encore parlé des Pods ! Il va être temps de décrire le fonctionnement de Kubernetes. 

# Les objets Kubernetes

Les éléments de base d'un cluster Kubernetes sont les *Nodes* et les *Pods*. Un Node est tout simplement une machine faisant parti de notre cluster. En fait, vous ne manipulerez jamais de Node directement, et avec minikube, vous n'avez même pas besoin de les créer. Il s'agit simplement de puissance de calcul disponible. Un Pod, à l'inverse, représente une entité logique. Il s'agit de l'association d'une ou plusieurs images, Docker ou non, et de volumes, qui sont des unités de stockage. Tout le travail de Kubernetes consiste à programmer, démarrer, et gérer l'état de ces Pods, sur les différents Nodes à disposition. Pour chaque Pod, nous pouvons en spécifier le nombre voulu, ainsi que différents paramètres permettant de limiter leur accès au processeur, ou à la mémoire vive par exemple. 

En fait, nous n'allons pas vraiment manipuler de Pods non plus. Leur gestion sera déléguée à des objets de plus haut niveau.

## Les Deployments

Un Deployment est l'un des objets que l'on manipule couramment. En plus d'y spécifier toutes les caractéristiques d'un Pod, on y ajoute le nombre d'instance qui doivent être programmées sur le cluster. 

## Les Statefulsets

Un Statefulset ressemble beaucoup à un Deployment. On y spécifie aussi les caractéristiques d'un Pod, et le nombre d'instances à utiliser. Cependant, ces objets sont conçus pour des éléments ayant besoin de retenir un état. Contrairement au worker que nous allons utiliser, qui a seulement besoin de nouvelles instances pour être plus efficace, certains éléments, comme les bases de données, ont besoin de plus de stabilité. Ainsi, dans un Statefulset, les noms de domaines sont conservés, et les instances sont toujours démarrées les une après les autres, dans le même ordre. Ils permettent aussi de retenir des données, même lorsque le Pod associé rencontre une erreur. 

## Les Services

Un Service est un point d'accès unique à un groupe de Pod. Imaginons que nous lancions plusieurs instances de notre API. Quand une requête est effectuée, l'instance qui la reçoit n'est pas importante. Tout ce que nous voulons, c'est que la requête soit traitée par l'une des instances disponibles. Un Service permet de réaliser cela. Il dispose d'un unique nom de domaine à l'intérieur du cluster, ou bien d'une unique adresse IP à l'extérieur de celui-ci. Les requêtes peuvent alors être réparties entre les différentes instances appartenant au Service, selon certaines règles qui sont configurables.

## Les ConfigMap et les Secrets

Un ConfigMap est simplement un fichier écrit à l'intérieur d'un fichier yaml. Ce fichier est ensuite disponible à l'intérieur du cluster, pour être monté là où il sera nécessaire. Cela permet par exemple de remplacer un fichier de configuration générique, par un fichier de configuration spécifique au cluster.

Pour certaines valeurs sensibles, comme des mots de passe par exemple, on peut aussi utiliser des Secrets. Ce sont simplement des valeurs encodées en base64, accessibles aux Pods dans lesquels ils sont montés, mais qui ne sont pas visibles pour tous les utilisateurs du cluster.

## Les Volumes et les VolumeClaims

Ce sont les équivalents des Nodes et des Pods pour le stockage. Un Volume représente une unité de stockage disponible, comme l'était un Node. Une VolumeClaim est une demande d'accès au stockage, comme un Pod était une demande d'accès à des ressources de calculs. Des explications plus détaillées seront données dans l'article consacré au stockage.


# Conclusion

D'autres objets existent, et nous n'avons présenté ici qu'un nombre succinct des paramètres possibles de ces objets. Des explications plus approfondies seront données au fur et à mesure de cette série. Justement, et si nous commencions à déployer quelque chose dans le cluster?
