---
layout: post
title:  "Jenkins Job DSL - Pipeline de release"
date:   2015-04-28 20:00:46
categories: CI
comments: true
---

## Introduction

Votre projet est prêt à être releasé. Cela implique traditionnellement :

- Une entrée utilisateur indiquant la version suivante   
- La suppression des qualifiers `SNAPSHOT` et la vérification que toutes les dépendances sont bien des versions non `SNAPSHOT`
- Une execution du pipeline de `compilation -> test -> package`
- Et quand tout ce passe bien, le passage à la version suivante
- Et en cas d'erreur, un rollback à la version courante

<!--more-->

Soit: `prepare release -> compilation -> test -> package -> (next iteration | rollback)`

Ces étapes peuvent se faire *simplement* en utilisant [le plugin release de maven](http://maven.apache.org/maven-release/maven-release-plugin/).
**La problématique est qu'il bypass complétement notre pipeline de `compilation -> test -> package`** avec toutes les spécificités qu'il peut contenir.


> **La difficulté avec Jenkins va être d'ajouter en queue et en tête de notre pipeline usuel les deux étapes susnommées.**
> **Jenkins ne permet pas, avec un systeme upstream/downstream basé sur des triggers de type post build, d'attendre la fin d'un pipeline (entendez avec une profondeur > 1).**


Cela doit passer par :

- Un freestyle job avec step de build bloquant de type "Trigger/call builds on other projects"
- Un job de type [MultiJob](https://wiki.jenkins-ci.org/display/JENKINS/Multijob+Plugin),
- Un job de type [Build flow plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Flow+Plugin)
- Un job de type [Workflow plugin](https://github.com/jenkinsci/workflow-plugin) qui semble remplacer à terme le Build flow plugin
- Autre?

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


Pour plus d'information sur la big picture derrière ce plugin je vious invite à consulter la présentation de CloudBees
[Jenkins Workflow Webinar - Dec 10, 2014](http://www.slideshare.net/cloudbees/jenkins-workflow-webinar-dec-10-2014)

 
## Pipeline de release

Le pipeline de release sera articulé autour de 5 steps: `prepare release -> compilation -> test -> package -> (next iteration | rollback)`. 
Ces jobs utiliseront deux paramètres.
 
* Le premier `REPOSITORY` définit le repository sur lequel effectuer la release
* le second `NEXT_VERSION` est la version de la prochaine itération. Il sera saisi par l'utilisateur. 
 
### Job : Prepare release  
Ce step clone le projet, supprime le qualifier SNAPSHOT de **l'ensemble des versions**, dependences comprises, puis il commit les modifications. 
Pour ce faire je vais utiliser un script shell. Pourquoi ne pas utiliser le plugin versions de maven? 

Ce plugin ne permet pas de supprimer le qualifier SNAPSHOT de la version du projet ni de supprimer ce qualifier dans un projet multi module définissant
une version au travers d'une propriété définie dans le POM root.

{% highlight groovy linenos %}
sh 'rm -Rf * .git'
git url:REPOSITORY
sh 'git checkout master'
sh 'find . -name "pom.xml" | xargs -I file sed -i.bak file -e "s/-SNAPSHOT//"'
sh 'git commit --allow-empty -am "Release"'
sh 'git push'
{% endhighlight %}

Ce snippet utilise 2 commandes DSL,`sh` et `git`, dont le nom est suffisamment explicite pour se passer d'explication. 
  
La ligne 1 est l'équivalent d'un clean workspace. Les lignes 2 et 3 sont atypiques. Pourquoi ne pas simplement cloner le répertoire via un git clone? Jenkins crée des répertoires dans le workspace courant (des `.[hash]`)
et git n'apprecie pas de cloner dans un répertoire non vide. A l'inverse la commande DSL `git` se positionne sur la référence du master en mode détaché.
Bref...
  
  
### Jobs : compilation -> test -> package
  
Les jobs construits dans la partie 1 de cette suite utilisent des relations upstream/downstream basées sur des triggers de type post build. 
Comme précisé dans l'introduction, cette relation doit être deconstruite au profit d'une relation qui sera définie dans le job de release. Ormis cette différence,
les jobs restent inchangés.

{% highlight groovy linenos %}
build 'Project 1 - Compile'
build 'Project 1 - Test'
build 'Project 1 - Package'
{% endhighlight %}

La commande dsl `build` invoque un build. Elle peut prendre différents paramètres comme `Wait for completion`, `Propagate errors` ainsi que les paramètres du job. Par exemple, et cela est à ma connaissance la seule manière de faire:
{% highlight groovy linenos %}
build job:'Foo', parameters: [[$class: 'StringParameterValue', name: 'FOO', value: 'BAR']]
{% endhighlight %}

`[['FOO' : 'BAR']]` serait plus élégant.
  
La prochaine itération, `(next iteration | rollback)`, est conditionnée au résultat de la présente. Si les trois jobs passent, le projet est releasé, si au des 3 
failed, le projet est rollbacké.

Cela peut s'exprimer simplement par un block `try-catch`
{% highlight groovy linenos %}
def success = true
try {
    build 'Project 1 - Compile'
    build 'Project 1 - Test'
    build 'Project 1 - Package'
} catch(e){
    success = false
}
{% endhighlight %}

Il y a différent manière de faire, par exemple en utilisant le retour de build qui est de type [`RunWrapper`](https://github.com/jenkinsci/workflow-plugin/blob/master/support/src/main/java/org/jenkinsci/plugins/workflow/support/steps/build/RunWrapper.java)
{% highlight groovy linenos %}
    def compileBuild = build job: 'Project 1 - Compile', propagate: false
    def success = compileBuild.result == 'SUCCESS'
{% endhighlight %}
Le propagate à false est indispensable pour bloquer l'envoi d'une exception en cas de failure.

Il y a également le keyword `catchError` qui a une portée plus globale, voir [l'aide du snippet generator] (https://github.com/jenkinsci/workflow-plugin/blob/master/basic-steps/src/main/resources/org/jenkinsci/plugins/workflow/steps/CatchErrorStep/help.html) 
 
  
  
### Job : Next Iteration  

Ce step modifie la version en utilisant celle saisie par l'utilisateur. Pour ce faire j'utilise simplement 
le plugin Maven Versions.
 
{% highlight groovy linenos %}
def mvnHome = tool 'Maven 3.2.2'
sh "${mvnHome}/bin/mvn versions:set -DnewVersion=${NEXT_VERSION}  -DgenerateBackupPoms=false"
sh 'git commit --allow-empty -am "Next Version"'
sh 'git push'
{% endhighlight %}
  
La ligne 1 permet de déclarer et d'utiliser un outil (ici Maven) préalablement définie dans les settings Jenkins  

![JobDsl](/assets/2015-04-28-jenkins-job-dsl-release-part2/settings-maven.png)

  
### Job : Rollback

Ce step rollback la version en utilisant celle initiale.
 
{% highlight groovy linenos %}
def mvnHome = tool 'Maven 3.2.2'
sh "${mvnHome}/bin/mvn versions:set -DnewVersion=${currentVersion}  -DgenerateBackupPoms=false"
sh 'git commit --allow-empty -am "Rollback"'
sh 'git push'
{% endhighlight %}


### Orchestrateur  

Les 5 steps sont mis en musique par un job de type `Workflow` qui se charge de l'orchestration. 
Ce job possède un paramètre qui sera à saisir par l'utilisateur: `NEXT_VERSION` est la version de la prochaine itération. `REPOSITORY` quant à lui est hard codé. 
  
{% highlight groovy linenos %}
node {
    //workspace cleanup
    sh 'rm -Rf * .git'

    //checkout the repository
    git url: 'https://github.com/nithril/jenkins-jobdsl-project1.git'
    sh 'git checkout master'

    //extract the current version
    def pom = readFile 'pom.xml'
    def currentVersion = new XmlParser().parseText(pom).version.text()

    //remove the snapshot qualifier
    sh 'find . -name "pom.xml" | xargs -I file sed -i.bak file -e "s/-SNAPSHOT//"'

    //push the change
    commitAndPush("Release ${currentVersion}")


    def success = true

    //compile -> test -> package
    try {
        build 'Project 1 - Compile'
        build 'Project 1 - Test'
        build 'Project 1 - Package'
    }
    catch (e) {
        success = false
        echo "Error during the compile -> test -> package : ${e}"
        echo 'The release will be rollbacked'
    }

    if (success) {
        mavenSetVersion(NEXT_VERSION)
        commitAndPush("Next Version ${NEXT_VERSION}")
    } else {
        mavenSetVersion(currentVersion)
        commitAndPush("Rollback to ${currentVersion}")
        error 'Error during the compile -> test -> package'
    }
}


def commitAndPush(message) {
    sh "git commit --allow-empty -am \"${message}\""
    sh 'git push'
}

def mavenSetVersion(newVersion) {
    def mvnHome = tool 'Maven 3.2.2'
    sh "${mvnHome}/bin/mvn versions:set -DnewVersion=${newVersion}  -DgenerateBackupPoms=false"
}

{% endhighlight %}

La commande `node` permet d'allouer un executor et un workspace sur un noeud Jenkins.
L'extraction du POM de la version courante se fait au travers un parsing XML groovy du résultat de la commande `readFile` (ligne 10 et 11). 
On retrouve ensuite les snippet élaborés dans les steps ci-dessus. L'avantage d'un DSL groovy, c'est que l'on à la possibilité de créer des fonctions. 
Pour le besoin de ce job j'ai créé `commitAndPush` et `mavenSetVersion`.
  
 
 
## Résultat

L'execution du job donne [le resultat suivant en sortie de console](https://gist.github.com/nithril/c9bec727e22a48cc3464).


Le menu `Running Steps` permet de voir les steps executés.
 
![JobDsl](/assets/2015-04-28-jenkins-job-dsl-release-part2/orchestrator-job-steps.png)

Le clique sur l'icone de la console affiche les logs du step correspondant. Seul petit bémol, la sortie d'un step de type `build` se résume
à l'affichage de `Starting building project: Project 1 - Compile` et non pas à la sortie du job sous jacent.











