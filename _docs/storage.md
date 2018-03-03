---
title: Storage
order: 4
---
Nos services sont presque en place, et nous allons bientôt pouvoir passer à notre application proprement dite. Il nous reste encore un dernier point à éclaircir : notre solution de stockage.

# Choix d'un système de fichier
Notre application travaille sur des fichiers. Il faut donc les stocker quelque part. Mais nous ne pouvons pas les attacher à un Pod, ni même à un Nœud. Les fichiers doivent être simultanément accessibles à tous les workers en lecture, et à toutes les instances de l'API en écriture, pour ajouter de nouveaux fichiers. De plus, ces fichiers doivent se trouver en dehors de notre cluster Kubernetes.

On aurait pu penser aux fournisseurs habituels, comme Amazon, Google Cloud, ou Azure, mais ceux-ci ne permettent pas d'avoir plusieurs écritures simultanées. Si nous avions seulement voulu la lecture simultanée, nous aurions pu utiliser Google Cloud. NFS aurait pu être une solution, mais c'est un système de fichier qui est très lent, et donc pas adapté à notre usage. Nous allons ici utiliser GlusterFS, qui répond à tous nos critères.

# Création du cluster
Le plus simple serait bien sûr de déjà disposer de volumes GlusterFS. Dans le cas contraire, il faudra se référer à la documentation, de votre distribution pour en installer un localement. Nous allons ici détailler les instructions pour ArchLinux, afin de détailler les opérations génériques à effectuer pour disposer d'un volume en bon état de fonctionnement.

Pour ArchLinux, il faut commencer par lancer le service glusterd. Cela doit normalement aussi lancer le service rpc-bind.
```bash
sudo pacman -S glusterfs
sudo systemctl start glusterd
```

Vous aurez ensuite besoin d'une partition vierge sur votre disque. Pour moi, il s'agit de `/dev/sda6`. Attention à bien remplacer cette valeur par votre partition vierge sous peine de perdre des données ! Je l'ai ensuite montée dans le répertoire `/data` créé pour l'occasion:

```bash
sudo mkdir -i /data
sudo mkfs.xfs /dev/sda6
sudo mount /dev/sda6 /data
sudo mkdir -p /data/brick1/gv1
```

Très bien, nous pouvons maintenant nous servir de ce nouveau dossier pour créer un volume GlusterFS. Attention, pour cela, vous devez disposer d'un hostname autre que localhost. Consultez la documentation de votre distribution pour cela, et remplacer ensuite ce hostname dans la commande suivante :

```bash
sudo gluster volume create gv1 hostname:/data/brick1/gv1
sudo gluster volume start gv1
```

Et voilà, votre volume GlusterFS est prêt ! Pour ceux qui veulent utiliser un volume GlusterFS déjà existant, nous avons ici utiliser gv1 comme nom de volume.  Vous pouvez utiliser le nom que vous voulez, mais veillez à bien le remplacer dans les fichiers de configuration qui vont suivre. 

# Création des volumes
Nous pouvons maintenant revenir à Kubernetes. Il va falloir expliquer à notre cluster comment utiliser ces volumes de stockage externes. Dans le cas d'un véritable stockage externe, l'IP du point d'accès sera connue. Ici, nous avons besoin de l'adresse IP de notre machine, vue par le cluster. Elle peut être obtenue dans la section `vboxnet0` de la commande `ifconfig`. Dans la plupart des cas, ce devrait être 192.16.99.1. Cela nous donne donc le point d'accès suivant :

```yaml
apiVersion: v1
kind: Endpoints
metadata:
    name: glusterfs
subsets:
    - addresses:
      - ip: 192.168.99.1
      ports:
      - port: 1
```

Pour sauvegarder ce Endpoint, nous devons aussi créer un service associé:
```yaml
apiVersion: v1
kind: Service
metadata:
    name: glusterfs
spec:
    ports:
    - port: 1
```

Ce point d'accès va maintenant pouvoir être utilisé pour provisionner un **PersistentVolume**. Cela donnera une possibilité de stockage au cluster, tout comme un Nœud ajoute une possibilité de calcul. Nous aurons ensuite besoin d'ajouter une **PersistentVolumeClaim**, qui pourra demander tout ou partie de cet espace de stockage, tout comme un Pod demande de la puissance de calcul à un Nœud. Commençons donc par notre PersistentVolume :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
    name: glusterfs
spec:
    capacity:
        storage: 10Gi
    accessModes:
    - ReadWriteMany
    glusterfs:
        endpoints: glusterfs
        path: /gv1
        readOnly: false
    storageClassName: gluster-storage
    persistentVolumeReclaimPolicy: Retain
```

Ici, nous provisionnons 10G de stockage. Adaptez au besoin cette valeur à la taille de votre partition. Nous précisons ensuite que cet espace est disponible en utilisant le Endpoint que nous venons de définir, et que le nom du volume est `gv1`. Encore une fois, adaptez ce nom si besoin. Nous allons maintenant pouvoir réclamer cet espace de stockage pour être utilisé par nos Pods.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: glusterfs
spec:
    accessModes:
    - ReadWriteMany
    resources:
        requests:
             storage: 10Gi
    storageClassName: gluster-storage
```

Les deux fichiers se ressemblent beaucoup, car nous demandons tout le stockage disponible, et que nous ne modifions pas les droits d'accès par rapport à ce que le PersistentVolume propose. Nous pourrions par exemple, pour le Worker, ne demander que des droits en lecture. Grâce à l'option *storageClassName*, on précise quel type de Volume utiliser.

# Conclusion

Et voilà, tout est prêt ! Je sais rien de bien sensationnel pour l'instant. Mais tous les éléments commencent à être en place, alors pourquoi ne pas utiliser ce stockage dans notre Worker?
