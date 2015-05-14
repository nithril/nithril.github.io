---
layout: post
title:  "Jenkins Workflow - Pipeline de release"
date:   2015-05-14 14:58:46
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


> La difficulté avec Jenkins va être d'ajouter en queue et en tête de notre pipeline usuel les deux étapes susnommées.
> Jenkins ne permet pas, avec un systeme upstream/downstream basé sur des triggers de type post build, d'attendre la fin d'un pipeline (entendez avec une profondeur > 1).


Cela doit passer par :

- Un freestyle job avec step de build bloquant de type "Trigger/call builds on other projects"
- Un job de type [MultiJob](https://wiki.jenkins-ci.org/display/JENKINS/Multijob+Plugin),
- Un job de type [Build flow plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Flow+Plugin)
- Un job de type [Workflow plugin](https://github.com/jenkinsci/workflow-plugin) qui semble remplacer à terme le Build flow plugin
- Autre?

Pour cet article je vais utiliser le dernier: [Workflow plugin](https://github.com/jenkinsci/workflow-plugin).


## Workflow plugin

Le workflow plugin ajoute un nouveau type de job nommé `Workflow`.

![JobDsl](/assets/2015-05-14-jenkins-workflow-release-part2/newjob-workflow.png)
 
Un job de type `Freestyle` permet de décrire au travers d'une interface utilisateur les steps constituants un job. 
L'approche par interface utilisateur, si elle a l'avantage d'être visuelle et de guider l'utilisateur, a l'inconvénient d'une certaine rigidité. 
Le job de type workflow permet de décrire au travers d'un DSL Groovy les steps constituants un job. On retrouve donc les steps d'un job classique sous la forme d'une DSL
augmenté de la puissance d'un langage de programmation (variable, condition, boucle...). Une approche donc plus flexible, mais plus technique. 
Le script peut, comme pour un job de type DSL, être  stocké dans le job ou dans un SCM.
 
![JobDsl](/assets/2015-05-14-jenkins-workflow-release-part2/newjob-workflow-script.png)
 
La documentation est spartiate voir inexistante, heureusement l'interface offre un snippet generator:  

![JobDsl](/assets/2015-05-14-jenkins-workflow-release-part2/newjob-workflow-generator.png)


Pour plus d'information sur la big picture de ce plugin je vous invite à consulter la présentation de CloudBees
[Jenkins Workflow Webinar - Dec 10, 2014](http://www.slideshare.net/cloudbees/jenkins-workflow-webinar-dec-10-2014)

 
## Pipeline de release

Le pipeline de release sera articulé autour de 5 steps: `prepare release -> compilation -> test -> package -> (next iteration | rollback)`. 
Ce pipeline utilisera un paramètre `NEXT_VERSION` qui est la version de la prochaine itération. Il sera saisi par l'utilisateur. 
 
### Step 1 : Prepare release  
Ce step clone le projet, supprime le qualifier SNAPSHOT de **l'ensemble des versions**, dependances comprises, puis il commit les modifications. 
Pour ce faire je vais utiliser un script shell. Pourquoi ne pas utiliser le plugin versions de maven ? 

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
  
La ligne 1 est l'équivalent d'un clean workspace. Les lignes 2 et 3 sont atypiques. Pourquoi ne pas simplement cloner le répertoire via un git clone? 
Jenkins crée des `dot` répertoires dans le workspace courant et git n'apprécie pas de cloner dans un répertoire non vide. 
A l'inverse la commande DSL `git` se positionne sur la référence du master en mode détaché.
Bref...
  
  
### Step 2,3,4 : compilation -> test -> package
  
Les jobs construits dans la partie 1 de cette suite utilisent des relations upstream/downstream basées sur des triggers de type post build. 
Comme précisé dans l'introduction, cette relation doit être deconstruite au profit d'une relation qui sera définie dans le job de release. Ormis cette différence,
les jobs restent inchangés.

{% highlight groovy linenos %}
build 'Project 1 - Compile'
build 'Project 1 - Test'
build 'Project 1 - Package'
{% endhighlight %}

La commande DSL `build` invoque un build. Elle peut prendre différents paramètres comme `Wait for completion`, `Propagate errors` ainsi que les paramètres du job. 
Par exemple, et cela est à ma connaissance la seule manière de faire:
{% highlight groovy linenos %}
build job:'Foo', parameters: [[$class: 'StringParameterValue', name: 'FOO', value: 'BAR']]
{% endhighlight %}

En passant `[['FOO' : 'BAR']]` serait plus élégant.
  
Le step suivant, `(next iteration | rollback)`, est conditionné au résultat du présent step. Si les trois jobs passent, le projet est releasé, si un des trois 
failed, le projet est rollbacké à sa version courante.

Cela peut s'exprimer simplement par un block `try-catch` car l'echec d'un build lance une exception:
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

Il y a différente manière de procéder, par exemple en utilisant le retour de build qui est de type 
[`RunWrapper`](https://github.com/jenkinsci/workflow-plugin/blob/master/support/src/main/java/org/jenkinsci/plugins/workflow/support/steps/build/RunWrapper.java):
{% highlight groovy linenos %}
    def compileBuild = build job: 'Project 1 - Compile', propagate: false
    def success = 'SUCCESS' == compileBuild.result 
{% endhighlight %}

Le `propagate` mis à false est indispensable pour bloquer le lancement d'une exception en cas d'échec.

Notons également le keyword `catchError` qui a une portée plus globale au bloc en cours d'exécution, voir [l'aide du snippet generator] (https://github.com/jenkinsci/workflow-plugin/blob/master/basic-steps/src/main/resources/org/jenkinsci/plugins/workflow/steps/CatchErrorStep/help.html) 
 
  
  
### Job : Next Iteration  

Ce step modifie la version en utilisant celle saisie par l'utilisateur. Pour ce faire j'utilise simplement 
le plugin Maven Versions:
 
{% highlight groovy linenos %}
def mvnHome = tool 'Maven 3.2.2'
sh "${mvnHome}/bin/mvn versions:set -DnewVersion=${NEXT_VERSION}  -DgenerateBackupPoms=false"
sh 'git commit --allow-empty -am "Next Version"'
sh 'git push'
{% endhighlight %}
  
La ligne 1 permet de déclarer et d'utiliser un outil (ici Maven) préalablement défini dans les settings Jenkins  

![JobDsl](/assets/2015-05-14-jenkins-workflow-release-part2/settings-maven.png)

  
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
Ce job possède un paramètre qui sera à saisir par l'utilisateur: `NEXT_VERSION` est la version de la prochaine itération.
  
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

La commande `node` permet d'allouer un `executor` et un `workspace` sur un noeud Jenkins.
L'extraction de la version courante à partir du `POM` se fait au travers un parsing XML Groovy du résultat de la commande `readFile` (ligne 10 et 11). 
On retrouve ensuite les snippet élaborés dans les steps ci-dessus. Pour le besoin de ce job j'ai créé deux fonctions `commitAndPush` et `mavenSetVersion`.
   
 
## Résultat

L'execution du job donne [le resultat suivant en sortie de console](https://gist.github.com/nithril/c9bec727e22a48cc3464).


Le menu `Running Steps` permet de voir les steps executés.
 
![JobDsl](/assets/2015-05-14-jenkins-workflow-release-part2/orchestrator-job-steps.png)

Le clique sur l'icone de la console associé à un step affiche les logs du step correspondant. Seul petit bémol, la sortie d'un step de type `build` se résume
à l'affichage de `Starting building project: Project 1 - Compile` et non pas à la sortie du job sous jacent.



## Conclusion

Il m'a demandé de revoir la conception classique que j'avais des pipelines Jenkins à base de relations upstream/downstream non bloquantes.
Le code est relativement concis et surtout localisé et auto suffisant pour comprendre l'entièreté du workflow sans avoir à naviguer dans les relations downstreams ou
à utiliser des plugins pour mettre en oeuvre des conditions.
 
Le manque de documentation rend la conception fastidieuse. C'est encore un plugin jeune et la compatibilité avec les plugins existants n'est pas automatique
[mais va en s'améliorant](https://github.com/jenkinsci/workflow-plugin/blob/master/COMPATIBILITY.md)
> For architectural reasons, plugins providing various extensions of interest to builds cannot be made automatically compatible with Workflow. Typically they require use of some newer APIs, large or small.
 
La visualisation de l'ensemble pourrait être travaillée. La vue `Running Steps` pourrait compléter une vue de plus haut niveau où l'utilisateur aurait la capacité
 de définir des steps de haut niveau à l'image du pipeline `prepare release -> compilation -> test -> package -> (next iteration | rollback)` et du plugin Build Pipeline.
 
 
Dans la partie 3, je reprendrai la partie 1 et la partie 2 suivant ce nouveau paradigme pour générer le pipeline nominal
  `compilation -> test -> package` et celui de release `prepare release -> compilation -> test -> package -> (next iteration | rollback)`





