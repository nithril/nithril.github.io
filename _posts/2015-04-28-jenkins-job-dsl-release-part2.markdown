---
layout: post
title:  "Jenkins Job DSL - Création automatisée de pipelines"
date:   2015-04-22 21:45:46
categories: continuous integration
comments: true
---


## Introduction

Jenkins peut rapidement devenir une usine à jobs. Sans être nécessairement Netflix aux 1001 projets, l'application de certains paradigmes peut faire 
augmenter significativement le nombre de jobs :    
- La modularisation applicative 
- La réutilisation de briques techniques
- Les utils que l'on peut être amené à développer
- Le découpage du pipeline d'un projet en jobs jenkins associés aux étapes d'intégration et de déploiement continue: On part de `compilation`, `test` et `deployment` 
pour arriver à des projets associés à N jobs. 

Suivant le dernier point, 10 projets vont générer au moins 30 jobs. Enjoy.  


## Objectif

L'idée face à cette configuration est de maintenir un pool de job type **maitrisé** et de ne pas multiplier les projets avec des configurations spécifiques.

Il reste tout de même la problématique de création et de maintenance des jobs. 
Même si créer un job à *partir de* reste simple (quoi que source du syndrome du copier/coller), configurer le tout sous la forme d'un pipeline reste fastidieux. 
Vient ensuite la maintenance de ces jobs où la modification d'un job type va toucher l'ensemble des jobs de ce type.
Sans évoquer l'ajout d'un nouveau job type s'intercalant entre deux types.


Il existe plusieurs moyens qui apportent des solutions à des niveaux différents:

