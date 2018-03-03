---
title: Kubernetes
order: 1
---
Entrons maintenant dans le monde de Kubernetes. Pour ensuite faire passer notre application à l'échelle, nous ne pouvons pas nous contenter de lancer à la main de multiples instances de notre nouveau container. Nous avons besoin d'un orchestrateur, à la fois pour s'assurer que nous disposons du nombre souhaité d'instances de notre application, mais aussi pour gérer la communication entre les différents composants. Pour fonctionner efficacement, la mise à l'échelle doit en effet se faire à partir d'une architecture de *micro-service*. Gérer les multiples instances de ces multiples services va alors être le rôle de l'orchestrateur.

Si docker-swarm a un temps semblé prometteur, une majorité des applications fortement distribuées s'appuient désormais sur Kubernetes. La configuration est alors un peu plus complexe à mettre en place, mais nous allons maintenant en détailler les principaux éléments.

# Pré-requis


