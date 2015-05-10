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


## Workflow plugin




## Deconstruction des jobs

Jes jobs construits dans la partie 1 de cette suite utilisent des relations upstream/downstream basées sur des triggers de type post build. 
Comme précisé dans l'introduction, cette relation doit être deconstruite au profit d'une relation

 
 
## Job de release
 
Ce job checkout le projet, supprime le qualifier SNAPSHOT de **l'ensemble des versions**, dependences comprises, puis il commit les modifications. 
Pour ce faire je vais utiliser un simple script shell. Pourquoi ne pas utiliser le plugin versions de maven? 
Ce plugin ne permet pas de supprimer le qualifier SNAPSHOT de la version du projet ni de supprimer ce qualifier dans un projet multi module définissant
une version au travers d'une propriété définie dans le POM root.

{% highlight shell %}
git clone ${REPOSITORY} --depth 1 .
find . -name "pom.xml" | xargs -I file sed -i.bak file -e "s/-SNAPSHOT//"
git commit -am "Release"
git push 
{% endhighlight %}
  
 
 
 
 
 
 Je crée les jobs unitairement que je vais ensuite utiliser dans un job de type Workflow.

{% highlight groovy %}

projects.each { project ->

    //Define projects name
    def compileProjectName = "${project.name} - Compile"
    def testProjectName = "${project.name} - Test"
    def packageProjectName = "${project.name} - Package"

    //Compile Job
    mavenJob(compileProjectName) {
        projectScm(delegate, project)
        goals "compile"
    }

    //Test Job
    mavenJob(testProjectName) {
        projectScm(delegate, project)
        goals "test"
    }

    //Package Job
    freeStyleJob(packageProjectName) {
        projectScm(delegate, project)
        steps {
            maven {
                goals "package"
                mavenInstallation "Maven 3.2.2"
            }
            shell "cp submodule/target/*.jar /dev/null"
        }
    }
}
{% endhighlight %}







