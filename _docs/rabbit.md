---
title: RabbitMQ
order: 2
---
Nous y voilà, nous allons déployer nos premiers **Pod** avec Kubernetes ! Ce n'est pas le plus simple à mettre en place, mais nous allons commencer par **RabbitMQ**, car nos autres composants auront besoin de s'y connecter

# RabbitMQ

RabbitMQ est un **broker**. Concrètement, il propose de mettre en place des tunnels, appelés **queues**, dans lesquels vont transiter des messages. Les applications peuvent alors publier, ou bien écouter ces queues, et donc se transmettre des messages.

Un premier intérêt réside dans le fait que les messages vont pouvoir être gardés en mémoire par RabbitMQ. On peut donc s'en servir dans un modèle maitre/esclave, et donner des t^aches à un **worker**. Si celui-ci n'est pas disponible, alors le message va être conservé, jusqu'à ce que qu'une application viennent le lire, et effectuer la tâche demandée. En y ajoutant un système de réponse, dit **d'acknowledgement**, on s'assure qu'aucune tâche ne sera oubliée.

Mais dans notre cas, c'est surtout la possibilité de distribuer le travail qui va nous intéresser. En effet, plusieurs applications différentes peuvent publier sur une queue, et plusieurs applications différentes peuvent aussi écouter. Dans ce cas, une seule d'entre elles accusera réception du message, et effectuera donc la tâche demandée. Avec 5 workers, nous allons donc pouvoir effectuer 5 t$aches en parallèle, et c'est bien tout le but de notre cluster.

Bien sûr, si l'ont veut déployer une instance de RabbitMQ dans un cluster, on ne peut pas se contenter de lancer plusieurs fois l'application. Celles-ci vont devoir partager des données, comme les queues et les messages. Il s'agit donc principalement de répartir la charge en cas de flux trop important de messages. Heureusement, RabbitMQ propose nativement un fonctionnement en mode cluster, et il suffira donc de connecter les différentes instances entre elles.

Les données doivent aussi être persistantes. En effet, en cas problème avec l'une des instances, elle va être reprogrammée, peut être sur un autre nœud. Or, nous ne voulons pas perdre les messages qui n'avaient pas encore été envoyés. Nous allons donc devoir recourir aux **Statefulsets** de Kubernetes, afin de retenir ces informations.

Voyons maintenant comment mettre tout cela en place !

# Configuration

Commençons par le plus simple ici, avec les fichiers de configuration de notre cluster RabbitMQ.

```yaml
apiVersion: v1
data:
  rabbitmq.conf: |
       default_vhost       = "/"
       #default_user        = "test"
       #default_pass        = "test"
kind: ConfigMap
metadata:
  name: rabbitmq
```

Rien de particulier ici: sous le nom de *rabbitmq*, nous préparons le fichier *rabbitmq.conf* qui ne contient que quelques valeurs par défaut. Nous le présentons avant tout à titre d'exemple, pour le cas où une configuration plus avancée serait nécessaire. À noter tout de même que nous utilisons ici le nouveau format de configuration pour RabbitMQ, qui ne sera pas compatible avec toutes les versions de RabbitMQ.

Nous n'avons pas spécifié l'utilisateur ni le mot de passe, car nous allons pour cela utiliser les *Secrets* de Kubernetes. 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
type: Opaque
data:
  rabbitmq-user: dXNlci1yYWJiaXQ=
  rabbitmq-password: cmFuZG9tLXBhc3N3b3JkLXJhYmJpdA==
  rabbitmq-erlang-cookie: cmFuZG9tLWNvb2tpZS1yYWJiaXQ=
```

Il s'agit simplement de valeur encodée en base64, obtenir à partir de commandes du type:
```bash
echo -n value | base64
```

Attention à bien utiliser le flag **-n** pour ne pas prendre en compte le retour à la ligne introduit normalement par *echo*.

En plus de l'utilisateur et du mot de passe, on spécifie aussi un cookie pour la machine virtuelle erlang. C'est ce cookie, s'il est identique sur toutes les instances, qui va permettre de les identifier, et de les autoriser à communiquer entre elles. Attention, si vous voulez par la suite changer ce cookie, il faudra penser à supprimer soit manuellement, soit en détruisant simplement le volume, le fichier `.erlang.cookie` du volume persistent, car le nouveau cookie ne sera sinon pas pris en compte.

# Services

Passons maintenant aux deux services à mettre en place. Le premier servira simplement à accéder à l'interface web permettant de surveiller et diriger le cluster une fois celui-ci en place. Le deuxième est le plus important, puisqu'il va permettre de répartir les requêtes entre les différentes instances, et de fournir un unique point d'accès à ces instances pour toutes les applications qui auront besoin de s'y connecter.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-management
  labels:
    app: rabbitmq
spec:
  ports:
  - port: 15672
    name: http
  selector:
    app: rabbitmq
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
spec:
  ports:
  - port: 5672
    name: amqp
  - port: 4369
    name: epmd
  - port: 25672
    name: rabbitmq-dist
  clusterIP: None
  selector:
    app: rabbitmq
```

Dans le premier Service, qui sera accessible via le nom `rabbitmq-management`, va retransmettre les requêtes vers le port 15672 de l'un des Pods portant le nom de `rabbitmq`, et cela correspond au port utilisé par le plugin de management.
Dans le deuxième service, nous allons pouvoir exposer les différents ports utilisés par RabbitMQ, et rediriger les requêtes vers les Pods du nom de `rabbitmq`. On ne donne pas d'IP au niveau du cluster à ce Service, et il ne sera donc accessible que depuis l'intérieur du cluster, contrairement au service de management qui doit pouvoir être accessible depuis une machine extérieure au cluster, afin qu'un administrateur puisse effectuer la maintenance nécessaire.

