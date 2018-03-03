---
title: Démonstration
order: 7
---

# Utilisation de l'API

Maintenant que tout est en place, il est temps que je vous montre comment se servir de cette API. Commençons par envoyer quelques textes, qui serviront de base de comparaison. Vous pouvez utiliser n'importe quel fichier texte, et envoyer plusieurs fois le même. Le résultat de la recherche n'est pas vraiment ce qui nous intéresse ici. Mais où devons-nous envoyer ces fichiers?

Pour trouver l'URL utilisé par l'API, nous devons demander des informations sur le service à minikube :

```bash
$ API_URL=$(minikube service api --url)
```

Il est alors possible d'envoyer des fichiers en ligne, en remplaçant filepath par le chemin du fichier à uploader.

```bash
$ curl -X POST $API_URL/api/Texts/upload -F "file=@filepath"
```

Commencez par envoyer une dizaine de fichiers. On peut alors effectuer une recherche dans ces fichiers grâce à la requête suivante :

```bash
$ curl -X POST $API_URL/api/Texts/search?word=test
```

VOus pouvez bien sur changer le mot recherché. Si vos textes ne sont pas trop long, et étant donné que nous utilisons 3 Workers pour 10 textes, cela devrait prendre un peu plus de 4 secondes. Pour rappel, à chaque requête, le orker est mis en pause une seconde, pour simuler une tâche intensive. 

# Adaptons nos ressources !

Supposons maintenant que ce délai ne soit pas acceptable. Si nous recevons des dizaines de requêtes à la seconde, nous ne serons pas capables de les traiter. Augmentons le nombre de nos Workers, pour en avoir autant que de textes à traiter :

```bash
$ kubectl scale --replicas=10 deployment worker
```

En effectuant une nouvelle requête, vous devriez remarquer une diminution signicative du temps de réponse. En une simple ligne, nous avons fortement augmenté les capacités de notre application. Comme promis, les étapes pour démarrer une première fois notre architecture auront été longues et tortueuses, mais déployer de nouvelles instances est maintenant un jeu d'enfant. Voilà qui démontre les apports d'un cluster Kubernetes 
