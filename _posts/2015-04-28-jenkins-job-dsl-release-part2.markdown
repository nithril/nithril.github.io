---
layout: post
title:  "Jenkins Job DSL - Pipeline de release"
date:   2015-04-28 20:00:46
categories: continuous integration
comments: true
---

## Introduction

Votre projet est pret à être releasé. La version déployée en QA a été validée , il ne vous reste plus qu'à fixer votre release.
  
Cela implique traditionnellement:
  
- La suppression des qualifiers `SNAPSHOT` et la vérification que toute les dépendances sont bien des versions non snapshot
- Une execution du pipeline de compilation-test-deploiement.
- Et quand tout ce passe bien, le passage à la version suivante

`release -> compilation -> test -> deploiement -> snapshot`

Ces étapes peuvent se faire *simplement* en utilisant [le plugin release de maven](http://maven.apache.org/maven-release/maven-release-plugin/).
**La problématique est qu'il bypass complétement notre pipeline de `compilation-test-deploiement`**

La difficulté avec Jenkins va être d'ajouter en queue et en tête de notre pipeline usuelle les deux étapes susnommées.
 
Jenkins ne permet pas, avec un systeme upstream/downstream basé sur des triggers de type post build, d'attendre la fin d'un pipeline (entendez avec une profondeur > 1).

Cela doit passer par :

- un freestyle job avec step de build de type "Trigger/call builds on other projects"
- un job de type [MultiJob](https://wiki.jenkins-ci.org/display/JENKINS/Multijob+Plugin),
- un job de type [Build flow plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Flow+Plugin)
- ce qui semble remplacer à terme le plugin ci-dessus [Workflow plugin](https://github.com/jenkinsci/workflow-plugin)

Pour cet article je vais utiliser le dernier: [Workflow plugin](https://github.com/jenkinsci/workflow-plugin).


## Deconstruction des jobs


La contrainte d'attente énoncée ci-dessous est une killer contrainte pour le [Build Pipeline Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Pipeline+Plugin).







