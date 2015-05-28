---
layout: post
title:  "Splunk"
date:   2015-05-14 14:58:46
categories: Splunk
comments: true
---

# Introduction

> What Is Splunk?
> You see servers and devices, apps and logs, traffic and clouds. We see data—everywhere. Splunk® offers the leading platform for Operational Intelligence. It enables the curious to look closely at what others ignore—machine data—and find what others never see: insights that can help make your company more productive, profitable, competitive and secure. What can you do with Splunk? Just ask.

<!--more-->


* Combien coute Splunk? [Un modèle basé sur la volumétrie de log/jour et sur les éditions/features](http://www.splunk.com/en_us/products/pricing.html)
* Quelle sont les différentes éditions? [Enterprise/Cloud/Free](http://www.splunk.com/en_us/products/splunk-enterprise/free-vs-enterprise.html) [Light](http://www.splunk.com/en_us/products/splunk-light/splunk-light-vs-splunk-enterprise.html) 
* Existe t il une version free? [La version free](http://www.splunk.com/en_us/products/splunk-enterprise/free-vs-enterprise.html) est la version Enterprise bridée. Elle limite les features et la volumétrie de logs 
à 500MB/day.


# Existant Open Source ## 

La présentation [Monitoring Open Source pour Java avec JmxTrans, Graphite et Nagios](http://fr.slideshare.net/cyrille.leclerc/open-source-monitoring-for-java-with-graphite) par par Cyrille Le Clerc, Henri Gomez
est une bonne base présentant des outils Open Source.

## Metrics
* [Jmxtrans](http://www.jmxtrans.org/): extraction des metriques exporté via JMX
* [Graphite](http://graphite.wikidot.com/): stockage des métriques, exploitation   des metrics (calculs...), rendu (image, texte...). Graphite se compose de carbon (listener), graphite (UI), whisper (stockage RRD)
** [The architecture of clustering Graphite](https://grey-boundary.io/the-architecture-of-clustering-graphite/=
* [Graphana](http://grafana.org/): graphite frontend

## Logs
* [Logstash](https://www.elastic.co/products/logstash), [flume](https://flume.apache.org/): collecte des logs 
* [Elasticsearch](https://www.elastic.co/products/elasticsearch): stockage
* [Kibana](https://www.elastic.co/products/kibana): query, visualization

## Alerting
* [Seyren](https://github.com/scobal/seyren): nécessite mongodb


Je pense avoir fait le tour. Il y a bien sur des variations, [InfluxDB](http://influxdb.com/) au lieu de graphite, Nagios au lieu de Seyren...


# Challenge

Peut on faire le même niveau fonctionnel en utilisant Splunk? 

Exploitation des logs applicatifs d'une application Java
Les logs seront directement écrits au format JSON via un appender/layout logback

Exploitation des metrics d'une application
Les metrics Java seront extraits en utilisant Metrics de DropWizard

# Installation

{% highlight text linenos %}
FROM ubuntu:14.04
ADD package/splunk-6.2.3-264376-linux-2.6-amd64.deb /tmp/splunk-6.2.3-264376-linux-2.6-amd64.deb
RUN sudo dpkg -i /tmp/splunk-6.2.3-264376-linux-2.6-amd64.deb
{% endhighlight %}

{% highlight text linenos %}
sudo docker build -t splunk .
sudo docker run -t -p 8000:8000 -v $HOME/splunk/var:/opt/splunk/var/lib/splunk  -v /$HOME/splunk/apps:/opt/splunk/etc/apps -v $HOME/splunk/log:/var/log/myapp -i splunk  /bin/bash
/opt/splunk/bin/splunk start --accept-license --answer-yes
{% endhighlight %}

Pour un controle plus fin, je lance le container sur la commande bash puis je lance en daemon splunk


# Configuration
 
Splunk peut être configuré...


indexes.conf
{% highlight text linenos %}
[nlab]
homePath   = $SPLUNK_DB/nlab/db
coldPath   = $SPLUNK_DB/nlab/colddb
thawedPath = $SPLUNK_DB/nlab/thaweddb
{% endhighlight %}


inputs.conf
{% highlight text linenos %}
[monitor:///var/log/myapp/*.log]
{% endhighlight %}