* [Template project plugin](https://wiki.jenkins-ci.org/display/JENKINS/Template+Project+Plugin) permet de partager les builders d'un job 
* [Templates Plugin](https://www.cloudbees.com/products/jenkins-enterprise/plugins/templates-plugin) permet de définir des templates réutilisable de builder, job, folder,
auxiliary. Il est disponible dans la version Enterprise de Cloudbees 
* [Job DSL Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Job+DSL+Plugin)
* Et sûrement d'autres plugins  


## Dont celui qui nous interesse, le `Job DSL Plugin`

> The job-dsl-plugin allows the programmatic creation of projects using a DSL. Pushing job creation into a script allows you to automate and standardize 
> your Jenkins installation, unlike anything possible before.

Le tout exprimé dans le langage [Groovy](http://www.groovy-lang.org/) qui propose 
[des fonctionnalités poussées pour créer des DSL](http://docs.groovy-lang.org/docs/latest/html/documentation/core-domain-specific-languages.html).



Par exemple le script ci-dessous (tiré de la page du plugin) 
permet de générer autant de job qu'il existe de branche sur le projet GitHub:

{% highlight groovy %}
def project = 'quidryan/aws-sdk-test'
def branchApi = new URL("https://api.github.com/repos/${project}/branches")
def branches = new groovy.json.JsonSlurper().parse(branchApi.newReader())
branches.each {
    def branchName = it.name
    job {
        name "${project}-${branchName}".replaceAll('/','-')
        scm {
            git("git://github.com/${project}.git", branchName)
        }
        steps {
            maven("test -Dproject.name=${project}/${branchName}")
        }
    }
}
{% endhighlight %}

Clair et concise, la configuration tient dans 1/3 d'écran avec juste ce qu'il y a à configurer.

### Prerequisite

Obviously, un jenkins (au moins le LTS) et le `Job DSL Plugin`.


## Définition des job types

Je vais prendre ici le cas le plus simple, 3 job types différents : `compilation`, `test`, `package` (pour s'abstraire du déploiement). 
L'objectif premier est de qualifier ces types et de définir la structure décrivant ces jobs. 

Nous avons donc une liste de projets qui sont caractérisés par un identifiant, un nom et une url scm.

{% highlight json %}
[
  {
    "id": "project1",
    "name": "Project 1",
    "scm": "https://github.com/nithril/jenkins-jobdsl-project1.git"
  },
  {
    "id": "project2",
    "name": "Project 2",
    "scm": "https://github.com/nithril/jenkins-jobdsl-project2.git"
  }
]
{% endhighlight %}


* Le type `compilation` est un job maven qui lance une compilation.
* Le type `test` est un job maven qui lance les tests.
* Le type `package` est un freestyle job qui package l'artefact maven et copie le tout dans `/dev/null`


## Création du job de génération

1. Je crée donc un job de type `Freestyle project` que je nomme `Generate Jobs`
1. Je lui ajoute un build step de type `Process Job DSLs`.
Première option intéressante, le script peut être mis directement dans la configuration du job ou stocké sur le filesystem suite, par exemple, à un clone de son repository.  
1. J'ajoute le SCM Git sur l'url [du projet](https://github.com/nithril/jenkins-jobdsl.git) contenant le script groovy
![JobDsl](/assets/jobdsl/job-dsl-scm.png)
1. Et je paramêtre le chemin vers le fichier
![JobDsl](/assets/jobdsl/jobstep.png)

## Description du script

### Chargement de la structure des projets

J'utilise pour cela le [JsonSlurper](http://docs.groovy-lang.org/latest/html/gapi/groovy/json/JsonSlurper.html): `JSON slurper parses text or reader content into a data structure of lists and maps.` 
{% highlight groovy %}
import groovy.json.JsonSlurper

def projects = new JsonSlurper().parseText(readFileFromWorkspace("src/main/groovy/project.json"))
{% endhighlight %}


### Iteration sur les projets et création des jobs


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
        publishers {
            downstream testProjectName
        }
    }

    //Test Job
    mavenJob(testProjectName) {
        projectScm(delegate, project)
        goals "test"
        publishers {
            downstream packageProjectName
        }
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

Les 2 projects ont respectivement leurs 3 jobs de définis avec leurs relations `downstream`.
![JobDsl](/assets/jobdsl/list-jobs.png)


### Description de `projectScm`

`projectScm` est la factorisation du bloc de configuration du SCM ci-dessous qui serait autrement à répeter dans chaque job:
{% highlight groovy %}
scm {
    git {
        remote {
            url project.scm
        }
        branch "master"
        createTag false
    }
}
{% endhighlight %}  
   
J'ai donc sorti ce bloc que j'ai mis dans une closure Groovy pour pouvoir l'utiliser dans la DSL Jenkins.
 Il est cependant nécessaire de définir par délégation le scope d'évalutation de cette closure qui est mis au scope du DSL Jenkins en cours d'execution.
  Pour plus d'information voir cet excellent article: [Groovy Closures: this, owner, delegate Let's Make a DSL](http://java.dzone.com/articles/groovy-closures-owner-delegate).

{% highlight groovy %}   
def projectScm = { owner, project ->
    delegate = owner
    scm {
        git {
            remote {
                url project.scm
            }
            branch "master"
            createTag false
        }
    }
}
{% endhighlight %}  


# Et pour finir
 
Je crée une vue de type Pipeline que je mets en fin d'itération
{% highlight groovy %} 
//Create the Pipeline view
buildPipelineView("${project.name}") {
    selectedJob compileProjectName
    displayedBuilds 3
    showPipelineParameters true
    showPipelineParametersInHeaders true
    showPipelineDefinitionHeader true
}
{% endhighlight %}  

![JobDsl](/assets/jobdsl/pipeline-project1.png)


# En conclusion

En 55 lignes de code et 12 lignes de fichier de description, j'ai pu définir un process de génération d'un pipeline de projet 
qui dans notre exemple a créé 6 jobs et 2 vues de type pipeline.
 
 Dans le prochain article, j'utiliserai un Job dsl pour générer les jobs de release.



<div id="disqus_thread"></div>
<script type="text/javascript">
    /* * * CONFIGURATION VARIABLES * * */
    var disqus_shortname = 'nithril';
    
    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript" rel="nofollow">comments powered by Disqus.</a></noscript>

