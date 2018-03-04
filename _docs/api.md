---
title: Api
order: 6
---
Nous voilà enfin au dernier de composant de notre architecture, l'API qui nous permettra de contrôler nos Workers. À nouveau, nous ne rentrerons pas dans les détails du code utilisé, qui pourra être trouvé sur GitHub.

Dans les grandes lignes, il s'agit d'une API NodeJS, fonctionnant avec le framework Loopback. Il va en particulier créer automatiquement les points d'accès à l'API, et gérer les accès à la base de données. Cette API permet d'uploader des textes en ligne, et d'effectuer une recherche par mot-clé sur tous les textes disponibles. Bien sûr, dans une véritable application, différents contrôles d'accès seraient effectués, mais il ne s'agit ici que d'un exemple. De plus, nous nous servons de notre cluster RabbitMQ pour transmettre les requêtes à nos Workers.

# Dockerfile

Cette fois ci, l'image de base sera donc celle de NodeJS. On crée un répertoire pour stocker le code, dans lequel on se place. Les dépendances de l'application, spécifiées dans le fichier *package.json* sont ensuite installées avec *npm*. Loopback utilise par défaut le port 300, c'est donc celui-ci que nous exposons. Il n'y a alors plus qu'à lancer le serveur NodeJS, en utilisant à nouveau notre script de démarrage, qui va d'abord monter le volume GlusterFs.

```dockerfile
FROM node:7.10

RUN mkdir -p /api
RUN mkdir -p /data

WORKDIR /api

COPY . /api

RUN npm install
EXPOSE 3000

ENTRYPOINT ["bash", "start.sh"]
```

Nous pouvons maintenant créer l'image Docker de l'API, depuis le répertoire où le code se situe :

```bash
$ eval $(minikube docker-env)
$ docker build -t api-example .
```

Comme dans le cas du Worker, nous avons ici des fichiers de configuration qui ne devraient pas être suivis par Git, car ils contiennent des paramètres spécifiques à notre environnement Kubernetes. Ils sont donc réservés à l'utilisation sur le cluster, et devraient être modifiés par chaque développeur pour refléter la configuration de son environnement de développement. Ces deux fichiers sont *server/config.json* et *server/datasources.json*.

# Deployment

Commençons par la configuration évoquée plus haut :

```yaml
apiVersion: v1
data:
  api.config: |
    {
        "restApiRoot": "/api",
        "host": "0.0.0.0",
        "port": 3000,
        "remoting": {
            "context": false,
            "rest": {
            "normalizeHttpPath": false,
            "xml": false
            },
            "json": {
            "strict": false,
            "limit": "100kb"
            },
            "urlencoded": {
            "extended": true,
            "limit": "100kb"
            },
            "cors": false,
            "errorHandler": false
        },
        "legacyExplorer": false,

        "rabbitMqProtocol":"amqp",
        "rabbitMqHost":"rabbitmq",
        "rabbitMqPort":5672,
        "rabbitMqUser":"user-rabbit",
        "rabbitMqPassword":"random-password-rabbit",
        "rabbitMqControlQueue":"textQueue"
    }
kind: ConfigMap
metadata:
  name: api-config
```

```yaml
apiVersion: v1
data:
  api.datasource: |
    {
        "db": {
            "name": "db",
            "connector": "memory"
        },
        "mongodb": {
            "name": "mongodb",
            "connector": "mongodb",
            "url": "mongodb://user-mongo:random-password-mongo@mongod-0.mongodb-service.default.svc.cluster.local:27017,mongod-1.mongodb-service.default.svc.cluster.local:27017,mongod-2.mongodb-service.default.svc.cluster.local:27017/demo?replicaSet=rs0"
        },
        "textStorage": {
            "name": "textStorage",
            "connector": "loopback-component-storage",
            "provider": "filesystem",
            "root": "/data/",
            "nameConflict": "makeUnique"
        }
    }
kind: ConfigMap
metadata:
  name: api-datasource
```

Il s'agit ici uniquement de quelques paramètres pour la connexion à RabbitMQ, et à MongoDB. Comme annoncé, l'URL utilisée pour la connexion à MongoDB précise toutes les instances du Replica Set, et ce fichier devrait donc être généré à partir d'un script. De la même manière, dans ces deux fichiers, les mots de passe ne devraient pas être écrits en clair. Nous avons choisi ici de simplifier la configuration.

Le Deployment va lui aussi beaucoup ressembler à celui utilisé pour le Worker :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    role: api
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      role: api
  template:
    metadata:
      labels:
        role: api
    spec:
      containers:
      - image: api-example
        name: api
        ports:
        - containerPort: 3000
        securityContext:
          privileged: true
        imagePullPolicy: IfNotPresent
        volumeMounts:
          #- name: gluster-persistent-storage
          #  mountPath: /data
          - name: api-config
            mountPath: /api/server/config.json
            subPath: api.config
          - name: api-datasource
            mountPath: /api/server/datasources.json
            subPath: api.datasource
      securityContext:
        supplementalGroups: [0]
      volumes:
        #- name: gluster-persistent-storage
        #  persistentVolumeClaim:
        #    claimName: glusterfs
        - name: api-config
          configMap:
            name: api-config
        - name: api-datasource
          configMap:
            name: api-datasource
```

On retrouve la création d'un container à partir de l'image qui vient d'être envoyée à Docker, ainsi que tous les éléments permettant de monter notre volume GlusterFS, et le fichier de configuration qui vient lui aussi d'être créé. 

# Accès à l'API

Tout est maintenant en place, mais notre API n'est accessible que depuis l'intérieur du cluster. Or, nous voulons qu'elle puisse être utilisée ! Pour cela, nous allons utiliser un Service de type LoadBalancer, qui va répartir les requêtes entrantes entre les différentes instances de l'API.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  labels:
    role: api
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
  selector:
    role: api
```

Nous voyons ici que toutes les requêtes arrivant sur le port 80 du Service seront redirigées vers les ports 3000 des instances de l'API. La commande suivante indique alors à quelle adresse envoyer nos requêtes, pour contacter le service. 

```bash
$ minikube service api --url
```

Et voilà ! Cette fois ci, nous en avons bel et bien fini avec la configuration ! Je vais maintenant vous montrer comment utiliser cette API, pour vérifier les performances de notre cluster.