# StatefulSet

Nous arrivons maintenant au cœur du problème, avec nos premiers véritables Pods, définis à l'intérieur d'un Statefulset. Commençons par ce (long) fichier de configuration:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  selector:
    matchLabels:
      app: rabbitmq
  serviceName: "rabbitmq"
  replicas: 3
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
      - name: rabbitmq
        image: rabbitmq:3.7.3-management-alpine
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - >
                if [ -z "$(grep rabbitmq /etc/resolv.conf)" ]; then
                  sed "s/^search \([^ ]\+\)/search rabbitmq.\1 \1/" /etc/resolv.conf > /etc/resolv.conf.new;
                  cat /etc/resolv.conf.new > /etc/resolv.conf;
                  rm /etc/resolv.conf.new;
                fi;
                until rabbitmqctl node_health_check; do sleep 1; done;
                if [[ "$HOSTNAME" != "rabbitmq-0" && -z "$(rabbitmqctl cluster_status | grep rabbitmq-0)" ]]; then
                  rabbitmqctl stop_app;
                  rabbitmqctl join_cluster rabbit@rabbitmq-0;
                  rabbitmqctl start_app;
                fi;
                rabbitmq-plugins enable rabbitmq_management --online;
                rabbitmqctl set_policy ha-all "." '{"ha-mode":"exactly","ha-params":3,"ha-sync-mode":"automatic"}'
        env:
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              name: rabbitmq
              key: rabbitmq-erlang-cookie
        - name: RABBITMQ_DEFAULT_USER
          valueFrom:
            secretKeyRef:
              name: rabbitmq
              key: rabbitmq-user
        - name: RABBITMQ_DEFAULT_PASS
          valueFrom:
            secretKeyRef:
              name: rabbitmq
              key: rabbitmq-password
        ports:
        - containerPort: 5672
          name: amqp
        volumeMounts:
        - name: rabbitmq
          mountPath: /var/lib/rabbitmq
        - name: config
          mountPath: /etc/rabbitmq
      volumes:
      - name: config
        configMap:
          name: rabbitmq
  volumeClaimTemplates:
  - metadata:
      name: rabbitmq
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

Nous retrouvons tout d'abord les *metadata* t les *selectors* habituels, qui permette de sélectionner spécifiquement ce StatefulSet. On précise ensuite le nom du service responsable de sa gestion, ainsi que le nombre d'instances du Pod que nous voulons utiliser.

La section `template` va ensuite définir le Pod qui sera à répliquer au sein du StatefulSet. À nouveau, on spécifie donc les metadata. Au même niveau, on retrouve ensuite la définition des containers et des volumes utilisés. Commençons par les volumes: il n'y en a qu'un, et il correspond au *ConfigMap* que nous avons défini précédemment. Il sera disponible aux containers sous le nom de *config*.

Pour les containers, il n'y en aura à nouveau qu'un seul, du nom de *rabbitmq*. On spécifie alors l'image Docker à utiliser, qui est ici une image officielle prête à l'emploi, ainsi qu'un script à executer après le démarrage de celle-ci. Nous reviendrons sur ce script plus tard.

Les trois prochaines sections à concerner le container sont *env*, *ports*, et *volumeMounts*. Dans la première, nous spécifions quelques variables d'environnement à transmettre à l'image. Celle-ci est configurée pour s'en servir pour définir le cookie, ainsi que l'utilisateur par défaut. Ces valeurs sont sélectionnées dans le *Secret* que nous avons défini auparavant. Dans la section *ports*, nous spécifions que le seul port à exposer est le 5672, utilisé par le protocole amqp de RabbitMQ. Enfin, dans *volumeMounts*, on précise où monter les volumes mis à disposition du container.  Le second est le fichier de configuration que nous avions défini dans la section *volumes*. Mais d'où provient le premier? De la dernière section de ce fichier, *volumeClaimTemplates*. Elle met à disposition de notre container un espace de stockage de 1G, qui sera persistant, et monté dans le dossier `/var/lib/rabbitmq`, qui est le dossier où sont sauvegardées toutes les données utiles.

# Script de lancement

Revenons sur le hook *postStart*, qui est exécuté une fois que le container est démarré. La première partie édite le fichier **resolv.conf**, afin que les différentes instances puissent communiquer en se contactant via une requête DNS. 

Ensuite, nous attendons que l'instance de RabbitMQ soit réellement démarrée, et si nous ne sommes pas dans la première instance, *rabbitmq-0*, il faut rejoindre le cluster que cette première instance va former. On peut ensuite activer le plugin de management, et configurer les tests à réaliser pour vérifier que les autres instances du cluster sont toujours actives.

# Conclusion

Vous devriez maintenant avoir un cluster RabbitMQ prêt à l'emploi. Grâce à kubectl, on peut facilement ajouter de nouvelles instances dynamiquement, qui rejoindront automatiquement le cluster. Ce cluster est accessible à toutes les autres applications via le nom de domaine du service, *rabbitmq*, via le port 5672 qu'il expose.

C'était un gros morceau, surtout en introduction, et la suite devrait être plus facile. Mais nous avons tout de même un cluster MongoDB à mettre en place, et ce ne sera pas non plus une mince affaire ! Rendez-vous au prochain numéro !
