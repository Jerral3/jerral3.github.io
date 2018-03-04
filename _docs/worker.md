---
title: Worker
order: 5
---
Nous commençons enfin à nous approcher du but, puisque nous allons mettre en place l'un des éléments actifs de notre application ! L'implémentation de ce Worker n'étant pas le sujet principal de cette présentation, le code est disponible directement sur le GitHub du projet.

On y trouvera en particulier un fichier `requirements.txt` qui indiquera à Python les dépendances du projet, et le fichier `config.json` qui contient la configuration pour une utilisation *locale* du projet. Ce fichier ne devrait donc normalement pas être suivi par git, mais simplement être adapté à la configuration de la machine de chaque développeur. Il sera réécrit, comme nous le verrons plus tard.

Enfin, on y trouve le code à proprement parler. Nous ne rentrerons pas dans le détail de l'établissement de la connexion avec RabbitMQ. Contentons nous de savoir que ce Worker attend qu'on lui envoie un chemin de fichier, ainsi qu'un mot à y rechercher. Avec les fichiers que nous allons utiliser, la recherche est très rapide. Et comme nous utilisons minikube, c'est de toute façon le temps du même processeur qui est partagé par les Workers. Pour simuler une tâche longue, et un environnement multi-machine, on ajoute donc une pause de une seconde dans le programme.

Voilà, vous savez tout ce qu'il est essentiel de savoir sur ce Worker !

# Revenons au stockage

Après avoir mis en place un beau cluster GlusterFs à l'étape précédente, je vais devoir vous décevoir... Avec la version actuelle de minikube, nous ne pouvons pas utiliser les volumes définis. Attention, je ne parle bien que des Volumes au sens de Kubernetes, et pas du volume GlusterFs, qui va lui bien nous servir !

J'ai quand même tenu à présenter la marche à suivre, car celle-ci permet de bien comprendre comment sont censés fonctionner les Volumes, et aussi car cette configuration peut vous servir si vous n'utilisez pas minikube. Le problème vient de la machine virtuelle utilisée : elle ne contient pas de client GlusterFs. Il serait possible de l'y installer, mais l'OS utilisé est TinyLinux, et il faudrait en fait recompiler entièrement le client, et donc installer avant cela tous les outils nécessaires à la compilation.

Cette installation dépassant totalement le cadre de ce tutoriel, nous allons utiliser une autre approche : nous allons utiliser un script de démarrage, qui montera le volume GlusterFs une fois le Pod démarré, et juste avant de lancer la commande démarrant réellement le Worker.

# Les Dockerfiles

Pour le moment, notre code n'est... Qu'un code. Or, nous ne savons utiliser que des images Docker. Il va donc falloir en construire une à partir de notre application. Pour cela, on utilise le Dockerfile suivant:

```dockerfile
FROM python:3.6.4-slim-stretch

RUN mkdir -p /worker

WORKDIR /worker

COPY . /worker

RUN pip install -r requirements.txt

ENTRYPOINT ["bash", "start.sh"]
```

La première ligne indique l'image Docker de base, sur laquelle nous allons greffer notre application. Ici, on utilise Python 3.6.4. Nous créons ensuite un dossier, qui sera à partir de maintenant le dossier courant. Puis, on y installe tout le code de l'application. Les chemins utilisés sont relatifs, il faudra donc construire l'image depuis le dossier du projet. 

Les fichiers sont maintenant disponibles, et nous pouvons installer les dépendances que nous avions spécifiées. Enfin, on termine avec le point d'entrée, qui indique à notre image la tâche principale qu'elle doit effectuer, à savoir lancer notre application. Enfin, dans notre cas, le script qui va tout d'abord monter le volume GlusterFs, et ensuite seulement lancer l'application ! Pas si compliqué non? Mais nous n'avons pour le moment qu'un fichier, et pas encore d'image Docker..

# Utiliser le Dockerfile

