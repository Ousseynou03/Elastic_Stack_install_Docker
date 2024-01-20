<h1> Bienvenue dans cette installation de Elastic sStack avec Docker Comose </h1>

Alors que l'Elastic Stack a évolué au fil des années et que les ensembles de fonctionnalités se sont élargis, la complexité pour commencer ou tenter une preuve de concept (POC) localement a également augmenté. Bien que Elastic Cloud reste le moyen le plus rapide et le plus facile pour démarrer avec Elastic, la nécessité de développer et de tester localement est toujours très présente. En tant que développeurs, nous sommes attirés par des configurations rapides et un développement rapide avec des résultats peu exigeants. Rien n'égale une configuration rapide et une POC aussi efficacement que Docker, c'est sur cela que nous allons nous concentrer pour mettre en place l'intégralité d'un déploiement Elastic Stack pour votre plaisir local.

Dans la première partie de cette série en deux parties, nous plongerons dans la configuration des composants d'un Elastic Stack standard composé d'Elasticsearch, Logstash, Kibana et Beats (ELK-B), sur lequel nous pouvons commencer immédiatement le développement.

Dans la deuxième partie, nous améliorerons notre configuration de base et ajouterons de nombreuses fonctionnalités différentes qui alimentent notre pile en évolution, telles que APM, Agent, Fleet, Integrations et Enterprise Search. Nous examinerons également comment les instrumenter dans notre nouvel environnement local pour des fins de développement et de POC.


En tant que prérequis, Docker Desktop ou Docker Engine avec Docker-Compose doit être installé et configuré. Pour ce tutoriel, nous utiliserons Docker Desktop.

Notre attention pour ces conteneurs Docker se concentrera principalement sur Elasticsearch et Kibana. Cependant, nous utiliserons Metricbeat pour obtenir des informations sur le cluster, ainsi que Filebeat et Logstash pour des fonctionnalités basiques d'ingestion.


File structure
Tout d'abord, commençons par définir la structure globale de notre système de fichiers.

├── .env

├── docker-compose.yml

├── filebeat.yml

├── logstash.conf

└── metricbeat.yml