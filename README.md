# STAGE 1 CEFIM : Mise en oeuvre d'un SIEM - ELK

## ELK : Elasticsearch - Logstash - Kibana

Elastic Stack - anciennement connue sous le nom de ELK Stack - est une collection de logiciels open-source produite par Elastic qui permet de rechercher, d’analyser et de visualiser des journaux générés à partir de n’importe quelle source dans n’importe quel format, une pratique connue sous le nom de journalisation centralisée. La journalisation centralisée peut être utile pour identifier des problèmes avec des serveurs ou des applications, car elle permet de consulter tous les journaux en un seul endroit. Elle est également utile car elle permet d’identifier les problèmes qui concernent plusieurs serveurs, en corrélant leurs journaux pendant une période spécifique.

![Stack ELK](img/Stack_ELK.png)

La Elastic Stack comporte quatre composants principaux :

- Elasticsearch : un moteur de recherche RESTful distribué qui stocke toutes les données recueillies.
- Logstash : le composant traitement des données de Elastic Stack, qui envoie les données entrantes à Elasticsearch.
- Kibana : une interface web pour la recherche et la visualisation des journaux.
- Beats : des expéditeurs de données légers qui peuvent envoyer des données provenant de centaines ou de milliers de machines à Logstash ou à Elasticsearch.
  - Filebeat  : recueille et expédie les fichiers journaux.
  - Metricbeat : collecte les métriques de vos systèmes et services.
  - Packetbeat : recueille et analyse les données du réseau.
  - Winlogbeat : collecte les journaux des événements Windows.
  - Auditbeat : collecte les données du framework de vérification Linux et surveille l’intégrité des fichiers.
  - Heartbeat : surveille activement la disponibilité des services.

## Installation

### Prérequis

- java openJDK/JRE
- nginx en reverse proxy

### Script de bascule d'état des services

```bash
#!/bin/bash

ES=$(systemctl is-active elasticsearch)      # echo active ou inactive
KI=$(systemctl is-active kibana)
LG=$(systemctl is-active logstash)
CLR='\e[0m'
RED='\e[0;31m'
GRE='\e[0;32m'
YEL='\e[0;33m'

if [[ $ES != "active" ]]; then
    systemctl start elasticsearch > /dev/null 2>&1
        if [[ $? != 0 ]]
        then
        echo -e "${RED}ElasticSearch failed to start${CLR}"
        else
        echo "${GREEN}ElasticSearch is active${CLR}"
        fi
else
    systemctl stop elasticsearch
    echo "${YEL}ElasticSearch is inactive${CLR}"
fi

if [[ $KI != "active" ]]; then
    systemctl start kibana > /dev/null 2>&1
    if [[ $? != 0 ]]
    then
        echo "${RED}Kibana failed to start${CLR}"
    else
        echo "${GRE}Kibana is active${CLR}"
    fi
else
    systemctl stop kibana
    echo "${YEL}Kibana is now inactive${CLR}"
fi

if [[ $LG != "active" ]]; then
    systemctl start logstash > /dev/null 2>&1
    if [[ $? != 0 ]]
    then
        echo "${RED}Logstash failed to start${CLR}"
    else
        echo "${GRE}Logstash is active${CLR}"
    fi
else
    systemctl stop logstash
    echo "${YEL}Logstash is now inactive${CLR}"
fi
```

### Installations

Installer la stack dans l'ordre `elasticsearch -> kibana -> logstash` afin de respecter les dépendances.

#### 1 - Elasticsearch

Récupérer la dernière version d'elasticsearch (8.5)

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.5.2-amd64.deb
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.5.2-amd64.deb.sha512
shasum -a 512 -c elasticsearch-8.5.2-amd64.deb.sha512 
dpkg -i elasticsearch-8.5.2-amd64.deb

```

Démarrer le service une 1ère fois afin de créer le superuser et **noter le mot de passe généré** avec :

```bash
systemctl start elasticsearch
```

Pour arrêter le service :

```bash
systemctl stop elasticsearch
```

Et pour qu'il se lance à chaque démarrage de la machine :

```bash
systemctl enable elsticsearch
```

Il est également possible *à tout moment* de :

- Réinitialiser le mot de passe super-utilisateur :

```bash
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

- Générer un token pour Kibana :

```bash
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

- Générer untoken pour ajouter un node Elasticsearch

```bash
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node
```

Le fichier de configuration (TLS...) se trouve dans `/etc/elasticsearch/elasticsearch.yml`.
Spécifiquement pour le paquet Debian (.deb) un fichier de configuration (variables d'environnement) est créé à `/etc/defaut/elasticsearch`.

---

#### 2 - Kibana

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.5.2-amd64.deb
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.5.2-amd64.deb.sha512
shasum -a 512 -c kibana-8.5.2-amd64.deb.sha512
sudo dpkg -i kibana-8.5.2-amd64.deb
```

Pour activer le lancement au démarrage du system et activer le service :

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

#### 3 - Logstash

Logstash est un middleware (intergicielle) ETL (Extract-Transform-Load) qui a pour rôle de recevoir les données, les filtrer selons les besoins et les charger ensuite dans ElasticSearch.
Considérer Logstash comme un pipeline qui prend les données à une extrémité, les traite d’une manière ou d’une autre et les envoie à leur destination (dans ce cas, la destination est Elasticsearch).

![Logstash Pipeline](img/logstash_pipeline_updated.png)

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install logstash
```

Les fichiers de configuration de Logstash se trouvent dans le répertoire `/etc/logstash/conf.d`

#### 4 - Nginx

Nginx en reverse proxy permet d'accéder à l'interface web de Kibana *(normalement accessible uniquement en localhost)* depuis l'extérieur.