Nous allons nous servir de ce fichier pour construire une image. Dans le cadre d'un vrai projet, nous disposerions d'un *Docker repository*, qui est l'équivalent Docker d'un *git repository*. Nous pourrions ensuite utiliser cette image à partir de n'importe quelle machine. Dans notre cas, c'est à un processus Docker interne à minikube que nous allons envoyer notre image. Il faut donc utiliser les commandes suivantes, en se plaçant dans le répertoire correspondant au code du Worker :

```bash
eval $(minikube docker-env)
docker build -t worker-example .
```

Avec le flag `-t`, l'image reçoit un nom qui nous servira à l'identifier. Le Dockerfile présent dans le répertoire courant est ensuite automatiquement utilisé pour construire l'image, et l'ajouter à la liste des images connues de minikube. Et voilà, il ne nous reste plus qu'à créer des Pods !

# Créons nos Pods

Enfin, presque. Avant de créer des Pods, nous devons adapter la configuration à celle de notre cluster. Pour le moment, le Worker est configuré pour chercher RabbitMQ à l'adresse *localhost* qui est indiquée dans le fichier de configuration. Mais dans notre cluster, RabbitMQ est placé derrière un service du même nom. C'est donc à l'adresse *rabbitmq* qu'il va falloir le chercher. De la même manière, nous allons utiliser un utilisateur et un mot de passe qui ne seront pas forcément les mêmes que sur l'ordinateur d'un développeur. Pour toutes ces raisons, nous allons à nouveau avoir recours à un ConfigMap :

```yaml
apiVersion: v1
data:
  worker.config: |
    {
        "host": "rabbitmq",
        "port": 5672,
        "user": "user-rabbit",
        "password": "random-password-rabbit",
        "queue": "textQueue"
    }
kind: ConfigMap
metadata:
  name: worker
```

Il a exactement la même forme que la configuration utilisée par le Worker, mais avec des valeurs différentes. Tout est maintenant prêt, attaquons ! Tout comme les Pods RabbitMQ et MongoDB étaient gérés par des Statefulsets, les Pods de nos Workers, qui eux n'ont pas d'état, sont gérés par un Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    role: worker
  name: worker
spec:
  replicas: 3
  selector:
    matchLabels:
      role: worker
  template:
    metadata:
      labels:
        role: worker
    spec:
      containers:
      - image: worker-example
        name: worker
        imagePullPolicy: IfNotPresent
        securityContext:
          privileged: true
        volumeMounts:
          #- name: gluster-persistent-storage
          #  mountPath: /data
          - name: config
            mountPath: /worker/config.json
            subPath: worker.config
      securityContext:
        supplementalGroups: [0]
      volumes:
        #- name: gluster-persistent-storage
          #persistentVolumeClaim:
            #claimName: glusterfs
        - name: config
          configMap:
            name: worker
```

Ces fichiers devraient commencer à vous être familiers. En plus des metadata, nous précisons le nombre de Workers voulus, et dans la section *spec.template*, la nature exacte du Pod est définie. Nous utilisons l'image que nous venons de créer en indiquant simplement son nom. Celle-ci ne doit être retéléchargée que si elle n'était pas déjà présente. 

Nous devons ensuite autoriser le container à s'exécuter dans un contexte privilégié, car il va devoir monter notre volume GlusterFS. Celui-ci devra être monté dans le répertoire */data* de nos Pods. Il faut aussi monter le fichier de configuration adapté au cluster. Cette fois, nous ne montons pas un dossier mais bien un fichier, attention donc au changement de syntaxe !

Enfin, nous indiquons d'où proviennent ces deux volumes. Le premier s'obtient grâce à la *PersistentVolumeClaim* que nous avions définie lors du précédent article. Ou plutôt, ce serait le cas si nous n'avions pas commenté cette partie, pour utiliser à la place notre script de démarrage. Le deuxième vient d'être créé ci-dessus. 

Nous avons maintenant 5 Workers prêts à travailler, mais il nous manque toujours la tête pensante de notre application !
