---
layout: post
title:  "Splunk"
date:   2015-05-14 14:58:46
categories: Splunk
comments: true
---

# Introduction

Splunk, qu'est ce? 

> You see servers and devices, apps and logs, traffic and clouds. We see data—everywhere. Splunk® offers the leading platform for Operational Intelligence. It enables the curious to look closely at what others ignore—machine data—and find what others never see: insights that can help make your company more productive, profitable, competitive and secure. What can you do with Splunk? Just ask.

En résumé, Splunk est un applicatif closed source. Il ingère des datas de type logs et offre des features de data mining, expoitation, visualisation et extraction. 


* Combien coute Splunk? 
    * Il applique un modèle fondé sur la volumétrie de log/jour et sur les éditions/features. [Cf. cette page](http://www.splunk.com/en_us/products/pricing.html).
* Quelle sont les différentes éditions? 
    * Il existe plusieurs éditions: [Enterprise / Cloud / Free](http://www.splunk.com/en_us/products/splunk-enterprise/free-vs-enterprise.html) [/ Light](http://www.splunk.com/en_us/products/splunk-light/splunk-light-vs-splunk-enterprise.html) 
* Existe t il une version free? 
    * [La version free](http://www.splunk.com/en_us/products/splunk-enterprise/free-vs-enterprise.html) est la version Enterprise bridée. Elle limite les features et la volumétrie de logs à 500MB/day.

<!--more-->

# Existant Open Source ## 

La présentation [Monitoring Open Source pour Java avec JmxTrans, Graphite et Nagios](http://fr.slideshare.net/cyrille.leclerc/open-source-monitoring-for-java-with-graphite) 
est une bonne base présentant des outils Open Source.

### Metrics
* [Jmxtrans](http://www.jmxtrans.org/): Extraction des metriques exporté via JMX
* [Graphite](http://graphite.wikidot.com/): Stockage et exploitation des metrics (calculs...), rendu (image, texte...). Graphite se compose de carbon (listener), graphite (UI), whisper (stockage RRD)
* [Graphana](http://grafana.org/): Il permet de constituer des dashboards autour de metrics Graphite (entre autre); ce projet est une perle.  

### Logs
* [Logstash](https://www.elastic.co/products/logstash), [flume](https://flume.apache.org/): Collecte des logs 
* [Elasticsearch](https://www.elastic.co/products/elasticsearch): Stockage, query
* [Kibana](https://www.elastic.co/products/kibana): Exploitation des logs, visualisation et extraction. 

### Alerting
* [Seyren](https://github.com/scobal/seyren): Application d'alerting qui se branche à graphite. Il possède un nombre appréciable de canaux. Il nécessite MongoDB.


### Conclusion

Je pense avoir fait le tour. Il y a bien sur des variations, [InfluxDB](http://influxdb.com/) au lieu de graphite, Nagios au lieu de Seyren...
La liste des fonctionnalités des projets listés ci-dessous, une fois mis bout à bout, est conséquente. 
Quoi qu'il en soit le nombre d'applicatif impliqué dans la chaine est conséquente et la mise en haute disponibilité de chacun de ces élements pourrait faire l'objet d'un sujet dédié 
(eg. pour graphite [The architecture of clustering Graphite](https://grey-boundary.io/the-architecture-of-clustering-graphite/)). 

# Scenario
 
Peut on avoir le même niveau fonctionnel en utilisant Splunk?     

L'objectif va être d'exploiter les logs applicatifs et les metrics d'une application Java 

Les logs et les metrics seront écrits dans des fichiers de logs en utilisant logback. Il y aura deux types de fichiers en sortie (et donc deux appenders). 
Le premier stockera les logs dans un format faiblement structuré texte, il a pour but d'être lisible par des Humains. Le second stockera dans le format JSON
pour être lu par les Machines. Pour plus d'information voir l'article en définir portant sur le [logging]() 
 
Pour contraindre l'article à une taille raisonnable, Splunk et l'application seront colocalisées sur le même container. Je développerai cette partie dans la conclusion.

# Installation de Splunk

## Installation du package

DockerFile

{% highlight text linenos %}
FROM ubuntu:14.04
ADD package/splunk-6.2.3-264376-linux-2.6-amd64.deb /tmp/splunk-6.2.3-264376-linux-2.6-amd64.deb
RUN sudo dpkg -i /tmp/splunk-6.2.3-264376-linux-2.6-amd64.deb
{% endhighlight %}


Build and run the image

{% highlight bash linenos %}
sudo docker build -t splunk .
sudo docker run -t -p 8000:8000 -v $HOME/splunk/var:/opt/splunk/var/lib/splunk  -v /$HOME/splunk/apps:/opt/splunk/etc/apps -v $HOME/splunk/log:/var/log/myapp -i splunk  /bin/bash
/opt/splunk/bin/splunk start --accept-license --answer-yes
{% endhighlight %}

Pour un controle plus fin, je lance le container sur la commande bash puis je lance en daemon splunk


## Configuration
 
Splunk peut être configuré de plusieurs façon: ligne de commande, fichiers de configuration, interface REST, interface web. 
Les fichiers de configuration de splunk sont de [type ini](http://en.wikipedia.org/wiki/INI_file)   

Il est conseillé de centraliser les ajouts dans une application Splunk plutôt que de modifier directement les fichiers de configuration "$SPLUNK/etc/system".
La création d'une application est simple et peut se faire en ligne de commande ou via [l'interface web](http://docs.splunk.com/Documentation/Splunk/latest/AdvancedDev/BuildApp).


### Application

Je crée mon application `nlab` via l'interface web `/opt/splunk/etc/apps/nlab/`
![Splunk](/assets/2015-05-26-operational-intelligence-splunk/create-app.png)


### Indexes

Je créé deux indexes `NLAB_LOGS` et `NLAB_METRICS` qui vont servir à stocker respectivement les logs et les metrics  

La configuration des indexes est stockée dans fichier [`indexes.conf`](http://docs.splunk.com/Documentation/Splunk/latest/Admin/Indexesconf)
qui est stocké dans le répertoire de l'application `/opt/splunk/etc/apps/nlab/local`. 
Pas de fioriture, je ne mets que le strict minimum: le répertoire de stockage des données suivant leurs états. 

{% highlight ini linenos %}
[NLAB_LOGS]
homePath   = $SPLUNK_DB/nlab_logs/db
coldPath   = $SPLUNK_DB/nlab_logs/colddb
thawedPath = $SPLUNK_DB/nlab_logs/thaweddb

[NLAB_METRICS]
homePath   = $SPLUNK_DB/nlab_metrics/db
coldPath   = $SPLUNK_DB/nlab_metrics/colddb
thawedPath = $SPLUNK_DB/nlab_metrics/thaweddb
{% endhighlight %}

Le nom de la section permet de nommer l'index. Les propriétés `homePath`, `coldPath` et `thawedPath` définissent le répertoire de stockage des données suivant leurs états. 
Pour plus d'information sur la notion de staging et de bucket, voir la page suivante [How the indexer stores indexes](http://docs.splunk.com/Documentation/Splunk/6.2.3/Indexer/HowSplunkstoresindexes)


### Inputs

L'application va générer deux fichiers de logs: `app.log.json` et `metrics.log.json`

La configuration des inputs est stockée dans le fichier [`inputs.conf`](http://docs.splunk.com/Documentation/Splunk/latest/Admin/Inputsconf) 
 qui est stocké dans le répertoire de l'application `/opt/splunk/etc/apps/nlab/local`.

Les fichiers à monitorer sont définis dans le nom de la section.
> [monitor://<path>]
> * This directs Splunk to watch all files in <path>. 
> * <path> can be an entire directory or just a single file.
> * You must specify the input type and then the path, so put three slashes in your path if you are starting 
> at the root (to include the slash that goes before the root directory).

Des wildcards peuvent être utilisés dans le path pour monitorer des logs d'applicatifs suivant le même formalisme ou par typologie eg. `/var/log/httpd/*_access`.
 
{% highlight ini linenos %}
[monitor:///var/log/myapp/app.json]
index=NLAB_LOGS
sourcetype=NLAB_JSON

[monitor:///var/log/myapp/metrics.json]
index=NLAB_METRICS
sourcetype=NLAB_JSON

{% endhighlight %}


La propriété `index` permet de définir l'index de destination et la propriété `sourcetype` permet de caractériser le type de traitement à appliquer. 
> Primarily used to explicitly declare the source type for this data, as opposed
> to allowing it to be determined via automated methods.  This is typically
> important both for searchability and for applying the relevant configuration for this
> type of data during parsing and indexing.

C'est également une bonne pratique d'indiquer à Splunk le type de logs / traitement à appliquer plutôt que de le laisser inférer (même s'il est relativement bon à ce jeu là).
 

## Processing Properties (Props)

Cette section permet de définir le type de source que l'on vient d'utiliser dans la section précédante.

La configuration des props est stockée dans le fichier [`props.conf`](http://docs.splunk.com/Documentation/Splunk/latest/Admin/Inputsconf) 
qui est encore une fois stocké dans le répertoire de l'application `/opt/splunk/etc/apps/nlab/local`.

{% highlight ini linenos %}
[NLAB_JSON]
INDEXED_EXTRACTIONS=JSON
KV_MODE=none
AUTO_KV_JSON=false
{% endhighlight %}

`INDEXED_EXTRACTIONS: Tells Splunk the type of file and the extraction and/or parsing method Splunk should use on the file.`
`KV_MODE: Used for search-time field extractions only. Specifies the field/value extraction mode for the data.`
`AUTO_KV_JSON: Used for search-time field extractions only. Specifies whether to try json extraction automatically.`

L'extraction des fields est donc faite à l'indexation.


# Exploitation







