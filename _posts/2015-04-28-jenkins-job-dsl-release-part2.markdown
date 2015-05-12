---
layout: post
title:  "Jenkins Job DSL - Pipeline de release"
date:   2015-04-28 20:00:46
categories: continuous integration
comments: true
---

* TOC
{:toc}

## Introduction

Votre projet est prêt à être releasé. Cela implique traditionnellement :

- Une entrée utilisateur indiquant la version suivante   
- La suppression des qualifiers `SNAPSHOT` et la vérification que toutes les dépendances sont bien des versions non snapshot
- Une execution du pipeline de `compilation-test-package`.
- Et quand tout ce passe bien, le passage à la version suivante

Soit: `prepare release -> compilation -> test -> package -> next iteration`

Ces étapes peuvent se faire *simplement* en utilisant [le plugin release de maven](http://maven.apache.org/maven-release/maven-release-plugin/).
**La problématique est qu'il bypass complétement notre pipeline de `compilation -> test -> package`** avec toutes les spécificités qu'il peut contenir.

La difficulté avec Jenkins va être d'ajouter en queue et en tête de notre pipeline usuel les deux étapes susnommées.
 
Jenkins ne permet pas, avec un systeme upstream/downstream basé sur des triggers de type post build, d'attendre la fin d'un pipeline (entendez avec une profondeur > 1).

Cela doit passer par :

- Un freestyle job avec step de build bloquant de type "Trigger/call builds on other projects"
- Un job de type [MultiJob](https://wiki.jenkins-ci.org/display/JENKINS/Multijob+Plugin),
- Un job de type [Build flow plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Flow+Plugin)
- Un job de type [Workflow plugin](https://github.com/jenkinsci/workflow-plugin) qui semble remplacer à terme le Build flow plugin

Pour cet article je vais utiliser le dernier: [Workflow plugin](https://github.com/jenkinsci/workflow-plugin).


## Workflow plugin

Le workflow plugin ajoute un nouveau type de job nommé `Workflow`.

![JobDsl](/assets/2015-04-28-jenkins-job-dsl-release-part2/newjob-workflow.png)
 
Un job de type Freestyle permet de décrire au travers une interface utilisateur les steps constituants un job. L'approche par interface, si elle a l'avantage d'être visuel 
et de guider l'utilisateur, a l'inconvénient d'une certaine rigidité rigidité. 
Le job de type workflow permet de décrire au travers une DSL Groovy les steps constituants un job. On retrouve donc les steps d'un job sous la forme d'une DSL
augmenté de la puissance d'un langage de programmation (variable, condition, boucle...). Une approche donc plus flexible, mais plus technique. 
Le script peut, comme pour un job de type DSL, être
 stocké dans le job ou dans un SCM.
 
 ![JobDsl](/assets/2015-04-28-jenkins-job-dsl-release-part2/newjob-workflow-script.png)
 
La documentation est spartiate voir inexistante, heuresement l'interface offre un snippet generator:  

 ![JobDsl](/assets/2015-04-28-jenkins-job-dsl-release-part2/newjob-workflow-generator.png)

 
## Pipeline de release

Le pipeline de release sera articulé autour de 5 jobs: `prepare release -> compilation -> test -> package -> next iteration`. 
Ces jobs utiliseront deux paramètres.
 
* Le premier `REPOSITORY` définit le repository sur lequel effectuer la release
* le second `NEXT_VERSION` est la version de la prochaine itération. Il sera saisi par l'utilisateur. 
 
### Job : Prepare release  
Ce job de type `Workflow` clone le projet, supprime le qualifier SNAPSHOT de **l'ensemble des versions**, dependences comprises, puis il commit les modifications. 
Pour ce faire je vais utiliser un script shell. Pourquoi ne pas utiliser le plugin versions de maven? 

Ce plugin ne permet pas de supprimer le qualifier SNAPSHOT de la version du projet ni de supprimer ce qualifier dans un projet multi module définissant
une version au travers d'une propriété définie dans le POM root.

{% highlight groovy linenos %}
node {
    sh 'rm -Rf * .git'
    git url:REPOSITORY
    sh 'git checkout master'
    sh 'find . -name "pom.xml" | xargs -I file sed -i.bak file -e "s/-SNAPSHOT//"'
    sh 'git commit --allow-empty -am "Release"'
    sh 'git push'
}
{% endhighlight %}
  
La ligne 2 est l'équivalent d'un clean workspace. Les lignes 3 et 4 sont atypiques. Pourquoi ne pas simplement cloner le répertoire via un git clone? Jenkins crée des répertoires dans le workspace courant (des `.[hash]`)
et git n'apprecie pas de cloner dans un répertoire non vide. A l'inverse la commande DSL `git` se positionne sur la référence du master en mode détaché.
Bref...
  
  
### Jobs : compilation -> test -> package
  
Les jobs construits dans la partie 1 de cette suite utilisent des relations upstream/downstream basées sur des triggers de type post build. 
Comme précisé dans l'introduction, cette relation doit être deconstruite au profit d'une relation qui sera définie dans le job de release. Ormis cette différence,
les jobs restent inchangés.
  
### Job : Next Iteration  

Ce job de type `Workflow` clone le projet et modifie la version en utilisant celle saisie par l'utilisateur. Pour ce faire j'utilise simplement 
le plugin Maven Versions.
 
{% highlight groovy linenos %}
node {
    sh 'rm -Rf * .git'
    git url:REPOSITORY, branch:'master'
    sh 'git checkout master'
    def mvnHome = tool 'Maven 3.2.2'
    sh "${mvnHome}/bin/mvn versions:set -DnewVersion=${NEXT_VERSION}  -DgenerateBackupPoms=false"
    sh 'git commit --allow-empty -am "Next Version"'
    sh 'git push'
}
{% endhighlight %}
  
La ligne 5 permet d'utiliser une version Maven préalablement définie dans les settings Jenkins  

![JobDsl](/assets/2015-04-28-jenkins-job-dsl-release-part2/settings-maven.png)

### Orchestrateur  

Les 5 jobs sont mis en musique par un job de type `Workflow` qui se charge de l'orchestration. 
Ce job possède un paramètre qui sera à saisir par l'utilisateur: `NEXT_VERSION` est la version de la prochaine itération. `REPOSITORY` quant à lui est hard codé. 
  
{% highlight groovy linenos %}
def repository = 'https://github.com/nithril/jenkins-jobdsl-project1.git'
build job:'Release - Prepare', parameters: [[$class: 'StringParameterValue', name: 'REPOSITORY', value: repository]]
build 'Project 1 - Compile'
build 'Project 1 - Test'
build job: 'Project 1 - Package'
build job:'Release - Next Iteration', parameters: [[$class: 'StringParameterValue', name: 'REPOSITORY', value: repository], [$class: 'StringParameterValue', name: 'NEXT_VERSION', value: NEXT_VERSION]]
{% endhighlight %}

La définition des paramètres d'un job est particulièrement peu intuitive et redontante; `repository` est déjà de type `String`
 
 
## Résultat

L'execution du job donne le resultat suivant en sortie de console:
 
![JobDsl](/assets/2015-04-28-jenkins-job-dsl-release-part2/orchestrator-job.png)

C'est sommaire.
 
Le menu `Running Steps` permet de voir les steps executés.
 
![JobDsl](/assets/2015-04-28-jenkins-job-dsl-release-part2/orchestrator-job-steps.png)

Le clique sur l'icone de la console n'affiche pas les logs du step correspondant, à moins que je sois passé à coté de quelque chose.

