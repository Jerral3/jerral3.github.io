---
title: Aller plus loin
order: 8
---

Si l'on regarde le travail accompli jusqu'à présent, nous pouvons voir tout le chemin qu'il reste encore à parcourir pour obtenir un cluster prêt pour la production.

# Continuous Delivery / Continuous Integration

Tout d'abord, un tel projet devrait être encore bien plus automatisé. Nos fichiers de configuration eux-mêmes devraient être parametrisés, afin de pouvoir être générés par des scripts, qui ne remplaceraient que quelques valeurs. 

De nombreux outils sont disponibles, le plus connu d'entre eux étant *Ansible*. Disposer de tels outils est indispensable si l'on veut utiliser un cluster non trivial en production. Le principe est simple: Ansible agit comme une sortie de glue entre tous les éléments de votre architecture. Alors, tout devient automatisable. Vous devez ajouter une instance de MongoDB à votre cluster, et cela nécessite quelques actions dans les Pods eux même? Vous pouvez créer un script Ansible pour cela. À chaque fois qu'un développeur envoie un commit sur la branche *master* de votre projet, vous devez lancer les tests unitaires, reconstruire l'image Docker correspondante, et effectuer un changement de version dans le cluster? Ansible peut aussi automatiser cela. Et bien plus encore, puisqu'il peut aussi être interfacé avec Nagios pour le monitoring, ou bien Slack et IRC pour vous prévenir directement.

Des alternatives à Ansible existe bien sûr, comme Puppet, ou Chef, qui était utilisé par Facebook. Le principe rest néanmoins assez similaire.

Pour gérer l'aspect intégration continue, ce sont des outils comme Jenkins, ou bien Travis qui pourront être utilisés.

# Agrégation de logs

Si vous avez suivi scrupuleusement les instructions présentées, vous ne devriez pas avoir rencontré trop de problèmes au cours de ce tutoriel. Pourtant, j'en ai moi même rencontré énormément.  Il est très probable que vous en rencontriez aussi, et dans ces situations, on se rend compte qu'il est bien plus difficile de debugger dans ces conditions. En effet, les Pods sont détruits et reprogrammés à chaque fois qu'une erreur fatale se produit. Même quand l'erreur est simplement informative, il faut retrouver par quel Pod elle a été émise. Imaginez que vous vous rendiez compte d'une défaillance émise par un des workers, mais que vous en ayez 10 000?

Avant de présenter de vraies solutions, je dois quand même vous informer de quelques astuces simples, utiles pour des clusters de petite taille, et donc pour du debugging. Tout d'abord, l'état de votre cluster est accessible avec `$ minikube dashboard`. Cela vous donne un aperçu de l'état global du cluster, et même quelques logs.

Si un Pod a connu une erreur fatale, et a été reprogrammé, il est aussi possible d'accéder à des traces de l'erreur, en utilisant la commande `$ kubectl logs --previous podName`.

Ces deux solutions restent très artisanales. Pour être vraiment efficace, il faut agréger les logs provenant des différents Pods. Pour cela, on peut utiliser un agent fluentd, qui fonctionnera sur chacun des Nodes du cluster. Il pourra alors envoyer les logs vers un système centralisé, comme ElasticSearch par example, et l'on pourra ensuite utiliser Kibana pour les visualiser. Pour s'assurer qu'un agent de logging est fonctionnel sur chaque Nodes, on pourra utiliser l'objet DaemonSet de Kubernetes.

# PodDisruptionBudget et autres indications

Un PodDisruptionBudget est indispensable dès lors que le cluster est composé de plusieurs Nodes. Son principal intérêt est de définir le nombre minimal d'instance qui doivent être fonctionnel en même temps. Imaginons que nous ayons à tout prix besoin d'au moins deux Workers fonctionnels, et que nous n'en ayons justement que deux. Sans PodDisruptionBudget, si l'un des Pods doit être reprogrammé, alors il sera d'abord supprimé de l'un des Nodes, avant d'être redémarré sur un autre. Avec un PodDisruptionBudget, il sera d'abord démarré, pour s'assurer que deux instances seront toujours en état de marche.

De la même manière, de nombreuses autres indications peuvent être apportées aux différents Pods du cluster. On peut par exemple gérer assez finement les besoins en ressources de calcul, de mémoire, ou d'accès au stockage. Il est important d'estimer correctement les besoins des différents composants en fonction du volume de traitement attendu.
