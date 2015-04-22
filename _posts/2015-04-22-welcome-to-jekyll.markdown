---
layout: post
title:  "Jenkins JobDSL"
date:   2015-04-22 21:45:46
categories: continuous integration
---


# Introduction

Jenkins peut rapidement contenir un nombre de jobs important. Sans être nécessairement Netflix aux 1001 projets, l'application de certains paradigmes peut faire 
augmenter significativement le nombre de jobs :    
- La modularisation applicative 
- La réutilisation de briques techniques
- Les petits utils que l'on peut être amené à développer
- Le découpage du pipeline d'un projet en jobs jenkins associés aux étapes de `compilation`, de `test`, de `deployment` qui peut devenir
`compilation`,  `unit-test`, `integration-test`, `packaging`, `deployment`. 

Suivant le dernier point, 10 projets vont générer au moins 30 jobs. Easy.  

L'idée devant cette configuration est de maintenir un pool de job type maitrisé et de ne pas multiplier les projets avec des configurations spécifiques.

Il reste tout de même la problématique de création et de maintenance des jobs. 
Même si créer un job à "partir de" reste simple, configurer le tout sous la forme d'un  pipeline est fastidieux. 
La modification d'un job type, va devoir être appliqués à l'ensemble des jobs de ce type.
Sans parler de l'ajout d'un nouveau job type s'intercalant entre deux types.


Il existe plusieurs moyens qui apportent des solutions à des niveaux différents:

* [Template project plugin](https://wiki.jenkins-ci.org/display/JENKINS/Template+Project+Plugin) permet de partager les builders d'un job 
* [Templates Plugin](https://www.cloudbees.com/products/jenkins-enterprise/plugins/templates-plugin) permet de définir des templates réutilisable de builder, job, folder,
auxiliary. Il est disponible dans la version Enterprise de Cloudbees 
* [Job DSL Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Job+DSL+Plugin)
* Et surement d'autres plugins  


# Dont celui qui nous interesse, le `Job DSL Plugin`

> The job-dsl-plugin allows the programmatic creation of projects using a DSL. Pushing job creation into a script allows you to automate and standardize 
> your Jenkins installation, unlike anything possible before.

Le tout exprimé dans le langage [Groovy](http://www.groovy-lang.org/). Par exemple le script ci-dessous (tiré de la page du plugin) 
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



# Prerequisite

Obviously, un jenkins au moins le LTS et le `Job DSL Plugin`.


# Définition des jobs type

Je vais prendre ici le cas le plus simple, 3 jobs types différents: `compilation`, `test`, `package` (pour s'abstraire du déploiement). **L'objectif premier est de qualifier ces types et de définir la structure décrivant ces jobs.** 

Nous avons donc une liste de projets qui sont caractérisés par un identifiant, un nom et une url SVN.

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


* Le type `compilation` est un job maven qui lance une compilation maven.
* Le type `test` est un job maven qui lance les tests.
* Le type `package` est un freestyle job qui package l'artefact en un TAR.GZ et copie le tout dans /dev/null


# Création du job de génération


Création d'un job de type `Freestyle project` que je nomme `Generate Jobs`

Ajout d'un build step de type `Process Job DSLs`

Première option interessante, le script peut être mis directement dans la configuration du job ou stocké sur le filesystem suite à un clone de son repository.  
![JobDsl](/assets/jobdsl/jobstep.png)

Pour 





You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
