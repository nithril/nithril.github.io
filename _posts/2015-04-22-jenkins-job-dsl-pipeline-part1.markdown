---
layout: post
title:  "Jenkins Job DSL - Pipeline de release"
date:   2015-04-22 21:45:46
categories: continuous integration
comments: true
---

## Introduction

Votre projet est pret à être releasé. La version déployée en QA a été validée , il ne vous reste plus qu'à fixer votre release.
  
Cela implique traditionnellement:
  
- La suppression des qualifiers `SNAPSHOT` et la vérification que toute les dépendances sont bien des versions non snapshot
- Une execution du pipeline de compilation-test-deploiement.
- Et quand tout ce passe bien, le passage à la version suivante

Ces étapes peuvent se faire *simplement* en utilisant [le plugin release de maven](http://maven.apache.org/maven-release/maven-release-plugin/).
**La problématique est qu'il bypass complétement notre beau pipeline de `compilation-test-deploiement`**



