<h1> Bienvenue dans cette installation de Elastic sStack avec Docker Comose </h1>

Alors que l'Elastic Stack a évolué au fil des années et que les ensembles de fonctionnalités se sont élargis, la complexité pour commencer ou tenter une preuve de concept (POC) localement a également augmenté. Bien que Elastic Cloud reste le moyen le plus rapide et le plus facile pour démarrer avec Elastic, la nécessité de développer et de tester localement est toujours très présente. En tant que développeurs, nous sommes attirés par des configurations rapides et un développement rapide avec des résultats peu exigeants. Rien n'égale une configuration rapide et une POC aussi efficacement que Docker, c'est sur cela que nous allons nous concentrer pour mettre en place l'intégralité d'un déploiement Elastic Stack pour votre plaisir local.

Dans la première partie de cette série en deux parties, nous plongerons dans la configuration des composants d'un Elastic Stack standard composé d'Elasticsearch, Logstash, Kibana et Beats (ELK-B), sur lequel nous pouvons commencer immédiatement le développement.

Dans la deuxième partie, nous améliorerons notre configuration de base et ajouterons de nombreuses fonctionnalités différentes qui alimentent notre pile en évolution, telles que APM, Agent, Fleet, Integrations et Enterprise Search. Nous examinerons également comment les instrumenter dans notre nouvel environnement local pour des fins de développement et de POC.


En tant que prérequis, Docker Desktop ou Docker Engine avec Docker-Compose doit être installé et configuré. Pour ce tutoriel, nous utiliserons Docker Desktop.

Notre attention pour ces conteneurs Docker se concentrera principalement sur Elasticsearch et Kibana. Cependant, nous utiliserons Metricbeat pour obtenir des informations sur le cluster, ainsi que Filebeat et Logstash pour des fonctionnalités basiques d'ingestion.


File structure :
Tout d'abord, commençons par définir la structure globale de notre système de fichiers.

├── .env

├── docker-compose.yml

├── filebeat.yml

├── logstash.conf

└── metricbeat.yml


D'accord, commençons par élaborer le fichier docker-compose.yml qui permettra de démarrer Elasticsearch et Kibana. Ensuite, nous ajouterons les fichiers de configuration YAML spécifiques pour Filebeat, Metricbeat et Logstash.

Très bien, créons un fichier .env pour définir les variables d'environnement qui seront utilisées dans notre fichier docker-compose.yml. Ces paramètres nous permettront de définir des ports, des limites de mémoire, des versions de composants, etc.

NB : Voir le fichier ***.env***

Notez que ici nous utilisons le mot de passe "elastic-stack" pour tous les mots de passe et la clé d'exemple sont utilisés à des fins de démonstration uniquement. 
Ils devraient être modifiés même pour vos besoins de POC locaux.

Comme vous pouvez le voir dans le fichier .env, nous spécifions les ports 9200 et 5601 respectivement pour Elasticsearch et Kibana. C'est également là que vous pouvez changer du type de licence "basic" au type "trial" afin de tester des fonctionnalités supplémentaires.

Nous utilisons la variable d'environnement STACK_VERSION ici pour la passer à chacun des services (conteneurs) dans notre fichier docker-compose.yml. Lors de l'utilisation de Docker, choisir de coder en dur le numéro de version plutôt que d'utiliser quelque chose comme la balise :latest est un bon moyen de maintenir un contrôle positif sur l'environnement. Pour les composants de l'Elastic Stack, la balise :latest n'est pas prise en charge et nous avons besoin de numéros de version pour extraire les images.

Voici le fichier docker compose de base 

<h2>
Mise en place d'un nœud Elasticsearch
</h2>


L'une des premières difficultés auxquelles on se heurte souvent lors du démarrage est la configuration de la sécurité. À partir de la version 8.0, la sécurité est activée par défaut. Par conséquent, nous devons nous assurer que le certificat CA est correctement configuré en utilisant un nœud "setup" pour établir les certificats. Activer la sécurité est une pratique recommandée et ne devrait pas être désactivée, même dans des environnements de POC.


<code>
version: "3.8"


volumes:
 certs:
   driver: local
 esdata01:
   driver: local
 kibanadata:
   driver: local
 metricbeatdata01:
   driver: local
 filebeatdata01:
   driver: local
 logstashdata01:
   driver: local


networks:
 default:
   name: elastic
   external: false


services:
 setup:
   image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
   volumes:
     - certs:/usr/share/elasticsearch/config/certs
   user: "0"
   command: >
     bash -c '
       if [ x${ELASTIC_PASSWORD} == x ]; then
         echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
         exit 1;
       elif [ x${KIBANA_PASSWORD} == x ]; then
         echo "Set the KIBANA_PASSWORD environment variable in the .env file";
         exit 1;
       fi;
       if [ ! -f config/certs/ca.zip ]; then
         echo "Creating CA";
         bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
         unzip config/certs/ca.zip -d config/certs;
       fi;
       if [ ! -f config/certs/certs.zip ]; then
         echo "Creating certs";
         echo -ne \
         "instances:\n"\
         "  - name: es01\n"\
         "    dns:\n"\
         "      - es01\n"\
         "      - localhost\n"\
         "    ip:\n"\
         "      - 127.0.0.1\n"\
         "  - name: kibana\n"\
         "    dns:\n"\
         "      - kibana\n"\
         "      - localhost\n"\
         "    ip:\n"\
         "      - 127.0.0.1\n"\
         > config/certs/instances.yml;
         bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
         unzip config/certs/certs.zip -d config/certs;
       fi;
       echo "Setting file permissions"
       chown -R root:root config/certs;
       find . -type d -exec chmod 750 \{\} \;;
       find . -type f -exec chmod 640 \{\} \;;
       echo "Waiting for Elasticsearch availability";
       until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
       echo "Setting kibana_system password";
       until curl -s -X POST --cacert config/certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_PASSWORD}\"}" | grep -q "^{}"; do sleep 10; done;
       echo "All done!";
     '
   healthcheck:
     test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
     interval: 1s
     timeout: 5s
     retries: 120
</code>


At the top of the docker-compose.yml we set the compose version, followed by the volumes and default networking configuration that will be used throughout our different containers.

We also see that we're standing up a container labeled “setup” with some bash magic to specify our cluster nodes. This allows us to call the elasticsearch-certutil, passing the server names in yml format in order to create the CA cert and node certs. If you wanted to have more than one Elasticsearch node in your stack, this is where you would add the server name to allow the cert creation.

Note: In a future post, we’ll adopt the recommended method of using a keystore to keep secrets, but for now, this will allow us to get the cluster up and running.

This setup container will start up first, wait for the ES01 container to come online, and then use our environment variables to set up the passwords we want in our cluster. We’re also saving all certificates to the “certs” volume so that all other containers can have access to them.

Since the Setup container is dependent on the ES01 container, let's take a quick look at the next configuration so we can start them both up: