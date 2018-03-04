---
title: MongoDB
order: 3
---
Avant de pouvoir nous attaquer à notre application proprement dite, il nous reste un démon à mettre en place, notre cluster MongoDB. Il s'agit encore une fois d'une application ayant besoin de données persistantes, qui va donc être représentée par un Statefulset. En fait, la plupart des étapes vont beaucoup ressembler la mise en place du cluster RabbitMQ. 

# MongoDB

Le mode de fonctionnement en cluster de MongoDB est plus complexe que celui de RabbitMQ. Alors que toutes les instances du broker peuvent traiter un message, seule une instance de la base de données, le *Primary Node*, peut y écrire, pour assurer la cohérence des données. Dans la terminologie propre à MongoDB, on parle donc plutôt de Replica Set, et pas de cluster. 

Le principal intérêt sera donc d'améliorer la vitesse de lecture des données, mais aussi d'être résistant à la panne d'une machine. Le Primary Node d'un Replica Set n'est pas figé, et peut changer en cas de défaillance de celui-ci. 

# Configuration

Contrairement à RabbitMQ, nous n'allons pas utiliser de fichier de configuration, car toutes les options nécessaires sont disponibles directement en argument de la commande de lancement du démon MongoDB. Néanmoins, pour des configurations plus avancées, cette solution reste toujours possible, sur le modèle de notre installation de RabbitMQ. Nous avons tout de même besoin de Secret, pour définir un mot de passe et un utilisateur.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo
  labels:
    app: mongo
type: Opaque
data:
  mongo-user: dXNlci1tb25nbw==
  mongo-password: cmFuZG9tLXBhc3N3b3JkLW1vbmdv
```

Cette fois encore, ils sont encodés en base64. Pour authentifier le Replica Set, MongoDB utilise une clé, semblable au cookie Erlang de RabbitMQ. 

Nous avons aussi besoin de définir le fichier qui sera l'équivalent de notre erlang-cookie.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret.key
  labels:
    app: mongo
type: Opaque
data:
  mongo-secret.key: bW9uZ29zZWNyZXQK
```

À nouveau, il s'agit simplement d'une valeur encodée en base64, qui sera placée dans un fichier.


Nous pouvons alors définir notre Statefulset, accompagné de son Service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongod
spec:
  selector:
    matchLabels:
      role: mongo
  serviceName: mongodb-service
  replicas: 3
  template:
    metadata:
      labels:
        role: mongo
        replicatset: rs0
    spec:
      containers:
        - name: mongod-container
          image: mongo
          command:
            - mongod
            - "--bind_ip"
            - "0.0.0.0"
            - "--replSet"
            - rs0
            - "--auth"
            - "--clusterAuthMode"
            - "keyFile"
            - "--keyFile"
            - "/etc/secrets-volume/mongo-secret.key"
            - "--setParameter"
            - "authenticationMechanisms=SCRAM-SHA-1"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: secrets-volume
              readOnly: true
              mountPath: /etc/secrets-volume
            - name: mongo-persistent-storage
              mountPath: /data/db
            - name: init-script
              mountPath: /data/init.sh
              subPath: init.sh
          env:
          - name: MONGO_USER
            valueFrom:
              secretKeyRef:
                name: mongo
                key: mongo-user
          - name: MONGO_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mongo
                key: mongo-password
          - name: MONGO_DATABASE
            value: "demo"
      volumes:
        - name: secrets-volume
          secret:
            secretName: mongo-secret.key
            defaultMode: 256
        - name: init-script
          configMap:
            name: mongo-init-script
  volumeClaimTemplates:
  - metadata:
      name: mongo-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

À nouveau, on définit un Service qui sert à gérer le Statefulset, en précisant le port de destination. Comme on ne lui attribue pas d'IP au niveau du cluster, la base de données n'est pas accessible depuis l'extérieur. On utilise ensuite 3 replicas de notre base de données. L'image utilisée est l'image officielle de MongoDB, et il est possible de spécifier la commande exacte qui doit être utilisée pour lancer le Replica Set. 

La plupart des options sont utilisées pour imposer l'authentification. Celle-ci est réalisée à partir du Secret que nous avons créé auparavant, exactement de la même manière que le cookie Erlang identifiait le cluster RabbitMQ. On notera aussi l'option *bind_ip*, sans laquelle MongoDB n'écouterait que les requêtes venant de la machine elle-même.

On retrouve les variables d'environnement associées à l'utilisateur, et au mot de passe, ainsi que quelques volumes à monter: le fichier de clé permettant l'authentification du Replica Set, un stockage persistant pour sauvegarder les données contenues dans la base, ainsi qu'un script d'initialisation dont nous n'avons pas encore parlé. On ne peut donc pas encore créer le Statefulset.

# Initialisation du Replica Set

Un Replica Set doit ensuite être initialisé manuellement. Cette opération ne doit pas être effectuée dans un hook post-démarrage, car toutes les instances doivent avoir démarré pour pouvoir effectuer la synchronisation. Il va donc falloir lancer ce script manuellement. Commençons par insérer ce script dans un ConfigMap:

```yaml
apiVersion: v1
data:
  init.sh: |
    if [ "$HOSTNAME" == "mongod-0" ]; then
      mongo --eval 'rs.initiate({_id: "rs0", version: 1, members: [ { _id: 0, host : "mongod-0.mongodb-service.default.svc.cluster.local:27017" }, { _id: 1, host : "mongod-1.mongodb-service.default.svc.cluster.local:27017" }, { _id: 2, host : "mongod-2.mongodb-service.default.svc.cluster.local:27017" } ]});';
      sleep 20;
      mongo --eval "db = db.getSiblingDB('admin'); db.createUser({ user: \"$MONGO_USER\", pwd: \"$MONGO_PASSWORD\", roles: [{ role: 'userAdminAnyDatabase', db: 'admin' }]});";
      mongo -u $MONGO_USER -p $MONGO_PASSWORD --authenticationDatabase admin --eval "db = db.getSiblingDB(\"$MONGO_DATABASE\"); db.createUser({ user: \"$MONGO_USER\", pwd: \"$MONGO_PASSWORD\", roles: [{ role: 'dbOwner', db: \"$MONGO_DATABASE\" }]});";
    fi;
kind: ConfigMap
metadata:
  name: mongo-init-script
```

C'est la première instances qui va initier le cluster. On remarque déjà qu'il faut préciser comment joindre toutes les autres instances du Replica Set. Concrètement, cela signifie que notre script fera l'affaire pour une simple démonstration, mais que dans la pratique, il devrait être généré pour être cohérent avec la configuration voulue. De la même manière, si nous voulons ajouter dynamiquement une instance, il faudra reconfigurer manuellement le Replica Set. Ces différentes actions peuvent être automatisées par des outils comme Ansible, mais ne rentre pas dans le cadre de cette présentation.

Le StatefulSet peut maintenant être créé. Une fois toutes les instances démarrées, on peut executer la commande:

```bash
$ kubectl exec mongod-0 /bin/bash -- /data/init.sh
```

En fait, pour une raison que je n'ai pas encore comprise, cette commande doit même être lancé deux fois. L'utilisateur ne parvient pas à être créé dans la même commande que l'initialisation du Replica Set..

Notre Replica Set est maintenant opérationnel pour une utilisation basique, mais beaucoup reste à faire pour qu'il soit réellement utilisable en production.
